# Chat Handoff Guides

This folder is used to maintain context files for starting new AI coder chats. Whenever you switch context, export a brief status handoff file here.

---

## Active Session Status Handoff

*This section will be updated by the agent at the end of each major session to ensure smooth continuation.*

### Current State
- Monorepo folder layout decided: `Sleeve/frontend` and `Sleeve/backend`.
- Python virtual environment created in `backend/.venv`.
- Core Technical Design Specification drafted and approved: `docs/superpowers/specs/2026-06-27-sleeve-design.md` outlining the conversational **Sleeve Deep Agent** built on LangChain's `deepagents` harness and LangGraph.
- Architectural Decisions Records updated: `docs/decisions/2026-06-27-architecture-decisions.md` documenting monorepo setup, FastAPI, deterministic allocation math, and rich interactive components.

### Next Steps
1. Create step-by-step implementation plan using the `writing-plans` skill.
2. Initialize Backend (`requirements.txt`, basic FastAPI setup, SQLAlchemy + Alembic, and `deepagents` loop).
3. Initialize Frontend (Next.js with TypeScript, Tailwind, shadcn/ui).
4. Implement the deterministic paycheck allocation engine.
5. Create the Deep Agent tool schemas and wire up Gemini.
6. Create the frontend chat dashboard with interactive card interceptions.
