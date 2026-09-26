# Capabilities and boundaries

[Overview](README.md) · [Workflow and evidence](WORKFLOWS.md) · [API examples](P5_API_EXAMPLES.md)

The documented baseline is 0.9.0-dev, reviewed on 2026-09-25. Implementation availability, application adoption, and live-server validation are distinct: a capability in the library does not establish that every consumer uses it or that it has been tested against every P5 version.

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
| Volumes, one volume, and its jobs | `GET general/volumes`, `GET general/volumes/{volume-id}`, `GET general/volumes/{volume-id}/jobs` |
| List one level of an index | `GET archive/indexes/{index-id}/inventory/{path}` |
| Archive overview | `GET archive/overview` |
| Read job state | `GET general/jobs/{job-id}` |
| Read job report | `GET general/jobs/{job-id}/report` |
| Read job protocol | `GET general/jobs/{job-id}/protocol` |

Archive requests support metadata attached to individual paths. Entry lookup resolves an opaque handle for the selected client/path and optional index. It distinguishes recognized missing-entry responses from other failures; an arbitrary HTTP 500 is not proof that a file is absent.

Two things about the lookup were measured against a live server and are easy to get wrong. Without a `database` header it does not reach an imported-volumes index. And for that index the path is the source path with a leading slash and **without** the volume's label: a path that starts with the label is answered as an unknown entry.

Restore selections can carry relative target paths. P5Kit rejects empty, absolute, and dot-component targets in that field. The calling application still owns destination selection, existing-file checks, and authorization.

Job state and completion are parsed separately. Reports can be requested as plain text; protocol requests can ask for JSON. A job identifier is needed for later reconciliation and evidence.

## Label-rooted indexes, such as Imported-Volumes

An index that holds imported volumes is addressed differently from an ordinary archive index, and P5 does not say so in the responses. Measured against a live server (an Imported-Volumes index of 15 volumes among 169):

- **Listing is rooted at the plain volume label.** `inventory/<label>/Volumes/…` lists folder by folder. `inventory/Volumes`, the index root, and a name shaped `<label>-<uuid>` (as the P5 web application can show it) are all answered 404, with the body `{}`.
- **Entry lookup is not rooted at the label** (see above). A listing path and a lookup path for the same file therefore differ by the label.
- **P5 does not list the labels.** The index root is a 404 and a volume record carries no pool, successor or index field.

P5Kit's part is to make that manageable:

- `discoverIndexLabels` tries each volume's label, and its barcode where it has one, as a top-level name and returns the ones the index answers for, with the volumes that offered each name and its first-level folders. It is read-only, bounds its concurrency, reports progress, treats a 404 as "not in this index" and stops on a rejected login. A label offered by two volumes is marked ambiguous, and volumes that hold nothing are skipped unless asked. On the measured server it found 14 of the 15 labels in 487 requests.
- A pasted list of labels is parsed and merged with the discovered ones. This is how a label that has no volume record, which probing cannot find, gets in.
- A label-to-volume map records the volume IDs that an index name's entries carry, and reports where the index name differs from the volume's own label. The volume ID, not the name, is the key to use for restore.

What this does not do: find a label that has no volume record, search by file name or by folder prefix (P5 offers neither over REST), or decide which of two volumes sharing a label is meant. Applications that need a file-name search use exported inventories, or list folders under the labels within a bounded number of requests.

## Errors

P5 answers a wrong password with **HTTP 400** and the plain-text body `Wrong username or password.`, not 401. P5Kit keeps a plain-text error body as the error's message and classifies a rejected login whichever status carries it, including when the client has redacted a password that is part of the message. An unknown entry is HTTP 500 with a text body, and a missing inventory path is HTTP 404 with `{}`.

## Evidence and responsibility

P5Kit supplies captured-response values, redaction support, mutation receipts, and an atomic archive-receipt writer. These help applications preserve what was requested and what P5 reported.

Redaction is not complete anonymization: paths, metadata, host information, and other operational details can still be sensitive. Applications control evidence retention, access, and any sharing.

The library's mutation receipt policy does not allow automatic replay. If delivery is uncertain, the next step is reconciliation. The low-level REST methods do not themselves prompt an operator or inspect the full safety policy of an archive plan.

## Developing or outside this baseline

- Further application migrations are future work. Several applications still parse P5's responses with their own code.
- A search by file name or by folder prefix is not something P5 offers over REST, so it is not something P5Kit can supply.
- A general monitoring interface for devices, jukeboxes, backup, and synchronization resources is not part of this baseline.
- Shared Keychain storage, application databases, and an `nsdchat` integration are not supplied by this baseline.
- UI, scheduling, archive-plan approval, bounded polling policy, search experiences, and independent file-hash verification remain owned by applications.

These are capability boundaries, not delivery commitments. This public repository provides no implementation downloads or package installation instructions.
