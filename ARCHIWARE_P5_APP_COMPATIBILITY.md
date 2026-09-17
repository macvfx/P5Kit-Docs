# Archiware P5 Workflow Compatibility

**Which of these macOS apps talk to Archiware P5, how they talk to it, and what they add to a P5 site.**

Versions checked against published GitHub releases on 2026-09-17. Live-server behaviour tested
against **Archiware P5 8.0.4**.

---

## The short version

P5 is the source of truth for what is on tape. It is not, however, a desk-side tool. Three
questions come up every week in a post facility and P5 answers none of them quickly:

1. *This project folder is 14 TB and the volume is full — is every file already on tape, so I can delete it?*
2. *We just offloaded eight camera cards. How does the camera, reel, card and ingest hash get into the archive index so somebody can search for it in two years?*
3. *Which tape is `A047_C013_0812XY.R3D` on, when the P5 server is on the other side of a VPN?*

These apps answer those three, and they do it without changing how P5 is administered.
**Everything is read-only against P5 except two clearly-marked write paths** — submitting an
archive job, and submitting a restore job. Four of the eleven apps can take one of those two
paths, each of them only when explicitly asked.

---

## Compatibility at a glance

| App | Version | How it reaches P5 | Writes to P5? |
| --- | --- | --- | --- |
| **P5 Archive Manager API** | 4.0.0 build 25 · beta | REST API v1 | Yes — archive submit (opt-in) |
| **P5 Archive Manager** (nsdchat edition) | 3.7.1 build 5 | `nsdchat` CLI, local | No |
| **CopyTrust** | 2.8.3 build 24 | REST API v1 | Yes — archive submit (opt-in) |
| **P5 Archive Overview** | 2.0.2 build 10 | REST `/archive/overview` | No |
| **P5 Archive Search** | 2.6 build 17 | REST API v1 | Yes — restore submit |
| **P5 Archive Browser** | 0.34 build 50 · beta | TSV inventories + REST API v1 | Yes — restore submit (off by default) |
| **P5 Archive Export** | 1.5 build 4 | `resources.db` read-only + `nsdchat` | No |
| **P5 Health Check** (Mac / menu bar / iPhone / CLI) | 1.7.1 build 3 | REST API v1 | No |
| **P5 Search Jumper** | 0.2.2-beta build 6 | REST `/restore/restoreselections` | Yes — restore submit |
| **Drop Verify**, **MHL Verify**, **Folder Copy Compare** | 2.8.3 build 24 / 2.6.0 | No P5 connection | No |

### Endpoints used

```
GET  /rest/v1/archive/indexes/{index}/inventory/{path}   is this file archived, and where
GET  /rest/v1/archive/overview                           current and recent archive jobs
GET  /rest/v1/general/volumes/{id}                       volume label, location, barcode, media type
GET  /rest/v1/general/clients                            archive clients
GET  /rest/v1/archive/plans                              archive plans, and whether they delete the source
GET  /rest/v1/archive/entries                            resolve a path to an archive entry
GET  /rest/v1/general/jobs/{jobID}                       job monitoring
POST /rest/v1/archive/plans/{planID}/archiveselections   submit an archive job
POST /rest/v1/restore/restoreselections                  submit a restore job
```

Those two `POST` calls are the only writes in the whole toolkit. Four apps can make one:
CopyTrust and P5 Archive Manager API submit archive jobs, and P5 Archive Browser,
P5 Archive Search and P5 Search Jumper submit restore jobs. Every other app makes `GET` calls
only, reads `resources.db` in read-only mode, or reads TSV inventories exported from `nsdchat`.

---

## Workflow 1 — Camera card to LTO, with the ingest metadata intact

**Apps:** CopyTrust → Archiware P5

The problem this solves is not copying. It is that everything known about a clip at the moment
it comes off the card — which camera, which reel, which card, what codec, what start timecode,
what the verified hash was — is known only by the ingest station, and P5 normally receives a
path and a byte count.

### The sequence

1. **Detect the card.** CopyTrust recognises camera-card structures (DCIM and camera folder
   signatures) rather than treating the volume as an anonymous folder tree.
2. **Copy to every destination at once.** Multi-source, multi-destination, with a per-destination
   preflight that refuses a read-only or over-full target before a byte moves.
3. **Verify with xxHash64** — None, Quick or Full. The hash is computed while the source is being
   read, so verification costs one pass, not two.
4. **Check the manifest the card arrived with.** If another tool (ShotPut Pro, OffShoot,
   Silverstack, YoYotta, or the camera) left an MHL beside the footage, CopyTrust reads it and
   reports whether the footage still matches what that tool recorded. This is kept separate from
   CopyTrust's own verification, because they are different claims — a card that arrived damaged
   copies and verifies perfectly, and only the arrival check can say otherwise.
5. **Write an MHL v1.1 manifest** plus a receipts folder: contact sheet PDF, EXIF CSV, per-ingest
   logs, session summary, and a JSON/TXT receipt.
6. **Archive the chosen destination to P5.** A preset says *which* destination archives to P5 and
   which one makes proxies, and it can carry the P5 server, index, client and plan — so the
   operator supplies only a password. CopyTrust reads the plan first and knows whether that plan
   deletes its source.

### What P5 actually gains

Before archiving, CopyTrust checks that the target archive index carries twelve custom metadata
keys, and refuses to archive without them. (An administrator installs them once, from Settings.)
Each archived file then carries:

| Key | Shown in P5 as | Holds |
| --- | --- | --- |
| `CT_ASSET` | CT Asset | Asset identifier |
| `CT_XXH64` | CT xxHash64 | The xxHash64 recorded at ingest |
| `CT_FRAME` | Frame Size | Frame or image size |
| `CT_KIND` | Media Kind | Media kind |
| `CT_CODEC` | Codec | Codec |
| `CT_FPS` | Frame Rate | Frame rate |
| `CT_TC` | Start TC | Start timecode |
| `CT_REEL` | Reel | Reel or clip |
| `CT_CARD` | Source Card | Source card |
| `CT_CAMERA` | Camera | Camera ID |
| `CT_DATE` | Capture Date | Capture date |
| `CT_STATE` | CT State | Verify state at ingest |

These are ordinary P5 index metadata fields. They are searchable in P5 Web, they survive in the
index, and they mean an archive search two years later can be *"every clip from camera B on
2026-08-12"* rather than a path guess.

`CT_XXH64` is a searchable contextual copy of the ingest hash — useful for finding and diagnosing
one item. It does not replace verifying a restored delivery against its preserved MHL; the MHL
remains the file-by-file integrity authority.

### Closing the loop after a restore

`card → verified copy → MHL → P5 archive → P5 restore → MHL verification`

Three separate proofs, all of which should pass before a restored copy is accepted:

- **P5 job completion** — P5 says the restore job finished.
- **Restore reconciliation** — the expected paths, file count and bytes arrived.
- **MHL verification** — each restored file's xxHash64 is recalculated and compared with the value
  recorded at ingest, using CopyTrust's *Verify Using MHL*, MHL Verify, or the `mhl-tool` CLI.

---

## Workflow 2 — Reclaim disk space you can prove is on tape

**App:** P5 Archive Manager API → Archiware P5

This is the folder-check-and-delete loop, for projects that are archived and restored repeatedly
and where the disk copy must be proven to match the tape copy before the space is freed.

### The sequence

1. **Drop the folder.** The app enumerates the folder on disk, then queries only the P5 archive
   directories that could hold those files — local-driven, so a large index is not walked.
2. **Map the path if storage has moved.** Per-server prefix substitutions handle
   `/Volumes/storageA` → `Volumes/storageB`, longest match wins. An ordered list of *historical*
   archived locations is tried when the current path returns nothing, so a project that moved
   between volumes since it was archived is still found.
3. **Search every index, optionally.** A file is found wherever it lives; the matching index is
   recorded per file.
4. **Read the per-file verdict.** Each file comes back as `exact`, `close` (within 64 KB),
   `size mismatch`, or `not archived`, with disk size and modification date against P5 size,
   archive date, volume, barcode, location and media type. Symlinks are flagged. Case differences
   are auto-corrected (`BC CANCER` → `BC Cancer`) with sibling suggestions when a folder is not found.
5. **Write the proof report before touching anything.** A CSV and a TXT report, built from the P5
   JSON, covering per-file evidence plus a not-archived section. It is written before any delete,
   and can be saved on demand at any time.
6. **Delete, verify-first.** The delete chain is deep-verify → open receipts → delete now. It
   re-checks each candidate immediately before removing it, deletes only files verified in P5, and
   exports a human-readable receipt of what was removed and why it was safe.
7. **Archive the remainder.** Files that came back `not archived` can be submitted to a chosen P5
   plan and client, the job is monitored, and the folder is re-checked afterwards so the result
   reflects the new state.

Delete and Archive are one-shot: after running they grey out and relabel, so a stale result cannot
be acted on twice.

Every API call, archive job step and error is written to a per-launch log, and *Zip logs to
Desktop* bundles them for support.

### Two editions, deliberately separate

**P5 Archive Manager 3.7.1** drives `nsdchat` and therefore has to run on a machine with local CLI
access to P5. **P5 Archive Manager API 4.0.0** connects over the network, searches every configured
archive index, and needs no CLI access at all. Different apps, different bundle identifiers,
different settings, different release channels — running both is fine.

---

## The rest of the toolkit

| Question | App |
| --- | --- |
| Is the LTO system clean and ready to archive right now? | **P5 Health Check** — server uptime, warning and failed jobs, drives needing cleaning, jukebox status, licences, and Appendable / Readonly / Full volume counts. Mac app, menu bar monitor, iPhone app and CLI script. |
| What has been archived lately, and how full are the pools? | **P5 Archive Overview** — current and completed jobs, pool usage, SQLite history, CSV/JSON export. |
| Where is this folder inside the archive, and how big is it — and can I get it back? | **P5 Archive Search** — browse the archive index with breadcrumbs, search a local SQLite copy across every configured server, calculate folder sizes, show volume barcodes. Restores a single file or a whole folder, reporting which tape is needed and whether P5 can reach it before anything is submitted. |
| Which tape is this file on — while the server is unreachable? | **P5 Archive Browser** — imports per-volume TSV inventories into a local SQLite catalog and searches across every tape offline. Archive groups carry LTO generation, barcode, location and health. Optional REST checks refresh volume metadata, and **Restore Folder from P5 Archive** recovers a whole folder subtree in one request. |
| I need every archive job, or a per-volume inventory, as a file. | **P5 Archive Export** — 13 analytical SQL queries against `resources.db` (read-only), `nsdchat`-driven per-volume TSV inventories, volume list CSV, and compressed P5 config/log backups, on a schedule. Windowed app and menu bar companion share one settings store. |
| Two folders should be identical — are they? | **Folder Copy Compare** — full-tree xxHash64 comparison with per-subfolder drill-down, and P5 stub file cleanup so a restored folder validates against what you actually need. |
| I have an MHL and want to re-check the media. | **MHL Verify** — reads MHL v1.1 from any compatible tool; matched, mismatched and missing reported explicitly. |
| One folder needs trust artifacts, without a copy. | **Drop Verify** — MHL manifest, contact sheet PDF and EXIF CSV from a single dropped folder. |
| I know the shot but not the filename, and it is on tape. | **P5 Search Jumper** — semantic visual and transcript search over a Jumper analysis cache; when the source file is offline, restore that exact ArchiveEntry from P5. |

### Restoring from P5 Archive Search

Search restores a single file or one whole folder, from a search result or while browsing. Before
anything is submitted it resolves the item to an archive entry, so what is confirmed is something
that was found rather than assumed, and it asks P5 which volume the item is on, whether that volume
is online and where it is. An offline tape does not block the restore — P5 waits and asks for it —
but the operator is told that is what will happen.

A folder is restored with one directory handle and P5 performs the recursion, so a folder restore
can cover more than the app has itself scanned. A restore is submitted once and never retried,
because a second request would run a second job; a submission that times out is reported as
delivery uncertain rather than as a failure. Unfinished jobs are picked up again when the app
reopens, and every attempt is recorded with its job number and outcome.

### Restoring from P5 Archive Browser

Browser's restore is worth calling out, because it does not trust P5's own answer.

Right-click a folder in the File Browser and choose **Restore Folder from P5 Archive…**. The
confirmation sheet shows the expected file count and byte total — taken from Browser's own imported
catalog rather than P5's directory listing — the tape's current online/offline state, and exactly
where the files will land, computed from measured `relocate` behaviour rather than assumed. One
directory handle restores the entire subtree in a single request, confirmed against a live P5 8.0.4
server.

When the job finishes, Browser independently reconciles what actually landed against what was
expected, byte-level rather than count-level, whenever the destination is readable from that Mac —
and reports missing or unexpected files. **P5 has been observed to report a completed job while
silently omitting files.** Every attempt is logged.

Restore is off until enabled in *Settings ▸ Restore ▸ Enable P5 Restore*.

---

## Requirements and honest caveats

**Requirements**

- macOS 14 or newer for most apps; P5 Archive Manager and P5 Archive Browser need macOS 14.6, and
  P5 Archive Export runs back to macOS 12.
- The REST apps need the P5 REST API reachable and an account that can read the archive indexes.
  Live behaviour is tested against P5 8.0.4.
- The `nsdchat` apps (P5 Archive Manager 3.x, P5 Archive Export volume inventory) need local CLI
  access to a P5 installation.
- Passwords are stored per server in the macOS Keychain.
- Transport is moving from `curl` to `URLSession`. Several apps still shell out to `curl`, which
  was originally chosen because App Transport Security blocks cleartext HTTP and most P5 hosts
  speak plain HTTP. That turned out to be avoidable: P5 Archive Search 2.6 runs entirely on
  `URLSession` against plain-HTTP servers, including over an overlay network, by declaring the
  exception in its `Info.plist`. The remaining `curl` apps are being moved the same way.

**Caveats — read these before pointing anything at production**

- **Archive submission is a write, and it is beta.** Both CopyTrust's post-copy submission and
  P5 Archive Manager API's *Archive the not-archived* call
  `POST /rest/v1/archive/plans/{planID}/archiveselections`. Test on a disposable plan first. Paths
  submit as the chosen client sees them.
- **Delete is real.** Test the delete chain on a throwaway folder and read the receipt before
  trusting it on a project volume.
- **These apps do not replace P5 Web.** Job review, plan configuration, retention and tape
  management stay where they belong.
- **Restore in P5 Archive Browser is off by default** and must be enabled in
  *Settings ▸ Restore ▸ Enable P5 Restore*. It restores whole folders only; individual-file
  restore is deliberately not built, because per-file selections are known to flatten the restored
  tree without per-entry `targetPath` containment, which is not yet verified.
- **Several apps are pre-release.** P5 Archive Manager API 4.0.0, P5 Archive Browser 0.31 and
  P5 Search Jumper 0.2.1 are betas.
- **Proxies are derivatives.** They never replace verified originals in an archive.

---

## Where to get them

All apps are free, distributed as signed and notarized DMGs from GitHub Releases, and listed at
`code.matx.ca`.

| App | Repository |
| --- | --- |
| P5 Archive Manager (both editions) | `macvfx/p5ArchiveManager` |
| P5 Archive Overview | `macvfx/p5ArchiveOverview` |
| P5 Archive Search | `macvfx/p5ArchiveSearch` |
| P5 Archive Browser | `macvfx/P5-Archive-Browser` |
| P5 Archive Export | `macvfx/p5ArchiveExport` |
| P5 Health Check | `macvfx/p5HealthCheck` |
| P5 Search Jumper | `macvfx/P5-Search-Jumper` |
| CopyTrust, Drop Verify, Folder Copy Compare | `macvfx/MHL` |
| MHL Verify | `macvfx/MHL-Verify` |
| P5 bash scripts | `macvfx/Archiware` |
