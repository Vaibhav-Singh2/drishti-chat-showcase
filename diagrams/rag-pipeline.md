# RAG Ingestion & Query Pipeline Diagram

```mermaid
flowchart TB
    %% Definitions
    subgraph UI["Admin Dashboard / Frontend"]
        UPLOAD["Document Upload UI\n(/ai-settings)"]
        CHAT_UI["Omnichannel Chat Inbox\n(WhatsApp · IG · FB)"]
    end

    subgraph Ingestion["Ingestion Pipeline (Asynchronous)"]
        VQ["vector-ingest-queue\n(BullMQ: Concurrency=1)"]
        VW["VectorWorker\n(Bun/Express)"]
        EXTRACT["Text Extraction\n(pdf-parse / TXT utf-8)"]
        CHUNK["Sentence & Boundary-Aware Chunking\n(Natural linguistic breakpoints)"]
        CLEAN["Sanitize Chunk\n(UTF-16/Emoji Surrogate Repair)"]
        HASH["MD5 Hashing\n(toUUID Deterministic Point ID)"]
        EMBED_INGEST["OpenAI Embedding API\n(text-embedding-3-small)"]
    end

    subgraph Query["Query Pipeline (Autonomous Agent Loop)"]
        IW["InboundWorker\n(inbound-msg-queue)"]
        RAG["ragPipeline.ts\n(Context Retrieval)"]
        EMBED_QUERY["OpenAI Embedding API\n(text-embedding-3-small)"]
        AGENT["Native Function-Calling Loop\n(searchKnowledgeBase Tool)"]
        LLM["AI Switchboard\n(Claude 3.5 / GPT-4o / Gemini 1.5 Pro)"]
    end

    subgraph Storage["Data Stores"]
        QDRANT[("Qdrant Vector DB\n(drishti-knowledge collection\n1536-dim Cosine)")]
        REDIS[("Redis 7\n(Queue backplane + Lock)")]
        TEMP_DISK["Local Temp Disk\n(/temp/ uploads)"]
    end

    %% Ingestion Flow
    UPLOAD -->|"POST /documents/upload"| TEMP_DISK
    UPLOAD -->|"Enqueue job"| VQ
    VQ -.->|"Redis Backend"| REDIS
    VQ -->|"Process Sequentially"| VW
    VW -->|"Read File"| TEMP_DISK
    VW --> EXTRACT
    EXTRACT --> CHUNK
    CHUNK --> CLEAN
    CLEAN --> HASH
    CLEAN --> EMBED_INGEST
    EMBED_INGEST -->|"1536-dim Vector"| QDRANT
    HASH -->|"MD5-based UUID (Deduplication)"| QDRANT
    VW -->|"Differentiated Cleanup (Preserve on retry)"| TEMP_DISK

    %% Query Flow
    CHAT_UI -->|"Inbound Customer Msg"| IW
    IW -->|"Agent Initialization"| AGENT
    AGENT -->|"searchKnowledgeBase(query)"| RAG
    RAG --> EMBED_QUERY
    EMBED_QUERY -->|"Query Vector"| QDRANT
    QDRANT -->|"Top Chunks (Cosine > 0.3)"| RAG
    RAG -->|"Semantic Chunks"| AGENT
    AGENT <-->|"Multi-Step Reasoning"| LLM
    AGENT -->|"Final Customer Message"| IW
```
