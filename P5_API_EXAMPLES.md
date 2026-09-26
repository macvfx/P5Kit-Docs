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

## Read volumes

```http
GET /rest/v1/general/volumes HTTP/1.1
Host: p5.example.invalid
Authorization: Basic <base64-credentials>
Accept: application/json
```

The list returns identifiers and links only. Read each volume with `GET /rest/v1/general/volumes/<volume-id>`, which returns its label, barcode, online state, location and sizes. Sizes are in kilobytes.

## List one level of an imported-volumes index

An index that holds imported volumes is rooted at the volume's plain label:

```http
GET /rest/v1/archive/indexes/Imported-Volumes/inventory/Example_Import.0001/Volumes HTTP/1.1
Host: p5.example.invalid
Authorization: Basic <base64-credentials>
Accept: application/json
```

This lists the folders under the label. `inventory/Volumes` and the index root are answered 404 with the body `{}`, and so is a name shaped `Example_Import.0001-<uuid>`, which is how the P5 web application can display the same folder. P5 does not list the labels; see the capabilities page for how applications find them.

## Resolve an exact path in an imported-volumes index

```http
GET /rest/v1/archive/entries HTTP/1.1
Host: p5.example.invalid
Authorization: Basic <base64-credentials>
Accept: application/json
client: <client-id>
database: Imported-Volumes
path: /Volumes/Example_Source/Example_Project/Example_Clip.mov
```

The path has a leading slash and does **not** start with the volume's label. A path such as `/Example_Import.0001/Volumes/Example_Source/…` is answered HTTP 500 with a text body saying the entry could not be resolved. Without the `database` header the lookup does not reach this index at all.

## Error responses

| Situation | Response |
| --- | --- |
| Wrong password | HTTP 400, plain-text body `Wrong username or password.` (not 401) |
| Entry lookup found nothing | HTTP 500, plain-text body saying the handle could not be resolved: unknown entry |
| Inventory path not in the index | HTTP 404, body `{}` |

A 404 carries no reason, so "no such label", "label in another index" and "nothing imported" look the same.

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
