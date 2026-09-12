# Drishti Marketing OS — Omnichannel AI Customer Engagement & Marketing Platform

> **Disclaimer**: This repository is a technical case study. The original implementation is proprietary and owned by the employer. No confidential source code, credentials, or sensitive business information is included.

---

## What Is This?

**Drishti Marketing OS** is a production-grade, omnichannel AI customer engagement and marketing platform built as a complete replacement for legacy BSP subscriptions (WATI/AISensy). It connects directly to **Meta's WhatsApp Cloud API**, **Facebook Messenger**, and **Instagram Direct**, routing all inbound customer interactions through an autonomous, native function-calling AI agent loop (Claude 3.5, GPT-4o, Gemini 1.5 Pro). 

The platform enriches every conversation with a self-hosted **Retrieval-Augmented Generation (RAG)** knowledge base (Qdrant), real-time bidirectional **Zoho CRM/Books** integration, atomic Redis **distributed concurrency locking**, automated **in-chat conversational checkout**, proactive **token & infrastructure health monitoring**, and comprehensive **anti-spam marketing guardrails**.

The result: median customer response time dropped from **4–12 hours → ~53 seconds**, with 24/7 autonomous multi-channel coverage and a **96.94% autonomous AI resolution rate** across 7,600+ conversations.

---

## 📺 Live Demo Video Walkthrough (Early Launch Recording)

> 🔗 **[Watch Full Platform Walkthrough on Google Drive](https://drive.google.com/file/d/1qsN0ZLRJXgkSQzIgWk0Hy7fjwcmx8TiG/view?usp=sharing)**
>
> *A recorded walkthrough demonstrating the platform's core UX: real-time WhatsApp & social inboxes, autonomous 26-tool agent reasoning, live Zoho CRM sidebars, conversational in-chat checkout, Meta template broadcasting, and the early analytics dashboard.*

> [!NOTE]
> **Media vs. Production Telemetry**: This recorded demo video and the UI screenshots in [screenshots.md](screenshots.md) capture the platform during its **initial launch milestone** (early 30-day token validation). The section below documents the platform's mature performance after **90 days of sustained production scaling** (growing from the early ~25M token baseline to over 475M tokens and 7,600+ conversations).

---

## Live Production Metrics (90-Day Telemetry)

> *Aggregated from live telemetry on [chat.maxfate.com/dashboard](https://chat.maxfate.com/dashboard) across 90 days of production operation.*

### Executive Performance Scorecard

| Metric | Legacy BSP Baseline | Production Platform (90-Day Live) | Impact / Delta |
|--------|---------------------|-----------------------------------|----------------|
| **Median Response Time** | 4–12 hours | **53.5 seconds** | **~99.8% speedup** |
| **Total Conversations Handled** | ~50/day per human agent | **7,616 conversations** | **Unlimited concurrency** |
| **Autonomous AI Coverage** | 0% (Manual only) | **99.9%** (7,612 of 7,616 chats) | **Full 24/7 coverage** |
| **Autonomous Resolution Rate** | N/A | **96.94%** (only 233 escalated to human) | **< 3.1% escalation rate** |
| **In-Chat Sales Conversion Rate** | 0% (Web redirect drop-offs) | **31.1%** (221 paid of 711 prospects) | **Direct chat revenue** |
| **Customer Satisfaction (CSAT)** | Unmeasured / anecdotal | **4.54 / 5.0 Stars** (66.7% solved) | **High consumer trust** |
| **AI Cost Efficiency** | Recurring per-seat fees | **$0.062 per conversation** ($477 total) | **Zero recurring BSP fees** |
| **Outbound Delivery Read Rate** | Unknown | **88.4%** (26,070 read of 29,492 delivered) | **High engagement** |
| **Marketing Opt-Out Rate** | Uncontrolled | **0.30%** (23 opt-outs / 7,715 contacts) | **Healthy account tier** |

---

### Omnichannel Channel Breakdown

The same platform routes WhatsApp, Facebook Messenger, and Instagram Direct with isolated token lifecycle management and channel-specific UTM sales attribution:

| Channel | Conversations | Share (%) | Open Chats | Closed Chats | Escalated to Human | Paid Chat Sales | Conversion Rate |
|---------|---------------|-----------|------------|--------------|--------------------|-----------------|-----------------|
| 🟢 **WhatsApp** | 6,604 | 86.7% | 31 | 6,573 | 192 (2.9%) | **154 paid** | **38.4%** |
| 🔵 **Facebook Messenger** | 693 | 9.1% | 10 | 683 | 10 (1.4%) | **55 paid** | **32.2%** |
| 🟣 **Instagram Direct** | 305 | 4.0% | 8 | 297 | 31 (10.2%) | **12 paid** | **8.7%** |
| 🌐 **Website Chat** | 14 | 0.2% | 0 | 14 | 0 (0.0%) | 0 paid | 0.0% |
| **Total Omnichannel** | **7,616** | **100%** | **49** | **7,567** | **233 (3.06%)** | **221 paid** | **31.1%** |

---

### In-Chat Conversational Sales Funnel

Rather than dropping off on external website checkout links, prospective customers are guided through a 7-stage conversational slot-filling state machine directly within chat:

```
[Prospects]      711  (100%)    ████████████████████████████████████████
[Understand]     711  (100%)    ████████████████████████████████████████  (100% step conversion)
[Recommend]      709  (99.7%)   ███████████████████████████████████████   (99.7% step conversion)
[Add to Cart]    619  (87.1%)   ████████████████████████████████          (87.3% step conversion)
[Form Progress]  594  (83.5%)   ██████████████████████████████            (96.0% step conversion)
[Order Created]  451  (63.4%)   █████████████████████                     (75.9% step conversion)
[Paid Purchases] 221  (31.1%)   ███████████                               (49.0% step conversion)
```

- **Step Conversion Highlights**: **96%** of users who start entering birth details finish all slots; **49%** of generated payment links convert to verified purchases.
- **Attribution Hygiene**: Only orders created directly *in-chat* (`createReportOrder`) count toward chat sales, filtering out organic website traffic.

---

### AI Model Spend & Token Economics (90 Days)

**Total Spend**: **$477.00 USD** | **Total Tokens**: **475,220,060** (~475.2 Million tokens) across 22,520 LLM completions:

| Model | Provider | Input Tokens | Output Tokens | Invocations | Total Cost (USD) | Cost Share | Primary Role |
|-------|----------|--------------|---------------|-------------|------------------|------------|--------------|
| `gpt-5.4-mini-2026-03-17` | OpenAI | 249,326,934 | 1,120,119 | 10,558 | **$195.40** | 41.0% | Primary Agent Tool Loop |
| `gpt-5.6-luna` | OpenAI | 126,038,853 | 1,780,316 | 7,093 | **$136.72** | 28.7% | Complex Inquiries & Nuance |
| `gpt-5.1-2025-11-13` | OpenAI | 82,745,627 | 976,015 | 4,122 | **$113.19** | 23.7% | Stable High-Volume Responder |
| `gpt-5.5-2026-04-23` | OpenAI | 3,169,863 | 48,801 | 183 | **$17.31** | 3.6% | Evaluation & Tone Tuning |
| `gpt-5.4-2026-03-05` | OpenAI | 5,194,312 | 24,434 | 242 | **$13.35** | 2.8% | Pre-Release Benchmarking |
| `gpt-5.4-nano-2026-03-17` | OpenAI | 4,736,406 | 45,949 | 321 | **$1.00** | 0.2% | Lightweight Classification |
| `gemini-3.1-pro-preview` | Google | 12,361 | 70 | 1 | **$0.02** | < 0.1% | Failover Provider Probe |
| **Total Aggregated** | — | **471,224,356** | **3,995,704** | **22,520** | **$477.00** | **100%** | **$0.062 / conversation** |

---

### Outbound Messaging & Delivery Funnel

Across **38,346 outbound messages** dispatched via Meta Graph API:
- **Delivered**: 29,492 messages (76.9% delivery rate)
- **Read**: 26,070 messages (**88.4% read rate** of delivered messages)
- **Customer Replies**: 672 direct inbound replies generated
- **Hard Delivery Failures**: 574 messages (1.5% failure rate, automatically suppressed by `MarketingGuard`)
- **Total Active Contacts**: 7,715 contacts managed with only 23 opt-outs (0.30%) and 56 permanent suppressions.

---

## Key Features

| Category | Capability |
|----------|-----------|
| 🌐 **Omnichannel Gateways** | Direct integration with Meta WhatsApp Cloud API (v20.0), Facebook Messenger API, and Instagram Graph API (v21.0) |
| 🧰 **Native AI Tool-Calling Loop** | 26 specialized tools enabling the LLM to autonomously inspect CRM contacts, verify payments, geocode birth places, generate payment links, share reports, and schedule CSAT surveys |
| 🛒 **Conversational In-Chat Checkout** | End-to-end slot-filling state machine (`saveCheckoutDetails`, `geocodeBirthPlace`, `createReportOrder`) with read-back confirmation gates |
| 🔒 **Distributed Concurrency Lock** | Redis atomic locking (`SET PX NX`) with tokenized Lua script release to eliminate duplicate AI executions on rapid user message bursts |
| 🛡️ **Marketing Guardrails & Anti-Spam** | Enforces frequency capping (1/24h per template, 3/7d per contact), STOP opt-out compliance, and tiered delivery-failure suppressions (permanent vs 24h Meta ecosystem caps) |
| 🏥 **Proactive Health Monitor** | Continuous background validation of Meta/Instagram tokens (with auto-refresh for 60-day `IGAA` tokens at day 20) and geocoding services with instant WhatsApp admin alerts |
| 🗣️ **Deterministic Script & Language Engine** | Code-level script detection (Devanagari, Bengali, Punjabi, Gujarati, Odia, Tamil, Telugu, Kannada, Malayalam, Urdu, Roman Hinglish, English) ensuring strict language fidelity |
| 📚 **Semantic RAG Knowledge Base** | Self-hosted Qdrant vector store; 1536-dim embeddings via OpenAI `text-embedding-3-small`; sentence-boundary chunking & MD5 point deduplication |
| 💬 **Real-time Shared Inboxes** | WebSocket-synced 4-column WhatsApp inbox plus dedicated `SocialInboxLayout` for Facebook Messenger & Instagram Direct |
| 🔀 **Dual AI Operating Modes** | Autonomous (instant AI send) or Supervised Draft mode (human operator review/edit before dispatch) |
| 🤝 **Bidirectional CRM Sync** | Live Zoho CRM & Books sidebar; customer profile resolution, deal stage tracking, invoice lookups, and whitelisted CRM actions (e.g. `"Request Refund"`) |
| 📊 **Sales Funnel & Attribution Analytics** | Multi-channel sales attribution (`whatsapp_chat`, `instagram_chat`, `facebook_chat`), drop-off stage analysis, CSAT ratings, and token cost tracking |
| ⏱️ **24h Window Auto-Close Service** | Automatic batch closing of conversations exceeding Meta's 24-hour customer service window, with seamless AI-mode preservation on reopen |

---

## Architecture Overview

```
┌────────────────────────────────────────────────────────────────────────┐
│               Meta Graph APIs (WhatsApp · Messenger · Instagram)       │
└──────────────▲───────────────────────────────────┬─────────────────────┘
 Outbound Post │                                   │ Inbound Webhooks
               │                                   ▼
┌────────────────────────────────────────────────────────────────────────┐
│  DRISHTI MARKETING OS (Turborepo Monorepo)                             │
│                                                                        │
│  ┌──────────────────────┐  WebSocket  ┌─────────────────────────────┐  │
│  │  Next.js 16 Web Apps │◄───────────►│  Express API (Bun Runtime)  │  │
│  │  • WhatsApp Inbox    │    HTTP     │  • HMAC Signature Gate      │  │
│  │  • Social Inbox (IG) │◄───────────►│  • Distributed Lock Manager │  │
│  │  • Marketing / Dash  │             │  • Proactive Health Monitor │  │
│  └──────────────────────┘             └──────────────┬──────────────┘  │
│                                                      │                 │
│                               ┌──────────────────────┼─────────────┐   │
│                               ▼                      ▼             ▼   │
│                          ┌─────────┐            ┌─────────┐   ┌──────┐ │
│                          │ MongoDB │            │ Qdrant  │   │Redis │ │
│                          │(23 Colls│            │ Vectors │   │Queue+│ │
│                          │ Atlas)  │            │(Self-   │   │ Lock │ │
│                          │         │            │ Hosted) │   │Backpl│ │
│                          └─────────┘            └─────────┘   └──────┘ │
└───────────────────────────────────────────────────────────────┬────────┘
             │ LLM Function-Calling Loop       │ Live APIs      │ Alerts
             ▼                                 ▼                ▼
     Claude 3.5 / GPT-4o /             Zoho India DC      Admin WhatsApp
     Gemini 1.5 Pro (Tools x26)        (CRM + Books)      Error Alerts
```

---

## Tech Stack

### Backend
- **Runtime**: Bun (Fast execution & rapid cold startup)
- **Framework**: Express v5
- **Concurrency**: Redis Atomic Distributed Locking with Lua script release
- **Queue System**: BullMQ + Redis (4 dedicated background queues with failed-job retention caps)
- **Primary DB**: MongoDB Atlas with Mongoose ODM (23 schemas with compound indexes)
- **Vector DB**: Qdrant (Self-hosted, open-source 1536-dim cosine similarity)
- **AI Tooling & SDKs**: OpenAI (`gpt-4o`, `text-embedding-3-small`), Anthropic (Claude 3.5 Sonnet), Google GenAI (Gemini 1.5 Pro) with native multi-step function calling
- **Media & Storage**: Cloudflare R2 (S3-compatible) + EC2 local disk storage

### Frontend
- **Framework**: Next.js 16 (App Router, Standalone Docker build)
- **State Management**: Zustand v5
- **Real-time Engine**: Socket.io-client with room-based multi-tenant event isolation
- **Styling**: Pure Vanilla CSS with custom obsidian/emerald design tokens
- **Data Visualization**: Recharts SVG analytics (Area, Donut, Funnel breakdowns)

### Infrastructure & Operations
- **Deployment**: AWS EC2 (Dockerized multi-container composition), Nginx reverse proxy
- **CI/CD**: GitHub Actions → Amazon ECR → Zero-downtime SSH deploy
- **Health & Monitoring**: Background health monitor for tokens, workers, and geocoding; automated WhatsApp template alerts

---

## Repository Structure

```
project-showcase/
├── README.md                     ← High-level overview, live 90-day telemetry, & core highlights
├── screenshots.md                ← Comprehensive 12-screenshot gallery & walkthrough
├── docs/
│   ├── architecture.md           ← Complete multi-channel architecture & sequence flows
│   ├── system-design.md          ← Database indexes, locking, queues, & anti-spam design
│   ├── engineering-decisions.md  ← Key architectural choices & trade-off evaluations
│   ├── challenges-and-solutions.md ← Deep-dive into real production engineering problems
│   └── my-contributions.md       ← Personal technical ownership & delivered systems
└── diagrams/
    ├── system-architecture.md    ← High-level system topology diagram
    ├── request-flow.md           ← Detailed inbound message lifecycle sequence
    ├── component-interactions.md ← Frontend, backend, worker, & tool interaction map
    ├── database-relationships.md ← Complete Entity-Relationship (ER) model
    ├── deployment-architecture.md← AWS EC2, Docker Compose, & CI/CD topology
    └── rag-pipeline.md           ← Sentence-aware RAG ingestion & query flow
```

---

## Screenshots & Live Demo

> 📺 **Video Demo**: [Watch Full Platform Walkthrough on Google Drive](https://drive.google.com/file/d/1qsN0ZLRJXgkSQzIgWk0Hy7fjwcmx8TiG/view?usp=sharing)  
> 🖼️ **Full UI Gallery**: See [screenshots.md](screenshots.md) for the complete 12-screenshot gallery with detailed technical captions.  
> *(Note: Visual media captured during initial launch rollout; see 90-day scorecard above for mature production scale).*

| Inbox (WhatsApp + Zoho CRM) | Token Cost Analytics |
|---|---|
| ![Inbox](assets/01-inbox-whatsapp-ai-response.png) | ![Analytics](assets/12-token-cost-analytics-dashboard.png) |

| Facebook Messenger Inbox (Auto AI) | Instagram DM Inbox (Auto AI) |
|---|---|
| ![Facebook Messenger](assets/02-inbox-facebook-auto-mode.png) | ![Instagram DM](assets/03-inbox-instagram-auto-mode.png) |

| AI Service Providers & Routing | BullMQ Queue Monitor |
|---|---|
| ![AI Providers](assets/05-ai-config-providers.png) | ![Queues](assets/08-queue-infrastructure-manager.png) |

---

## Documentation Index

| Document | Description |
|----------|-------------|
| [Architecture](docs/architecture.md) | Omnichannel gateways, BullMQ queues, distributed locking, and deployment topology |
| [System Design](docs/system-design.md) | Concurrency patterns, 23 MongoDB schemas, indexing strategy, and anti-spam guardrails |
| [Engineering Decisions](docs/engineering-decisions.md) | Why Bun, Qdrant, atomic locking, code-level language detection, and native tool-calling |
| [Challenges & Solutions](docs/challenges-and-solutions.md) | Real production bugs solved: race conditions, silent token expiry, language drift, and payment lag |
| [My Contributions](docs/my-contributions.md) | Complete inventory of systems designed, implemented, and owned |

---

## Core Engineering Highlights

1. **Sub-2s Webhook Ingestion** — HMAC-SHA256 verified payloads are enqueued to Redis instantly, returning `200 EVENT_RECEIVED` in < 15ms before Meta retry timers fire.
2. **Distributed Concurrency Control** — Atomic Redis locking (`SET PX NX`) with tokenized Lua release prevents race conditions and duplicate AI replies when customers send rapid consecutive messages.
3. **26-Tool Native Agent Loop** — Fully replaces prompt-stuffing with a reasoning loop where the LLM fetches CRM records on demand, geocodes addresses, verifies payments, and triggers whitelisted actions.
4. **Conversational In-Chat Checkout** — Slot-filling state machine collects and validates customer details (name, DOB, TOB, geocoded POB, report language), enforces a read-back confirmation gate, and creates the order directly in chat.
5. **Deterministic Script & Language Engine** — Code-level Unicode and Hinglish marker detection eliminates model language drift, preserving exact script and tone (Hinglish, Devanagari, English, regional Indic languages).
6. **Multi-Channel Social Inboxes** — Unified backend powers WhatsApp plus dedicated `SocialInboxLayout` for Facebook Messenger and Instagram Direct, complete with channel-specific UTM attribution.
7. **Proactive Health Monitoring & Token Auto-Refresh** — Automated watchdog inspects Meta tokens, warns 7 days prior to expiry, auto-renews 60-day Instagram tokens at day 20, and WhatsApps admins on silent failures.
8. **Anti-Spam Marketing Guardrails** — Enforces frequency capping (1/24h per template, 3/7d per contact), handles STOP opt-outs, and applies tiered suppressions without ever blocking transactional utility messages.
9. **Stale Payment Conflict Resolution** — Heuristic pattern matching prioritizes recent customer payment claims over lagging CRM status reads, preventing awkward arguments with paying customers.
10. **24-Hour Meta Window Auto-Close Service** — Background batch worker closes expired customer service sessions to prevent inbox clutter, with seamless AI state restoration upon customer reply.

---

*Built as an end-to-end full-stack engineering project: omnichannel architecture, distributed systems, backend, frontend, DevOps, AI integration, and database design.*
