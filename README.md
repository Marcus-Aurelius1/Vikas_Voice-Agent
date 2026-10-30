<div align="center">

# 🎙️ Vikas: Autonomous Voice & Multimodal Chat Agent

**A low-latency, conversational voice and chat assistant powered by LangGraph, Groq, Hybrid Qdrant RAG, and Model Context Protocol (MCP).**

[![Python 3.11+](https://img.shields.io/badge/Python-3.11%2B-blue?logo=python&logoColor=white)](https://www.python.org/)
[![FastAPI](https://img.shields.io/badge/FastAPI-0.112%2B-009688?logo=fastapi&logoColor=white)](https://fastapi.tiangolo.com/)
[![LangGraph](https://img.shields.io/badge/LangGraph-Agentic%20Workflow-FF6F00?logo=langchain&logoColor=white)](https://github.com/langchain-ai/langgraph)
[![Groq](https://img.shields.io/badge/Groq-LPU%20Inference-f55036)](https://groq.com/)
[![Qdrant](https://img.shields.io/badge/Qdrant-Hybrid%20Vector%20Search-dc244c?logo=qdrant&logoColor=white)](https://qdrant.tech/)
[![Edge-TTS](https://img.shields.io/badge/Edge--TTS-Free%20Streaming%20Audio-2b7489)](https://github.com/rany2/edge-tts)
[![Docker](https://img.shields.io/badge/Docker-Ready-2496ED?logo=docker&logoColor=white)](https://www.docker.com/)
[![License: MIT](https://img.shields.io/badge/License-MIT-yellow.svg)](https://opensource.org/licenses/MIT)

[Features](#-key-features) • [System Architecture](#-system-architecture) • [Getting Started](#-getting-started) • [Environment Variables](#-environment-variables) • [Docker & Deployment](#-docker--deployment) • [API Reference](#-api-reference)

---

</div>

## 🌟 Overview

**Vikas** is a production-ready conversational voice agent designed to provide fluid, human-like voice conversations in natural Hinglish and English. Rather than waiting for an entire response to generate before synthesizing speech, Vikas utilizes **sentence-level streaming over NDJSON** to transcribe, think, and stream audio chunks concurrently with minimal latency.

Beyond conversation, Vikas possesses an agentic reasoning loop that can retrieve knowledge from uploaded PDFs (using dense vector + BM25 sparse hybrid search), query real-time stock prices, conduct web searches, and communicate with external servers through the **Model Context Protocol (MCP)**.

---

## ✨ Key Features

- ⚡ **Low-Latency Voice Streaming (`/turn`)**:
  - Speech-to-Text via **Groq Whisper** (`whisper-large-v3-turbo`).
  - Connector-word detection (`and`, `aur`, `lekin`, `ki`, `because`) to avoid awkward speech interruptions while the user is still formulating thoughts.
  - Sentence-by-sentence text generation and **Edge-TTS** audio chunk streaming over NDJSON.
  - Real-time barge-in and speech interruption support (`/interrupt`).

- 🧠 **LangGraph Agentic State Machine**:
  - Cyclical LLM-to-tool routing powered by `openai/gpt-oss-120b` on Groq.
  - Multi-tenant conversation checkpointer using **PostgreSQL** (`AsyncPostgresSaver`) on Neon, with an automated fallback to local **SQLite** (`AsyncSqliteSaver`).
  - Thread namespacing (`user:{user_id}:{call_id}`) ensures strict multi-tenant data isolation.

- 📄 **Multimodal Hybrid PDF RAG**:
  - PDF document structure, text, and tables parsed using `pymupdf4llm`.
  - Figures and visual diagrams extracted and captioned using **Groq Vision** (`meta-llama/llama-4-scout-17b-16e-instruct`).
  - Hybrid embeddings generated using ONNX-based **FastEmbed** (`all-MiniLM-L6-v2` dense + `Qdrant/bm25` sparse).
  - Reciprocal Rank Fusion (**RRF**) querying against **Qdrant Cloud** collections.

- 🔌 **Extensible Toolset via Model Context Protocol (MCP)**:
  - Built-in live tools: `DuckDuckGoSearchRun` for real-time web browsing and `get_stock_price` via Alpha Vantage.
  - Remote MCP server integration via `MultiServerMCPClient` (`deepwiki` MCP and `expense` FastMCP).

- 🔐 **Zero-Secret Authentication (Neon Auth)**:
  - Managed Google OAuth and session management via Neon Auth (Better Auth).
  - Public JWKS verification of short-lived EdDSA JWTs with automatic key-rotation caching.
  - User whitelist control via `ALLOWED_EMAILS`.

- 🎨 **Modern Web Interface**:
  - Single-page application styled with dark glassmorphism, responsive controls, and an animated audio orb.
  - In-browser microphone recording, audio queue playback, and instant drag-and-drop document upload.

---

## 🏗️ System Architecture

```mermaid
flowchart TD
    subgraph Client ["Client Layer (Web Dashboard)"]
        UI["Modern UI (index.html)"]
        MIC["Audio Capture (MediaRecorder)"]
        PLAYER["Sequential Audio Player"]
    end

    subgraph Auth ["Authentication"]
        NEON_AUTH["Neon Auth (Better Auth)"]
        JWKS["Public JWKS (EdDSA / Ed25519)"]
    end

    subgraph API ["FastAPI Backend (server.py)"]
        AUTH_DEP["Token Verification"]
        TURN_ENDPOINT["/turn (NDJSON Stream)"]
        CHAT_ENDPOINT["/chat (REST)"]
        INGEST_ENDPOINT["/ingest (PDF Upload)"]
    end

    subgraph CoreBrain ["LangGraph Core (backend_mcp.py)"]
        LLM["ChatGroq (gpt-oss-120b)"]
        STATE_GRAPH["StateGraph Workflow"]
        TOOL_NODE["ToolNode Executor"]
        CHECKPOINTER[("Neon Postgres / SQLite Checkpointer")]
    end

    subgraph Tools ["Tools & Capabilities"]
        WEB["DuckDuckGo Search"]
        STOCK["Alpha Vantage Stocks"]
        MCP["MCP Servers (DeepWiki, Expense)"]
        RAG_TOOL["retrieve_relevant_chunks"]
    end

    subgraph RAG ["Multimodal RAG Pipeline (retrieval.py)"]
        PDF["PyMuPDF4LLM"]
        VISION["Groq Vision (Image Captioning)"]
        FASTEMBED["FastEmbed (Dense + BM25)"]
        QDRANT[("Qdrant Vector DB (RRF Fusion)")]
    end

    subgraph AudioEngine ["Voice Services"]
        STT["Groq Whisper STT"]
        TTS["Edge-TTS Synthesis"]
    end

    %% Audio and Interaction Flow
    MIC -->|Raw Audio (webm)| TURN_ENDPOINT
    UI -->|Bearer JWT| AUTH_DEP
    AUTH_DEP -.->|Verify against| JWKS
    TURN_ENDPOINT -->|Audio Bytes| STT
    STT -->|Transcript| STATE_GRAPH

    %% State & Reasoning Flow
    STATE_GRAPH --> LLM
    LLM -->|Tool Call| TOOL_NODE
    TOOL_NODE --> WEB & STOCK & MCP & RAG_TOOL
    TOOL_NODE -->|Observation| LLM
    STATE_GRAPH <--> CHECKPOINTER

    %% RAG Pipeline Flow
    INGEST_ENDPOINT --> PDF
    PDF --> VISION
    VISION & PDF --> FASTEMBED
    FASTEMBED --> QDRANT
    RAG_TOOL <--> QDRANT

    %% Speech Synthesis Flow
    LLM -->|Sentence Chunks| TTS
    TTS -->|Base64 Audio + Text| TURN_ENDPOINT
    TURN_ENDPOINT -->|Streamed NDJSON| PLAYER
```

---

## 📁 Project Structure

```bash
voice-agent/
├── server.py              # FastAPI application, route handlers, and NDJSON streaming
├── backend_mcp.py         # LangGraph workflow, ChatGroq LLM, tools, and checkpointer
├── retrieval.py           # Multimodal PDF extraction, FastEmbed embeddings, and Qdrant RAG
├── stt.py                 # Groq Whisper speech-to-text integration
├── Tts.py                 # Edge-TTS multi-voice synthesis helper
├── neon_auth.py           # Neon Auth EdDSA JWT validation with JWKS key rotation
├── index.html             # Vanilla HTML/JS frontend dashboard with audio orb & auth
├── requirements.txt       # Python dependencies
├── Dockerfile             # Multi-stage production container with pre-cached models
├── .env.example           # Example configuration template with documented variables
└── .gitignore             # Git ignore file for secrets, DBs, and virtual environments
```

---

## ⚙️ Environment Variables

Create a `.env` file in the project root based on [`.env.example`](file:///.env.example):

| Variable | Required | Description | Example / Link |
| :--- | :---: | :--- | :--- |
| `GROQ_API_KEY` | **Yes** | Groq API Key for Whisper STT, LLaMA-4 Vision, & ChatGroq | [Groq Console](https://console.groq.com/keys) |
| `QDRANT_URL` | **Yes** | Qdrant Cluster Endpoint | `https://xxxx.gcp.cloud.qdrant.io:6333` |
| `QDRANT_API_KEY` | **Yes** | Qdrant API Key | [Qdrant Cloud](https://cloud.qdrant.io/) |
| `NEON_AUTH_BASE_URL` | **Yes** | Base URL for Neon Auth endpoint | `https://your-auth.neon-auth.com` |
| `POSTGRES_URI` | Optional | Connection URI for persistent Postgres checkpointer (falls back to local SQLite if unset) | `postgresql://user:pass@host/db?sslmode=require` |
| `ALPHA_VANTAGE_API_KEY` | Optional | API Key for real-time stock quotes | [Alpha Vantage](https://www.alphavantage.co/support/#api-key) |
| `ALLOWED_EMAILS` | Optional | Comma-delimited list of whitelisted user emails | `user@example.com,admin@company.com` |
| `AGENT_VOICE` | Optional | Default Edge-TTS voice model | `en-IN-PrabhatNeural` (default) |
| `ALLOWED_ORIGINS` | Optional | Allowed CORS origins (comma-separated) | `*` |
| `PORT` | Optional | Port for the Uvicorn HTTP server | `8000` |

---

## 🚀 Getting Started

### 1. Clone & Set Up Environment

```bash
# Clone the repository
git clone https://github.com/AbhishekAnand2894/voice-agent.git
cd voice-agent

# Create and activate a Python virtual environment
python3 -m venv .venv
source .venv/bin/activate  # On Windows: .venv\Scripts\activate

# Install dependencies
pip install --upgrade pip
pip install -r requirements.txt
```

### 2. Configure Environment

Copy the example configuration and fill in your keys:

```bash
cp .env.example .env
```

### 3. Run Locally

Start the FastAPI server using Uvicorn:

```bash
uvicorn server:app --reload --port 8000
```

The API will be accessible at `http://localhost:8000` (interactive API documentation available at `http://localhost:8000/docs`).

### 4. Access the Dashboard

Open `index.html` in your web browser, or serve it using any static server:

```bash
python3 -m http.server 3000
```
Navigate to `http://localhost:3000/index.html` to log in and start talking to Vikas.

---

## 🐳 Docker & Deployment

### Running with Docker

The included [Dockerfile](file:///Dockerfile) pre-downloads all FastEmbed models during build time and runs with `HF_HUB_OFFLINE=1` for rapid, zero-lag container startup:

```bash
# Build the Docker image
docker build -t vikas-voice-agent .

# Run the container with environment variables
docker run -p 8000:8000 --env-file .env vikas-voice-agent
```

### Deploying to Render

1. Create a **New Web Service** on Render and link this repository.
2. Choose **Docker** as the runtime.
3. In the **Environment Variables** tab, add the required variables listed in the [Environment Variables](#-environment-variables) section.
4. Set the port to `8000` (or leave Render's default `$PORT`).
5. Deploy service.

---

## 📡 API Reference

### Audio & Conversation Endpoints

| Method | Endpoint | Description |
| :--- | :--- | :--- |
| `POST` | `/turn?call_id={id}` | Streams an NDJSON response containing transcripts, sentence breakdowns, and Base64-encoded audio for a given voice turn. |
| `POST` | `/chat` | Standard REST chat endpoint returning JSON reply and optional base64 audio. |
| `POST` | `/interrupt` | Notifies the backend that the user has interrupted playback. |
| `POST` | `/ingest` | Uploads a PDF file, parses text/images, and indexes chunks into Qdrant for RAG. |

### Utilities & Auth Endpoints

| Method | Endpoint | Description |
| :--- | :--- | :--- |
| `GET` | `/me` | Validates Bearer token and returns authenticated user claims. |
| `GET` | `/config` | Returns public configuration (e.g. `neon_auth_base_url`) needed by the frontend. |
| `GET` | `/voices` | Returns a list of all available Edge-TTS neural voices. |
| `GET` | `/health` | Service health status check. |

---

## 🎤 Available TTS Voices

Vikas supports multiple regional and international voices out of the box via Microsoft Edge Neural TTS:

- **English (India)**: `en-IN-PrabhatNeural`, `en-IN-NeerjaNeural`
- **Hindi (India)**: `hi-IN-SwaraNeural`, `hi-IN-MadhurNeural`
- **Marathi (India)**: `mr-IN-AarohiNeural`, `mr-IN-ManoharNeural`
- **English (US/UK)**: `en-US-AriaNeural`, `en-US-GuyNeural`, `en-GB-RyanNeural`, `en-GB-SoniaNeural`
- **Japanese**: `ja-JP-NanamiNeural`, `ja-JP-KeitaNeural`

---

## 📄 License

This project is licensed under the MIT License - see the [LICENSE](LICENSE) file for details.

---

<div align="center">
  <sub>Built with ❤️ by Abhishek Anand</sub>
</div>
