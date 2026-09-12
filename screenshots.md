# Screenshots

> **Disclaimer**: This repository is a technical case study. The original implementation is proprietary and owned by the employer. No confidential source code, credentials, or sensitive business information is included. Screenshots have been reviewed for sensitive data before inclusion.

---

## 📺 Live Video Walkthrough

> 🔗 **[Watch Full Platform Walkthrough on Google Drive](https://drive.google.com/file/d/1qsN0ZLRJXgkSQzIgWk0Hy7fjwcmx8TiG/view?usp=sharing)**
>
> *A recorded walkthrough showcasing the real-time WhatsApp & social inboxes, autonomous 26-tool agent reasoning in action, live Zoho CRM sidebar integration, conversational in-chat checkout flow, template broadcasting, and the 90-day production analytics dashboard.*

---

## 1. Shared Inbox — WhatsApp (AI Auto-Response + Zoho CRM Sidebar)

The 4-column inbox layout (`InboxLayout.tsx`): navigation sidebar, conversation thread list with Open/Closed filters and AI mode badges, the active chat canvas, and the live Zoho CRM contact + deal sidebar. Shows the AI auto-response (labeled **DRISHTI AI**), the Manual/Supervised/Auto mode toggle, the 24h Meta reply window countdown timer, and CRM actions — **View/Share Report**, **Report Correction Link**, and **Request Refund**.

![WhatsApp inbox with AI response and Zoho CRM sidebar](assets/01-inbox-whatsapp-ai-response.png)

---

## 2. Shared Inbox — Facebook Messenger (Autonomous AI Mode)

The specialized social inbox (`SocialInboxLayout.tsx`) serving a **Facebook Messenger** channel via Meta's Graph API. The **Auto** mode indicator is active — all replies are sent autonomously by the native tool-calling AI agent. The Zoho CRM panel shows "No Zoho CRM profile linked" for this contact (PSID not yet linked to phone).

![Facebook Messenger inbox in Auto AI mode](assets/02-inbox-facebook-auto-mode.png)

---

## 3. Shared Inbox — Instagram DM (Autonomous AI Mode)

The social inbox serving an **Instagram Direct** channel with all conversations running in full **Auto** mode. Demonstrates the omnichannel architecture — the same backend engine routes WhatsApp, Facebook Messenger, and Instagram DMs simultaneously while isolating account credentials and webhook challenge tokens.

![Instagram DM inbox in Auto AI mode](assets/03-inbox-instagram-auto-mode.png)

---

## 4. AI Configuration — Guidelines & RAG Settings

The **AI Configuration** page → **Guidelines & RAG** tab. Shows the system brand prompt editor, the **RAG Cosine Similarity Cutoff** slider (set to 0.30), and the **Context History Message Depth** slider (set to 10 messages). All settings are persisted to `GlobalConfig` in MongoDB Atlas and apply globally across all AI provider sessions.

![AI Configuration - Guidelines and RAG settings](assets/04-ai-config-guidelines-rag.png)

---

## 5. AI Configuration — AI Service Providers & Routing

The **AI Service Providers** tab. Shows the multi-provider panel with **OpenAI (GPT Models)** currently set as the **Active Override** (model: `gpt-5.4-nano`), with per-million-token input/output pricing configured. **Anthropic Claude** and **Google Gemini** are shown as System Default (.env) — available for override with scoped rollout options (new, old, or all chats).

![AI Configuration - Service providers and routing](assets/05-ai-config-providers.png)

---

## 6. AI Configuration — RAG Knowledge Base Documents

The **Knowledge Base** tab showing the drag-and-drop PDF/TXT uploader and the ingested vector files table. Documents display chunk counts and completion status. Each chunk is processed via sentence-aware chunking, sanitized of emoji surrogate splits, and embedded as a 1536-dim vector in the self-hosted Qdrant instance with MD5-derived UUIDs.

![AI Configuration - Knowledge Base document ingestion](assets/06-ai-config-knowledge-base.png)

---

## 7. AI Configuration — Real-Time Context Simulation Playground

The **Context Playground** tab — an administrative testing tool that lets engineers simulate how brand instructions, RAG chunks, conversation history, and live Zoho CRM context are merged before invoking LLM tool reasoning. Used to validate agent behavior without sending live customer messages.

![AI Configuration - Context simulation playground](assets/07-ai-config-playground.png)

---

## 8. Queue & Infrastructure Manager

The **BullMQ Queue Monitor** showing all 4 background queues: `inbound-msg-queue`, `outbound-msg-queue`, `status-msg-queue`, and `vector-ingest-queue`. Each queue shows real-time **Active**, **Waiting**, **Delayed**, and **Failed** job counts with automatic 24h failed-job retention pruning to prevent Redis memory leaks.

![BullMQ Queue and Infrastructure Manager](assets/08-queue-infrastructure-manager.png)

---

## 9. Meta WhatsApp Templates — Broadcast Console

The **Templates** page showing synced Meta-approved WhatsApp templates in an interactive card grid. Templates show their approval status (APPROVED / REJECTED), language, category (MARKETING, UTILITY), and variable placeholders. Admins can sync from Meta Graph API and trigger bulk campaigns gated by `MarketingGuard`.

![Meta WhatsApp Templates broadcast console](assets/09-templates-broadcast.png)

---

## 10. Create WhatsApp Template — Multi-Step Modal

The **Create Template** modal with a 3-step creation flow (Basic Info → Content → Preview). Supports template categories (**Marketing**, **Utility**, and **Authentication**) and dynamic button configurations before submitting to Meta for approval via Graph API.

![Create WhatsApp Template modal](assets/10-templates-create-modal.png)

---

## 11. Security Settings — Operator Profile & 2FA

The **Security Settings** page showing the operator profile card (name, email, role, account status) and the **Two-Factor Authentication (2FA)** setup panel. Operators configure TOTP authentication via Google Authenticator using the QR code enrollment flow powered by `otplib`.

![Security Settings and 2FA configuration](assets/11-security-settings-2fa.png)

---

## 12. Token Cost & Messaging Analytics Dashboard

The **Token Cost Analytics** dashboard on `chat.maxfate.com/dashboard` showing AI spend metrics. Displays total calculated cost, per-provider breakdown (Claude, OpenAI, Gemini), a **Daily Cost Trend** area chart, and a **Token Volume Share** donut chart broken down by exact model name. 

While this specific snapshot captures an earlier 30-day window, the live 90-day production telemetry reflects:
- **Total AI Spend**: **$477.00 USD** across 22,520 LLM completions
- **Total Volume**: **475.2 Million tokens** (471.2M input / 4.0M output)
- **Unit Economics**: **$0.062 per conversation**
- **Core Model**: `gpt-5.4-mini` as the primary tool agent ($195.40 / 249M input tokens) with `gpt-5.6-luna` for complex reasoning.

![Token Cost Analytics Dashboard](assets/12-token-cost-analytics-dashboard.png)
