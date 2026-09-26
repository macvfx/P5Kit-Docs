# Workflow and evidence

[Overview](README.md) · [Capabilities](CAPABILITIES.md) · [API examples](P5_API_EXAMPLES.md)

P5Kit contributes shared connection and operation behavior. An application decides what work is appropriate, obtains authorization, and presents progress and evidence to the operator.

## Archive workflow

1. The application identifies the intended server, client, archive plan, and source paths.
2. It checks access and plan behavior, including any consequences for source files, and obtains authorization.
3. Where integrated, P5Kit's resolver evaluates the configured routes and records the chosen endpoint and identity evidence.
4. The application pins the mutation and submits the archive selection through the REST client.
5. It retains the returned job identifier and available entry information, then follows job state, report, and protocol evidence.
6. It records the outcome. Any independent content verification or later source deletion is a separate decision.

Submitting an archive request and receiving a job ID establishes that work was accepted, not that it finished successfully. A completed job likewise does not replace content verification.

## Restore workflow

1. Identify the archived path, originating client, and archive index where needed.
2. Resolve the exact entry handle. A missing-entry result is distinct from a network or server failure.
3. Select and check the destination, including existing files. Relative target paths must satisfy P5Kit's validation.
4. Authorize and submit the restore selection, retaining the selected endpoint and job identifier.
5. Follow the job and inspect the resulting files. Verify content separately when the workflow requires it.

Opaque archive entry handles come from P5. Applications should preserve them exactly rather than reconstructing them from a filename or barcode.

## Finding media in an imported-volumes index

An imported-volumes index is addressed by volume label, and P5 does not list the labels. An application that needs to check or restore media there does this:

1. **Find the labels.** Use P5Kit's discovery, and let the operator add any it missed from the P5 web application. Keep them per server.
2. **Look the exact path up** with the `database` header naming the index, using the source path with a leading slash and without the label. A hit returns the entry handle and the volume ID.
3. **If it is not there, the media may have been moved before it was archived** (into a "To Archive" folder, say), so the path in the index is not the path a project names. List the volumes under their plain labels for the project's folders, within a bounded number of requests, and confirm each candidate path with a lookup. Nothing counts as found until P5 has confirmed it.
4. **Restore from the entry handle**, preserved exactly as P5 returned it. Ask which volume holds the item and whether it is online; the volume ID is the reliable link, since an index name can differ from the volume's own label.

A misspelled label is answered 404 like a label that is not in the index, so an application should report a label it could not list rather than skip it silently. One misspelled label once hid the only volume that held a whole project.

## An uncertain submission

A timeout can occur after a server has accepted work but before the client receives the response. Repeating the request immediately could create another archive or restore job.

P5Kit's mutation receipt distinguishes requests known not to have been sent, uncertain delivery, and received responses. Known non-delivery calls for fresh resolution and authorization. Uncertain delivery calls for reconciliation before retry. Received responses are recorded without automatic replay.

Applications must implement the reconciliation step using available job and operation evidence. P5Kit does not promise exactly-once execution or automatically decide that a failed connection means no job exists.

## Validation boundaries

The private package includes automated tests for connection models, endpoint resolution, mutation decisions, request encoding, response parsing, redaction, and transports. Some transport tests exercise a local loopback server; those are not live P5 compatibility tests.

Six applications consume the package (see the overview). Adoption does not mean each uses every connection or safety foundation. The examples here are reviewed request shapes and were not executed against a P5 server as part of publishing this documentation. The imported-volume behaviour described above was measured against a live P5 server on 2026-09-25.
