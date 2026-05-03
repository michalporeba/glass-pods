# GlassPods — Research Hypothesis

This document sets out the research hypothesis behind GlassPods, the technical reasoning that led to the current architecture, and the open questions the prototype is intended to investigate.

---

## Context and motivation

Government data is fragmented. Personal information about a citizen is held across many organisations, each with their own systems, formats, and governance processes. The result is duplication, inconsistency, and friction when services need to share information — even when there is a clear legal basis to do so.

Attempts to solve this problem typically reach for centralisation: shared platforms, common identifiers, aggregated data stores. These approaches can work, but they concentrate risk, create single points of failure, and raise legitimate concerns about surveillance, mission creep, and the long-term governance of sensitive personal data.

At the same time, there is growing policy and legal emphasis on individual data rights — transparency, access, and portability. These pressures are often treated as being in tension with interoperability. GlassPods explores whether they can instead be made mutually reinforcing.

---

## The Solid starting point

The [Solid](https://solidproject.org/) project, initiated by Tim Berners-Lee, proposes that individuals should control their own data through personal online datastores called pods. Applications request access to pods rather than accumulating data centrally. Governance rules are explicit and machine-readable. Interoperability comes from shared standards rather than shared infrastructure.

The values behind Solid are well-aligned with the problem. The implementations are not.

Current Solid implementations have two fundamental problems at government scale:

**The file-based storage model.** Pods store data as RDF documents (Turtle files). Querying across multiple resources requires multiple HTTP fetches; there are no native joins. Any aggregation either requires replicating data out of pods — undermining the sovereignty model — or accepting severe performance penalties. This is not an engineering limitation that better tooling will fix; it reflects a deep tension between the document-oriented storage model and the relational query patterns that government data exchange requires.

**The consent model.** Solid assumes that individuals approve or deny access to their data. In a government context, this is often legally inappropriate. Legislation establishes the lawful basis for processing personal data; individuals do not have a veto over, for example, the use of their records for fraud detection or benefit calculation. Asking citizens to approve every government data access would be both impractical and legally incoherent.

These are not reasons to abandon the values of Solid. They are reasons to look for an alternative implementation of those values.

---

## The core hypothesis

> Interoperability between government organisations can be achieved while preserving individual data sovereignty — not by requiring citizens to own and host their data, but by requiring government organisations to store, access, and exchange data in citizen-centric, transparent, and governable ways.

More specifically, GlassPods tests the hypothesis that:

1. **Named graphs partitioned by citizen** can serve as the organisational unit for personal data storage within government systems, preserving the conceptual benefits of the pod metaphor without the performance costs of distributed file storage.

2. **Machine-readable interchange agreements** encoding permitted purposes, accessing organisations, and permitted result shapes can replace ad hoc data sharing arrangements, making governance computable rather than procedural.

3. **Transparency rather than consent** is the appropriate model for citizen engagement with government data use — citizens should be able to see what is held about them and who has accessed it, rather than being asked to approve access that the law already permits.

4. **Inference on write** can populate purpose-specific analytical graphs from citizen graphs automatically, separating analytical workloads from personal data stores and making the purpose limitation principle technically enforceable rather than merely policy-level.

5. **This architecture is horizontally scalable** because citizen graph operations are independent of each other, enabling sharding by citizen identifier without cross-citizen coordination costs.

---

## Architecture

### The storage model

Each government organisation maintains a triplestore in which personal data is stored as named graphs, one per citizen. The named graph URI encodes both the citizen identifier and the holding organisation. Internally, this is a quad store (subject, predicate, object, graph); externally, it exposes a Linked Data Platform interface so that citizen-facing tooling can navigate it using familiar Solid pod conventions.

This separation — internal quad store, external LDP interface — is the key architectural move. It preserves the user experience of the pod metaphor (folders, files, navigable structure) without binding the storage layer to a file-based implementation.

The citizen's view is read-only. They can see what is held about them by each organisation. They cannot modify it — because the authoritative source of the data is the organisation, not the citizen.

### The event log

All writes to citizen named graphs are recorded in a timestamped triple event log. Each log entry records the asserted or retracted triple, the time, and the asserting system. The log is append-only; deletions are represented as retraction events rather than physical removals.

This gives the system two important properties. First, any point-in-time state of any citizen graph can be reconstructed from the log — useful for audit and for demonstrating what data was available at the time a decision was made. Second, the log can be compacted periodically by removing assertion/retraction pairs that fall outside the retention window, keeping storage manageable without losing auditability.

### Analytical graphs

Purpose-specific analytical graphs are materialised on demand from the event log rather than maintained continuously. Each analytical graph contains only the triples relevant to a specific, declared purpose — for example, a graph containing only the triples needed to assess eligibility for a specific service.

Inference rules written in Datalog populate these graphs from citizen graph updates. Because the rules are explicit and the output graphs are purpose-bounded, the purpose limitation principle becomes technically enforceable: an analytical process can only access the triples that the inference rules project for its declared purpose.

### Interchange agreements

Cross-organisational data exchange is governed by interchange agreements — machine-readable artefacts encoded using ODRL (for permissions and constraints) and DPV (for purposes and lawful bases). An interchange agreement specifies:

- The permitted querying organisation
- The data-holding organisation
- The purpose of the exchange, referencing a controlled vocabulary of lawful bases
- The permitted result shape, expressed as a SHACL shape

When an organisation requests data, it presents a purpose declaration. The receiving system checks whether the declared purpose matches a current interchange agreement, executes the query in a sandboxed environment, validates the result against the agreed shape, and — if both checks pass — returns the result and writes an audit entry to the citizen's transparency graph.

No runtime human approval is required, because the legal basis was established when the interchange agreement was made. The citizen is not asked for permission; they are shown that access occurred.

### The transparency graph

Each citizen has a transparency graph — a named graph containing audit entries for accesses to their data. This is written to automatically by the transparent access interface whenever a cross-organisational query is executed against their citizen graphs.

Each audit entry records: the accessing organisation, the declared purpose, the lawful basis, the shape of data accessed, and the timestamp. The transparency graph is citizen-readable via the pod interface.

This is the most significant departure from conventional Solid. In Solid, citizens approve access before it happens. In GlassPods, citizens see access after it has happened. This is not a weakening of citizen rights — it is an alignment with how government data use actually works legally, combined with a technical guarantee of visibility that currently does not exist in most government systems.

A second, privileged access interface exists for cases where legislation permits data access without citizen notification (law enforcement, national security, serious fraud). This interface logs internally for organisational accountability but does not write to the citizen transparency graph. The existence of this interface is itself publicly documented — what is not disclosed is which individual records it has been used to access.

### Access control

The citizen-facing interface authenticates via standard web identity mechanisms, compatible with existing government identity infrastructure. The organisational interface authenticates by organisation identifier, with endpoint-level controls determining which operations each organisation may perform.

Named graph access control enforces the partition between citizen-visible graphs, internal-only graphs, and analytical graphs. The same triple may appear in multiple graphs with different access rules — the authoritative assertion is in the citizen graph; derived triples in analytical or interchange graphs are projections, not copies.

---

## Technical stack

The prototype uses:

- **Community Solid Server (CSS)** — implements the Solid/LDP protocol for the citizen-facing layer, configured with a SPARQL backend rather than file storage
- **Apache Jena Fuseki** — SPARQL 1.1 triplestore with TDB2 persistent storage, holding all named graphs
- **Nemo** — Datalog-based rule engine for inference on write, populating analytical graphs from citizen graph updates
- **pySHACL** — SHACL validation for interchange agreement enforcement
- **Python middleware** — connects the layers, implements the interchange agreement logic, manages the event log

For scaling experiments, Fuseki can be substituted with **Halyard** — an open source, horizontally scalable triplestore built on Apache HBase — without changes to the middleware layer, because both implement the RDF4J SPARQL interface. This substitutability is itself an architectural property worth demonstrating.

---

## Open questions

The prototype is intended to investigate the following questions:

**Technical viability**
- Can the CSS SPARQL backend be reliably configured to use Fuseki as its storage layer, with named graphs as the partitioning unit?
- Can a shape-conformant query enforcer be built as middleware — a component that executes a SPARQL query, validates the result against a SHACL shape, and either returns the result or rejects the query — using available open source components?
- Can Nemo inference rules be triggered on write events in a way that keeps analytical graphs consistent with citizen graph updates, with acceptable latency?

**Scaling**
- Does per-citizen read latency remain flat as total citizen count increases? This is the key empirical test of the horizontal scalability claim.
- At what scale does Fuseki become the bottleneck, and does substituting Halyard resolve that bottleneck without architectural changes?
- What are the storage costs per citizen at realistic triple counts, and how does compaction of the event log affect those costs over time?

**Governance expressiveness**
- Is the combination of ODRL and DPV expressive enough to encode realistic government interchange agreements, or are there lawful bases and purposes that require vocabulary extensions?
- Can the SHACL shape of a permitted result be specified precisely enough to prevent over-disclosure, while remaining flexible enough to be practically useful?

**Limitations and scope**
- The tamper-evidence of the audit log is out of scope for this prototype. The mechanism exists; the cryptographic guarantee does not. This is identified as future work.
- The prototype does not attempt to model every category of government data. It uses a simplified domain — a small number of entity types and relationships — to demonstrate the architectural principle.
- The legal framework analysis (mapping specific legislative bases to DPV terms) is illustrative rather than comprehensive.

---

## Relationship to existing work

GlassPods draws on and extends several lines of existing work:

- **Solid and LDP** — for the pod metaphor and the citizen-facing interface model
- **Linked Data Fragments** — for the insight that query decomposition and server burden can be traded off, and that SPARQL-at-scale requires architectural rethinking
- **SHACL-AF** — for SPARQL-based triple rules as an inference mechanism
- **ODRL and DPV** — for machine-readable policy and purpose vocabulary
- **Data spaces** (IDSA, Gaia-X) — for the concept of machine-readable data sharing agreements in federated environments
- **Event sourcing** — for the append-only log pattern applied to RDF
- **RDF-star** — for clean reification of log entries without awkward named graph workarounds

The novel contribution is the composition of these elements into a coherent architecture specifically addressing the government interoperability problem, together with the empirical investigation of its technical viability and scaling properties.

---

## Name

**GlassPods** — the transparency of glass, the sovereignty of the pod. Citizens can see in. Data stays where it is held, governed by those accountable for it, visible to those it concerns.

The name **Cist** is reserved for a potential future server implementation. In Welsh, a *cist* is a chest or coffer — a safe place of keeping. In archaeology, a cist is a stone-lined grave: deeply personal, carefully bounded, not to be disturbed without cause.
