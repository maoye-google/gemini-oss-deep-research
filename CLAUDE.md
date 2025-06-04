# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Development Commands

### Quick Start
- `make dev` - Start both frontend and backend development servers concurrently
- `make dev-frontend` - Start only frontend (Vite) on http://localhost:5173
- `make dev-backend` - Start only backend (LangGraph) on http://localhost:2024

### Frontend (React + Vite + TypeScript)
Navigate to `frontend/` directory:
- `npm run dev` - Development server with hot reload
- `npm run build` - Build for production (runs TypeScript check first)
- `npm run lint` - Run ESLint
- `npm run preview` - Preview production build

### Backend (LangGraph + Python)
Navigate to `backend/` directory:
- `langgraph dev` - Development server with auto-reload and LangGraph UI
- `make test` - Run pytest unit tests
- `make test_watch` - Run tests in watch mode
- `make lint` - Run ruff linting and mypy type checking
- `make format` - Auto-format code with ruff

### Installation
- Backend: `cd backend && pip install .` (or use uv)
- Frontend: `cd frontend && npm install`

## Architecture Overview

This is a fullstack research agent application with a React frontend and LangGraph-powered backend that performs iterative web research using Google Gemini models.

### Backend Architecture (LangGraph Agent)
The core is a stateful research agent in `backend/src/agent/graph.py` that follows this flow:
1. **Query Generation**: Generates initial search queries from user input
2. **Web Research**: Uses Google Search API to find relevant sources  
3. **Reflection**: Analyzes results to identify knowledge gaps
4. **Iterative Refinement**: Generates follow-up queries if needed (max configurable loops)
5. **Answer Synthesis**: Creates final answer with citations

Key components:
- `state.py` - Defines TypedDict state schemas for the graph nodes
- `tools_and_schemas.py` - Pydantic models for structured outputs
- `prompts.py` - System prompts for different agent roles
- `configuration.py` - Graph configuration and parameters
- `app.py` - FastAPI application that serves the graph

### Frontend Architecture (React + LangGraph SDK)
- Uses `@langchain/langgraph-sdk/react` for streaming communication with backend
- Real-time activity timeline showing agent's research progress
- Components follow Shadcn UI patterns with Tailwind CSS
- Key files: `App.tsx` (main state), `ActivityTimeline.tsx` (progress), `ChatMessagesView.tsx` (results)

### State Management
The LangGraph agent maintains state through:
- `OverallState`: Main state with messages, search results, loop counts
- `ReflectionState`: Tracks knowledge gaps and follow-up queries  
- `WebSearchState`: Individual search query state
- State is persisted in Postgres (production) or in-memory (development)

## Environment Setup

Backend requires `GEMINI_API_KEY` in `backend/.env` (copy from `backend/.env.example`).

Frontend automatically connects to:
- Development: http://localhost:2024 
- Production: http://localhost:8123

## Production Deployment

Uses Docker multi-stage build:
1. Builds optimized React frontend 
2. Copies frontend build to LangGraph API container
3. Requires Redis (pub-sub) and Postgres (state persistence)
4. Run with: `docker-compose up` (needs GEMINI_API_KEY and LANGSMITH_API_KEY)