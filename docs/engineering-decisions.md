# Engineering Decisions

> **Disclaimer**: This repository is a technical case study. The original implementation is proprietary and owned by the employer. No confidential source code, credentials, or sensitive business information is included.

---

This document captures the key architectural decisions made during the evolution of Drishti Marketing OS into an enterprise omnichannel platform, detailing the technical context, evaluated alternatives, and engineering trade-offs.

---

## Decision 1: Bun Runtime for API Backend

**Decision**: Run the Express API on the **Bun runtime** (`oven/bun:1.1-alpine`) instead of standard Node.js.

**Context**: Fast container cold-starts are vital during auto-scaling and zero-downtime rolling redeployments.

**Rationale**:
- Bun delivers sub-second cold starts and reduced memory footprint compared to vanilla Node.js.
- Native TypeScript execution during local development and script utilities.
- Node.js API compatibility for standard libraries (`bcryptjs`, `otplib`, `ioredis`).

**Tradeoff & Mitigation**: Compatibility with native C++ npm bindings was verified. Production Docker images use a multi-stage build where TypeScript is compiled to JS via Node, then executed on Bun's runtime for maximum reliability.

---

## Decision 2: Redis-Backed BullMQ for Asynchronous Ingestion

**Decision**: Place a BullMQ queue layer between webhook ingestion and message processing across 4 dedicated queues (`inbound-msg-queue`, `outbound-msg-queue`, `vector-ingest-queue`, `status-msg-queue`).

**Context**: Meta's WhatsApp and Instagram APIs enforce a strict **2-second webhook acknowledgement window**. Failure to reply within 2s triggers Meta retry storms.

**Rationale**:
- Separates sub-15ms ingestion from multi-second LLM reasoning, RAG vector searches, and CRM synchronization.
- Durable persistence (Redis AOF/RDB) prevents message loss across application restarts.
- Concurrency and rate-limiting can be tuned per queue.

**Memory Optimization**: Configured all queues with `removeOnFail: { age: 86400, count: 1000 }` to prevent failed jobs from exhausting Redis memory over time.

---

## Decision 3: Atomic Redis Distributed Locking (`DistributedLock`)

**Decision**: Implement a distributed locking mechanism using atomic Redis commands (`SET PX NX`) with tokenized Lua script release around inbound conversation processing.

**Context**: When users send rapid bursts of messages ("Hi", "Where is my report?", "Order #12345"), concurrent BullMQ workers picked up adjacent jobs for the same conversation simultaneously. This caused duplicate AI completions, conflicting draft suggestions, and out-of-order responses.

**Alternatives Considered**:
- *In-memory mutex*: Fails across multiple worker processes or containers.
- *Database row locking (MongoDB transaction)*: High latency overhead and potential write-conflict aborts.
- *Single worker concurrency*: Destroys system throughput for all other conversations.

**Implementation**:
- Acquire: `redis.set("lock:conv:" + id, token, "PX", 30000, "NX")`
- Release: Atomic Lua script checking that the lock token matches before deletion.
- Result: Perfectly serial processing per conversation with zero cross-conversation blocking.

---

## Decision 4: Self-Hosted Qdrant Over Cloud-Managed Pinecone

**Decision**: Migrate the vector store from Pinecone to self-hosted **Qdrant** running as a Docker service on the same host.

**Context**: The initial prototype used Pinecone, which incurred recurring vector storage costs, query billing, and round-trip cloud network latency.

**Rationale**:
- Self-hosted Qdrant runs on the local Docker network (`drishti-network`), reducing vector search latency to < 10ms.
- Zero ongoing vector storage subscription fees.
- Point deduplication using MD5 hash UUIDs prevents duplicate vector creation on document re-upload.

---

## Decision 5: Native Function-Calling Agent Loop Over Static Prompt-Stuffing

**Decision**: Replace upfront static context injection with a 26-tool **native function-calling agent loop** (`aiSwitchboard.ts` and `agentTools.ts`).

**Context**: The early system fetched CRM records and RAG chunks upfront and stuffed them into a massive prompt. This caused:
- Token bloat on simple greetings like "Hi".
- AI hallucinations on payment status (assuming payment succeeded when it hadn't).
- Accidental exposure of marketing checkout links as delivered report links.

**Rationale**:
- The LLM reasons dynamically: it fetches data only when needed (e.g. calling `getDealById` or `verifyPayment`).
- Tools enforce server-side validation: private report links are gated by ownership verification inside `shareReportLink` before reaching the model.
- Reduces median prompt token size on routine conversational turns.

---

## Decision 6: Deterministic Code-Level Script & Language Detection

**Decision**: Implement code-level script and marker analysis in `replyLanguage.ts` rather than relying on LLM prompting to infer the reply language.

**Context**: Prompts instructing LLMs to "mirror the customer's language" frequently failed in production. Indian customer names or simple English greetings ("Hi") caused models to flip into Hinglish or Devanagari Hindi.

**Rationale**:
- Unicode script ranges reliably detect Devanagari, Bengali, Punjabi, Gujarati, Tamil, Telugu, etc., with zero ambiguity.
- Roman Hinglish marker dictionaries (`aap`, `kaise`, `kya`, `chahiye`, `batao`) matched as whole words prevent false positives.
- Passing a deterministic constraint (`"You MUST reply strictly in English"`) prevents model drift completely.

---

## Decision 7: Anti-Spam Marketing Guardrails & Tiered Delivery Suppression

**Decision**: Centralize all outbound marketing templates through `MarketingGuard` with frequency capping, STOP opt-outs, and error-code-driven suppressions.

**Context**: WhatsApp closely monitors template rejection and block rates. Uncontrolled abandoned-cart blasts or sending to invalid numbers degraded business account health ratings.

**Rules Established**:
- Frequency capping: Maximum 1 of the same template per 24 hours; maximum 3 marketing templates per 7 days per contact.
- Opt-outs: Any reply containing "STOP" sets `marketingOptOut = true`, permanently halting marketing templates.
- Tiered Delivery Suppression:
  - Error 131050 (opted out at Meta) / 131026 (undeliverable): Permanent suppression (`marketingHardSuppressed = true`).
  - Error 131049 (Meta ecosystem frequency cap): 24-hour temporary pause (`marketingSuppressExpiresAt = now + 24h`).
- **Transactional Immunity**: UTILITY and AUTHENTICATION templates (order status, payment confirmation, report delivery) are never blocked.

---

## Decision 8: Conversational In-Chat Checkout Slot-Filling

**Decision**: Build an in-chat conversational checkout state machine (`saveCheckoutDetails`, `geocodeBirthPlace`, `createReportOrder`) instead of redirecting users to an external website.

**Context**: Over 60% of potential buyers who received external website links dropped off without completing the order due to mobile browser context switches.

**Rationale**:
- Customers share birth details directly in chat, which the AI validates and geocodes in real time.
- The AI enforces a read-back confirmation step before invoking `createReportOrder`.
- Produces a direct, personalized UPI/Razorpay payment link for the generated deal, drastically improving checkout conversion rates.

---

## Decision 9: Proactive Health Watchdog & Token Auto-Renewal

**Decision**: Run a 30-minute background health watchdog (`healthMonitor.ts`) that validates access tokens and third-party geocoding APIs, sending automated WhatsApp alerts to admins.

**Context**: Meta access tokens occasionally expire or are revoked. When an Instagram token expired, the bot remained visually "active" in the dashboard, but hundreds of customer DMs went unanswered for days.

**Rationale**:
- Automated Graph API `debug_token` checks warn 7 days prior to token expiration.
- Instagram-login tokens (`IGAA...`, 60-day lifespan) are automatically refreshed via Graph API once they exceed 20 days.
- Geocoding API health is verified with lightweight probe queries.
- High-severity outages trigger instant WhatsApp messages to engineering leads using Meta-approved `web_error_alert` templates.

---

## Decision 10: Stale Payment Claim Conflict Resolution Heuristic

**Decision**: Prioritize customer payment assertions over lagging CRM "pending" statuses when deciding whether to share a payment recovery link.

**Context**: Zoho CRM payment updates lag behind actual Razorpay transactions by 1–3 minutes. Customers who just paid often replied "I already paid". The bot would read the stale CRM record ("Payment Pending") and argue with the customer, damaging brand trust.

**Rationale**:
- Regex patterns detect recent customer payment claims ("paid", "payment ho gaya", "debited", "UTR").
- While a claim is active, the bot suppresses payment links and reassures the customer that the payment is being verified, preventing customer conflict.

---

## Decision 11: 24-Hour Meta Window Batch Auto-Close Engine

**Decision**: Implement a background service that auto-closes conversations whose 24-hour Meta customer service window has expired.

**Context**: Meta blocks free-form business messaging outside the 24-hour window. Leaving thousands of expired threads open in the shared inbox confused human operators and bloated frontend queries.

**Rationale**:
- Background worker scans `customerReplyWindowExpiresAt <= now` and closes threads in batches of 500 every 60 seconds.
- Preserves threads where a human operator is actively working an escalation.
- Seamlessly reopens threads when the customer sends a new message.
