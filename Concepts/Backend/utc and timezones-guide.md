# UTC & Timezones — Everything You Need to Know
> A complete reference built from first principles. Written for backend engineers.

---

## 1. The Core Mental Model

> **Time is one universal truth. Timezones are just display preferences.**

There is only one moment happening right now in the world. Different countries just label it differently.

```
When it is 06:00 UTC:

  Riyadh    →  09:00  (UTC+3)
  London    →  06:00  (UTC+0)
  New York  →  01:00  (UTC-5)
  Singapore →  14:00  (UTC+8)
```

Same moment. Different labels.

Your backend deals with the **moment** (UTC). Your frontend deals with the **label** (local time).

---

## 2. What Is UTC?

**UTC (Coordinated Universal Time)** is the world's time standard. It has no offset — it is the reference point that all other timezones are measured against.

- `UTC+3` means 3 hours ahead of UTC → Riyadh
- `UTC-5` means 5 hours behind UTC → New York
- `UTC+0` means no difference → London (in winter)

UTC never changes. It has no Daylight Saving Time. It is always stable. That is exactly why we use it as the storage standard.

---

## 3. Why Store UTC in the Database?

> **Rule: Always store timestamps as UTC in the database. No exceptions.**

If you store local time instead, you get these problems:

- You cannot sort timestamps correctly — `09:00 Riyadh` and `09:00 London` are different moments but look the same
- You cannot do date range filtering correctly
- If a user changes their timezone, all their historical data becomes ambiguous
- Some local times don't even exist or occur twice because of Daylight Saving Time (explained in Section 5)

UTC has none of these problems. It is always unambiguous, always sortable, always correct.

```
BAD  → store "2024-03-31 09:00:00"         (which timezone? unknown)
GOOD → store "2024-03-31T06:00:00.000Z"    (UTC, always clear)
```

**The database is a source of truth, not a display layer. It stores facts, not presentations.**

---

## 4. The Format: ISO 8601

Always use **ISO 8601** for datetime strings in your API. Never use custom or human-readable formats.

```
2024-03-31T06:00:00Z             → UTC (Z = Zulu = UTC+0)
2024-03-31T09:00:00+03:00        → UTC+3 (Riyadh), same moment as above
2024-03-31T06:00:00.000Z         → UTC with milliseconds
```

These are the only formats you should accept or return in an API.

---

## 5. What Is Daylight Saving Time (DST)?

Some countries **shift their clocks forward by 1 hour in summer**, then back in winter. This is called Daylight Saving Time.

**Example — London:**
- Winter → UTC+0
- Summer → UTC+1 (clocks moved forward 1 hour)

**Saudi Arabia has no DST.** Riyadh is always UTC+3, all year.

This matters because if you store a fixed offset like `+01:00` for a London user, you will be wrong for half the year. That is why we use **IANA timezone names** instead of fixed offsets for anything involving date ranges or recurring logic:

```
"Europe/London"   ✅  the library knows when DST applies
"+01:00"          ⚠️  only correct for one fixed moment, not ranges
```

For Yaqoot's users (Saudi Arabia), DST is not your immediate concern since Riyadh never changes. But knowing the concept is important for when you work with international users.

---

## 6. The Three Types of Datetime Strings (Important Names)

Knowing these names lets you speak precisely with other engineers and in interviews.

### UTC Timestamp
```
"2024-03-31T06:00:00Z"
```
The `Z` stands for **Zulu** (NATO alphabet for zero), meaning UTC+0. This is already normalized to UTC. Also called "Zulu time."

### Offset Datetime (Timezone-Aware Timestamp)
```
"2024-03-31T09:00:00+03:00"
```
Contains the local time **plus** the UTC offset. The backend can convert it to UTC trivially. Also called a **datetime with UTC offset**.

### Naive Datetime
```
"2024-03-31T09:00:00"
```
No `Z`, no offset. The backend has **no idea** what timezone this belongs to. This is the dangerous one.

"Naive" is the actual industry term — you will hear it in Python docs, database documentation, and engineering discussions. It means the datetime is "naive" about where it belongs in the world.

> **In an interview or team discussion you can say:**
> *"We should never accept naive datetimes on the API — everything must be either a UTC timestamp or an offset datetime."*

---

## 7. UTC Timestamp vs Offset Datetime — Which Is Better?

**Both are valid. Both are common. The backend result is identical.**

```
"2024-03-31T09:00:00+03:00"  ──┐
                                ├──► new Date() ──► "2024-03-31T06:00:00.000Z"
"2024-03-31T06:00:00Z"       ──┘
```

JavaScript's `new Date()` normalizes both to UTC automatically. Same value, same storage.

**Why do clients often send offset datetimes?**

Because it requires less work on the frontend. The date picker gives them `09:00`, they append their browser's offset (`+03:00`), and send it. No conversion needed.

**Why do some clients send UTC directly?**

Because it is slightly cleaner — the server does zero work, and every request looks the same regardless of the user's timezone.

Neither approach is wrong. It is a frontend team convention. What matters is that **naive datetimes are never accepted.**

---

## 8. The Full Flow — End to End

```
USER
  picks "March 31 at 9:00 AM" in the UI
        ↓
FRONTEND
  sends either:
    "2024-03-31T09:00:00+03:00"   (offset datetime)
  or:
    "2024-03-31T06:00:00Z"        (UTC timestamp, converted by frontend)
        ↓
BFF / BACKEND
  normalizes to UTC:
    new Date("2024-03-31T09:00:00+03:00") → 2024-03-31T06:00:00.000Z
  validates, processes
        ↓
DATABASE
  stores: 2024-03-31T06:00:00.000Z  (UTC)
        ↓
API RESPONSE
  returns: "2024-03-31T06:00:00.000Z"  (always UTC)
        ↓
FRONTEND
  converts UTC to user's local time for display:
  "March 31, 2024 at 9:00 AM"
```

**The backend only speaks UTC. The frontend handles local display.**

---

## 9. Storing the User's Timezone

Every app with users across different timezones should store the timezone on the user record.

```sql
ALTER TABLE users ADD COLUMN timezone VARCHAR(64) NOT NULL DEFAULT 'UTC';
-- example value: "Asia/Riyadh"
```

**Why?**

When a user requests "today's orders" or "this week's activity", your backend needs to know what "today" means *for them*. Without their stored timezone, you would have to:

- Ask them to send their timezone in every single request (annoying, easy to forget)
- Assume UTC for everyone (wrong for most users)
- Guess from IP address (inaccurate)

Storing it once on the user record means your BFF always has it via `req.user.timezone` — no extra work per request.

**This is the key use case:** you take the user's local date input, look up their timezone from their profile, convert to UTC, and query the database correctly.

```js
// User says "show me March 31 orders"
// BFF reads: req.user.timezone → "Asia/Riyadh"
// Converts:  March 31 in Riyadh → UTC range
//   start: 2024-03-30T21:00:00Z
//   end:   2024-04-01T20:59:59Z
// Queries DB with that UTC range ✅
```

Always store **IANA timezone names**, not fixed offsets:

```
"Asia/Riyadh"    ✅  correct, handles DST automatically
"+03:00"         ⚠️  works for Saudi Arabia (no DST), but not reliable globally
"AST"            ❌  ambiguous (Arabia Standard Time? Atlantic Standard Time?)
```

---

## 10. Date Range Filtering — The Tricky Part

This is where most timezone bugs happen.

**Scenario:** User wants orders placed "on March 31" — they are in Riyadh (UTC+3).

"March 31 in Riyadh" in UTC is:
```
Start: 2024-03-31T00:00:00+03:00  =  2024-03-30T21:00:00Z
End:   2024-03-31T23:59:59+03:00  =  2024-04-01T20:59:59Z
```

One local day spans **two UTC days**. If you filter by UTC midnight to UTC midnight, you get the wrong data.

```js
// BAD — filtering March 31 in UTC, not in Riyadh time
WHERE created_at >= '2024-03-31T00:00:00Z'
  AND created_at <  '2024-04-01T00:00:00Z'
// You're missing 3 hours at the start and adding 3 wrong hours at the end

// GOOD — convert the user's local date to UTC first
import { zonedTimeToUtc } from 'date-fns-tz';

const userTz = req.user.timezone; // "Asia/Riyadh"

const startUTC = zonedTimeToUtc('2024-03-31T00:00:00', userTz);
const endUTC   = zonedTimeToUtc('2024-03-31T23:59:59', userTz);

// startUTC → 2024-03-30T21:00:00.000Z
// endUTC   → 2024-04-01T20:59:59.000Z

WHERE created_at >= startUTC AND created_at <= endUTC  ✅
```

---

## 11. API Design Rules

### Responses — always return UTC ISO 8601

```json
{
  "id": "order_123",
  "created_at": "2024-03-31T06:00:00.000Z",
  "updated_at": "2024-03-31T06:05:12.345Z",
  "scheduled_at": "2024-04-02T09:00:00.000Z"
}
```

Always with `Z`. Never a bare datetime. Never local time.

### Requests — accept UTC or offset datetime, reject naive

```
✅  "2024-03-31T06:00:00Z"          UTC timestamp
✅  "2024-03-31T09:00:00+03:00"     Offset datetime
❌  "2024-03-31T09:00:00"           Naive datetime — reject this
❌  "2024-03-31 09:00:00"           Not ISO 8601 — reject this
```

### Validation in Node.js

```js
function validateTimestamp(value, fieldName) {
  if (!value) throw new Error(`${fieldName} is required`);

  const parsed = new Date(value);
  if (isNaN(parsed.getTime())) {
    throw new Error(`${fieldName} must be a valid ISO 8601 datetime with timezone`);
  }

  return parsed; // JavaScript Date is UTC internally
}
```

---

## 12. Database Column Types

### PostgreSQL

```sql
-- Use TIMESTAMPTZ for any "when did this happen" column
-- Stores UTC, always timezone-aware
created_at   TIMESTAMPTZ NOT NULL DEFAULT NOW(),
updated_at   TIMESTAMPTZ NOT NULL DEFAULT NOW(),
scheduled_at TIMESTAMPTZ,

-- Use DATE only when the calendar date itself is the data
-- e.g. a user's birthday, a public holiday
birth_date   DATE,

-- Never use TIMESTAMP (no timezone) for event timestamps
-- It stores local time with no offset info — a footgun
```

### MySQL

```sql
-- Use DATETIME(3) and always insert UTC explicitly
created_at DATETIME(3) NOT NULL DEFAULT UTC_TIMESTAMP(3),

-- Set timezone on DB connection to be safe
SET time_zone = '+00:00';
```

---

## 13. Common Mistakes

### Storing local time in the DB
```js
// BAD
const now = new Date().toLocaleString(); // server's local time, ambiguous
await db.query('INSERT INTO orders (created_at) VALUES (?)', [now]);

// GOOD
const now = new Date().toISOString(); // "2024-03-31T06:00:00.000Z"
await db.query('INSERT INTO orders (created_at) VALUES (?)', [now]);
// Or just use DEFAULT NOW() / DEFAULT CURRENT_TIMESTAMP in the DB
```

### Filtering "today" in UTC instead of the user's timezone
```js
// BAD — "today" in UTC, not in the user's local today
const start = new Date();
start.setUTCHours(0, 0, 0, 0);

// GOOD — "today" in the user's timezone
import { utcToZonedTime, zonedTimeToUtc } from 'date-fns-tz';
import { startOfDay, endOfDay } from 'date-fns';

const userTz = req.user.timezone;
const nowInUserTz = utcToZonedTime(new Date(), userTz);
const startUTC = zonedTimeToUtc(startOfDay(nowInUserTz), userTz);
const endUTC   = zonedTimeToUtc(endOfDay(nowInUserTz), userTz);
```

### Returning local time or ambiguous format in the API
```json
// BAD
{ "created_at": "2024-03-31 09:00:00" }        ← naive, ambiguous
{ "created_at": "Sun Mar 31 2024 09:00:00 GMT+0300" }  ← not ISO 8601

// GOOD
{ "created_at": "2024-03-31T06:00:00.000Z" }   ← always UTC ISO 8601
```

### Accepting naive datetimes silently
```js
// BAD — silently storing an ambiguous value
const dt = req.body.scheduled_at; // "2024-03-31T09:00:00" (naive)
await db.insert({ scheduled_at: dt }); // what timezone is this? unknown

// GOOD — reject it
if (!dt.endsWith('Z') && !dt.match(/[+-]\d{2}:\d{2}$/)) {
  throw new Error('scheduled_at must include a timezone offset or Z');
}
```

---

## 14. Quick Reference

| Question | Answer |
|---|---|
| What to store in DB? | UTC always |
| What column type in Postgres? | `TIMESTAMPTZ` |
| What to return in API? | UTC ISO 8601 with `Z` suffix |
| What to accept in API? | UTC timestamp or offset datetime (not naive) |
| What is a naive datetime? | A datetime with no `Z` and no offset — dangerous |
| What is Zulu time? | UTC+0, same as `Z` in ISO 8601 |
| Where to store user's timezone? | On the user record as an IANA name |
| Why store IANA name not offset? | IANA handles DST automatically |
| Who converts UTC to local for display? | The frontend, not the backend |
| Is `+03:00` worse than `Z` in a request? | No — both are equally valid |

---

## 15. The Names — For Interviews and Team Discussions

| Situation | Term to use |
|---|---|
| `"2024-03-31T06:00:00Z"` | **UTC timestamp** or **Zulu time** |
| `"2024-03-31T09:00:00+03:00"` | **Offset datetime** or **timezone-aware timestamp** |
| `"2024-03-31T09:00:00"` | **Naive datetime** — always flag this |
| Clock shift in summer | **Daylight Saving Time (DST)** |
| `"Asia/Riyadh"` format | **IANA timezone name** |

---

## 16. Indexing Timestamps — Tradeoffs You Must Know

This is one of the most important practical decisions when designing tables that involve filtering by time.

### What is a B-Tree Index?

The default index type in PostgreSQL (and most databases) is a **B-Tree index**. It organizes values in a sorted tree structure, which makes range queries (`>=`, `<=`, `BETWEEN`) very fast.

When you index a `TIMESTAMPTZ` column, the B-Tree stores the full UTC value — year, month, day, hour, minute, second, millisecond. This makes it extremely precise and powerful for filtering.

```sql
-- This index stores the full UTC timestamp
CREATE INDEX idx_orders_created_at ON orders (created_at);
```

---

### TIMESTAMPTZ Index — Full Precision

**What it stores:** The exact UTC moment — `2024-03-30T21:00:00.000Z`

**What it is good at:**

```sql
-- "All orders in the last 24 hours" → extremely fast
WHERE created_at >= NOW() - INTERVAL '24 hours'

-- "All orders between two exact UTC moments" → extremely fast
WHERE created_at >= '2024-03-30T21:00:00Z'
  AND created_at <  '2024-04-01T20:59:59Z'

-- Sorting by time → extremely fast
ORDER BY created_at DESC
```

**The tradeoff — it knows nothing about local calendar dates:**

```sql
-- "All orders on March 31 in Riyadh" — the index can't answer this directly
-- Because "March 31 Riyadh" = "March 30 21:00 UTC" to "April 1 20:59 UTC"
-- The DB sees a range spanning two UTC calendar days
-- You MUST convert to UTC first (as covered in Section 10), then the index works perfectly
WHERE created_at >= '2024-03-30T21:00:00Z'
  AND created_at <  '2024-04-01T20:59:59Z'
-- ✅ Index is used efficiently here
```

The index itself is fine — the burden is on **you** to give it the right UTC range. If you give it the correct UTC bounds, it will perform perfectly.

---

### The Problem: Filtering on a Computed Expression

This is where developers make a critical mistake. They try to filter by extracting the date from the timestamp directly in the query:

```sql
-- BAD — this breaks the index
WHERE DATE(created_at) = '2024-03-31'

-- Also BAD — same problem
WHERE created_at::DATE = '2024-03-31'

-- Also BAD
WHERE EXTRACT(DAY FROM created_at) = 31
```

**Why is this bad?**

When you wrap a column inside a function (`DATE()`, `::DATE`, `EXTRACT()`), the database **cannot use the B-Tree index** on that column. It has to scan every single row, apply the function, and then check the result. This is called a **sequential scan** and it is very slow on large tables.

```
Table with 10 million orders:

Using index on created_at with UTC range  →  reads ~hundreds of rows  ✅ fast
Using DATE(created_at) = '2024-03-31'    →  reads all 10 million rows ❌ slow
```

**The fix is always the same:** convert your filter to a UTC range in your application code, then query with `>=` and `<`. The index is used and performance is excellent.

---

### When to Use a DATE Column Instead

Sometimes the calendar date itself *is* the data — not a precise moment. In those cases, a `DATE` column is the right choice.

```sql
-- Use DATE when the date itself is the business fact
birth_date      DATE,   -- a user's birthday
holiday_date    DATE,   -- a public holiday
subscription_expiry DATE, -- expires "on" a date, not at a specific moment
report_date     DATE    -- this report is "for" March 31, regardless of time
```

**Benefits of a DATE column:**

- Simpler to filter — `WHERE birth_date = '2024-03-31'` uses the index directly, no conversion needed
- Smaller storage — 4 bytes vs 8 bytes for TIMESTAMPTZ
- No timezone ambiguity — a birthday is March 31 everywhere, it has no "time"

**When NOT to use DATE:**

- Any "when did this event happen" column → use TIMESTAMPTZ
- Any column you will sort chronologically → use TIMESTAMPTZ
- Any column where hour/minute matters → use TIMESTAMPTZ

---

### The Full Comparison

| | `TIMESTAMPTZ` | `DATE` |
|---|---|---|
| Stores | Exact UTC moment | Calendar date only |
| Precision | Year, month, day, hour, minute, second, ms | Year, month, day |
| Storage | 8 bytes | 4 bytes |
| Index filter | Must use UTC range (`>=`, `<`) | Direct equality or range on date |
| Sorting | Perfect chronological sort | Sorts by date only (no time) |
| Timezone filtering | Requires UTC conversion in app code | No conversion needed (no time component) |
| Best for | Events, logs, orders, scheduled tasks | Birthdays, expiry dates, report periods |
| Biggest mistake | Wrapping in `DATE()` function — kills index | Storing events that have a time component |

---

### The Decision Rule

Ask yourself: **"Is the time of day part of what I'm recording?"**

```
"When was this order created?"       → TIMESTAMPTZ  (time matters)
"When does this subscription end?"   → DATE          (just the date matters)
"When was this user born?"           → DATE          (just the date matters)
"When did this payment happen?"      → TIMESTAMPTZ  (time matters)
"What date is this report for?"      → DATE          (just the date matters)
"When was this token issued?"        → TIMESTAMPTZ  (time matters)
```

---

### Key Takeaway for Your BFF

Your BFF layer is responsible for giving the database the correct UTC range — not the database. The database's index just needs a proper `>=` and `<` on a `TIMESTAMPTZ` column and it will be fast. The work of converting "March 31 in Riyadh" → UTC range lives in your Node.js code, not in the SQL query.

```js
// BFF does the conversion
const startUTC = zonedTimeToUtc('2024-03-31T00:00:00', 'Asia/Riyadh');
const endUTC   = zonedTimeToUtc('2024-03-31T23:59:59', 'Asia/Riyadh');

// DB query uses index efficiently
WHERE created_at >= $1 AND created_at < $2
// → index scan, fast, correct ✅
```

Never push the timezone logic into the SQL. Do it in your BFF, pass UTC bounds to the query, let the index do its job.

---

## Summary — The Rules in One Place

1. **Database stores UTC. Always. No exceptions.**
2. **API always returns UTC ISO 8601 with `Z` suffix.**
3. **Never accept naive datetimes. Require `Z` or an offset.**
4. **Both UTC timestamps and offset datetimes from the client are fine — normalize on arrival.**
5. **Date range filters must convert the user's local dates to UTC using their timezone.**
6. **Store the user's timezone as an IANA name on their profile.**
7. **The frontend is responsible for displaying times in local format.**
8. **The backend is a timezone-free zone — UTC in, UTC out.**
9. **Index `TIMESTAMPTZ` columns and always query with UTC ranges (`>=`, `<`) — never wrap the column in `DATE()` or `EXTRACT()` or you kill the index.**
10. **Use `DATE` columns only when the calendar date itself is the data (birthdays, expiry dates) — not for events.**
