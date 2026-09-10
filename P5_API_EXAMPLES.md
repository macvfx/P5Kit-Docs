# P5 REST API examples

[Overview](README.md) · [Capabilities](CAPABILITIES.md) · [Workflow and evidence](WORKFLOWS.md)

These are illustrative HTTP requests, not P5Kit source code. They show the P5 operations behind the documented integration. Compare them with the [official P5 REST API reference](https://blog.archiware.com/redoc/p5_rest_api/awp5api.html) and the API version enabled on your server.

All hosts, identifiers, paths, and credentials below are fictional or placeholders. The example base is `https://p5.example.invalid/rest/v1`; use your configured host, port, and version. `<base64-credentials>` represents HTTP Basic authentication supplied securely by the client. Use HTTPS with normal certificate validation. No example was submitted to a live P5 server during documentation publication.

## Read server information

```http
GET /rest/v1/general/srvinfo HTTP/1.1
Host: p5.example.invalid
Authorization: Basic <base64-credentials>
Accept: application/json
```

This reads server information. A successful response alone does not prove that a particular archive plan is safe or that all client paths are accessible.

## Discover archive plans

```http
GET /rest/v1/archive/plans HTTP/1.1
Host: p5.example.invalid
Authorization: Basic <base64-credentials>
Accept: application/json
```

Use identifiers returned by the server when reading a plan or submitting work. Display labels and plan identifiers need not be interchangeable.

## Resolve an exact archived path

```http
GET /rest/v1/archive/entries HTTP/1.1
Host: p5.example.invalid
Authorization: Basic <base64-credentials>
Accept: application/json
client: <client-id>
database: <index-id>
path: /Volumes/Example_Archive/Example_Project/Example_Clip.mov
```

The `database` header selects the archive index for this lookup. P5Kit's exact lookup supports omitting it when an index is not supplied. Preserve the returned opaque entry handle. Paths containing spaces or other special characters need appropriate header encoding; this example deliberately avoids them.

## Submit an archive selection

This request starts work. Review the selected plan, client, paths, and plan-specific source handling before submitting it.

```http
POST /rest/v1/archive/plans/<plan-id>/archiveselections HTTP/1.1
Host: p5.example.invalid
Authorization: Basic <base64-credentials>
Accept: application/json
Content-Type: application/json
client: <client-id>
time: now

{
  "paths": [
    {
      "path": "/Volumes/Example_Archive/Example_Project/Example_Clip.mov",
      "meta": []
    }
  ],
  "description": "Example archive selection"
}
```

Save the returned job identifier and entry information. An accepted request is not a completed archive. If the response is lost, reconcile before retrying.

## Read job status

```http
GET /rest/v1/general/jobs/<job-id> HTTP/1.1
Host: p5.example.invalid
Authorization: Basic <base64-credentials>
Accept: application/json
```

Inspect both execution state and completion result. Applications should bound polling and retain enough evidence to revisit uncertain outcomes.

## Read the job report

```http
GET /rest/v1/general/jobs/<job-id>/report HTTP/1.1
Host: p5.example.invalid
Authorization: Basic <base64-credentials>
Accept: text/plain
```

The report is requested as plain text. It can include operational details; review it before sharing.

## Submit a restore selection

This request writes restored data. Replace the entry placeholder with the exact handle returned by P5 and check the destination before submission.

```http
POST /rest/v1/restore/restoreselections HTTP/1.1
Host: p5.example.invalid
Authorization: Basic <base64-credentials>
Accept: application/json
Content-Type: application/json
client: <client-id>
relocate: /Volumes/Example_Restore
time: now

{
  "entries": [
    {
      "ID": "<opaque-entry-handle>",
      "targetPath": "Example_Project/Example_Clip.mov"
    }
  ],
  "description": "Example restore selection"
}
```

Here `targetPath` is relative; `relocate` names the chosen restore root. Keep the returned job identifier, follow the job, and inspect the restored file. Matching the expected content requires separate verification.
