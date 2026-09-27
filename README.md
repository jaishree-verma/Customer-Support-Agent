# Customer Support Agent

Built a multi-agent retrieval-augmented generation system for enterprise customer support.

- Live Web Application: https://customersupportagent-ten.vercel.app/
- Live Backend API: https://customer-support-agent-backend-p97d.onrender.com/
- API Documentation: https://customer-support-agent-backend-p97d.onrender.com/docs

---

## Problem Statement

Standard linear Retrieval-Augmented Generation (RAG) architectures suffer from significant operational limitations when deployed in enterprise customer support environments:

1. Single-Pass Query Failures: Naive RAG pipelines execute a single vector lookup for complex user prompts. If the initial search query misses key terms, the retriever fetches irrelevant document chunks, leading to incomplete or incorrect answers.
2. Hallucinations Under Low Context: When retrieved contexts lack necessary details, standard LLM pipelines attempt to fill gaps by generating plausible but unverified statements (hallucinations), exposing enterprises to compliance and brand risks.
3. Lack of Automated Evaluation: Linear RAG systems operate without feedback loops. There is no automated mechanism to check if retrieved documents contain sufficient facts to answer the user request before synthesizing a response.

### Scenario Example

A customer asks a complex support query: "How do I troubleshoot Wi-Fi connection issues and check the circuit breaker for the QuantumFlow QF-2025?"

- Naive RAG Approach: Performs a single vector search on the combined query. It fetches a generic Wi-Fi user manual page while omitting power circuit breaker details. The LLM attempts to guess power troubleshooting steps, leading to incorrect instructions.
- Multi-Agent Approach: The Router decomposes the query into targeted sub-queries ("QuantumFlow QF-2025 Wi-Fi troubleshooting" and "QuantumFlow QF-2025 circuit breaker power checks"). The Retriever fetches chunks for each sub-query. The Evaluator detects that the power context is initially missing and triggers a self-correction loop to re-fetch document chunks. Once the Evaluator approves sufficiency, the Response Generator synthesizes a verified answer with citations.

---

## Solution & Approach

Customer Support Agent replaces linear RAG with a stateful multi-agent network built on LangGraph and Google Gemini. The system introduces self-correcting feedback loops, multi-step query decomposition, and a modern streaming web interface.

### Execution Flow & Workflow Phases

1. Phase 1: Knowledge Base Construction
   - Document Loading: Supports PDF, Markdown, and Text documents.
   - Text Chunking: Splits documents into manageable segments using RecursiveCharacterTextSplitter.
   - Embedding Generation: Converts text chunks into vector embeddings using GoogleGenerativeAIEmbeddings.
   - Vector Store Persistence: Stores chunks and embeddings in a local persistent ChromaDB instance.
2. Phase 2: Multi-Step Query Decomposition & Research Orchestration
   - Router Agent (Supervisor): Central orchestrator directing state flow between agent nodes based on next_agent_to_call.
   - Research Agent: Breaks complex queries into ordered, focused sub-queries and manages research progression.
3. Phase 3: Intelligent Retrieval & Self-Correction
   - Retriever Agent: Fetches relevant document chunks from ChromaDB (with keyless local search fallback).
   - Evaluator Agent: Assesses the sufficiency and relevance of retrieved chunks for the current sub-query.
   - Self-Correction Loop: If retrieved information is deemed insufficient, the Evaluator provides feedback and loops back to the Retriever (up to MAX_RETRIEVAL_ATTEMPTS).
4. Phase 4: Answer Synthesis & Refinement
   - Synthesizer Agent: Combines all verified document chunks into a comprehensive draft answer.
   - Formatter Agent: Polishes the draft answer for clarity, conciseness, grammar, and professional presentation.

---

## Why It Is Important & Key Benefits

Enterprise automated support requires strict factual grounding, auditability, and predictability. Unchecked AI responses increase operational workload when users escalate failed automated interactions to human support teams.

Key benefits include:

- Reduction in Support Hallucinations: Evaluator gates ensure responses are synthesized exclusively from verified knowledge base contexts.
- Verifiable Citations and Grounding: Every answer includes source attribution cards referencing document names, sections, and relevance match scores.
- Automated Quality Checks: Built-in self-correction loops retry retrieval before delivering answers to users.
- Lower Escalation Costs: Resolving complex multi-part queries automatically reduces support ticket volume and human agent intervention costs.

---
## Architecture 

![RAG Agent Architecture Diagram](images/output.png)

## System Architecture

The workflow is managed as a stateful graph where each agent acts as a specialized node operating on a central `AgentState`.

```
                        +----------------------------+
                        |         User Query         |
                        +----------------------------+
                                      |
                                      v
                        +----------------------------+
                        |      Router Agent          |
                        |      (Supervisor)          |
                        +----------------------------+
                          /           |            \
                         /            |             \
                        v             v              v
           +-----------------+  +-----------------+  +------------------+
           | Research Agent  |  | Retriever Agent |  | Evaluator Agent  |
           | (Decomposition) |  |   (ChromaDB)    |  |  (Sufficiency)   |
           +-----------------+  +-----------------+  +------------------+
                        \             |              /
                         \            |             /
                          v           v            v
                        +----------------------------+
                        |  Self-Correction Evaluator  |
                        |     Feedback Loop Loop     |
                        +----------------------------+
                                      |
                                      v
                        +----------------------------+
                        |  Synthesizer & Formatter   |
                        |     (Response Generator)   |
                        +----------------------------+
                                      |
                                      v
                        +----------------------------+
                        | Grounded Final Answer + SSE|
                        +----------------------------+
```

### Component Responsibilities

- Router Agent (Supervisor): Central orchestrator reading state signals (`next_agent_to_call`) to direct execution to downstream nodes or signal workflow completion.
- Research Agent: Decomposes complex user queries into an ordered list of clear sub-queries using Gemini.
- Retriever Agent: Queries ChromaDB vector stores or local document stores to extract relevant document chunks for each sub-query.
- Evaluator Agent: Performs LLM-assisted context verification to approve or reject retrieved document chunks, managing retry thresholds.
- Response Generator (Synthesizer & Formatter): Combines all verified document chunks into a clear, formatted customer support report with source attributions.

---

## Tech Stack

- Multi-Agent Framework: LangGraph, LangChain Core, LangChain Community
- Language: Python 3.10+
- Frontend: Next.js 15 (App Router), React 19, Tailwind CSS, Framer Motion, Zustand, Lucide Icons
- API / Transport: FastAPI, Server-Sent Events (SSE), REST
- Vector Database: ChromaDB, LangChain Chroma, Local Document Fallback Search
- LLM Provider: Google Gemini (Gemini 2.0 Flash / Gemini 2.5)

---

## Setup and Installation

### Prerequisites

- Python 3.10 or higher
- Node.js 18.0 or higher
- Google Gemini API Key

### 1. Clone the Repository

```bash
git clone https://github.com/jaishree-verma/-Customer-Support-Agent.git
cd -Customer-Support-Agent
```

### 2. Configure Environment Variables

Create a `.env` file in the project root directory:

```env
GEMINI_API_KEY="YOUR_GOOGLE_GEMINI_API_KEY"
GOOGLE_API_KEY="YOUR_GOOGLE_GEMINI_API_KEY"
```

### 3. Install Python Dependencies

```bash
python -m pip install -r requirements.txt
```

### 4. Build Knowledge Base Index

Place your support documents (`.txt`, `.md`, `.pdf`) in the `data/` directory and run:

```bash
python ingest.py
```

This processes raw documents, generates embeddings, and creates the persistent ChromaDB vector store.

### 5. Start Backend FastAPI Server

```bash
python -m uvicorn app.main:app --reload --port 8000
```

The REST and SSE API server will run on `http://127.0.0.1:8000`.

### 6. Install and Start Frontend Web Portal

In a new terminal window:

```bash
cd frontend
npm install
npm run dev
```

Open `http://localhost:3000` in your web browser.

---

## Repository Structure

```
.
├── .env                              (Environment variables and Gemini API Key)
├── requirements.txt                  (Python dependencies)
├── ingest.py                         (Builds ChromaDB vector index from data/)
├── main.py                           (Console execution entry point)
├── app/
│   ├── main.py                       (FastAPI server with CORS & SSE endpoints)
│   └── schemas.py                    (Pydantic payload schemas)
├── data/                             (Raw support manuals and document files)
├── chroma_db/                        (ChromaDB vector store files)
├── frontend/                         (Next.js App Router web application)
│   ├── src/
│   │   ├── app/                      (App Router layout and pages)
│   │   ├── components/               (Navbar, AgentTracker, ChatInterface, SourceCard)
│   │   ├── store/                    (Zustand chat state management)
│   │   ├── lib/                      (SSE streaming client)
│   │   └── types/                    (TypeScript interface definitions)
│   ├── package.json
│   └── tailwind.config.js
├── notebooks/                        (Jupyter development notebooks)
└── src/
    ├── agents/                       (Router, Research, Retriever, Evaluator, Synthesizer, Formatter)
    ├── config/                       (Configuration manager)
    ├── constants/                    (System constants and attempt limits)
    ├── data_ingestion/               (Document loaders and chunking pipeline)
    ├── graph/                        (LangGraph workflow compilation)
    ├── llm_config/                   (Gemini model and embeddings setup)
    └── models/                       (AgentState TypedDict definition)
```

---

## Future Enhancements

- Human-in-the-Loop Escalation Workflows: Seamless handover to human support agents when queries exceed retrieval thresholds or request account modifications.
- Multi-Modal Query Support: Processing customer image uploads (error screenshots, hardware serial labels) alongside text queries.
- Automated Evaluation Benchmark Suites: Integration of RAGAS and TruLens benchmark metrics to track retrieval precision and hallucination rates.
- Multi-Tenant Vector Indexing: Tenant isolation and access control lists (ACLs) for enterprise multi-team document separation.
- Real-Time Token Streaming: Direct token-by-token streaming from Gemini models into the SSE event stream for lower initial token latency.
- External Tool Integration: Connecting web search APIs and ticketing systems (Zendesk, Jira) for out-of-knowledge-base queries.
