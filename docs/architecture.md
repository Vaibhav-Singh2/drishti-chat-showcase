# System Architecture

> **Disclaimer**: This repository is a technical case study. The original implementation is proprietary and owned by the employer. No confidential source code, credentials, or sensitive business information is included.

---

## 1. High-Level System Architecture

The system is structured as a **Turborepo monorepo** containing two main applications — a Next.js 16 web frontend and a Bun/Express API backend — backed by three data stores (MongoDB Atlas, self-hosted Qdrant, Redis) and integrated with Meta's Omnichannel Cloud APIs (WhatsApp Cloud API v20.0, Facebook Messenger, Instagram Graph API v21.0), Zoho's India DC APIs (CRM and Books), and the Drishti core server.

```mermaid
graph TB
    subgraph External["External Services & Gateways"]
        META_WA["Meta WhatsApp\nCloud API v20.0"]
        META_FB["Facebook\nMessenger API"]
        META_IG["Instagram\nGraph API v21.0"]
        ZOHO["Zoho India DC\nCRM + Books (Live)"]
        LLM["AI Providers\nClaude 3.5 · GPT-4o · Gemini 1.5 Pro"]
        R2["Cloudflare R2\nMedia Storage"]
        ECR["Amazon ECR\nContainer Registry"]
        DRISHTI_CORE["Drishti Core Server\nOrder & Geocoding APIs"]
    end

    subgraph Monorepo["Drishti Marketing OS — Turborepo Monorepo"]
        subgraph Frontend["apps/web — Next.js 16 (App Router + Standalone)"]
            LOGIN["/login\nAuth + TOTP 2FA"]
            INBOX["/inbox\nWhatsApp Shared Inbox"]
            SOCIAL_INBOX["/facebook & /instagram\nSocialInboxLayout"]
            DASH["/dashboard\nSales Funnel & Analytics"]
            MARKETING_UI["/marketing\nOpt-Outs & Suppressions"]
            AISETTINGS["/ai-settings\nModel Config + RAG KB"]
            QUEUES["/queues\nBullMQ Monitor"]
            TEMPLATES["/templates\nBroadcast Campaigns"]
        end

        subgraph Backend["apps/api — Express v5 + Bun Runtime"]
            WH["Webhook Controller\nHMAC Signature Gate"]
            META_WH["Meta Social Webhook\nFB / IG Routing"]
            AUTH["Auth Controller\nJWT + 2FA TOTP"]
            CHAT["Chat Controller\nInbox & Thread API"]
            MKTG["Marketing Controller\nAnti-Spam & Gating"]
            ANALYTICS["Analytics Controller\nFunnel & Cost Engine"]
            MEDIA["Media Controller\n16MB Upload + R2 Presign"]

            subgraph CoreServices["Infrastructure & Concurrency Services"]
                LOCK["DistributedLock\nRedis SET PX NX + Lua"]
                HEALTH["HealthMonitor\nToken Watchdog + Alerts"]
                WINDOW["ConversationWindow\n24h Meta Window Auto-Close"]
                LANG["ReplyLanguage\nDeterministic Script Detector"]
                MGUARD["MarketingGuard\nFrequency Capper & Suppressions"]
            end

            subgraph Queues["BullMQ Background Workers"]
                IQ["inbound-msg-queue\nInboundWorker"]
                OQ["outbound-msg-queue\nOutboundWorker"]
                VQ["vector-ingest-queue\nVectorWorker"]
                SQ["status-msg-queue\nStatusWorker"]
            end

            subgraph AILayer["Autonomous AI Agent Layer"]
                RAG["RAG Pipeline\nSentence-Aware Qdrant Search"]
                SWITCH["AI Switchboard\nProvider Router & Fallbacks"]
                TOOLS["Agent Tool Loop (x26 Tools)\nFunction-Calling Engine"]
                CHECKOUT["Checkout State Machine\nIn-Chat Slot-Filling"]
            end

            SOCKET["Socket Service\nRoom-Targeted WebSocket"]
        end

        subgraph DataStores["Data Stores"]
            MONGO["MongoDB Atlas\nPrimary Database\n23 Schemas"]
            QDRANT["Qdrant Vector DB\n1536-dim Cosine\ndrishti-knowledge"]
            REDIS["Redis 7\nQueue Backplane\n+ Lock + Cache"]
        end
    end

    %% Inbound Ingress
    META_WA -->|"POST /messages/webhook\nHMAC Signed"| WH
    META_FB -->|"POST /messages/webhook/meta/:id"| META_WH
    META_IG -->|"POST /messages/webhook/meta/:id"| META_WH

    WH -->|"Enqueue & ack < 2s"| IQ
    META_WH -->|"Enqueue & ack < 2s"| IQ

    %% Worker Execution & Concurrency
    IQ -->|"Acquire lock (conv:ID)"| LOCK
    IQ -->|"Detect Script"| LANG
    IQ -->|"Semantic retrieval"| RAG
    IQ -->|"Invoke agent loop"| SWITCH
    SWITCH --> TOOLS
    TOOLS -->|"Live contact/deal"| ZOHO
    TOOLS -->|"Geocode & order"| DRISHTI_CORE
    TOOLS --> CHECKOUT
    SWITCH -->|"Completions + tools"| LLM
    IQ -->|"Release lock (Lua)"| LOCK

    %% Outbound & Queues
    IQ -->|"Emit events"| SOCKET
    IQ -->|"Enqueue reply"| OQ
    OQ -->|"Frequency check"| MGUARD
    OQ -->|"POST /messages"| META_WA
    OQ -->|"POST /messages"| META_FB
    OQ -->|"POST /messages"| META_IG
    VQ -->|"Upsert vectors"| QDRANT

    %% Frontend Connections
    INBOX <-->|"HTTP / Cookie Auth"| CHAT
    INBOX <-->|"WebSocket"| SOCKET
    SOCIAL_INBOX <-->|"HTTP + WebSocket"| CHAT
    DASH -->|"GET /analytics"| ANALYTICS
    MARKETING_UI -->|"GET /marketing"| MKTG

    %% Proactive Health & Alerts
    HEALTH -->|"Audit tokens"| META_WA
    HEALTH -->|"Warn / Alert WhatsApp"| OQ

    %% Data Store Bindings
    Backend <-->|"Mongoose ODM"| MONGO
    RAG <-->|"REST Client"| QDRANT
    LOCK <-->|"ioredis"| REDIS
    Backend <-->|"ioredis"| REDIS
```

---

## 2. Inbound Message Lifecycle & Concurrency Flow

Every inbound message (WhatsApp, Messenger, or Instagram) traverses a 9-layer pipeline designed for sub-2s acknowledgement, distributed concurrency locking, script fidelity, and autonomous tool-driven response generation.

```mermaid
sequenceDiagram
    autonumber
    actor Customer as Omnichannel Customer (WA / IG / FB)
    participant Meta as Meta Graph APIs
    participant Webhook as Webhook Controller
    participant Redis as Redis / BullMQ Backplane
    participant Lock as DistributedLock (Redis)
    participant InWorker as Inbound Worker
    participant Lang as ReplyLanguage Engine
    participant RAG as RAG Pipeline (Qdrant)
    participant Switchboard as AI Switchboard
    participant Tools as Agent Tool Execution
    participant DB as MongoDB Atlas
    participant Socket as Socket Service
    actor Agent as Human Operator (Next.js)
    participant OutWorker as Outbound Worker

    Customer->>Meta: Sends Message ("When will my marriage report arrive?")
    Meta->>Webhook: POST /messages/webhook (HMAC-SHA256 signature)
    Note over Webhook: Validate signature (< 5ms)<br/>Push to Redis queue (< 10ms)
    Webhook->>Redis: Enqueue → inbound-msg-queue
    Webhook-->>Meta: HTTP 200 EVENT_RECEIVED (< 20ms)

    Redis->>InWorker: Dequeue job
    InWorker->>Lock: acquire("conv:conversationId", 30000ms)
    Note over Lock: Redis SET key token PX 30000 NX<br/>Guarantees only ONE worker processes thread
    InWorker->>DB: Idempotency check via metaWamid
    InWorker->>DB: Resolve Contact & Conversation (WA / IG / FB)
    InWorker->>DB: Save Message (direction: inbound)
    InWorker->>Socket: Emit message_new + conversation_update
    Socket-->>Agent: Real-time UI thread update

    InWorker->>Lang: detectMessageLanguage(text)
    Note over Lang: Deterministic script analysis<br/>(Devanagari / Hinglish / English / Indic)

    alt Conversation AI is Active
        InWorker->>Socket: Emit ai_thinking (starts UI pulsing)
        InWorker->>RAG: retrieveContext(query)
        RAG->>DB: OpenAI embedding + Qdrant search (> 0.3)
        InWorker->>Switchboard: invokeAgentLoop(prompt, tools, history, lang)

        loop Native Function-Calling Loop (1–3 roundtrips)
            Switchboard->>Tools: tool_call (e.g. lookupCustomerByPhone)
            Tools->>DB: Query Zoho CRM / Books / Drishti Server
            Tools-->>Switchboard: tool_result ({ deals, status: "Sent", ... })
        end

        Switchboard->>DB: Log tokens & costs → CostTracking

        alt Supervised Draft Mode
            InWorker->>DB: Save response (status: draft)
            InWorker->>Socket: Emit message_new (draft card)
            Socket-->>Agent: Render Suggested Reply card
            Agent->>Webhook: POST /messages/:id/approve
            Webhook->>Redis: Enqueue → outbound-msg-queue
        else Autonomous Mode
            InWorker->>DB: Save response (status: pending)
            InWorker->>Redis: Enqueue → outbound-msg-queue
        end
    end

    InWorker->>Lock: release(lockHandle)
    Note over Lock: Atomic Lua script verifies token and deletes key

    Redis->>OutWorker: Process outbound job
    Note over OutWorker: Check MarketingGuard frequency cap & opt-outs
    OutWorker->>Meta: POST /messages (Bearer / Page token)
    Meta-->>OutWorker: Return wamid / message_id
    OutWorker->>DB: Update status → sent + metaWamid
    OutWorker->>Socket: Emit message_status (sent)
    Socket-->>Agent: Update delivery indicators
```

---

## 3. Omnichannel Multi-Account Routing

To support multiple WhatsApp Business Accounts (WABAs), Facebook Pages, and Instagram Business profiles within a single tenant without cross-contamination:

```mermaid
graph TD
    INBOUND([Inbound Webhook]) --> ROUTE{Route Identifier}
    ROUTE -->|/messages/webhook| WA_DEFAULT[WhatsApp Default Client]
    ROUTE -->|/messages/webhook/meta/:id| META_RESOLVE[Lookup MetaAccount by ID]

    META_RESOLVE --> CHANNEL{channelType}
    CHANNEL -->|messenger| FB_MSG[Process Messenger Message\nMap sender PSID to Contact]
    CHANNEL -->|instagram| IG_MSG[Process Instagram DM\nMap sender IGSID to Contact]

    WA_DEFAULT & FB_MSG & IG_MSG --> CONV_LINK{Resolve Conversation}
    CONV_LINK -->|New Contact| CREATE_C[Create Contact with channelType + external ID]
    CONV_LINK -->|Existing Contact| ATTACH_C[Load Conversation with account reference]

    ATTACH_C --> WORKER[Push to BullMQ Inbound Queue]
```

### Account Models
- **`WhatsAppAccount`**: Stores WABA ID, Phone Number ID, App Secret, and dedicated Access Tokens for multi-number setups.
- **`MetaAccount`**: Stores `channelType` (`messenger` | `instagram`), `pageId`, `instagramAccountId`, and Page/IG access tokens. Includes auto-refreshed timestamps for `IGAA` Instagram User tokens.

---

## 4. The 26-Tool Native AI Agent Architecture

Instead of prompt-stuffing with a static dump, the AI Switchboard exposes an autonomous **Tool-Calling Registry** (`agentTools.ts`):

```mermaid
flowchart LR
    subgraph AgentLoop["Native Function-Calling Loop (LLM)"]
        PROMPT["System Brand Prompt\n+ Detected Language Directive\n+ 10 Messages History"] --> MODEL["LLM (Claude / GPT-4o / Gemini)"]
        MODEL --> DECIDE{Requires Data\nor Action?}
        DECIDE -->|Yes| CALL["Emit tool_call(name, args)"]
        CALL --> EXEC["Backend Tool Dispatcher"]
        EXEC --> RESULT["Return tool_result JSON"]
        RESULT --> MODEL
        DECIDE -->|No| FINAL["Generate Final Customer Message"]
    end

    subgraph ToolRegistry["26 Agent Tools by Category"]
        subgraph CRM["Customer & Deal Lookup"]
            T1["lookupCustomerByPhone"]
            T2["lookupCustomerByEmail"]
            T3["lookupCustomerByName"]
            T4["getDealById"]
            T5["findDealsByPaymentReference"]
            T6["getCustomerDeals"]
            T7["listCustomerReports"]
        end

        subgraph Payment["Payment & Invoicing"]
            T8["verifyPayment"]
            T9["getInvoices"]
        end

        subgraph Knowledge["Brand & Policy"]
            T10["searchKnowledgeBase"]
            T11["getReportCatalog"]
        end

        subgraph Checkout["Conversational In-Chat Checkout"]
            T12["geocodeBirthPlace"]
            T13["saveCheckoutDetails"]
            T14["createReportOrder"]
            T15["getCheckoutLink"]
            T16["getPaymentLink"]
        end

        subgraph Delivery["Delivery & Self-Service"]
            T17["shareReportLink"]
            T18["shareCorrectionLink"]
            T19["getReportContent"]
            T20["getReportStatus"]
        end

        subgraph Actions["Actions & Governance"]
            T21["requestRefund"]
            T22["escalateToHuman"]
            T23["requestCustomerFeedback"]
            T24["saveCustomerFeedback"]
            T25["setConversationLanguage"]
            T26["setFunnelStage"]
        end
    end

    EXEC -.-> ToolRegistry
```

---

## 5. Conversational In-Chat Checkout State Machine

A breakthrough capability in Drishti Marketing OS is **in-chat order booking**. Rather than redirecting users to an external website checkout where drop-offs occur, the AI collects details conversationally:

```mermaid
stateDiagram-v2
    [*] --> Discovery: Customer asks for guidance / report
    Discovery --> Recommending: AI recommends specific report type
    Recommending --> CollectingSlots: Customer agrees to proceed

    state CollectingSlots {
        [*] --> AskName: Report type selected
        AskName --> AskDOB: Name recorded
        AskDOB --> AskTOB: Date of birth recorded
        AskTOB --> AskPOB: Time of birth recorded
        AskPOB --> Geocoding: Place of birth given
        Geocoding --> AskPOB: Invalid / Ambiguous location
        Geocoding --> AskGender: geocodeBirthPlace confirmed coords
        AskGender --> AskQuestions: Gender confirmed
        AskQuestions --> ReadBack: 1-4 custom questions recorded
    }

    CollectingSlots --> ReadBack: All slots filled (saveCheckoutDetails)
    ReadBack --> ReadBackConfirmation: AI reads back complete details
    ReadBackConfirmation --> CollectingSlots: Customer modifies a detail
    ReadBackConfirmation --> OrderCreated: Customer confirms ("Yes, all correct")
    OrderCreated --> DeliverPaymentLink: createReportOrder generates order & payment link
    DeliverPaymentLink --> [*]: Customer completes payment via UPI / Card
```

---

## 6. Proactive Health Monitoring & Token Lifecycle Watchdog

Silent failures (such as expired tokens or third-party geocoding API outages) cause AI responders to fail silently while remaining "active" in the UI. A background service monitors system vitals and proactively alerts operations:

```mermaid
graph TD
    CRON["Health Monitor Watchdog\n(Runs every 30 minutes)"] --> T1{Check Meta Access Tokens}
    T1 -->|Valid| T2{Inspect Expiry Date}
    T1 -->|Code 190 Invalid| ALERT1[Trigger P1 WhatsApp Alert to Admins]
    T2 -->|< 7 Days Remaining| ALERT2[Trigger Warning WhatsApp Alert]
    T2 -->|Instagram Token > 20 Days| REFRESH[Auto-refresh IGAA Token\n(via Graph API)]

    CRON --> G1{Check Geocoder Service}
    G1 -->|Probe Fails| ALERT3[Trigger P2 WhatsApp Alert]

    ALERT1 & ALERT2 & ALERT3 --> SEND_ALERT["Outbound Dispatch via\nMeta-Approved web_error_alert Template"]
```

---

## 7. Database Relationship Diagram (MongoDB Atlas)

The platform is backed by 23 Mongoose collection schemas designed for high-concurrency messaging, state tracking, and auditing:

```mermaid
erDiagram
    User {
        ObjectId _id PK
        string email UK
        string passwordHash
        string name
        string role "admin | agent"
        string twoFactorSecret
        bool twoFactorEnabled
        bool isActive
    }

    Contact {
        ObjectId _id PK
        string phoneNumber UK
        string name
        string email
        string zohoContactId
        bool marketingOptOut
        bool marketingHardSuppressed
        date marketingSuppressExpiresAt
        bool isActive
    }

    MetaAccount {
        ObjectId _id PK
        string name
        string channelType "messenger | instagram"
        string pageId UK
        string instagramAccountId UK
        string pageAccessToken
        date tokenRefreshedAt
        string webhookVerifyToken
        bool isActive
    }

    WhatsAppAccount {
        ObjectId _id PK
        string name
        string phoneNumberId UK
        string wabaId
        string accessToken
        bool isDefault
    }

    Conversation {
        ObjectId _id PK
        ObjectId contactId FK
        ObjectId whatsAppAccountId FK
        ObjectId metaAccountId FK
        string channelType "whatsapp | messenger | instagram | website"
        bool isAiActive
        bool isAiDraftMode
        string status "open | closed"
        bool needsHumanAttention
        string escalationReason
        string feedbackStatus
        date lastMessageAt
        date lastCustomerMessageAt
        date customerReplyWindowExpiresAt
    }

    Message {
        ObjectId _id PK
        ObjectId conversationId FK
        ObjectId senderId FK
        string direction "inbound | outbound"
        string senderType "user | bot | contact"
        string text
        string messageType
        string status "sent | delivered | read | failed | pending | draft"
        string metaWamid UK
    }

    AISession {
        ObjectId _id PK
        ObjectId conversationId FK,UK
        string provider
        string modelName
        number temperature
    }

    CostTracking {
        ObjectId _id PK
        ObjectId conversationId FK
        string provider
        string modelName
        number inputTokens
        number outputTokens
        number calculatedCostUSD
    }

    Feedback {
        ObjectId _id PK
        ObjectId conversationId FK
        ObjectId contactId FK
        string status "sent | responded | completed"
        bool solved
        number rating
        string comment
    }

    MarketingSend {
        ObjectId _id PK
        string phoneNumber
        string templateName
        string category
        string status
        date sentAt
    }

    RefundRequest {
        ObjectId _id PK
        string dealId
        ObjectId conversationId FK
        string reason
        string category
        string status
    }

    Contact ||--o{ Conversation : "has"
    Conversation ||--o{ Message : "contains"
    Conversation ||--|| AISession : "configures"
    Conversation ||--o{ CostTracking : "tracks"
    Conversation ||--o{ Feedback : "gathers"
    MetaAccount ||--o{ Conversation : "routes"
    WhatsAppAccount ||--o{ Conversation : "manages"
```

---

## 8. Deployment Architecture (AWS EC2 & Docker Compose)

```mermaid
graph TB
    subgraph CI["GitHub Actions CI/CD Pipeline"]
        PUSH["git push → main"]
        LINT["Lint & Typecheck (Turborepo)"]
        BUILD["Docker BuildKit (Multi-stage)"]
        PUSH_ECR["Push Images to Amazon ECR"]
        SSH["SSH Deployment Hook"]
    end

    subgraph AWS["AWS EC2 Mumbai (ap-south-1)"]
        NGINX["Nginx Reverse Proxy\n(SSL Termination · 16MB Body Cap)"]

        subgraph Docker["Docker Compose Network (drishti-network)"]
            API_C["api container\nBun Runtime · Express v5\n:5005"]
            WEB_C["web container\nNext.js 16 Standalone\n:3005"]
            REDIS_C["redis:7-alpine\nPersistent Volume\nQueue + Lock Backplane"]
            QDRANT_C["qdrant/qdrant:v1.11\nPersistent Data Volume\n:6333"]
        end

        HEALTH_ROUTE["/healthz\nHTTP 200 (Mongo + Redis Healthy)\nHTTP 503 (Degraded)"]
    end

    subgraph Managed["Managed Cloud Services"]
        MONGO_ATLAS["MongoDB Atlas\nM10 Multi-AZ Cluster"]
        CF_R2["Cloudflare R2\nS3-Compatible Media Bucket"]
        CLOUDWATCH["AWS CloudWatch\nStructured JSON Logs"]
    end

    PUSH --> LINT --> BUILD --> PUSH_ECR --> SSH
    SSH -->|"docker compose pull api web\ndocker compose up -d"| Docker

    NGINX --> API_C
    NGINX --> WEB_C

    API_C --> REDIS_C
    API_C --> QDRANT_C
    API_C --> MONGO_ATLAS
    API_C --> CF_R2
    API_C -->|"stdout JSON"| CLOUDWATCH
```
