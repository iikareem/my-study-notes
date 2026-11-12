# Object Storage Design Patterns: Uploads, Consistency, and Delivery with S3 and TUS

I remember when I was learning AWS S3 and working on my first production system that needed to handle image uploads and retrieval at scale. I was going through the AWS docs, reading about presigned URLs, S3 buckets, IAM roles — trying to understand not just how to use the API, but how to design something that would actually hold up under real conditions.

The more I learned, the more questions surfaced. What happens if the upload succeeds in S3 but my server never finds out? What if two requests confirm the same upload at the same time? What if a user's connection drops halfway through a 2GB file? What if I need to serve private files fast without routing every byte through my server?

I organized everything I learned into this article — the architecture decisions, the failure modes, the tradeoffs, and the techniques for keeping S3 and your database in sync. It's meant to be a reference for building object storage upload and retrieval in a scalable, consistent way from the ground up.

---

## The Core Problem — Why You Can't Route Files Through Your Server

The first instinct when building file uploads is to route everything through your app server:

```
Client ──── 500MB ──▶ App Server ──── 500MB ──▶ S3
```

It works locally. In production it creates a set of problems that compound quickly:

| Problem | What It Looks Like |
|---|---|
| **Double bandwidth cost** | Your server pays for 500MB ingress from the client AND 500MB egress to S3 — billed twice for the same data |
| **Memory exhaustion** | Even streaming large bodies keeps chunks in memory — 50 concurrent uploads of 100MB each means 5GB of working memory just for file forwarding |
| **Thread starvation** | One large upload holds a worker thread for minutes, blocking hundreds of normal requests |
| **Timeout bombs** | Nginx, ALBs, and most reverse proxies default to 60–120 second timeouts — large uploads die silently mid-transfer |
| **Latency** | The client is blocked waiting for your server to fully forward the file before it gets any response |

The fix is architectural: your server should be in the **authorization path**, not the **data path**.

```
Step 1 — Client asks server for permission:
  Client ──────────────────▶ App Server
  Client ◀──── signed URL ───────────

Step 2 — Client uploads directly to storage:
  Client ─────── 500MB ──────────────▶ S3
```

The server signs a ticket. The client uses that ticket to talk to S3 directly. Your server never touches the bytes.

> **The server's job is to sign, not to carry.**

---

## Presigned URLs — How the Authorization Works

A **presigned URL** is a time-limited, cryptographically signed URL that delegates exactly one storage operation to whoever holds it. The client needs no AWS credentials — the URL itself is the proof of authorization.

When your server generates a presigned URL, S3 produces something like this:

```
https://s3.amazonaws.com/my-bucket/uploads/user-42/avatar.png
  ?X-Amz-Algorithm=AWS4-HMAC-SHA256
  &X-Amz-Credential=AKIAIOSFODNN7EXAMPLE...
  &X-Amz-Date=20250725T120000Z
  &X-Amz-Expires=300
  &X-Amz-SignedHeaders=content-type;host
  &X-Amz-Signature=<HMAC of all the above using your secret key>
```

When the client uses this URL, S3 re-derives the expected signature using your secret key. If they match and the expiry hasn't passed — the operation is allowed. **Your secret key never leaves your server.**

```js
const presignedUrl = s3.getSignedUrl('putObject', {
  Bucket: 'my-bucket',
  Key: `uploads/user-42/${uuid}.png`,
  Expires: 300,             // 5 minutes — URL is useless after this
  ContentType: 'image/png', // S3 rejects if client sends wrong type
});
```

| Property | What It Enforces |
|---|---|
| **Time-scoped** | Expires after N seconds — a leaked URL becomes worthless quickly |
| **Operation-scoped** | A PUT URL cannot be used to GET the same object |
| **Key-scoped** | Tied to one exact object path — can't be used for a different key |
| **Content-type enforced** | S3 returns 403 if the client sends the wrong content type |
| **Credential-free for client** | Client needs no AWS credentials whatsoever |

---

## S3 Versioning — Enable It Before You Need It

Before going into the upload flow and consistency strategies, there is one S3 setting you should turn on from the very beginning: **versioning**. It affects how you design your metadata schema, how you recover from failures, and how your DB records relate to actual objects in S3. Understanding it now makes everything that follows clearer.

S3 Versioning means that every time an object at a given key is overwritten or deleted, S3 doesn't throw away the old bytes. It keeps every version, each with its own unique `VersionId`. A normal GET returns the current version. Every previous version is still there, retrievable by ID.

```
PUT uploads/user-42/avatar.png  →  VersionId: "abc111"  ← current
PUT uploads/user-42/avatar.png  →  VersionId: "abc222"  ← current
PUT uploads/user-42/avatar.png  →  VersionId: "abc333"  ← current

GET uploads/user-42/avatar.png                    → returns "abc333" (latest)
GET uploads/user-42/avatar.png?versionId=abc111   → returns the original
```

### What Problems It Solves

**Accidental overwrites.** A bug in key generation, a migration script that reused an existing path, a re-upload that clobbered the wrong file — without versioning, that data is gone. With versioning, you restore it in one API call.

**Accidental deletes.** A DELETE on a versioned bucket doesn't remove the object. It places a "delete marker" — the object appears gone to normal GET requests, but every version is still there. Undo the delete by removing the marker.

**Audit trail for sensitive files.** For regulated industries — healthcare, finance, legal — being able to prove what a file contained at a specific point in time is sometimes a hard requirement. Versioning gives you this for free.

**Safer data migrations.** Running a script that transforms or moves files in S3? If something goes wrong mid-run, versioning gives you a clean rollback path instead of trying to reconstruct what was there before.

### The Cost Tradeoff

Every version is stored and billed independently. A 10MB image re-uploaded 5 times costs 50MB of storage, not 10MB. Manage this with a lifecycle rule:

```json
{
  "Rules": [{
    "ID": "ExpireOldVersions",
    "Status": "Enabled",
    "NoncurrentVersionExpiration": {
      "NoncurrentDays": 30
    }
  }]
}
```

This keeps the current version forever and deletes anything older than 30 days automatically.

### How Versioning Changes Your Metadata — Three Approaches

This is the part most articles skip. When versioning is on, the same S3 key can point to different bytes over time. How you track this in your DB depends entirely on what your application needs.

---

#### Approach A — Always Overwrite (Only Care About Latest)

One row per file. Each new upload overwrites the previous record's `s3_version_id`. You only ever need the current version, so there's no point keeping history in the DB.

```sql
-- Schema: one row per logical file
CREATE TABLE files (
  file_id       UUID PRIMARY KEY,
  user_id       UUID NOT NULL,
  s3_key        TEXT NOT NULL,
  s3_version_id TEXT,           -- updated on every re-upload
  status        TEXT NOT NULL,
  updated_at    TIMESTAMPTZ
);
```

On each new upload, you upsert the existing row:

```js
await db.query(`
  UPDATE files
  SET s3_version_id = $1, status = 'complete', updated_at = NOW()
  WHERE file_id = $2
`, [versionId, fileId]);
```

S3 still keeps old versions for 30 days (your lifecycle rule). Your DB only knows about the latest. Simple, no history overhead.

**Use when:** users replace files and only the current version matters — profile pictures, cover images, document drafts.

---

#### Approach B — Append New Row Per Upload (Full History)

Every upload creates a new row. Each row is a self-contained record pointing to an exact `VersionId`. Your DB and S3 both keep full history. Your application queries for "current" using ordering or a flag.

```sql
-- Schema: one row per upload event
CREATE TABLE file_uploads (
  upload_id     UUID PRIMARY KEY,
  file_id       UUID NOT NULL,       -- logical file grouping
  user_id       UUID NOT NULL,
  s3_key        TEXT NOT NULL,
  s3_version_id TEXT NOT NULL,       -- exact version this row describes
  status        TEXT NOT NULL,
  created_at    TIMESTAMPTZ NOT NULL
);

-- To fetch the current version:
SELECT * FROM file_uploads
WHERE file_id = $1
ORDER BY created_at DESC
LIMIT 1;
```

Each row is completely independent. Rolling back means pointing your application at an older `upload_id`. Auditing means querying the full history for a `file_id`.

**Use when:** you need a full upload history — document management, regulated content, audit logs, "restore previous version" features.

---

#### Approach C — Hybrid: Current Pointer + History Table

One row tracks the current state of a logical file. A separate history table logs every upload event. This separates "what is the file right now?" from "what has ever happened to this file?"

```sql
-- Current state: one row per logical file
CREATE TABLE files (
  file_id            UUID PRIMARY KEY,
  user_id            UUID NOT NULL,
  s3_key             TEXT NOT NULL,
  current_version_id TEXT NOT NULL,   -- points to the active S3 VersionId
  current_upload_id  UUID,            -- FK to upload_history
  status             TEXT NOT NULL,
  updated_at         TIMESTAMPTZ
);

-- History: one row per upload event
CREATE TABLE upload_history (
  upload_id     UUID PRIMARY KEY,
  file_id       UUID NOT NULL REFERENCES files(file_id),
  s3_key        TEXT NOT NULL,
  s3_version_id TEXT NOT NULL,
  uploaded_at   TIMESTAMPTZ NOT NULL
);
```

On a new upload you do two writes in one transaction: insert a row into `upload_history`, then update `files` to point `current_version_id` at the new version.

**Use when:** your application needs fast single-row lookups for the current version (no ordering, no LIMIT) but also needs a full audit trail. The tradeoff is two writes per upload and a join when you want history.

---

### Which Approach to Use

| Approach | DB rows per upload | History | Best For |
|---|---|---|---|
| **Always Overwrite** | 1 per file (updated) | None in DB | Profile pictures, replaceable assets |
| **Append New Row** | 1 per upload event | Full in DB | Documents, audit requirements, version restore |
| **Hybrid** | 1 current + 1 history | Full in DB | Apps needing both fast current lookup and full history |

> **Enable versioning on your upload bucket from the start.** Turning it on after the fact works — but you'll have existing objects with no version history, and you'll have missed the window where you most needed the safety net.

---

## The Upload Flow — A Three-Phase Handshake

The upload is a three-phase handshake between client, app server, and S3. Each phase has a specific responsibility.

```
Phase 1 — Initiate
  Client ──▶ App Server ──▶ S3 (presign)
  Client ◀── upload_id + URL ◀── App Server

Phase 2 — Direct PUT
  Client ── binary bytes ──▶ S3
  Client ◀── 200 OK ────────── S3

Phase 3 — Confirm
  Client ──▶ App Server ── HeadObject ──▶ S3
  App Server verifies object, updates DB
  Client ◀── { status: "complete" }
```

### Phase 1 — Initiation

```http
POST /api/upload/initiate
Content-Type: application/json

{
  "filename": "demo-video.mp4",
  "content_type": "video/mp4",
  "size_bytes": 104857600
}
```

Your server does five things in sequence:

1. **Authenticates** the request — is this user logged in?
2. **Validates** the upload — allowed content type? within size limit?
3. **Generates** a unique, unguessable storage key — never use user-supplied filenames as keys directly
4. **Creates a `pending` record** in your DB — this is the state machine entry point
5. **Calls S3** to generate a presigned PUT URL, then returns it

```json
{
  "upload_id": "upl_a3f9c2",
  "presigned_url": "https://s3.amazonaws.com/bucket/uploads/...",
  "expires_at": "2025-07-25T14:05:00Z"
}
```

> **Why create the DB record before the upload succeeds?** Because it gives you a handle to track state and clean up failures. Without this record, you have no idea an upload was ever started — no way to detect orphaned objects, no way to notify the user of a failure, no way to resume. The pending record is your source of truth from the moment intent is expressed.

### Phase 2 — Direct Upload

The client uses the presigned URL directly — your server is not involved:

```http
PUT https://s3.amazonaws.com/bucket/uploads/...?<signature>
Content-Type: video/mp4
Content-Length: 104857600

<binary data>
```

S3 validates the signature. If `Content-Type` doesn't match what was signed, S3 rejects with 403. Your server never sees this traffic.

### Phase 3 — Confirmation

After S3 returns `200 OK`, the client notifies your server:

```http
POST /api/upload/confirm
Content-Type: application/json

{
  "upload_id": "upl_a3f9c2"
}
```

Your server:

1. Calls `HeadObject` on S3 — verify the object actually exists and get its `VersionId`
2. Updates the DB: `pending → complete`, stores `s3_version_id`
3. Triggers downstream processing asynchronously

> **Why call HeadObject instead of trusting the client?** A client could send a fake confirmation without ever uploading. Or the upload could have partially succeeded and S3 rejected it silently. `HeadObject` is ground truth — S3 either has the object or it doesn't. You never mark a record complete based on what the client claims.

---

## Metadata Consistency — Keeping S3 and Your DB in Sync

The blob lives in S3. The metadata lives in your DB. They must always agree. Multiple failure scenarios can split them apart — and this is where most systems have silent bugs that surface days later.

### The Two Failure Modes

**Mode 1 — DB says complete, S3 object missing:**

```
DB:  { status: "complete", url: "s3://my-bucket/uploads/xyz.mp4" }
S3:  [object does not exist — upload failed silently]
```

Users get 404s or broken media players with no explanation.

**Mode 2 — S3 object exists, DB says pending:**

```
S3:  [object fully uploaded and sitting in S3]
DB:  { status: "pending" }  ← server crashed between HeadObject and DB write
```

The file is consuming storage, the user thinks the upload failed, and your app will never surface it.

---

### Strategy 1 — Confirmation Pattern (Standard)

The three-phase flow above. Server calls `HeadObject` before marking complete.

**What it handles:**

- Upload fails → pending record stays pending → cleanup job runs
- Confirmation crashes → object exists in S3, DB still says pending → reconciliation detects and fixes

**What it doesn't handle alone:** If the server crashes between `HeadObject` and the DB write, you're in Mode 2 until the reconciliation job runs. That's acceptable — the gap is seconds to minutes, not permanent.

---

### Strategy 2 — S3 Event Notifications (Async / Decoupled)

Configure S3 to emit an event every time an object is created. A Lambda or worker consumes it and updates your DB:

```
S3 ObjectCreated event
       │
       ▼
   SQS Queue
       │
       ▼
   Worker / Lambda
       │
       ▼
   DB: pending → complete
```

**Pros:**

- Completely decoupled — your app server doesn't need a confirmation endpoint
- Resilient — if the worker crashes, SQS retries automatically
- Fires regardless of how the object got there (migration, admin script, direct SDK call)

**Cons:**

- Eventual consistency — delay between upload and DB update
- Client can't get immediate synchronous confirmation
- More infrastructure to wire up

**When to use it:** When your client polls for status anyway (e.g. video processing pipelines), or when you want to decouple confirmation from post-processing entirely.

---

### What Goes on Top of Both Strategies

Both strategies share a common problem: the real world is messy. Clients retry. Networks duplicate requests. SQS delivers events more than once. The two techniques below don't replace Strategy 1 or Strategy 2 — you layer them on top of whichever one you pick.

---

#### Idempotent Writes

The DB write can be called more than once for the same upload:

- Client sends `/confirm`, gets a network timeout, retries
- SQS delivers the same S3 event twice (at-least-once delivery is a guarantee, not a bug)
- A load balancer retries a failed request automatically

This sounds scary, but an UPDATE with deterministic SET values is already idempotent by nature. Running the same UPDATE twice on the same row produces the same result both times:

```sql
UPDATE files
SET status = 'complete', updated_at = NOW()
WHERE upload_id = $1;
```

Run this 100 times — the row ends up identical every time. Same input, same output, no side effects.

**So when do you actually need the `AND status = 'pending'` guard?**

Only when you need to protect against a late or duplicate write clobbering a status that was legitimately changed by something else after the first write succeeded.

For example: the first `/confirm` call sets the row to `complete`. Then the virus scanner sets it to `quarantined`. Then a delayed retry of `/confirm` arrives and forces it back to `complete` — wiping out the quarantine. That's what the guard prevents:

```sql
UPDATE files
SET status = 'complete', updated_at = NOW()
WHERE upload_id = $1 AND status = 'pending'  -- ← only update if nothing else touched it
RETURNING *;
```

First call: row is `pending` → matches → updated to `complete`. Any retry: row is no longer `pending` → no match → 0 rows updated → safe no-op.

**The guard is not what makes it idempotent. The deterministic SET is.** The guard is an extra safety layer to prevent late retries from stomping on legitimate downstream state transitions.

---

#### The Reconciliation Job — Your Safety Net Under Everything

No matter which strategy you choose, run a periodic background job. It catches anything that slipped through:

```
Every 5 minutes:

1. SELECT * FROM files
   WHERE status = 'pending'
   AND created_at < NOW() - INTERVAL '15 minutes'

2. For each stale record:
   a. Call S3 HeadObject on the expected key
   b. Object exists  → mark complete, trigger processing
   c. Object missing → mark failed, notify user or queue retry
```

**What it catches:**

- Client crashed after upload, never sent `/confirm`
- Server crashed between HeadObject and DB write
- SQS event was lost before processing
- Any gap the primary strategy didn't cover

> **Design rule:** Never rely solely on the happy path. The reconciliation job is cheap to write and makes the entire system eventually self-healing.

---

### Which Strategy to Use

| Strategy | Consistency | Complexity | Best For |
|---|---|---|---|
| Confirmation Pattern | Strong (HeadObject as proof) | Low | Most applications |
| S3 Event Notifications | Eventual | Medium | Async pipelines, decoupled systems |
| Idempotent Writes | — (layered on top) | Low | Always add this to either strategy |
| Reconciliation Job | Eventually strong | Low | Always add this as a safety net |

---

## After the Upload — Two Independent Things Happen

Once a file lands in S3, two completely separate concerns kick in. They are triggered differently, owned by different parts of your system, and serve different purposes.

> Read this section carefully. The confusion between these two things is the most common reason this topic feels unclear.

---

### Thing 1 — Confirmation: "Did the Upload Happen?"

This is purely about **DB state**. The only question it answers is: does S3 have the object, and does my DB know about it?

```
Client uploads file to S3
       │
       ▼
Client calls POST /confirm
       │
       ▼
Your App Server calls HeadObject → S3 confirms object exists
       │
       ▼
DB: pending → complete
       │
       ▼
App Server returns { status: "complete" } to client immediately
```

No thumbnails, no virus scans, no processing. Just: the file is there, the DB knows about it, done. The client gets a synchronous response right away.

---

### Thing 2 — S3 Event: "Now Do Something With the File"

This is triggered by **S3 itself** the moment any object lands in the bucket — not by your confirm endpoint, not by your app server. AWS fires this automatically.

```
Object lands in S3
       │
       ▼  ← AWS infrastructure fires this, not your code
s3:ObjectCreated:* event
       │
       ▼
SQS Queue
       │
       ├──▶ Thumbnail Lambda     → generates preview image, writes thumbnail URL to DB
       ├──▶ Virus Scanner Lambda → scans the file, writes scan result to DB
       └──▶ (Notification)       → pushes WebSocket to client when all jobs finish
```

Your app server is **not in this flow at all**. This pipeline fires whether the object arrived through your upload flow, a migration, an admin script, or a direct SDK call.

---

### Why They Must Be Separate

| | Confirmation (`/confirm`) | S3 Event → Lambda |
|---|---|---|
| **Triggered by** | Client HTTP call | S3 automatically on every PUT |
| **Question it answers** | Did the upload happen? | Is the file ready to use? |
| **Owned by** | Your App Server | AWS Lambda (separate workers) |
| **Speed** | Synchronous — immediate response | Async — seconds or minutes |
| **Client waits?** | Yes — gets `{ status: "complete" }` back | No — client polls or receives WebSocket later |

They are not redundant. They are not alternatives. They solve different problems. If you generated the thumbnail inside `/confirm` synchronously, the client would sit waiting for minutes on every upload. That is why the split exists.

---

### What the Client Actually Experiences

```
Step 1: Client uploads binary directly to S3
Step 2: Client calls POST /confirm
        ← App Server responds immediately: { status: "complete" }
        ← Client knows the file landed. Upload flow is done.

Step 3: Client waits (polls GET /file/{id} every few seconds,
        or listens on a WebSocket channel)

Step 4: Lambda workers finish thumbnail generation and virus scan
        → Notification pushes WebSocket: { status: "ready" }

Step 5: Client shows the file as usable to the user
```

---

### How Status Flags Move Through the System

```
"pending"      → set by App Server          when POST /initiate is called
"complete"     → set by App Server          when POST /confirm succeeds
"processing"   → set by Lambda              when it picks up the S3 event and starts working
"ready"        → set by Lambda              when ALL processing jobs finish successfully
"quarantined"  → set by Lambda              when virus scan finds a threat
"failed"       → set by Reconciliation Job  when object is missing after timeout
```

---

### Do You Actually Need Multiple Lambdas?

Before jumping into coordination strategies, ask the question most articles skip: **do you actually need separate Lambdas in the first place?**

For most applications, a single Lambda that runs all post-processing jobs sequentially is completely fine:

```
S3 Event fires
       │
       ▼
Processing Lambda
  1. Generate thumbnail
  2. Run virus scan
  3. UPDATE status = 'ready'
  4. Push WebSocket to client
```

Simple, no coordination needed, easy to debug. This works well when your jobs are fast, your file volume is moderate, and one team owns the whole pipeline.

**You only need multiple Lambdas when you have a real reason:**

- Jobs take long enough that running them sequentially would time out a single Lambda (Lambda max timeout is 15 minutes)
- Jobs are genuinely independent and you want them to run in parallel to save wall-clock time — e.g. a 3-minute transcoding job and a 3-minute virus scan that you want done in 3 minutes total, not 6
- Different teams own different jobs and need to deploy them independently
- You want to scale each job separately based on its own load profile

If none of these apply, stay with one Lambda.

---

### Coordinating Multiple Lambdas — When You Actually Need It

If multiple Lambdas are the right call, the new problem is: how does the system know when **all** of them are done so it can set `status = 'ready'` and notify the user? Each Lambda reacts to the same S3 event independently. Neither knows the other exists.

---

#### Approach 1 — DB Checklist (Start Here)

Each Lambda writes its own result to dedicated columns, then checks whether all expected jobs are done. Whoever finishes last sets the final status and sends the notification.

**Thumbnail Lambda:**

```sql
UPDATE files
SET thumbnail_url = 'https://cdn/.../thumb.jpg', thumbnail_done = true
WHERE s3_key = $key;

SELECT thumbnail_done, virus_scan_done FROM files WHERE s3_key = $key;
-- if both true → set status = 'ready' and push WebSocket
```

**Virus Scanner Lambda:**

```sql
UPDATE files
SET scan_result = 'clean', virus_scan_done = true
WHERE s3_key = $key;

SELECT thumbnail_done, virus_scan_done FROM files WHERE s3_key = $key;
-- if both true → set status = 'ready' and push WebSocket
```

No extra infrastructure — just the DB you already have.

---

#### Approach 2 — AWS Step Functions (When You Need Visibility or Branching)

```
S3 Event fires
       │
       ▼
Step Functions State Machine starts
       │
       ├──▶ Thumbnail Lambda    ──┐
       ├──▶ Virus Scanner Lambda──┤  (runs both in parallel)
       │                          │
       └──────────── waits for BOTH to finish ──────────────┘
                    │
                    ▼
            Notification Lambda
            UPDATE files SET status = 'ready'
            Push WebSocket to client
```

Each Lambda just does its own job. The orchestrator handles "wait for all parallel branches", retries, and conditional branching — for example, "only run transcoding if the file is a video."

---

#### Approach 3 — SQS Fan-out + Aggregation Queue (When Teams Scale Independently)

```
S3 Event
       │
       ▼
SNS Topic  (broadcasts to multiple queues simultaneously)
       │
       ├──▶ Thumbnail Queue  → Thumbnail Lambda  → posts "thumbnail_done" to Results Queue
       └──▶ Scan Queue       → Scanner Lambda    → posts "scan_done"      to Results Queue

Results Queue
       │
       ▼
Aggregator Lambda
  reads each result message
  checks DB: are all expected jobs done?
  if yes → UPDATE status = 'ready', push WebSocket
```

No Lambda ever talks to another Lambda directly. Adding a new processing job is just a new queue and Lambda — nothing else changes.

---

#### Which Approach to Use

| Approach | Complexity | Reach For It When |
|---|---|---|
| **Single Lambda** | None | Jobs are fast, one team owns the pipeline, volume is moderate |
| **DB Checklist** | Low — just SQL | You need parallelism, 2–3 job types, small team |
| **Step Functions** | Medium — AWS config | Many job types, conditional branching, pipeline visibility needed |
| **SQS Fan-out** | High — many queues | Many job types, separate teams per Lambda, true deploy independence |

---

### Confirm Endpoint vs Lambda — Two Design Options

**Option A — Separation of concerns (recommended):**

```
POST /confirm  →  HeadObject  →  DB: pending → complete      (synchronous, immediate)
S3 Event       →  Lambda      →  processing → ready + notify  (async, decoupled)
```

`/confirm` owns existence confirmation. Lambda owns processing. The client gets an immediate synchronous answer the moment the upload lands.

**Option B — Lambda handles everything:**

```
POST /confirm  →  removed or made optional
S3 Event       →  Lambda  →  pending → complete → processing → ready + notify
```

One fewer endpoint. But the entire flow is eventual — the client must poll from the very beginning.

Use Option A when your client needs immediate confirmation. Use Option B when eventual consistency throughout is acceptable.

---

## Resumable Uploads — What Happens When the Connection Drops

For files over ~100MB, network interruptions are expected, not exceptional. Two approaches solve this at different levels.

### S3 Multipart Upload — Parallel and Part-Based

S3 natively supports splitting a file into independent parts that can be uploaded in any order, including in parallel.

**Rules:**

- Each part must be ≥ 5MB (except the last)
- Maximum 10,000 parts per upload
- Parts can be uploaded simultaneously across multiple threads or connections
- Nothing is visible as a complete object until you call `CompleteMultipartUpload`

**Full flow:**

```
1. Client asks server to initiate
      → Server: CreateMultipartUpload
      → S3 returns: UploadId = "VXBsb2FkIElEIGZvciA..."

2. Server generates one presigned URL per part
      → Returns all part URLs to client

3. Client uploads parts — in parallel, any order:
      Thread 1: PUT part_1 ──▶ ETag: "etag_1"
      Thread 2: PUT part_2 ──▶ ETag: "etag_2"
      Thread 3: PUT part_3 ──▶ ✗ failed
      Thread 4: PUT part_4 ──▶ ETag: "etag_4"

4. Retry only the failed part:
      Thread 3 (retry): PUT part_3 ──▶ ETag: "etag_3"

5. Client confirms all parts done:
      → Server: CompleteMultipartUpload(UploadId, [etag_1, etag_2, etag_3, etag_4])
      → S3 assembles the final object in part-number order
```

**Resume after a crash:**

```
Client has UploadId stored locally
       │
       ▼
Server calls: ListParts(UploadId)
       │
       ▼
S3 returns: Part 1 ✓, Part 2 ✓, Part 3 ✗, Part 4 ✓
       │
       ▼
Client uploads only Part 3 — nothing else
```

**Cleanup for abandoned multipart uploads** — incomplete uploads accumulate in S3 and cost money. Add a lifecycle rule:

```json
{
  "Rules": [{
    "ID": "AbortIncompleteMultipartUploads",
    "Status": "Enabled",
    "AbortIncompleteMultipartUpload": { "DaysAfterInitiation": 2 }
  }]
}
```

---

### TUS Protocol — Sequential and Byte-Precise

[TUS](https://tus.io/) is an open HTTP-based protocol for resumable uploads. Unlike S3 Multipart, it's not tied to any specific storage backend — it works with S3, GCS, local disk, or anything else you can append bytes to.

The core idea is deliberately simple: every upload gets a server-assigned URL, and the server tracks exactly one number — the byte offset of how much it has received so far. When a connection drops, the client asks "where did you stop?" and resumes from exactly that byte. No parts, no ETags, no UploadId — just an offset.

**The three HTTP methods TUS uses:**

| Method | Path | What It Does |
|---|---|---|
| `POST` | `/files` | Creates a new upload. Server responds with a `Location` header containing the upload URL. Client also sends `Upload-Length` so the server knows the total file size. |
| `HEAD` | `/files/{id}` | Client asks: how many bytes have you received? Server responds with `Upload-Offset: N`. This is how resume works — client reads this number and picks up from there. |
| `PATCH` | `/files/{id}` | Client sends the next chunk of bytes starting at the current offset. Server appends them and updates its internal offset counter. |

**What a resume actually looks like:**

```
Client sends PATCH: bytes 0 → 10MB
Connection drops at byte 8,388,608
       │
       ▼
Client reconnects and sends HEAD to /files/{id}
Server responds: Upload-Offset: 8388608
       │
       ▼
Client sends PATCH starting at byte 8,388,608
Server appends from that exact byte — nothing is re-sent
```

TUS is strictly sequential. The server always expects the next bytes to start exactly where the last successful chunk ended. If the client sends a PATCH with the wrong offset, the server rejects it with `409 Conflict`.

**Why this design exists:** tracking a single offset number means the server needs no AWS-specific state — no `UploadId`, no parts registry, no ETag map. Any backend that can append bytes can implement TUS. That storage-agnosticism is the whole point, and sequential ordering is what makes it possible.

---

### S3 Multipart vs TUS — The Real Difference

| | S3 Multipart | TUS |
|---|---|---|
| **Upload order** | Any order — fully parallel | Strictly sequential |
| **Resume granularity** | Part boundary (≥ 5MB) | Exact byte |
| **Throughput potential** | High — parallel threads | Lower — single stream |
| **Server state needed** | UploadId + parts list in S3 | One offset number in your DB |
| **Storage dependency** | S3 specifically | Any backend |
| **Best for** | AWS, want parallelism | Storage-agnostic, or byte-level precision matters |

On bad mobile connections, TUS can outperform S3 Multipart for resume precision. With 10MB parts, if you're 9.9MB into a part when the connection drops, you lose that entire part and re-upload all 10MB. TUS resumes from exactly byte 9,961,472.

**Rule of thumb:** If you're on AWS and S3 is your blob store, use S3 Multipart — it's native and requires minimal extra infrastructure. Use TUS when you need storage-agnostic resumability or byte-level precision on mobile networks matters more than throughput.

---

## Download Flow & CDN Strategy

How you serve blobs depends entirely on whether the content is public or private.

### Pattern A — Public Content

```
Client ──▶ CDN Edge Node
              │
              ├── Cache HIT  ──▶ Response (fast, no S3 cost)
              │
              └── Cache MISS ──▶ S3 Origin ──▶ CDN caches it ──▶ Response
```

Use content-addressed or version-stamped keys so you can cache forever:

```
# Mutable — hard to cache
/uploads/user-42/avatar.png

# Version-stamped — safe to cache forever
/uploads/user-42/avatar-v3.png

# Content-addressed — deduplicated and immutable
/uploads/abc123def456.png
```

Set `Cache-Control: max-age=31536000, immutable`. Old versions stay cached. New uploads get a new key. No cache invalidation ever needed.

### Pattern B — Private Content

```
Client ──▶ App Server (auth check)
              │
              └── generate CloudFront Signed URL (TTL: 15 min)
              │
Client ◀── signed URL
              │
Client ──▶ CDN (object cached at edge, but URL is time-limited)
```

The object is cached at the CDN edge — performance benefit. Each access requires a fresh signed URL from your server — security benefit. You don't have to choose between the two.

---

## Access Control — Preventing URL Sharing

A presigned URL is a bearer token. Whoever has it can use it. For sensitive content:

| Technique | How It Works | Trade-off |
|---|---|---|
| **Short TTL** | URL expires in minutes | User must re-request frequently |
| **IP binding** | CloudFront locks URL to a specific IP | Breaks for mobile users whose IP changes |
| **Rate-limit URL generation** | Server caps how many download URLs per user per hour | Prevents bulk scraping |
| **Watermarking** | Personalize file content per user — leaked files are traceable | Compute-intensive; only works for documents/images |
| **Logging** | Log every URL generation with user ID, IP, timestamp | Detection after the fact, not prevention |

---

## Storage Architecture Decisions

### Key Naming Strategy

Never use the user-supplied filename as your S3 key directly. Filenames can contain path traversal characters, whitespace, and duplicates.

```
# Bad — collides, guessable, enumerable
uploads/avatar.png

# Good — unique, non-guessable
uploads/{user_id}/{uuid}.{ext}

# Good — date-partitioned, enables time-based lifecycle rules
uploads/{year}/{month}/{uuid}

# Good — content-addressed, enables deduplication
uploads/{sha256_of_content}
```

### Storage Class Lifecycle

Most files are accessed heavily right after upload, then rarely. Use lifecycle rules to transition automatically:

```
Day 0:   Upload → S3 Standard                   (hot, fast)
Day 30:  Auto-transition → S3 Standard-IA       (~45% cheaper)
Day 90:  Auto-transition → S3 Glacier Instant   (~68% cheaper)
Day 365: Auto-transition → S3 Glacier Deep      (~95% cheaper)
```

---

## The Mental Model — One Paragraph

An app server has three jobs in a blob system: **sign**, **verify**, and **delegate**. It signs presigned URLs so clients talk directly to S3. It verifies via `HeadObject` so the DB reflects ground truth. It delegates all data transfer — both upload and download — to S3 and the CDN. The moment your app server starts carrying bytes, you've broken the model. The moment your DB and S3 can drift apart without a reconciliation mechanism, you've introduced a silent bug.

```
┌────────────────────────────────────────────────────┐
│                   LARGE BLOBS                      │
│                                                    │
│  App Server  →  authorization & metadata only      │
│  S3          →  durable storage                    │
│  CDN         →  delivery at the edge               │
│  DB          →  source of truth for state          │
│  Queue       →  buffer for async post-processing   │
│  Lambda      →  thumbnail, virus scan, notify      │
│                                                    │
│  Upload:   Client → App Server → S3 (direct)       │
│  Download: Client → CDN → S3 (on cache miss)       │
│  State:    pending → complete → processing → ready │
└────────────────────────────────────────────────────┘
```

---

## Interview Cheat Sheet

**"How do you handle a failed upload?"**

Track all upload state in DB using `pending/complete/failed`. When a client calls confirm, use `HeadObject` to verify the object exists before marking complete. A reconciliation job runs every few minutes, checking S3 for uploads stuck in `pending` beyond a threshold. S3 Lifecycle rules auto-delete orphaned objects after 1 day without any code.

---

**"How do you keep metadata consistent with blob storage?"**

Three-phase flow: initiate (DB record created, presigned URL issued) → upload (direct to S3, server not involved) → confirm (server calls `HeadObject` to verify, then updates DB). The UPDATE is idempotent by nature — same SET values produce the same result no matter how many times it runs. Add `AND status = 'pending'` to the WHERE clause to prevent late retries from clobbering legitimate downstream state changes. The reconciliation job is the safety net for any gaps.

---

**"How do you support resumable uploads?"**

S3 Multipart for parallel uploads on AWS: split into ≥5MB parts, presign each, upload independently. On resume after crash, call `ListParts` to find completed parts, upload only the missing ones. TUS protocol for byte-level resumability: server tracks one offset number, client asks for current offset and continues from exactly that byte — strictly sequential but byte-precise.

---

**"How do you serve private files at scale?"**

CloudFront Signed URLs: server generates time-limited (15 min) signed URLs after auth check. Client fetches directly from CloudFront — CDN caches the object at the edge. Binary never passes through your app server. Performance of CDN plus security of signed access.

---

**"What if many users upload the same file?"**

Content-addressed deduplication: hash the file client-side (SHA-256) before initiating. Send hash to server. Server checks if a record with that hash exists — if yes, skip upload entirely and return existing file ID. Store a reference counter; only delete from S3 when counter reaches zero.

---

**"How do you prevent malicious uploads?"**

Multiple layers: enforce `Content-Type` and max size in presign conditions — S3 enforces these, not just your server code. Run virus scan post-upload before marking the file usable. Validate magic bytes server-side — don't trust the `Content-Type` header. Serve uploaded content from a separate CDN domain — never your app domain — to prevent cookie hijacking and XSS from user-supplied HTML files.