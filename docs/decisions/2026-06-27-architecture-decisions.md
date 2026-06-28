# Key Architectural Decisions Log (ADR)

This document tracks all critical, high-level technology and architectural decisions made for the development of **Sleeve**.

---

## ADR 1: Monorepo Subfolder Directory Layout
- **Decision Date:** June 27, 2026
- **Status:** DECIDED
- **Context:** We need a clean, structured repository for a decoupled Next.js + Python stack during a fast-paced hackathon.
- **Decision:** Use a monorepo containing `frontend/` (Next.js) and `backend/` (FastAPI) directories. 
- **Consequence:** Keeps dependencies perfectly isolated, virtual environments separate from node modules, and allows running parallel development terminals under a single git repository structure.

---

## ADR 2: FastAPI with SQLAlchemy ORM and Alembic
- **Decision Date:** June 27, 2026
- **Status:** DECIDED
- **Context:** Decoupled Python backends require structural database mapping and flexible migration controls as requirements adapt.
- **Decision:** Build the backend on FastAPI, using SQLAlchemy 2.0 ORM declarations for models, and Alembic for schema migrations.
- **Consequence:** Ensures strict type safety, clean data model schemas, and robust migration steps.

---

## ADR 3: Deterministic Financial Allocation Engine
- **Decision Date:** June 27, 2026
- **Status:** DECIDED
- **Context:** We need a paycheck allocation engine. Relying on Large Language Models (LLMs) for raw math can result in hallucinations, precision errors, and unpredictable logic.
- **Decision:** Keep the core business calculations 100% deterministic, implemented in standard, auditable Python service code.
- **Consequence:** AI remains strictly an advisory and text-generating wrapper, ensuring financial transactions are safe, reliable, and predictable.

---

## ADR 4: Conversational "Deep Agent" Orchestration via LangGraph
- **Decision Date:** June 27, 2026
- **Status:** DECIDED
- **Context:** Rather than complex form-based UIs, we want an interface where users can describe income, edit values, and check off items in natural language. The AI needs a way to plan long-running stateful tasks (like paycheck entries to goal splits to transfer checklists) without losing track or executing incorrect calculations.
- **Decision:** Build Sleeve's conversational engine as an autonomous **Deep Agent** using LangChain's `deepagents` harness on the **LangGraph** runtime. 
- **Consequence:** Spawns a stateful ReAct agent loop that uses built-in task checklists (`write_todos`), filesystems, and custom Python database tools to process calculations deterministically. Gives the user a highly fluid, intelligent, conversational CFO experience.

---

## ADR 5: Rich Interactive Component Rendering (Markdown + JSON Blocks)
- **Decision Date:** June 27, 2026
- **Status:** DECIDED
- **Context:** Text-only chatbots are difficult for reviewing detailed tables, checklist items, or charts. We need a way to combine the speed of conversational chat with the structure of a rich web dashboard.
- **Decision:** When our Deep Agent calls a database tool (like `get_allocation_plan` or `get_checklist`), the backend returns both natural-language reasoning and a structured JSON payload. The Next.js frontend detects these payloads (e.g. wrapped in custom XML-like tags like `<allocation-card>` or `<checklist-card>`) and renders them as beautiful, interactive React components (like graphs, editable tables, or checklist boxes) right inside the chat window.
- **Consequence:** Achieves the ultimate user experience: natural language input with a visual, responsive interface that updates in real-time.
