# 🎙️ Mobile AI Voice Assistant (RAG + Django + PostgreSQL)

An intelligent, low-latency AI Voice Assistant accessible on mobile devices via a responsive Progressive Web App (PWA). The system integrates **Speech-to-Text (STT)**, **Retrieval-Augmented Generation (RAG)** over a custom knowledge base, and **Text-to-Speech (TTS)** powered by **Django** and **PostgreSQL (pgvector)**, designed for cloud hosting on a custom domain.

---

## 📐 System Architecture

### 1. High-Level Multi-Tier Architectural Diagram

```mermaid
graph TB
    subgraph Client_Layer ["📱 Client Layer (Mobile PWA)"]
        UI["Mobile Touch UI / Dark Mode"]
        Mic["Microphone (MediaRecorder API)"]
        Player["Audio Player & Visualizer (Web Audio API)"]
        PWA_Worker["Service Worker & PWA Manifest"]
    end

    subgraph Ingress_Layer ["🌐 Ingress & Security Tier"]
        DNS["Custom Domain (DNS A-Record)"]
        SSL["Cloudflare / Let's Encrypt (HTTPS Required)"]
        Nginx["Nginx Reverse Proxy & Static Cache"]
    end

    subgraph App_Layer ["⚙️ Django Backend Application Tier"]
        WSGI["Gunicorn / Uvicorn ASGI Server"]
        DRF["Django REST Framework API"]
        AudioHandler["Audio Ingestion & Formats Converter"]
        AuthThrottle["Rate Limiter & Session Manager"]
    end

    subgraph Voice_Layer ["🎙️ Speech Processing Services"]
        STT["STT Engine: OpenAI Whisper / Groq Whisper"]
        TTS["TTS Engine: gTTS / Edge-TTS"]
        Cleaner["Markdown Stripper & Speech Text Normalizer"]
    end

    subgraph AI_RAG_Layer ["🧠 RAG & LLM Orchestration"]
        Embedder["Embedding Generator (OpenAI / HuggingFace)"]
        Retriever["pgvector Hybrid / Cosine Retriever"]
        PromptBuilder["Grounding Prompt & History Assembler"]
        LLM["LLM Engine (Gemini 2.0 Flash / GPT-4o-mini / Groq)"]
    end

    subgraph Data_Layer ["🐘 Data & Vector Persistence"]
        PG["PostgreSQL Database"]
        PGVector["pgvector Extension (HNSW / IVFFlat Index)"]
        MediaStorage["Local / S3 Audio Media Storage"]
        RedisCache["Redis (Sessions, Rate Limits & Vector Cache)"]
    end

    %% Connections
    UI --> Mic
    Mic -->|Audio Blob (WebM/WAV)| DNS
    DNS --> SSL
    SSL --> Nginx
    Nginx --> WSGI
    WSGI --> DRF
    DRF --> AudioHandler
    AudioHandler --> STT
    STT -->|Transcribed Text| DRF
    DRF --> Embedder
    Embedder -->|Query Vector| Retriever
    Retriever <--> PGVector
    Retriever -->|Context Chunks| PromptBuilder
    PromptBuilder --> LLM
    LLM -->|Raw LLM Response| Cleaner
    Cleaner --> TTS
    TTS -->|MP3 / Audio Stream| MediaStorage
    MediaStorage --> Nginx
    Nginx -->|Audio URL + Text Transcript| Player
    Player --> UI
```

---

### 2. End-to-End Voice Request Lifecycle (Sequence Diagram)

```mermaid
sequenceDiagram
    autonumber
    actor MobileUser as 📱 Mobile User
    participant PWA as 🌐 Frontend (PWA)
    participant Nginx as 🛡️ Nginx / SSL
    participant Django as ⚙️ Django API
    participant STT as 🎙️ Whisper STT
    participant Embed as 🔢 Embedding Model
    participant PGVector as 🐘 PostgreSQL (pgvector)
    participant LLM as 🧠 LLM (Gemini/Groq)
    participant TTS as 🔊 gTTS / Edge-TTS

    MobileUser->>PWA: Taps Mic & Speaks (Press-to-Talk)
    PWA->>PWA: Records audio chunks (MediaRecorder API)
    MobileUser->>PWA: Releases Mic (Stop Recording)
    PWA->>Nginx: POST /api/voice/interact/ (Audio Blob)
    Nginx->>Django: Proxy request to Gunicorn/Django
    Django->>STT: Send audio file for transcription
    STT-->>Django: Returns transcribed query ("What are the admission requirements?")
    Django->>Embed: Embed query text into 1536-dim vector
    Embed-->>Django: Query vector array
    Django->>PGVector: Cosine similarity query (SELECT ... ORDER BY embedding <=> query LIMIT 4)
    PGVector-->>Django: Relevant knowledge base chunks
    Django->>Django: Assemble Grounded System Prompt (Context + History + Query)
    Django->>LLM: Stream or fetch completion
    LLM-->>Django: Generated answer text
    Django->>Django: Clean text (strip markdown symbols, bullets, URLs)
    Django->>TTS: Synthesize cleaned speech text
    TTS-->>Django: Output audio file (MP3/WAV)
    Django-->>Nginx: Response payload (JSON: audio_url, transcript, answer_text)
    Nginx-->>PWA: HTTP 200 OK
    PWA->>MobileUser: Renders text in chat & auto-plays synthesized speech
```

---

### 3. Knowledge Base Ingestion Pipeline (RAG Setup)

```mermaid
flowchart LR
    Docs["📄 Raw Documents (PDF, TXT, MD)"] --> Parser["Document Parser & Text Cleaner"]
    Parser --> Chunker["Recursive Chunker (500-800 tokens, 10% overlap)"]
    Chunker --> Embeddings["Embedding Model (text-embedding-3-small)"]
    Embeddings --> VectorStore[("🐘 PostgreSQL + pgvector\n(HNSW Index for O(log N) lookup)")]
```

---

### 4. Database Schema Architecture

```mermaid
erDiagram
    KnowledgeDocument ||--o{ DocumentChunk : contains
    ChatSession ||--o{ ChatMessage : records

    KnowledgeDocument {
        uuid id PK
        string title
        string source_type
        text raw_content
        datetime created_at
    }

    DocumentChunk {
        uuid id PK
        uuid document_id FK
        text chunk_text
        int token_count
        vector_1536 embedding "Indexed with HNSW"
        jsonb metadata
    }

    ChatSession {
        uuid id PK
        string session_token
        string client_user_agent
        datetime created_at
    }

    ChatMessage {
        uuid id PK
        uuid session_id FK
        text user_transcript
        string user_audio_path
        text ai_response_text
        string ai_audio_path
        float latency_stt_sec
        float latency_llm_sec
        float latency_tts_sec
        datetime timestamp
    }
```

---

## 🛠️ Technology Stack

| Component | Selected Technology | Purpose & Rationale |
|---|---|---|
| **Mobile Frontend** | **PWA (HTML5, Vanilla CSS, JS)** | Native app feel on iOS/Android without app stores; access to Web Audio & MediaRecorder APIs. |
| **Backend Framework** | **Django + Django REST Framework** | Robust API architecture, built-in ORM, secure authentication, and administrative controls. |
| **Database & Vector Store** | **PostgreSQL + `pgvector`** | Unified database storing relational app data (users, sessions) and vector embeddings (HNSW/IVFFlat). |
| **Speech-to-Text (STT)** | **OpenAI Whisper / Groq Whisper / Web Speech** | Accurate multi-accent mobile voice transcription. |
| **RAG & Orchestration** | **LangChain / LlamaIndex + HuggingFace/OpenAI Embeddings** | Document chunking, vector indexing, similarity search, and prompt augmentation. |
| **Text-to-Speech (TTS)** | **gTTS (Google TTS) / Edge-TTS** | Free, natural speech synthesis; Edge-TTS offers low-latency streaming options. |
| **Production & DevOps** | **Docker, Nginx, Gunicorn, Cloudflare / Certbot** | Containerized deployment, reverse proxy, custom domain DNS, and automatic SSL. |

---

## 🗺️ Phased Action Plan & Implementation Roadmap

### Phase 1: Environment & Database Setup
- [ ] Initialize Python virtual environment (`python -m venv venv`) and Git repository.
- [ ] Install core dependencies:
  - `django`, `djangorestframework`, `django-cors-headers`
  - `psycopg2-binary`, `pgvector`
  - `python-dotenv`
- [ ] Configure PostgreSQL with the `pgvector` extension:
  ```sql
  CREATE EXTENSION IF NOT EXISTS vector;
  ```
- [ ] Design Django Database Models:
  - **`KnowledgeDocument`**: Document title, source file/URL, raw content, upload date.
  - **`DocumentChunk`**: Foreign key to document, chunk content, token count, vector embedding field (`VectorField(dimensions=...)`).
  - **`ChatSession` & `ChatMessage`**: Session ID, user audio input path, transcribed text, AI response, generated audio path, latency metrics.

---

### Phase 2: RAG Pipeline & Vector Search
- [ ] **Document Ingestion Engine**:
  - Support PDF, TXT, DOCX, and Markdown parsing.
  - Implement recursive character chunking (e.g. 500–800 tokens with 10% overlap).
- [ ] **Embedding Generation**:
  - Connect embedding model (e.g. `text-embedding-3-small` or local HuggingFace sentence-transformers).
  - Bulk compute and store vector embeddings into PostgreSQL.
- [ ] **Vector Indexing & Retrieval**:
  - Create HNSW or IVFFlat index on the vector column for sub-second search times.
  - Implement top-$k$ semantic search with cosine or L2 similarity.
- [ ] **Context Injection & LLM Synthesis**:
  - Engineer system prompts with strict grounding rules to eliminate hallucinations.
  - Integrate LLM API (Google Gemini, OpenAI, or Groq for ultra-fast generation).

---

### Phase 3: Voice Processing Pipeline (STT & TTS)
- [ ] **Speech-to-Text (STT) Service**:
  - Setup Django endpoint to receive audio blobs (`multipart/form-data`).
  - Transcribe audio using Whisper API (Groq/OpenAI) or local `faster-whisper`.
- [ ] **Text-to-Speech (TTS) Service**:
  - Clean LLM output (strip markdown formatting, asterisks, URLs, and emojis for natural speech).
  - Synthesize voice using `gTTS` or `edge-tts`.
  - Save audio to media storage or stream directly to the client.
- [ ] **Unified Voice Chat Endpoint**:
  - `POST /api/voice/interact/`: Takes user audio $\rightarrow$ STT $\rightarrow$ RAG query $\rightarrow$ LLM generation $\rightarrow$ TTS $\rightarrow$ returns JSON with audio URL + transcription + answer text.

---

### Phase 4: Mobile-First Frontend (PWA)
- [ ] **Mobile-Optimized Interface**:
  - Fluid, responsive dark-mode layout designed for one-handed mobile use.
  - Pulsing voice visualizer / wave animation during speech input and playback.
  - Push-to-talk (PTT) and tap-to-record toggle modes.
- [ ] **Audio Capture & Compatibility**:
  - Implement `MediaRecorder` API with fallback codecs (`audio/webm`, `audio/mp4`, `audio/wav`).
  - Resolve iOS Safari autoplay restrictions via initial user-gesture audio context unlock.
- [ ] **PWA Configuration**:
  - Add `manifest.json` with app icons, theme colors, and standalone display mode.
  - Setup lightweight service worker for asset caching and offline shell.

---

### Phase 5: Latency & Performance Optimization
- [ ] **Latency Benchmarking**: Keep round-trip time under 2.5 seconds.
  - Leverage Groq or Gemini 2.0 Flash for sub-second LLM inference.
  - Stream TTS audio chunks if real-time playback is needed.
- [ ] **Caching**:
  - Add Redis caching for repetitive queries and static document embeddings.
- [ ] **Audio Compression**:
  - Compress audio recordings on the client-side before transmission to minimize network payload.

---

### Phase 6: Domain Setup, Production Hosting & Security
- [ ] **Containerization**:
  - Create `Dockerfile` and `docker-compose.yml` for Django, PostgreSQL with pgvector, and Nginx.
- [ ] **Server Deployment**:
  - Deploy to an Ubuntu VPS (DigitalOcean Droplet, Hetzner, AWS EC2, or Render).
  - Configure Gunicorn/Uvicorn as WSGI/ASGI application server.
  - Configure Nginx reverse proxy with static/media file caching.
- [ ] **Domain & SSL Setup**:
  - Buy custom domain (Cloudflare, Namecheap, Porkbun, etc.).
  - Configure DNS `A` records pointing to your server's public IP address.
  - Obtain and auto-renew free SSL certificate via Let's Encrypt / Certbot:
    ```bash
    sudo certbot --nginx -d yourdomain.com -d www.yourdomain.com
    ```
    *(Note: HTTPS is **mandatory** for browsers to permit mobile microphone access).*
- [ ] **Security Hardening**:
  - Set `DEBUG = False`, configure `ALLOWED_HOSTS` and CORS origins.
  - Store API keys and database credentials in `.env`.
  - Add API rate limiting via Django REST framework throttling.

---

## ⚠️ Key Technical Considerations & Gotchas

1. **HTTPS Is Mandatory for Microphones**: Mobile browsers (iOS Safari, Android Chrome) strictly block microphone access on insecure HTTP connections (except `localhost`). HTTPS on your custom domain is non-negotiable.
2. **iOS Safari Audio Autoplay**: Mobile Safari blocks programmatic `.play()` calls unless an initial audio interaction occurs via a direct user tap. Initialize the `AudioContext` on the first user interaction.
3. **TTS Text Pre-processing**: Raw markdown (like `**bold**`, `## headers`, or bullet points) sounds unnatural when read by TTS engines. Strip all markdown formatting prior to TTS conversion.
4. **gTTS vs. Edge-TTS**:
   - `gTTS` is stable and free, but saves complete MP3 files and has noticeable network round-trip time.
   - `edge-tts` provides asynchronous streaming and significantly more natural voices.

---

## 🚀 Getting Started (Initial Local Setup)

### 1. Clone & Set Up Virtual Environment
```bash
python -m venv venv
# Windows:
venv\Scripts\activate
# Linux/Mac:
source venv/bin/activate
```

### 2. Install Core Packages
```bash
pip install django djangorestframework django-cors-headers psycopg2-binary pgvector gTTS edge-tts openai python-dotenv
```

### 3. Create Django Project
```bash
django-admin startproject voice_assistant_backend .
python manage.py startapp assistant
```
