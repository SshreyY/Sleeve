# Sleeve — Personal Paycheck Allocation Planner
## Technical Design Specification

- **Date:** June 27, 2026
- **Status:** APPROVED / READY FOR IMPLEMENTATION
- **Product:** Sleeve
- **Slogan:** "Protect what you receive, and build what you believe."
- **Stack:** Decoupled Monorepo (Next.js + Python FastAPI + local PostgreSQL + SQLAlchemy + Alembic + LangChain `deepagents` on LangGraph + Gemini 3.5 Flash)

---

## 1. System Architecture

Sleeve is an AI-first, conversational personal finance application. Rather than filling out forms, the user manages their entire paycheck allocation flow through standard natural language chat.

The system is constructed as a decoupled monorepo. The Python FastAPI backend hosts the database, runs the deterministic financial calculations, and compiles our **Conversational Deep Agent** using the `deepagents` harness on **LangGraph**. The Next.js frontend delivers a modern, conversational UI capable of rendering rich, interactive components (like graphs, editable tables, or checklists) whenever the agent returns structured financial payloads.

```text
Sleeve/
├── backend/                  # Python FastAPI Backend (Host for API and Agent)
│   ├── .venv/                # Python Virtual Environment
│   ├── requirements.txt      # Python dependencies (includes deepagents, fastapi)
│   ├── run.py                # Backend startup script
│   ├── alembic.ini           # Database migration configuration
│   ├── alembic/              # Database migration history
│   └── app/
│       ├── __init__.py
│       ├── main.py           # FastAPI server entry point
│       ├── config.py         # Pydantic Settings env configuration
│       ├── database.py       # SQLAlchemy engine & SessionLocal session manager
│       ├── models.py         # SQLAlchemy DB models
│       ├── schemas.py        # Pydantic schemas (Request/Response validation)
│       ├── routers/          # API route handlers
│       │   ├── chat.py       # Streaming chat endpoint for the Deep Agent
│       │   └── health.py
│       └── services/         # Core business logic
│           ├── paycheck.py   # Paycheck CRUD & calculations
│           ├── allocation.py # Deterministic paycheck allocation engine
│           ├── goal.py       # Goal creation & progress calculation
│           └── ai.py         # Compiled Deep Agent & custom tool bindings
│
├── frontend/                 # Next.js Frontend
│   ├── package.json          # Node dependencies
│   ├── tsconfig.json         # TypeScript configurations
│   ├── tailwind.config.js    # Tailwind styling configurations
│   └── src/
│       ├── app/              # App router (pages, layouts)
│       │   ├── page.tsx      # Main Conversational Chat Dashboard
│       │   └── layout.tsx
│       ├── components/       # UI Components (chat-box, rich component cards)
│       │   ├── ChatInterface.tsx  # Dynamic streaming chat component
│       │   ├── AllocationTable.tsx # Interactive editable plan table
│       │   ├── ChecklistCard.tsx   # Interactive checkbox checklist
│       │   └── GoalOverview.tsx    # Visual goal progress graphs
│       └── lib/              # API Client & helper functions
│
└── docs/                     # Documentation Folder
    ├── plans/                # Step-by-step implementation plans
    ├── handoffs/             # Handoff notes for starting new agent chats
    ├── decisions/            # Log of architectural decisions
    ├── prs/                  # PR description templates
    └── superpowers/specs/    # System specs
```

---

## 2. Relational Database Schema

All decimal values use high-precision fields (e.g., `Numeric(12, 2)` or direct `Numeric` types mapping to Python's `Decimal`) to avoid floating-point calculations errors.

```sql
-- Enums
CREATE TYPE pay_frequency_enum AS ENUM ('weekly', 'biweekly', 'semimonthly', 'monthly');
CREATE TYPE goal_type_enum AS ENUM ('emergency_fund', 'roth_ira', 'brokerage', 'short_term_savings', 'debt', 'custom');
CREATE TYPE plan_status_enum AS ENUM ('draft', 'approved', 'completed');
CREATE TYPE allocation_category_enum AS ENUM ('needs', 'hysa', 'roth_ira', 'brokerage', 'savings_goal', 'flexible', 'debt');
CREATE TYPE checklist_status_enum AS ENUM ('todo', 'done', 'skipped');
```

### SQLAlchemy Models

#### `UserSettings` (Global Preferences - Single Record)
- `id`: Integer (Primary Key)
- `pay_frequency`: Enum (`weekly`, `biweekly`, `semimonthly`, `monthly`)
- `checking_buffer`: Numeric(12, 2) (Checking account cushion)
- `monthly_expenses`: Numeric(12, 2) (Average monthly needs)
- `emergency_fund_target_months`: Integer (e.g., 12)
- Default percentages for allocations:
  - `default_needs_percent`, `default_hysa_percent`, `default_roth_ira_percent`, `default_brokerage_percent`, `default_savings_goals_percent`, `default_flexible_percent`
- Targets:
  - `roth_ira_annual_target`: Numeric(12, 2)
  - `brokerage_monthly_target`: Numeric(12, 2)

#### `Paycheck`
- `id`: UUID (Primary Key)
- `pay_date`: Date
- `pay_period_start` / `pay_period_end`: Date
- `employer_name`: String
- `gross_pay` / `net_deposit` / `taxes_withheld`: Numeric(12, 2)
- Pre-deposit savings:
  - `contribution_401k` / `contribution_hsa` / `other_payroll_savings`: Numeric(12, 2)
- `notes`: Text
- `created_at` / `updated_at`: DateTime

#### `Goal`
- `id`: UUID (Primary Key)
- `name`: String
- `type`: Enum (`emergency_fund`, `roth_ira`, `brokerage`, `short_term_savings`, `debt`, `custom`)
- `target_amount`: Numeric(12, 2) (Optional)
- `current_amount`: Numeric(12, 2) (Current tracking balance)
- `target_date`: Date
- `priority`: Integer (Priority sort)
- `is_active`: Boolean

#### `AllocationPlan`
- `id`: UUID (Primary Key)
- `paycheck_id`: UUID (Foreign Key to `Paycheck`, 1:1)
- `status`: Enum (`draft`, `approved`, `completed`)
- Totals:
  - `net_deposit` / `total_allocated` / `unallocated_amount`: Numeric(12, 2)
  - `pre_deposit_savings` / `post_deposit_savings` / `total_savings`: Numeric(12, 2)
  - `total_savings_rate`: Numeric(5, 2)
- `warnings`: JSON (Array of Warning strings)
- `explanation`: Text (Gemini structured explanation summary)
- `created_at` / `updated_at`: DateTime

#### `AllocationItem`
- `id`: UUID (Primary Key)
- `allocation_plan_id`: UUID (Foreign Key to `AllocationPlan`)
- `category`: Enum (`needs`, `hysa`, `roth_ira`, `brokerage`, `savings_goal`, `flexible`, `debt`)
- `goal_id`: UUID (Optional Foreign Key to `Goal`)
- `label`: String (e.g., "Transfer to High-Yield Savings")
- `amount`: Numeric(12, 2)
- `reason`: String
- `editable`: Boolean (Flexible remainder is false, others true)
- `destination`: String (Account name or routing destination)
- `order`: Integer

#### `ChecklistItem`
- `id`: UUID (Primary Key)
- `allocation_plan_id`: UUID (Foreign Key to `AllocationPlan`)
- `label`: String
- `amount`: Numeric(12, 2)
- `destination`: String
- `status`: Enum (`todo`, `done`, `skipped`)
- `completed_at`: DateTime

---

## 3. The Deterministic Allocation Engine

Whenever the user inputs a paycheck, our backend runs a deterministic Python script. It calculates rates and reserves, processes cash splits, generates warnings, and saves the output.

```text
               Available Cash = Paycheck.net_deposit
                                 │
                                 ▼
                     ┌───────────────────────┐
                     │ 1. Needs/Bills        │ -> Deduct max(default_needs_%, monthly_expenses/2)
                     └───────────┬───────────┘
                                 ▼
                     ┌───────────────────────┐
                     │ 2. Checking Buffer    │ -> Warn if Available Cash falls below buffer
                     └───────────┬───────────┘
                                 ▼
                     ┌───────────────────────┐
                     │ 3. HYSA Emergency     │ -> Deduct default % capped at remaining gap
                     └───────────┬───────────┘
                                 ▼
                     ┌───────────────────────┐
                     │ 4. Roth IRA Pace      │ -> suggested_per_paycheck = gap / remaining_periods
                     └───────────┬───────────┘
                                 ▼
                     ┌───────────────────────┐
                     │ 5. Short-Term Goals   │ -> Loop through goals by priority order
                     └───────────┬───────────┘
                                 ▼
                     ┌───────────────────────┐
                     │ 6. Brokerage Target   │ -> Allocate remaining investment targets
                     └───────────┬───────────┘
                                 ▼
                     ┌───────────────────────┐
                     │ 7. Flexible Spending  │ -> Leftover cash assigned here
                     └───────────────────────┘
```

The algorithm returns an `AllocationPlan` with calculated percentages:
- `tax_rate = taxes_withheld / gross_pay`
- `pre_deposit_savings_rate = (401k + HSA + other_payroll) / gross_pay`
- `total_savings_rate = (pre_deposit_savings + post_deposit_savings) / gross_pay`

---

## 4. Deep Agent Tool Specifications

To orchestrate these database operations and calculations, the **Sleeve Deep Agent** is bound to a collection of specialized Python tools. The agent invokes these tools in response to NLP inputs.

### 1. `get_goals_summary()`
- **Args:** None
- **Behavior:** Fetches all active goals, calculates targets, gaps, and progress percentages.
- **Output:** Returns JSON summary. The frontend intercepts this to display a visual chart.

### 2. `create_or_update_goal(name, type, target_amount, current_amount, priority, target_date)`
- **Args:** Attributes of a `Goal`.
- **Behavior:** Creates a new goal or updates an existing one.

### 3. `register_paycheck_and_allocate(gross_pay, net_deposit, taxes_withheld, contribution_401k, contribution_hsa, other_payroll_savings, pay_date, notes)`
- **Args:** Paycheck attributes.
- **Behavior:** Creates a `Paycheck` record, runs the **Deterministic Allocation Engine**, creates a draft `AllocationPlan`, and saves it to the database.
- **Output:** Returns the full JSON payload of the paycheck and allocation items. The frontend intercepts this to render an **interactive editable table** in the chat window.

### 4. `modify_allocation_item_amount(plan_id, item_id, amount)`
- **Args:** Plan ID, Item ID, and new target amount.
- **Behavior:** Updates a specific item's amount, recalculates the remaining available cash, updates the `flexible` remainder, refreshes all total metrics (such as the total savings rate), regenerates active warnings, and saves.
- **Output:** Returns the revised plan JSON.

### 5. `approve_allocation_plan(plan_id)`
- **Args:** Plan ID.
- **Behavior:** Moves plan status to `approved`.
- **Output:** Generates a set of `ChecklistItems` corresponding to each non-discretionary allocation item and returns them. The frontend intercepts this to render the Checklist component.

### 6. `complete_allocation_plan(plan_id)`
- **Args:** Plan ID.
- **Behavior:** Transitions plan status to `completed`. For each allocation item tied to a `Goal`, it adds the allocated amount to the goal's `current_amount` balance. 

### 7. `toggle_checklist_item_status(item_id, status)`
- **Args:** Item ID, new status (`todo`, `done`, `skipped`).
- **Behavior:** Updates the checklist item status.

---

## 5. Conversational UI Component Interception

To prevent text clutter, when the backend streams a tool execution payload, it wraps the structured JSON output in custom XML tags. The Next.js frontend parses these tags during streaming and renders interactive React components inline:

### 1. Goals Overview Card
- **Trigger Tag:** `<goals-card>{JSON}</goals-card>`
- **Component:** Displays visual progress rings and progress bars for HYSA, Roth, and savings goals.

### 2. Allocation Table Card
- **Trigger Tag:** `<allocation-card plan_id="..." status="draft">{JSON}</allocation-card>`
- **Component:** Renders a clean table showing Gross, Net, Pre-deposit Savings, and individual Allocation Items.
- **Interactivity:** If the plan is in `draft` status, the user can type values directly into the input fields in the table. When the user changes a value, the frontend makes a background API call to `modify_allocation_item_amount` which returns the recalculated plan, updating the table and the total savings rate instantly.

### 3. Checklist Card
- **Trigger Tag:** `<checklist-card plan_id="..."> {JSON} </checklist-card>`
- **Component:** Displays a modern todo checklist.
- **Interactivity:** User can check boxes directly. The frontend triggers `toggle_checklist_item_status` in the backend to mark things complete.

---

## 6. Phase 1 Verification Checklist & Plan

We measure Phase 1 success by confirming:
1. User can say: *"I got paid $4,500 with $900 taxes and $450 in my 401k."* 
   - **Result:** Deep Agent plans the task, calls `register_paycheck_and_allocate`, runs the calculation, and renders the dynamic **Allocation Table Card**.
2. User can edit an amount in the Table Card.
   - **Result:** Table updates immediately, recalculates savings rate, and shows any warnings.
3. User says: *"Looks good, approve it."*
   - **Result:** Deep Agent plans, calls `approve_allocation_plan`, and renders the interactive **Checklist Card**.
4. User checks off a box.
   - **Result:** Checklist status is saved in PostgreSQL database.
