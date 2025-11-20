# **Life Insurance Support Assistant \- AI Agent Instructions**

This file provides context, guidelines, and instructions for AI coding agents working on the Life Insurance Support Assistant project.

## **1\. Project Context**

Objective: Build a "Context-Aware Policy Advisor" that assists users with life insurance inquiries.  
Core Logic: The system uses a RAG (Retrieval-Augmented Generation) pipeline orchestrated by LangGraph. It retrieves policy details from a local ChromaDB vector store populated with synthetic markdown documents.  
Key Use Case: The demo focuses on a "Nervous Father" persona asking about Term vs. Whole Life, Education funds, and specific exclusions (e.g., Skydiving).

## **2\. Tech Stack & Constraints**

### **Backend (API & Logic)**

* **Language:** Python 3.10+  
* **Package Manager:** uv (strictly enforced). Do not use pip directly.  
* **Framework:** FastAPI (Async) \- *Required*.  
* **AI Orchestration:** **LangGraph** (StateGraph) & **LangChain**.  
* **LLM Access:** Use \`from langchain.chat\_models import init\_chat\_model\` for provider agnosticism (default: OpenAI).  
* **Vector DB:** ChromaDB (Local persistence).

### **Frontend (UI & Demo)**

* **Framework:** **Chainlit** (Python).  
* **Reasoning:** chosen for its built-in support for "Chain of Thought" visualization and easy audio/voice integration.  
* **Visualization:** Must use Chainlit's "Steps" feature to visualize the RAG retrieval and Grading process.

### **DevOps**

* **Containerization:** Docker & Docker Compose.  
* **CI/CD:** GitHub Actions (linting & testing).

## **3\. Project Structure**

Maintain this folder structure when generating or modifying files:

life-insurance-agent/  
├── backend/  
│   ├── app/  
│   │   ├── agent/           \# LangGraph Logic (Shared by API and UI)  
│   │   │   ├── graph.py     \# StateGraph definition  
│   │   │   └── nodes.py     \# Retrieve, Grade, Generate functions  
│   │   ├── core/            \# Config & Env vars  
│   │   ├── db/              \# Vector store connection  
│   │   ├── models/          \# Pydantic schemas  
│   │   └── main.py          \# FastAPI entry point (for API requirement)  
│   ├── data/                \# Synthetic markdown policies (Source of Truth)  
│   ├── ingest.py            \# RAG Ingestion script  
│   └── pyproject.toml       \# Managed by uv  
├── frontend/  
│   └── app.py               \# Chainlit Application  
├── .env  
└── docker-compose.yml

## **4\. Setup & Run Commands**

### **Environment Setup**

* **Install Dependencies:** uv sync (Ensure chainlit is added: uv add chainlit)  
* **Ingest Data:** uv run python backend/ingest.py

### **Running the App**

* **Run API (Backend):** uv run uvicorn backend.app.main:app \--reload \--port 8000  
* **Run UI (Frontend):** uv run chainlit run frontend/app.py \-w

## **5\. Coding Standards & Guidelines**

### **General**

* **Type Hinting:** All Python code must use strict type hints.  
* **Async/Await:** Use asynchronous functions for all I/O bound operations.  
* **Environment Variables:** Load from .env using pydantic-settings.

### **LangGraph & AI Logic**

* **State Definition:** Define AgentState (TypedDict) clearly.  
* **Graph Import:** The Chainlit app should import the compiled graph (app) directly from backend.app.agent.graph to enable step streaming.  
* **Prompting:** Use System Prompts to enforce the "Helpful Insurance Advisor" persona.

### **RAG Pipeline (Important)**

* **Retrieval:** Use similarity\_search with k=3.  
* **Grading:** Verify document relevance before generation.  
* **Citations:** Format answers to include source references (e.g., \[Source: Policy\_A.md\]).

### **Chainlit UI Guidelines**

* **Step Visualization:** Use cl.Step to show the internal reasoning of the agent.  
  * Wrap the Retrieval node in a step named "Retrieving Policy Docs".  
  * Wrap the Grading node in a step named "Checking Relevance".  
* **Streaming:** Stream the final LLM response token-by-token using cl.Message(content="").send() and .stream\_token().  
* **Audio:** Enable Chainlit's audio features for the Voice Agent demo requirement.

## **6\. Data Handling**

* **Synthetic Data:** The backend/data/ directory contains the source of truth.  
* **Ingestion:** Run ingest.py whenever data is modified to update the local ChromaDB.

## **7\. Testing Instructions**

* **Unit Tests:** uv run pytest tests/  
* **Manual UI Test:** Verify that the "Thinking..." steps appear in Chainlit before the final answer is streamed.

## **8\. Commit Guidelines**

Follow the Conventional Commits specification:

* feat: New features  
* fix: Bug fixes  
* docs: Documentation changes  
* refactor: Code restructuring  
* chore: Config changes

## **9\. Security**

* Ensure OPENAI\_API\_KEY is never committed.  
* Do not log PII (Personally Identifiable Information) in the console.
