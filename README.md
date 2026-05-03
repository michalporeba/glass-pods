# GlassPods

**Citizen-transparent, graph-native data sovereignty for government interoperability**

GlassPods is a research prototype exploring whether the principles behind [Solid](https://solidproject.org/) — decentralised data ownership,
individual-controlled access, machine-readable governance — can be made technically viable at government scale 
without requiring citizens to host or manage their own data.

## The problem

Public sector data is fragmented across organisational boundaries, making joined-up services difficult.
Current solutions tend toward centralisation — shared platforms, aggregated stores — which raises legitimate concerns 
about governance, trust, and systemic risk.

At the same time, individual-centric approaches like Solid pods, while philosophically well-aligned with data rights, 
face practical barriers in government contexts: legal authority over data often rests with the state, not the individual; 
performance at scale is constrained by the file-based, document-oriented storage model; 
and the consent model (ask permission before use) is misaligned with how lawful government data processing actually works.

## The approach

GlassPods reframes the problem. Rather than asking citizens to own and host their data, it asks: 
what if government organisations stored data *about* citizens in citizen-partitioned, pod-like structures — and made 
access to that data transparent to the citizen, rather than subject to their prior approval?

The core ideas:

- Each government organisation stores personal data as **named graphs partitioned by individual** — one named graph per citizen, per organisation — rather than in process-centric tables or siloed databases
- Citizens see a **read-only, transparent view** of what each organisation holds about them, and a log of who has accessed it and why
- Cross-organisational data exchange is governed by **machine-readable interchange agreements** encoding permitted purposes, accessing organisations, and permitted result shapes (using SHACL and ODRL/DPV)
- **Inference on write** populates purpose-specific analytical graphs automatically from citizen graphs, keeping analytical workloads separated from personal data stores
- A **timestamped triple event log** provides point-in-time reconstruction and clean audit trails

The shift from consent-as-approval to **consent-as-transparency** reflects the legal reality of government data use: in most cases, 
the lawful basis is established by legislation, not individual permission. What citizens deserve is visibility, not a veto.

## Status

Early-stage research prototype. This repository accompanies a research investigation into the technical viability of this architecture.

See [HYPOTHESIS.md](HYPOTHESIS.md) for a detailed account of the research hypothesis, technical decisions, and open questions.

## Technology

- [Community Solid Server](https://github.com/CommunitySolidServer/CommunitySolidServer) — Solid protocol layer for citizen-facing pod access
- [Apache Jena Fuseki](https://jena.apache.org/documentation/fuseki2/) — SPARQL triplestore with named graph support
- [Nemo](https://github.com/knowsys/nemo) — Datalog-based inference engine
- [pySHACL](https://github.com/RDFLib/pySHACL) — SHACL validation for interchange agreement enforcement
- Python middleware connecting the layers

## Related

- [Solid Project](https://solidproject.org/)
- [W3C SHACL](https://www.w3.org/TR/shacl/)
- [W3C ODRL](https://www.w3.org/TR/odrl-model/)
- [Data Privacy Vocabulary (DPV)](https://w3c.github.io/dpv/dpv/)
- [Nemo rule engine](https://github.com/knowsys/nemo)
- [Halyard](https://github.com/Merck/Halyard) — horizontal scaling path for future work


