# My Engineering Contributions

> **Disclaimer**: This repository is a technical case study. The original implementation is proprietary and owned by the employer. No confidential source code, credentials, or sensitive business information is included.

---

## Overview

I was the **sole full-stack engineer** responsible for designing, building, and scaling Drishti Marketing OS from an initial concept into an enterprise-grade omnichannel customer engagement operating system. I owned every tier of the product: architecture design, backend infrastructure, concurrency control, AI tool calling, frontend UI engineering, database modeling, and DevOps automation.

This document outlines what I personally architected, built, and shipped into production.

---

## 1. System Architecture & Omnichannel Ingress

**What I designed and delivered:**
- Designed the full Turborepo monorepo architecture uniting Next.js 16 and Express (Bun runtime).
- Architected direct integrations with **Meta's WhatsApp Cloud API (v20.0)**, **Facebook Messenger API**, and **Instagram Graph API (v21.0)**, eliminating recurring third-party BSP subscription costs.
- Designed multi-account routing schemas (`MetaAccount`, `WhatsAppAccount`) enabling simultaneous management of multiple WhatsApp Business Accounts, Facebook Pages, and Instagram Business profiles within a unified system.
- Designed an async-first queue pipeline using BullMQ + Redis to satisfy Meta's 2-second webhook acknowledgement requirement, achieving sub-15ms ingest times.

---

## 2. Distributed Concurrency & Locking Engine

**What I built:**
- Authored `DistributedLock.ts`, an atomic Redis locking system (`SET PX NX`) preventing race conditions when customers send rapid consecutive message bursts.
- Implemented atomic lock release using custom Lua scripts to guarantee that only the worker holding the specific token can release the lock.
- Solved duplicate AI inferences, conflicting order states, and out-of-order responses across horizontally distributed container workers.

---

## 3. Autonomous 26-Tool AI Agent Loop & Conversational Checkout

**What I built:**
- Built the native function-calling agent loop in `aiSwitchboard.ts` and `agentTools.ts`, moving beyond static prompt-stuffing to an autonomous reasoning engine across Claude 3.5, GPT-4o, and Gemini 1.5 Pro.
- Authored **26 specialized native tools** spanning customer lookup, payment verification, birth place geocoding, report delivery, self-serve corrections, and CSAT feedback.
- Designed the **in-chat conversational checkout state machine** (`saveCheckoutDetails`, `geocodeBirthPlace`, `createReportOrder`):
  - In-chat slot-filling collecting name, DOB, TOB, POB, and custom questions.
  - Integration with Drishti core servers to geocode birth locations into canonical coordinates.
  - Strict read-back confirmation gate before order generation.
  - In-chat personalized UPI / Razorpay payment link generation, increasing checkout conversion rates.
- Implemented **server-side privacy guardrails** inside `shareReportLink` with tolerant first-name and date-of-birth validation, preventing unauthorized URL exposure.
- Authored the **stale payment claim heuristic**, detecting customer payment claims to prevent the AI from falsely claiming an order is unpaid while the CRM catches up.
- Implemented keep-alive typing indicator heartbeats on WhatsApp to prevent typing status timeouts during multi-step tool reasoning.

---

## 4. Deterministic Language & Script Detection Engine

**What I built:**
- Authored `replyLanguage.ts`, a code-level linguistic and script analyzer eliminating LLM language drift.
- Built Unicode range matchers for Devanagari Hindi, Bengali, Punjabi, Gujarati, Odia, Tamil, Telugu, Kannada, Malayalam, and Urdu.
- Built isolated word-boundary regex patterns for Roman-script Hinglish markers (`aap`, `kaise`, `kya`, `nahi`, `chahiye`, `batao`).
- Built neutral token filters ("Hi", "Hello", "OK", "9:30 AM") to preserve established conversation language.
- Guaranteed 100% language fidelity, completely eliminating unwanted flips to Hinglish on English chats.

---

## 5. Anti-Spam Marketing Guardrails & Suppressions

**What I built:**
- Built `MarketingGuard.ts` to protect WhatsApp Business Account quality ratings from spam penalties.
- Implemented strict frequency capping: max 1 template per 24 hours per template; max 3 marketing templates per 7 days per contact.
- Built automated STOP opt-out handling (`marketingOptOut = true`).
- Implemented **tiered delivery-failure suppression**:
  - Permanent suppression on Meta error codes 131050 (user stopped marketing) and 131026 (undeliverable number).
  - 24-hour temporary hold on Meta error code 131049 (Meta ecosystem healthy engagement cap).
  - Guaranteed transactional immunity so UTILITY and AUTHENTICATION templates are never blocked.

---

## 6. Proactive Health Watchdog & Token Auto-Renewal

**What I built:**
- Built `healthMonitor.ts` and `alertScheduler.ts` to detect silent failures before customer impact.
- Automated Graph API audits validating Meta access tokens and warning 7 days prior to expiration.
- Built an automatic renewal engine for 60-day Instagram User access tokens (`IGAA...`) at day 20, preventing silent DM auto-reply outages.
- Implemented automated P1 WhatsApp alerts to engineering leads using Meta-approved `web_error_alert` templates.

---

## 7. RAG Knowledge Base & Sentence-Aware Chunking

**What I built:**
- Built `ragPipeline.ts` using self-hosted Qdrant for 1536-dimensional vector search via OpenAI `text-embedding-3-small`.
- Implemented sentence- and boundary-aware text chunking for uploaded PDFs and TXT documents in `VectorWorker.ts`.
- Built Unicode text sanitization to strip emoji surrogate split characters that previously broke vector upserts.
- Built MD5 point deduplication ensuring document re-uploads do not create duplicate vector points.

---

## 8. Real-Time Frontend & Shared Inboxes

**What I built:**
- Built the Next.js 16 web application from scratch using the App Router and standalone Docker mode.
- Developed the 4-column WhatsApp inbox (`InboxLayout.tsx`) featuring real-time WebSocket syncing, 24h Meta reply window countdown timers, live Zoho CRM deal sidebars, and Suggested Reply draft cards.
- Developed `SocialInboxLayout.tsx`, a specialized inbox layout for Facebook Messenger and Instagram Direct channels.
- Built the **AI Copilot Composer** (`ChatComposer.tsx`) offering tone refinements, expansion, and bullet-point compression.
- Designed the full dark-mode design system (`globals.css`) with custom obsidian and emerald tokens.

---

## 9. Sales Funnel & Multi-Channel Attribution Analytics

**What I built:**
- Built `messagingAnalyticsController.ts` and `funnel.ts` to track and visualize customer journey drop-offs across 7 distinct stages.
- Implemented channel-specific UTM attribution (`whatsapp_chat`, `instagram_chat`, `facebook_chat`) to track exact revenue contribution per channel.
- Built frontend data visualizations (`MessagingAnalytics.tsx`, `SalesFunnel.tsx`, `MarketingOptOuts.tsx`) with universal date filtering.
- Built the CSAT feedback state machine (`Feedback.ts`) collecting problem resolution ratings and qualitative feedback.

---

## 10. Database Modeling & DevOps CI/CD

**What I built:**
- Designed all 23 Mongoose collection schemas in MongoDB Atlas with compound indexes optimized for high-concurrency filtering and sorting.
- Engineered zero-downtime GitHub Actions CI/CD deploying to AWS EC2 via Amazon ECR and multi-stage Docker builds.
- Configured Nginx reverse proxy with SSL termination and 16MB file upload streaming.

---

## 11. Measurable Production Outcomes (90-Day Telemetry)

The direct business and operational impact of these contributions over a 90-day live production window ([chat.maxfate.com/dashboard](https://chat.maxfate.com/dashboard)):

- **Autonomous Scale**: Successfully handled **7,616 customer conversations** across WhatsApp (86.7%), Facebook Messenger (9.1%), and Instagram Direct (4.0%) with a **99.9% active AI coverage rate** (7,612 threads).
- **High Autonomous Resolution**: Maintained a **96.94% autonomous AI resolution rate**, escalating only 233 conversations (3.06%) to human operators.
- **Speed & Latency**: Slashed median response time from 4–12 hours to **53.5 seconds** across 2,054 sampled inbound response pairs.
- **In-Chat Conversational Sales**: Converted **221 paid purchases** directly within chat out of 711 prospects entering buying flows — achieving an overall **31.1% in-chat conversion rate** (38.4% on WhatsApp, 32.2% on Messenger).
- **Token & Cost Efficiency**: Processed **475.2 Million tokens** across 22,520 LLM completions for **$477.00 USD total spend**, averaging just **$0.062 per conversation**.
- **Customer Satisfaction (CSAT)**: Achieved a **4.54 / 5.0 Star average rating** across 289 rated customer surveys, with **66.7%** reporting first-contact issue resolution.
- **Anti-Spam Quality**: Maintained an **88.4% read rate** on delivered outbound messages with only 23 opt-outs (0.3%) and 56 suppressions out of 7,715 total contacts.
