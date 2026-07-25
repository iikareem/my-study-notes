---
tags:
  - system-design
  - s3
  - cdn
  - uploads
  - storage
---

# Large Blobs — Complete System Design Guide

> **Core Problem:** Images, videos, and large documents cannot flow through application servers without overwhelming bandwidth and compute. This guide walks through the full design space — from first principles to interview-ready depth.

---

## Table of Contents

1. [The Problem, First Principles](#1-the-problem-first-principles)
2. [Core Architecture: Presigned URLs](#2-core-architecture-presigned-urls)
3. [Upload Flow — Step by Step](#3-upload-flow--step-by-step)
4. [Download Flow & CDN Strategy](#4-download-flow--cdn-strategy)
5. [Metadata Consistency](#5-metadata-consistency)
6. [Handling Failed Uploads](#6-handling-failed-uploads)
7. [Resumable Uploads](#7-resumable-uploads)
8. [Access Control](#8-access-control)
9. [Storage Architecture Decisions](#9-storage-architecture-decisions)
10. [Scaling Considerations](#10-scaling-considerations)
11. [Interview Cheat Sheet](#11-interview-cheat-sheet)

---

## 1. The Problem, First Principles

### Why Not Route Blobs Through Your App Server?

Imagine a user uploads a 500MB video. If it goes through your server:

```
Client ──── 500MB ──▶ App Server ──── 500MB ──▶ S3
```

**What breaks:**

- **Bandwidth**: Your server pays for ingress AND egress twice
- **Memory**: Buffering or streaming large bodies exhausts RAM
- **Latency**: Client waits for the full round-trip through your server
- **Concurrency**: One large upload can saturate worker threads, starving other requests
- **Timeouts**: Long uploads break on connection timeouts common in reverse proxies (Nginx, ALBs)

### The Right Shape

```
                 ┌─── presigned URL ───┐
Client ──────▶ App Server             │
Client ◀─────── URL ─────────────────┘
Client ─────────────── 500MB ──────────────▶ S3 (direct)
```

The server is involved only to authorize the operation — not to carry the data.

---

## 2. Core Architecture: Presigned URLs

A **presigned URL** is a time-limited, scoped URL signed with your storage credentials. It delegates a specific operation (PUT or GET) to the bearer for a limited window.

### How S3 Presigning Works (Conceptually)

```
presigned_url = sign({
  bucket:     "my-bucket",
  key:        "uploads/user-42/avatar.png",
  operation:  "PUT",
  expires_in: 300,           // 5 minutes
  conditions: {
    content_type:   "image/png",
    max_size:       5_000_000  // 5MB
  },
  signed_with: AWS_SECRET_KEY
})
```

S3 validates the signature and conditions when the client uses the URL. Your secret key never leaves your server.

### Key Properties

|Property|Detail|
|---|---|
|**Time-scoped**|URL expires (typically 5–15 min for uploads, longer for downloads)|
|**Operation-scoped**|A PUT presigned URL cannot be used to GET|
|**Key-scoped**|Only works for the exact object path|
|**Condition-enforced**|Can enforce content-type, size limits|
|**Credential-free**|Client needs no AWS credentials|

---

## 3. Upload Flow — Step by Step

### Phase 1: Initiation (Client → App Server)

```
POST /api/upload/initiate
{
  "filename": "demo-video.mp4",
  "content_type": "video/mp4",
  "size_bytes": 104857600
}
```

Your server:

1. **Validates** the request (auth, file type whitelist, size limits)
2. **Generates** a unique, unguessable storage key
3. **Creates a pending record** in your DB (status: `pending`)
4. **Calls S3** to generate a presigned PUT URL
5. **Returns** the presigned URL + your internal `upload_id`

```json
{
  "upload_id": "upl_a3f9c2",
  "presigned_url": "https://s3.amazonaws.com/bucket/uploads/...",
  "expires_at": "2025-07-25T14:05:00Z"
}
```

### Phase 2: Direct Upload (Client → S3)

```
PUT {presigned_url}
Content-Type: video/mp4
Content-Length: 104857600

[binary data]
```

S3 validates the signature. Your server is not involved.

### Phase 3: Confirmation (Client → App Server)

After S3 returns `200 OK`:

```
POST /api/upload/confirm
{
  "upload_id": "upl_a3f9c2"
}
```

Your server:

1. **Verifies** the object actually exists in S3 (`HeadObject` call)
2. **Updates** the DB record: `pending → complete`
3. **Triggers** downstream processing (thumbnail generation, virus scan, transcoding)

```
┌──────────┐   1. initiate    ┌─────────────┐   2. presign   ┌─────┐
│  Client  │ ───────────────▶ │  App Server │ ─────────────▶ │ S3  │
│          │ ◀─── URL+id ─── │             │ ◀─── URL ───── │     │
│          │                  └─────────────┘                │     │
│          │ ──── PUT binary ──────────────────────────────▶ │     │
│          │ ◀─── 200 OK ─────────────────────────────────── │     │
│          │   3. confirm     ┌─────────────┐                │     │
│          │ ───────────────▶ │  App Server │──HeadObject──▶ │     │
│          │                  │  (verify +  │ ◀─────────────  │     │
│          │                  │  update DB) │                └─────┘
└──────────┘                  └─────────────┘
```

---

## 4. Download Flow & CDN Strategy

### The Two Download Patterns

#### Pattern A: Public Content (Profile pictures, public assets)

```
Client ──▶ CDN Edge (cache hit) ──▶ response
                │ (cache miss)
                ▼
              S3 Origin ──▶ CDN ──▶ Client
```

CDN caches the object at the edge. No auth required. URL is stable and cacheable.

#### Pattern B: Private Content (User documents, private videos)

```
Client ──▶ App Server (auth check) ──▶ generate signed download URL
Client ◀── signed URL (short TTL)
Client ──▶ CDN/S3 directly with signed URL
```

**CloudFront Signed URLs** allow you to give authenticated users time-limited access to CDN-cached private content — you get both the security of signed access and the performance of CDN edge caching.

### CDN Configuration Decisions

|Decision|Consideration|
|---|---|
|**TTL**|Long TTL for immutable content (hash in key); short for mutable|
|**Cache key**|Exclude auth query params from cache key for shared objects|
|**Invalidation**|Prefer key-based versioning over cache invalidation (invalidation is slow and costs money)|
|**Origin shield**|Enable a single origin-facing CDN layer to protect S3 from cache-miss stampedes|

### Immutable URL Pattern

Instead of:

```
/uploads/user-42/avatar.png          ← mutable, hard to cache
```

Use a content-addressed or version-stamped key:

```
/uploads/user-42/avatar-v3.png       ← version in name
/uploads/abc123def456.png            ← content hash in name
```

This lets you set `Cache-Control: max-age=31536000, immutable` — browsers and CDNs cache forever.

---

## 5. Metadata Consistency

This is the interview-critical section. The blob (in S3) and its metadata (in your DB) must stay in sync. Several failure scenarios can break this.

### The Consistency Problem

```
DB record: { status: "complete", url: "s3://..." }
S3 object: [does not exist — upload failed silently]
```

Or inversely:

```
S3 object: [exists — upload succeeded]
DB record: { status: "pending" }  ← confirmation step crashed
```

### Strategy 1: App Server as Source of Truth (Confirmation Pattern)

The flow from Section 3 — client confirms after upload, server does `HeadObject` to verify — is the standard approach.

**Failure modes handled:**

- Upload fails → pending record stays pending → cleanup job deletes it
- Confirmation crashes → object exists but DB says pending → reconciliation job detects mismatch

### Strategy 2: S3 Event Notifications (Async)

Configure S3 to emit events on `s3:ObjectCreated:*`. A Lambda/worker consumes these and updates your DB.

```
S3 ──▶ SNS/SQS ──▶ Worker ──▶ DB update
```

**Pros:** Decoupled, resilient  
**Cons:** Eventual consistency; your app may serve a "processing" state briefly

### Strategy 3: Idempotent Metadata + Object Versioning

Use S3 Object Versioning to keep history. Your DB stores the S3 version ID. On any write:

```sql
UPDATE files SET s3_version_id = $1, status = 'complete'
WHERE upload_id = $2 AND status = 'pending'
```

This makes updates safe to retry.

### The Reconciliation Job

Run a periodic job (every few minutes) that:

```
1. Query DB for records in `pending` state older than X minutes
2. For each: call S3 HeadObject
   - Object exists → mark complete
   - Object missing → mark failed, notify user or retry
```

This is your safety net for any consistency gap.

---

## 6. Handling Failed Uploads

### Failure Taxonomy

|Failure Point|Symptom|Recovery|
|---|---|---|
|Presign request fails|Client never gets URL|Retry initiate call|
|Upload interrupted mid-stream|S3 returns 4xx/5xx|Resume (see Section 7) or restart|
|Upload succeeds, confirm fails|S3 has object, DB says pending|Reconciliation job|
|S3 eventually rejects (wrong content-type)|S3 returns error after upload|Client handles error, notifies server|
|Server crashes between S3 verify + DB write|Orphaned object in S3|Reconciliation job|

### Cleanup Strategy: Orphaned Objects

Objects in S3 that have no corresponding DB record (or a permanently failed record) are wasted storage. Handle with:

**S3 Lifecycle Rules:**

```json
{
  "Rules": [{
    "ID": "DeleteOrphanedUploads",
    "Filter": { "Prefix": "uploads/tmp/" },
    "Status": "Enabled",
    "Expiration": { "Days": 1 }
  }]
}
```

Upload to `uploads/tmp/{key}`. On confirmation, copy to `uploads/permanent/{key}`. Anything in `tmp/` older than 1 day is auto-deleted by S3.

**Alternatively:** your reconciliation job can call `DeleteObject` directly for records that have been `pending` for too long.

---

## 7. Resumable Uploads

For large files (100MB+), network interruptions during upload are common. Starting over is unacceptable. Two approaches:

### Approach A: S3 Multipart Upload

S3 natively supports splitting a file into parts (minimum 5MB each) and uploading them independently.

```
Initiate Multipart Upload → UploadId
├── Upload Part 1 (5MB) → ETag1
├── Upload Part 2 (5MB) → ETag2
├── Upload Part 3 (remaining) → ETag3
└── Complete Multipart Upload (UploadId + [ETag1, ETag2, ETag3])
```

**Resume flow:**

```
Client crashes after Part 2.
On restart → ListParts(UploadId) → sees Part 1 and 2 done
→ resumes from Part 3 only
```

**Your server's role:** generate a presigned URL per part. The client orchestrates the parts.

```
POST /api/upload/initiate-multipart
→ { upload_id, s3_multipart_id, parts: [{ part_number: 1, presigned_url }, ...] }
```

### Approach B: TUS Protocol

[TUS](https://tus.io/) is an open protocol for resumable uploads over HTTP. Your server implements the TUS endpoints; clients use TUS client libraries.

```
POST   /files        → create upload, get Location URL
HEAD   /files/{id}   → get current offset (for resume)
PATCH  /files/{id}   → upload chunk from offset
```

TUS handles chunk negotiation, checksum verification, and offset tracking. Good when you control the server stack and want a standardized resumable protocol beyond S3 specifics.

### Multipart vs TUS

||S3 Multipart|TUS|
|---|---|---|
|**Best for**|S3-backed storage|Any storage backend|
|**Client libraries**|aws-sdk, custom|tus-js-client, many SDKs|
|**Resume granularity**|Per part (≥5MB)|Per byte|
|**Server complexity**|Low (S3 handles state)|Medium (you track offsets)|
|**Part cleanup**|AbortMultipartUpload + Lifecycle|Handled by TUS server|

---

## 8. Access Control

### Public vs. Private Decision Matrix

```
Is this content user-specific or sensitive?
├── No  → Public CDN URL, no signing needed
│         Cache-Control: max-age=31536000
└── Yes → Signed URL (time-limited access)
          ├── S3 Presigned GET URL (bypass CDN)
          └── CloudFront Signed URL (CDN-cached private content)
```

### Signed Download URL Pattern

```
GET /api/files/{file_id}/download-url

Server:
1. Auth check: does this user own file_id?
2. Generate presigned GET URL (TTL: 15 min)
3. Return URL

Client uses URL directly with S3/CDN — server not in the download path
```

### Token-Binding Patterns (Advanced)

To prevent URL sharing:

- Include the requester's IP in the signed condition (CloudFront supports this)
- Use short TTLs (minutes, not hours)
- Log and rate-limit presigned URL generation
- For highly sensitive files: watermark or personalize the content

---

## 9. Storage Architecture Decisions

### Single Bucket vs. Multi-Bucket

|Strategy|When to Use|
|---|---|
|**Single bucket, prefix-based**|Simple systems; easier IAM; one lifecycle config|
|**Bucket per environment**|Always (dev/staging/prod must be separated)|
|**Bucket per tenant**|Multi-tenant SaaS with data isolation requirements|
|**Bucket per content type**|Different lifecycle/retention policies per type|

### Key Naming Strategy

Bad:

```
uploads/avatar.png          ← collides across users
uploads/user42/avatar.png   ← guessable, enumerable
```

Good:

```
uploads/{user_id}/{uuid}.{ext}
uploads/{date_prefix}/{uuid}         ← for time-based lifecycle
uploads/{content_hash}               ← for deduplication
```

### Storage Classes & Cost Optimization

```
Upload → S3 Standard (hot)
       ↓ 30 days (lifecycle rule)
       S3 Standard-IA (infrequent access, cheaper)
       ↓ 90 days
       S3 Glacier Instant Retrieval (archival, very cheap)
       ↓ 365 days
       S3 Glacier Deep Archive (cold, cheapest)
```

Define this in an S3 Lifecycle Policy. Your application code doesn't change — S3 handles the transitions automatically.

---

## 10. Scaling Considerations

### Bandwidth at Scale

|Approach|Benefit|
|---|---|
|Direct S3 upload|Offloads all upload bandwidth from your servers|
|CDN for downloads|Edge caching reduces S3 egress costs significantly|
|S3 Transfer Acceleration|Routes upload traffic over AWS backbone (faster for distant clients)|
|Regional buckets|Collocate bucket with your compute region|

### Metadata DB at Scale

The `files` table becomes a hotspot. Optimize:

```sql
CREATE INDEX idx_files_user_status ON files(user_id, status);
CREATE INDEX idx_files_pending_created ON files(created_at) 
  WHERE status = 'pending';  -- partial index for reconciliation job
```

For very high volume, consider separating blob metadata from application data into a dedicated service.

### Processing Pipeline

Large blobs usually require post-processing (thumbnails, transcoding, virus scanning). Don't do this synchronously in your confirm endpoint:

```
Upload confirmed
     │
     ▼
Publish to queue (SQS, RabbitMQ, etc.)
     │
     ├──▶ Thumbnail Worker
     ├──▶ Virus Scanner
     └──▶ Transcoding Pipeline (FFmpeg, AWS MediaConvert)
```

Return `status: processing` to the client immediately. Push a websocket/webhook notification when processing completes.

---

## 11. Interview Cheat Sheet

### Questions to Expect

**"How do you handle a failed upload?"**

> Track state in DB with `pending/complete/failed`. Run a reconciliation job that checks `HeadObject` for stale pending records. Use S3 lifecycle rules on a `tmp/` prefix to auto-delete orphans.

**"How do you keep metadata consistent with blob storage?"**

> Three-phase flow: initiate (DB record created, presigned URL issued) → upload (direct to S3) → confirm (server calls `HeadObject` to verify existence, then updates DB). Reconciliation job handles gaps.

**"How do you support resumable uploads?"**

> S3 Multipart Upload: split file into ≥5MB parts, presign each part, client uploads independently. On resume, call `ListParts` to find completed parts, upload only the remainder. Or use TUS protocol for byte-level resumability.

**"How do you serve private files at scale?"**

> CloudFront Signed URLs: generate time-limited signed URLs on your server (after auth check), let the CDN cache and serve the actual content. Client never hits your app server for the binary.

**"What if the same file is uploaded by many users?"**

> Content-addressed storage: hash the file client-side, check if hash already exists in DB before initiating upload. If it does, skip upload and link to existing object. Add a reference counter for safe deletion.

**"How do you prevent malicious uploads?"**

> Enforce content-type and size in presign conditions. Run virus scanning post-upload (ClamAV, cloud-native solutions). Validate magic bytes server-side (not just the extension). Never serve uploaded content from your own domain (use a separate CDN domain to prevent cookie hijacking).

---

### The Mental Model

```
┌─────────────────────────────────────────────────┐
│                  LARGE BLOBS                     │
│                                                  │
│  App Server = Authorization & Metadata only      │
│  S3 = Storage (upload destination)               │
│  CDN = Delivery (download source)                │
│  DB = Source of truth for state                  │
│  Queue = Async post-processing                   │
│                                                  │
│  Upload:  Client → App Server → S3 (direct)      │
│  Download: Client → CDN → S3 (on miss)           │
│  State:   pending → complete → (processing)      │
└─────────────────────────────────────────────────┘
```

> The server's job is to sign, not to carry.

---

## Related

- [[System Design MOC]] — scaling & realtime hub
- [[S3]] — AWS S3 service details (cert study)
- [[Realtime Updates]] · [[Scaling Reads vs Writes]]

← [[System Design MOC]] · [[Concepts MOC]]
