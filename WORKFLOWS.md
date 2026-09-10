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

## An uncertain submission

A timeout can occur after a server has accepted work but before the client receives the response. Repeating the request immediately could create another archive or restore job.

P5Kit's mutation receipt distinguishes requests known not to have been sent, uncertain delivery, and received responses. Known non-delivery calls for fresh resolution and authorization. Uncertain delivery calls for reconciliation before retry. Received responses are recorded without automatic replay.

Applications must implement the reconciliation step using available job and operation evidence. P5Kit does not promise exactly-once execution or automatically decide that a failed connection means no job exists.

## Validation boundaries

The private package includes automated tests for connection models, endpoint resolution, mutation decisions, request encoding, response parsing, redaction, and transports. Some transport tests exercise a local loopback server; those are not live P5 compatibility tests.

The current consumers are CopyTrust and P5 Search Jumper. Adoption does not mean each uses every connection or safety foundation. The examples here are reviewed request shapes and were not executed against a P5 server as part of publishing this documentation.
