# Capabilities and boundaries

[Overview](README.md) · [Workflow and evidence](WORKFLOWS.md) · [API examples](P5_API_EXAMPLES.md)

The documented baseline is 0.8.0-dev, reviewed on 2026-09-18. Implementation availability, application adoption, and live-server validation are distinct: a capability in the library does not establish that every consumer uses it or that it has been tested against every P5 version.

## Connections and transport

P5Kit models multiple routes to one logical server, including scheme, host, port, priority, network context, and API-version overrides. It supports versioned server-file serialization and migration from older server-file shapes. Credential references identify credentials managed elsewhere; the server model is not a password store.

The resolver accepts an injected read-only probe. It evaluates reachability and identity evidence, orders configured endpoints by priority, and prefers HTTPS within equal priorities. It supports verified read fallback and records endpoint decisions. Identity mismatches hold an operation rather than silently accepting another server.

Mutations and accepted-job follow-up have endpoint-pinning models. Moving job follow-up to another route requires an explicit verified transition. These models do not turn the REST client into an automatic network failover service.

The REST layer can use URLSession or a curl process. The curl transport puts request configuration into restricted temporary files rather than exposing credentials and request bodies as process arguments. HTTP failures retain their status separately from transport failures.

Either transport can address a P5 server over plain HTTP or TLS. P5 serves the same REST API on both, on port 8000 and 8443 respectively by default, though TLS is enabled per installation rather than universally. Because P5 ships a self-signed certificate that ordinary system verification rejects, the URLSession transport takes an explicit trust policy: system verification, or a certificate pinned by its SHA-256 fingerprint, which refuses a server that later presents a different certificate. A probe reports the fingerprint, subject, and system verdict of the certificate a server presents, so that decision can be put to an operator rather than assumed.

## Archive, restore, and job operations

| Operation | P5 route, relative to the configured REST base |
| --- | --- |
| Server information | `GET general/srvinfo` |
| Clients | `GET general/clients` |
| Archive plans and one plan | `GET archive/plans`, `GET archive/plans/{plan-id}` |
| Archive indexes | `GET archive/indexes` |
| Metadata keys and one key | `GET archive/indexes/{index-id}/keys`, `GET archive/indexes/{index-id}/keys/{key-id}` |
| Add metadata keys | `PUT archive/indexes/{index-id}/keys` |
| Submit archive selection | `POST archive/plans/{plan-id}/archiveselections` |
| Resolve an archive entry | `GET archive/entries` with client, path, and optional database headers |
| Submit restore selection | `POST restore/restoreselections` |
| Read job state | `GET general/jobs/{job-id}` |
| Read job report | `GET general/jobs/{job-id}/report` |
| Read job protocol | `GET general/jobs/{job-id}/protocol` |

Archive requests support metadata attached to individual paths. Entry lookup resolves an opaque handle for the selected client/path and optional index. It distinguishes recognized missing-entry responses from other failures; an arbitrary HTTP 500 is not proof that a file is absent.

Restore selections can carry relative target paths. P5Kit rejects empty, absolute, and dot-component targets in that field. The calling application still owns destination selection, existing-file checks, and authorization.

Job state and completion are parsed separately. Reports can be requested as plain text; protocol requests can ask for JSON. A job identifier is needed for later reconciliation and evidence.

## Evidence and responsibility

P5Kit supplies captured-response values, redaction support, mutation receipts, and an atomic archive-receipt writer. These help applications preserve what was requested and what P5 reported.

Redaction is not complete anonymization: paths, metadata, host information, and other operational details can still be sensitive. Applications control evidence retention, access, and any sharing.

The library's mutation receipt policy does not allow automatic replay. If delivery is uncertain, the next step is reconciliation. The low-level REST methods do not themselves prompt an operator or inspect the full safety policy of an archive plan.

## Developing or outside this baseline

- Volume listing, details, and volume job lookup are under development beyond the documented tag.
- Shared archive-index inventory support and further application migrations are future work.
- A general monitoring interface for devices, jukeboxes, backup, and synchronization resources is not part of this baseline.
- Shared Keychain storage, application databases, and an `nsdchat` integration are not supplied by this baseline.
- UI, scheduling, archive-plan approval, bounded polling policy, search experiences, and independent file-hash verification remain owned by applications.

These are capability boundaries, not delivery commitments. This public repository provides no implementation downloads or package installation instructions.
