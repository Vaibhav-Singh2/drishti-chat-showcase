# Challenges & Solutions

> **Disclaimer**: This repository is a technical case study. The original implementation is proprietary and owned by the employer. No confidential source code, credentials, or sensitive business information is included.

---

This document details the real engineering challenges encountered during the production lifecycle of Drishti Marketing OS, and the architectural solutions implemented to resolve them.

---

## Challenge 1: Meeting Meta's 2-Second Webhook Acknowledgement Constraint

### The Problem
Meta's WhatsApp Cloud API and Instagram Graph API require webhook endpoints to return HTTP 200 within **2 seconds**. If the server takes longer, Meta treats the delivery as failed and retries, triggering duplicate processing. The full message pipeline — HMAC verification, RAG vector embeddings, LLM reasoning, CRM lookups, and DB writes — takes 3–8 seconds.

### The Solution
**Immediate Enqueueing via BullMQ:**
1. Webhook controller verifies HMAC signature using constant-time `crypto.timingSafeEqual` (< 5ms).
2. Enqueues the raw payload to Redis `inbound-msg-queue` (< 10ms).
3. Immediately returns `HTTP 200 EVENT_RECEIVED` (< 20ms total).
4. Decoupled `InboundWorker` processes the job asynchronously without time boundaries.

**Result**: Zero webhook retry timeouts caused by application latency.

---

## Challenge 2: Race Conditions from Rapid Consecutive Inbound Messages

### The Problem
Customers frequently send rapid sequences of messages within seconds (e.g. "Hi", "I need help", "Here is my order"). In a multi-worker BullMQ cluster, adjacent messages for the same conversation were picked up concurrently by different workers. Both workers fetched the same conversation state, invoked the LLM in parallel, and dispatched duplicate or conflicting replies.

### The Solution
**Atomic Redis Distributed Locking (`DistributedLock`):**
1. Before processing, the worker attempts an atomic lock:
   ```typescript
   const lock = await DistributedLock.acquire(`conv:${conversationId}`, 30000);
   if (!lock) return; // Drop or defer redundant concurrent execution
   ```
2. The lock is held throughout RAG retrieval, LLM tool execution, and outbound enqueueing.
3. Upon completion, an atomic Lua script verifies the worker's unique token before deleting the lock:
   ```lua
   if redis.call("get", KEYS[1]) == ARGV[1] then
     return redis.call("del", KEYS[1])
   else
     return 0
   end
   ```

**Result**: Guaranteed sequential processing per conversation across distributed containers, eliminating duplicate AI responses.

---

## Challenge 3: Silent Meta & Instagram Token Expiry

### The Problem
Instagram-login access tokens (`IGAA...`) expire after 60 days. In early production, an Instagram token expired unnoticed. Because the frontend showed the channel as "Active", incoming Instagram DMs went unanswered for days before operators realized replies were failing.

### The Solution
**Proactive Health Monitor Watchdog (`healthMonitor.ts`):**
1. **Automated Graph API Audit**: Every 30 minutes, a watchdog probes `graph.facebook.com/debug_token` and `graph.instagram.com/me`.
2. **Auto-Refresh Engine**: Instagram tokens older than 20 days are automatically renewed via Meta Graph API token refresh endpoints (`tokenRefreshedAt`).
3. **7-Day Expiry Warnings**: If any token has < 7 days before expiration, an automated warning is triggered.
4. **WhatsApp Outage Alerts**: Invalid tokens (Code 190) trigger an immediate high-priority WhatsApp alert to engineering leads using the Meta-approved `web_error_alert` template.

**Result**: Zero silent token outages since deployment; 100% automated renewal of Instagram credentials.

---

## Challenge 4: AI Language Drift to Hinglish or Devanagari on English Messages

### The Problem
The system prompt instructed the model to "mirror the customer's language". However, when English-speaking customers with Indian names opened with "Hi" or "When will my report arrive?", LLMs frequently replied in Roman Hinglish or Devanagari Hindi. The model made incorrect inferences based on cultural preconceptions.

### The Solution
**Deterministic Code-Level Script & Marker Engine (`replyLanguage.ts`):**
1. **Script Range Matching**: Analyzes raw Unicode characters. Devanagari (`/[ऀ-ॿ]/`), Bengali, Tamil, etc., are identified decisively.
2. **Hinglish Marker Dictionary**: Evaluates Roman-script function words (`aap`, `kaise`, `kya`, `nahi`, `chahiye`) using word boundaries (`\b`).
3. **English Function Markers**: Evaluates English syntax (`the`, `is`, `when`, `report`, `order`).
4. **Neutral Token Isolation**: Greetings ("Hi", "Hello", "OK") are treated as neutral, preserving the thread's previously established language.
5. Injects an explicit, non-negotiable instruction: `"Customer language: English. You MUST reply in English."`

**Result**: Eliminated language drift completely across all conversation threads.

---

## Challenge 5: Stale CRM Payment Lag Causing AI to Deny Valid Payments

### The Problem
Zoho CRM payment status updates lag behind actual Razorpay transactions by 1–3 minutes. Customers who completed payment and texted "I already paid" were met with an AI reply stating "Your payment is still pending, please pay here" because the CRM deal still read "Proposal/Link Shared". Customers became rightfully frustrated by the contradiction.

### The Solution
**Stale Payment Claim Conflict Resolution Heuristic:**
1. **Payment Claim Detection**: Scans the customer's message for payment assertions ("paid", "payment ho gaya", "debited", "UTR", "transaction id").
2. **Assertion Override**: When a recent payment claim is detected within a 12-hour window, the customer's claim overrides the pending CRM status.
3. **Link Withholding**: The AI is strictly instructed to withhold payment links, reassure the customer that their payment is being confirmed, and notify them that report preparation begins shortly.

**Result**: Eliminated false payment-pending accusations and protected customer trust.

---

## Challenge 6: Marketing Delivery Failures & Frequency Cap Penalties

### The Problem
Automated marketing campaigns (e.g. abandoned-cart recovery) risked spamming numbers repeatedly if customers failed to convert. Additionally, blasting numbers that had opted out at the Meta OS level caused high delivery failure rates, jeopardizing the WhatsApp Business Account quality rating.

### The Solution
**Centralized Marketing Guardrails (`marketingGuard.ts`):**
1. **Strict Frequency Caps**: Gated at maximum 1 of the same template per 24 hours, and maximum 3 marketing templates per 7 days per contact.
2. **STOP Opt-Outs**: Incoming messages containing "STOP" immediately set `marketingOptOut = true`.
3. **Tiered Delivery-Failure Suppression**:
   - Meta Error 131050 / 131026: Permanent suppression (`marketingHardSuppressed = true`).
   - Meta Error 131049 (Meta ecosystem healthy engagement cap): 24-hour temporary hold (`marketingSuppressExpiresAt = now + 24h`).
4. **Transactional Protection**: UTILITY and AUTHENTICATION templates (order status, payment confirmation, report links) are never blocked.

**Result**: Preserved WhatsApp "High" tier quality rating with zero spam penalties.

---

## Challenge 7: WhatsApp "Typing..." Indicator Drop-Off During Multi-Step Reasoning

### The Problem
Meta WhatsApp allows sending a `"typing..."` chat status indicator. However, Meta automatically expires the typing indicator after 5–10 seconds. When the AI agent performed multi-step tool calling (e.g. searching CRM → verifying payment → querying RAG), the typing indicator would expire before the reply was sent, leading users to believe the bot had stalled.

### The Solution
**Keep-Alive Typing Indicator Heartbeat:**
1. Inbound worker initiates a typing status ping immediately upon acquiring the lock.
2. An interval timer sends heartbeat typing indicators to Meta every 4 seconds throughout LLM reasoning and tool execution.
3. The heartbeat is terminated only upon final outbound message dispatch.

**Result**: Smooth, continuous typing indicators for the customer throughout complex tool interactions.

---

## Challenge 8: Expired 24-Hour Meta Service Windows Bloating Shared Inbox

### The Problem
Meta prohibits free-form business messages past 24 hours from the customer's last message. In an active business, thousands of older conversations remained marked as "Open" in MongoDB, slowing down thread queries and confusing support staff.

### The Solution
**Batch Auto-Close Engine (`conversationWindow.ts`):**
1. Stores `customerReplyWindowExpiresAt` on every inbound message.
2. A background cron runs every 60 seconds, finding conversations where `customerReplyWindowExpiresAt <= now`.
3. Closes conversations in batches of 500 while preserving active human escalations (`isAiActive === false && needsHumanAttention === true`).
4. Reopening a closed chat upon a new customer reply seamlessly restores the previous AI configuration.

**Result**: Clean operator inbox with sub-100ms thread listing queries and zero stale open sessions.

---

## Challenge 9: Preventing Sensitive Link Exposure via AI Hallucination

### The Problem
Astrology reports contain private customer birth charts. If the AI injected private report download links based on simple phone matching, customers using shared family numbers or spoofed identities could access unauthorized reports.

### The Solution
**Ownership Guardrails in Native Tools (`agentTools.ts`):**
1. `shareReportLink` and `shareCorrectionLink` enforce server-side ownership verification before returning any URL to the model.
2. For primary phone numbers, verified deals match automatically.
3. For secondary accounts or differing numbers, the caller must supply both their registered email and matching Zoho Deal ID.
4. Includes tolerant first-name (`nameLooseMatch`) and date-of-birth (`dobLooseMatch`) verification tolerant of international date formats.

**Result**: 100% server-side enforcement; private URLs can never be hallucinated or leaked by the LLM.

---

## Challenge 10: Emoji Surrogate Characters Breaking Qdrant Vector Upserts

### The Problem
Customer support FAQs and astrology documents frequently contained emojis. Text extraction from PDFs produced split UTF-16 surrogate pairs, which caused Qdrant's REST API upsert endpoints to reject payloads with JSON parsing errors.

### The Solution
**Pre-Embedding Text Sanitization in `VectorWorker`:**
1. Regex sanitization strips broken surrogate pairs before embedding generation and hashing.
2. The clean text is hashed via MD5 to generate a deterministic Qdrant point UUID.
3. Chunk deduplication ensures identical document re-uploads do not create duplicate vector points.

**Result**: Zero vector ingestion failures from Unicode or emoji formatting errors.

---

## Challenge 11: Docker Hub Rate Limiting in CI/CD

### The Problem
The EC2 production server ran `docker compose pull`, which attempted to pull all images including third-party base images (`qdrant/qdrant`, `redis:7-alpine`). Docker Hub's 100 pulls / 6 hours rate limit on shared EC2 IPs frequently caused deployment pipelines to crash.

### The Solution
Targeted ECR image pulling in GitHub Actions:
```bash
docker compose pull api web
docker compose up -d
```
Base infrastructure images (Qdrant and Redis) are pinned to specific versions, pulled once during initial host provisioning, and never re-pulled during application code releases.

**Result**: 100% reliable CI/CD deployment runs with zero Docker Hub rate-limiting interruptions.
