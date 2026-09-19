# P5Kit

**A shared Swift library for Archiware P5 integrations in macOS applications.**

P5Kit brings common P5 connection, archive, restore, job, and operation-evidence behavior into one library. Applications can share that foundation while keeping their own interface, workflow decisions, local databases, and file verification.

This is the **public documentation repository**. The P5Kit implementation remains private. This repository contains documentation and illustrative P5 REST API requests only; it does not distribute a Swift package, application, or executable tooling.

## Why it exists

Archive applications repeatedly need to discover P5 resources, submit work, interpret responses, follow jobs, and retain enough evidence to explain what happened. Implementing these separately in every application creates inconsistent behavior and duplicated maintenance.

P5Kit provides a common foundation for these tasks. Its purpose is to make integrations more consistent and inspectable, especially when a network failure leaves the result of an archive or restore submission uncertain.

## What it does

| Area | Capability |
| --- | --- |
| Connections | Models one logical server with multiple configured endpoints, priorities, credential references, and identity evidence. |
| REST communication | Provides typed operations through URLSession or a curl-based transport. |
| Transport security | Supports plain HTTP and TLS, with a trust policy per server: ordinary system verification, or a pinned certificate identified by its SHA-256. A certificate probe reports what a server presents so an operator can decide. |
| Archive overview | Reads finished archive jobs and media-pool capacity, keeping each job's directory list intact. |
| Resource discovery | Reads server information, clients, archive plans, indexes, and metadata keys. |
| Archiving | Submits selected paths, supports per-path metadata, and extracts job and entry information. |
| Restoring | Resolves exact archive entry handles and submits restore selections with relative-target validation. |
| Jobs | Reads job state, reports, and protocol information. |
| Evidence | Captures HTTP results with redaction support and writes versioned archive receipts atomically. |
| Mutation safety | Provides endpoint pins and receipt decisions for uncertain delivery; receipt policy never permits automatic replay. |

The endpoint resolver and safety models are building blocks. Calling applications must connect them to their workflow; constructing a REST client alone does not enforce authorization or plan safety.

## Status and scope

This overview describes the **0.8.0-dev** development baseline, reviewed on **2026-09-19**. P5Kit targets macOS 13 or later and uses Swift tools 5.9. Its public API is still evolving.

Six applications consume it, pinned by exact tag and moved only when each application is next worked on, so they sit on different baselines by design: P5 Archive Overview, P5 Archive Search, P5 Archive Browser and P5 Archive Manager API on 0.8.0-dev, P5 Search Jumper on 0.4.0-dev, and CopyTrust on 0.3.0-dev. Project Folder Tracker remains a prospective adopter.

Consuming the package and adopting its typed surface are different things. P5 Archive Manager API resolves 0.8.0-dev for the network transport and TLS trust model only — it still parses P5's responses with its own code — so it is a consumer of the transport layer rather than of the typed REST API.

See [Capabilities and boundaries](CAPABILITIES.md) for implemented versus developing areas, [Workflow and evidence](WORKFLOWS.md) for application responsibilities, and [P5 API examples](P5_API_EXAMPLES.md) for illustrative HTTP requests.

[Archiware P5 workflow compatibility](ARCHIWARE_P5_APP_COMPATIBILITY.md) covers the applications themselves: which of them talk to P5, how they reach it, which two write to it, and the three site workflows they cover.

## What it does not replace

P5Kit requires an existing Archiware P5 installation and appropriate access. It does not supply P5 itself, tape hardware management, a graphical archive application, or a complete wrapper for every P5 REST endpoint.

A successful job is evidence about P5's reported execution. It is not, by itself, independent proof that archived or restored bytes match their source. File verification and deletion decisions remain separate application responsibilities.

## Vendor documentation

The examples supplement the [official Archiware P5 REST API reference](https://blog.archiware.com/redoc/p5_rest_api/awp5api.html). Consult the reference appropriate to your server version for the complete API contract. More manuals are available from [Archiware](https://www.archiware.com/manuals).

P5Kit is an independent integration project. Archiware P5 is the vendor platform it integrates with; this repository is not the official Archiware SDK or API documentation.
