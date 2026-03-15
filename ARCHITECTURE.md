# ARCHITECTURE.md — ArvyaX Journal System Design

> This document addresses the four required architectural questions and provides a complete system overview.

---

## System Overview

```
![alt text](<System_Overview.drawio.png>)

```

### Data Flow for Analyze

```
Client → POST /api/journal/analyze
           │
           ▼
     Hash input text (SHA-256)
           │
           ▼
     Check AnalysisCache (MongoDB)
      /              \
  HIT                MISS
   │                  │
   │            Call Gemini API
   │                  │
   │           Parse + Validate
   │                  │
   │           Store in Cache
   │                  │
   └──────────────────┘
           │
           ▼
   Return { emotion, keywords, summary }
```

---

## 1. How Would You Scale This to 100k Users?

### Backend Scaling

**Horizontal scaling with stateless API servers**
- Deploy multiple Express instances behind a load balancer (NGINX or AWS ALB).
- Since all state lives in MongoDB (no in-process sessions), any instance can serve any request.
- Use PM2 cluster mode locally; Kubernetes (EKS/GKE) for cloud deployments.

**MongoDB Scaling**
- Use **MongoDB Atlas** with auto-scaling clusters.
- **Replica sets** (1 primary + 2 secondaries) for high availability and read scaling.
- Route read-heavy queries (`GET /journal/:userId`, insights) to secondaries with `readPreference: secondaryPreferred`.
- **Sharding** on `userId` once a single replica set becomes a bottleneck — this keeps one user's data co-located on one shard.
- Existing indexes (`userId + createdAt`, `userId + emotion`, `textHash`) are already designed for shard-key efficiency.

**Async LLM Processing**
- At scale, synchronous Gemini calls (1–3 seconds each) will block connections. Replace with an async queue:
  1. Client submits text → API immediately returns `{ jobId, status: "queued" }`.
  2. A worker pool (Bull/BullMQ + Redis) pulls jobs and calls Gemini.
  3. Client polls `GET /api/journal/analyze/status/:jobId` or receives a WebSocket push.
- This decouples user experience from LLM latency entirely.

**Caching Layer (Redis)**
- Add Redis as an in-memory cache in front of MongoDB for:
  - Analysis results (faster than a DB lookup): `SETEX hash 86400 result`
  - Insights aggregations (expensive at scale): cache per `userId` with 5-minute TTL
  - User entry lists: short TTL (30 seconds) to absorb bursts

**CDN + Frontend**
- Serve the React build from CDN (CloudFront, Vercel, Netlify) — zero load on the API for static assets.

### Infrastructure Summary

```
Users → CDN (React)
      → ALB → Auto Scaling Group (Express nodes)
                    → Redis (hot cache)
                    → MongoDB Atlas (replica set → sharded at 100k)
                    → BullMQ Worker Pool → Gemini API
```

---

## 2. How Would You Reduce LLM Cost?

### Strategy 1: Content-Hash Caching (Already Implemented)
Every unique piece of text is hashed (SHA-256) and the Gemini result stored in MongoDB's `AnalysisCache` collection with a 30-day TTL. Identical or repeated text never re-calls the API.

**Expected impact:** Users often write similar entries after recurring sessions (e.g., "felt calm in the forest again"). Cache hit rates of 15–30% are realistic within a month.

### Strategy 2: Batch Processing
Instead of one API call per entry, accumulate 5–10 unanalyzed entries and send them in a single prompt:
```
Analyze these 5 journal entries and return a JSON array of results: [...]
```
Gemini 1.5 Flash charges per token, not per request. Batching reduces per-entry overhead by ~40% by amortizing the system prompt tokens.

### Strategy 3: Use Flash Over Pro
Gemini 1.5 **Flash** is used instead of Pro or Ultra. For structured emotion extraction (a clear, simple task), Flash is as accurate as Pro at ~10x lower cost. Never use a heavier model for a task the lighter model handles well.

### Strategy 4: Prompt Optimization
The current prompt is already minimal and structured. Further optimizations:
- Pre-truncate input to 500 characters (emotion is clear from the first few sentences).
- Remove few-shot examples (Flash doesn't need them for this task).
- Use JSON mode / response schema enforcement when the SDK supports it to cut output tokens.

### Strategy 5: Analyze-on-Demand, Not Automatically
Emotion analysis is triggered by user action ("Analyze" button), not automatically on save. This means entries users never analyze (drafts, short notes) never consume LLM quota. Estimated savings: 30–50% of calls never happen.

### Strategy 6: Semantic Deduplication (Advanced)
At very high scale, use embedding vectors to find *semantically similar* entries (not just identical ones). If a new entry is >90% similar to a cached analysis, reuse the cached result without an API call. This requires a vector database (Pinecone, Weaviate) or MongoDB Atlas Vector Search.

---

## 3. How Would You Cache Repeated Analysis?

### Current Implementation (MongoDB TTL Cache)

```javascript
// AnalysisCache schema
{
  textHash: "sha256(text)",     // unique lookup key
  emotion: "calm",
  keywords: ["rain", "nature"],
  summary: "...",
  expiresAt: Date (30 days)     // MongoDB TTL index auto-deletes
}
```

Flow:
1. Hash incoming text with SHA-256.
2. Query `AnalysisCache.findOne({ textHash })`.
3. If found → return cached result immediately (no Gemini call).
4. If not found → call Gemini → store result → return result.

The `expiresAt` field uses a MongoDB TTL index (`expireAfterSeconds: 0`) which runs a background cleanup task, keeping the cache lean without manual maintenance.

### Production Upgrade: Two-Level Cache

For 100k users, add Redis as an L1 cache in front of MongoDB:

```
Request
   │
   ▼
Redis GET (textHash)    ← ~0.5ms
   │ HIT → return
   │ MISS
   ▼
MongoDB findOne         ← ~5–15ms
   │ HIT → store in Redis → return
   │ MISS
   ▼
Gemini API              ← 1000–3000ms
   │
   ▼
Store in MongoDB + Redis → return
```

This gives:
- L1 (Redis): sub-millisecond, ephemeral, bounded memory
- L2 (MongoDB): persistent, durable, 30-day TTL
- L3 (Gemini): only on true cache miss

### Cache Invalidation Policy
- 30-day TTL is appropriate — emotions from identical text don't change.
- Manual invalidation endpoint `DELETE /api/admin/cache/:hash` for edge cases.
- On model upgrade (switching Gemini versions), flush the entire cache so old results don't pollute responses.

---

## 4. How Would You Protect Sensitive Journal Data?

Journal entries describe users' private thoughts and mental states — this is among the most sensitive data a system can store.

### Data in Transit
- All API traffic served over **HTTPS/TLS 1.3** in production (via reverse proxy or cloud load balancer).
- HTTP → HTTPS redirect enforced at the infrastructure level.
- `Strict-Transport-Security` header set via Helmet.

### Data at Rest — Encryption
- **MongoDB Atlas** offers encryption at rest using AES-256 by default.
- For self-hosted MongoDB, enable **Encrypted Storage Engine** (MongoDB Enterprise) or use filesystem-level encryption (LUKS on Linux).
- Highly sensitive fields (journal `text`) can be **field-level encrypted** using MongoDB's CSFLE (Client-Side Field Level Encryption) so even DB admins cannot read plaintext.

### Application-Level Security
- **No plain userId in URLs for sensitive operations** at scale — use authenticated JWT tokens. The current `userId` param is suitable for an MVP; in production, replace with a user identity extracted from a verified JWT.
- **Helmet.js** sets security headers: `X-Frame-Options`, `Content-Security-Policy`, `X-XSS-Protection`, etc.
- **Rate limiting** prevents brute-force enumeration of user IDs or bulk data harvesting.
- **Input validation** caps text at 5000 characters and validates ambience against an enum — no SQL/NoSQL injection surface.
- Request body size limited to `10kb` to prevent payload attacks.

### Access Control
- In production: every endpoint requires a Bearer JWT. The `userId` in the JWT (issued by Auth0, Cognito, or custom auth) is used server-side — the client never provides their own userId.
- Separate admin roles required to access aggregate analytics or raw exports.
- Principle of least privilege for MongoDB users: the API user can only read/write `journal` and `cache` collections — no drop/admin permissions.

### Logging & Audit
- Never log journal entry text in application logs (currently only `textHash` is logged for cache debugging).
- Structured logs (JSON) shipped to a SIEM (e.g., Datadog, Elastic) for anomaly detection.
- Audit log for exports or admin access to raw data.

### LLM Data Handling
- Journal text is sent to Google Gemini for analysis. In production:
  - Review Google's [Data Use Policy for Gemini API](https://ai.google.dev/gemini-api/terms) — confirm whether prompts are retained for training.
  - Enable **data residency** options if operating in GDPR jurisdictions.
  - Consider running a local/on-premise LLM (Ollama + Llama 3, Mistral) for maximum data sovereignty — the `geminiService.js` is abstracted behind a service interface making this swap straightforward.
  - Add a **user consent prompt** before first analysis: "Your text will be sent to Google Gemini for analysis."

### GDPR / Right to Deletion
- `DELETE /api/journal/user/:userId` endpoint (not in MVP but required at scale) must purge all entries AND cache entries associated with that user's text hashes.
- Data retention policy documented and enforced via MongoDB TTL indexes on inactive accounts.

---

## Database Schema Design

### `JournalEntry`

| Field | Type | Notes |
|-------|------|-------|
| `_id` | ObjectId | Auto-generated |
| `userId` | String | Indexed; in production = hashed/internal ID |
| `ambience` | String (enum) | forest, ocean, mountain, etc. |
| `text` | String | Up to 5000 chars |
| `analysis.emotion` | String | Populated after analyze call |
| `analysis.keywords` | [String] | From LLM |
| `analysis.summary` | String | From LLM |
| `analysis.analyzedAt` | Date | Timestamp of analysis |
| `textHash` | String | SHA-256 of text; used for cache lookup |
| `createdAt` | Date | Auto (timestamps) |
| `updatedAt` | Date | Auto (timestamps) |

**Indexes:** `{userId, createdAt}`, `{userId, analysis.emotion}`, `{userId, ambience}`, `{textHash}`

### `AnalysisCache`

| Field | Type | Notes |
|-------|------|-------|
| `textHash` | String (unique) | Cache key |
| `emotion` | String | |
| `keywords` | [String] | |
| `summary` | String | |
| `expiresAt` | Date | TTL index — auto-deletes after 30 days |

---

## API Route Summary

| Method | Endpoint | Description |
|--------|----------|-------------|
| POST | `/api/journal` | Create journal entry |
| GET | `/api/journal/:userId` | Get entries (paginated) |
| POST | `/api/journal/analyze` | Analyze text with Gemini |
| GET | `/api/journal/insights/:userId` | Aggregated insights |
| GET | `/health` | Service health check |
