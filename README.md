# 🏙️ ShehriLink

### AI-powered municipal complaint resolution with closed-loop verification

**ShehriLink** is a web application that takes a citizen's complaint about a city problem (a broken streetlight, a pothole, uncollected garbage, a water leak) and handles it from start to finish with a team of cooperating AI agents. The system classifies the complaint, finds its location, checks for duplicates, sets a priority, picks the responsible department, and assigns a field team.

What sets it apart is **closed-loop verification**: a team reporting "fixed" is not enough to close a complaint. A complaint is closed only when independent evidence confirms the problem is solved. If the check fails, the system replans, reassigns the complaint to a different team and verifies again. After a configurable number of failed attempts, it escalates the complaint to a human supervisor instead of looping forever.

Citizens can report problems and follow progress live. Municipal officers get a dashboard, a complaints table, an escalation queue, and manual verification controls.

------------------------------------------------------------------------

## 👩‍💻 Our Team

Built with teamwork by:

1.  **Syeda Taqiya Noman**
2.  **Zainab Siddiq**
3.  **Bushra Zafar**
4.  **Jawerya Shafi**
5.  **Hashmat Ali**

------------------------------------------------------------------------

## ✨ Features

### For citizens

-   Report a city problem in plain language, with an optional location.
-   Watch the agents work in a live **Agent Activity Timeline**, step by step.
-   Track any complaint by its ID with a progress bar and the full history.

### For municipal officers (`/admin`)

-   **Dashboard** with summary cards and charts by category, priority, zone and day.
-   **Complaints table** with search, filters, sorting and pagination, plus a detail page for each complaint.
-   **Escalation queue** listing complaints that need a human.
-   **Teams page** showing field teams, their availability and workload.
-   **Manual verification:** an officer can mark a complaint *Fixed* (closes it) or *Not fixed* (reassigns it, or escalates after the retry limit).

### Across the application

-   **English and Urdu** interface with a full right-to-left (RTL) layout for Urdu.
-   **Dark mode** that follows the system setting and remembers your choice.
-   Responsive, mobile-friendly layout.
-   Works **without any API key**: a deterministic mock mode runs all agents with rules. Add a key to use OpenAI, Anthropic (Claude), Google Gemini, or any OpenAI-compatible server (Groq, OpenRouter, Ollama).

------------------------------------------------------------------------

## 🔄 How It Works

```
Complaint -> Classify -> Locate -> Duplicate check -> Priority -> Department -> Assign
          -> Verify resolution -> Close | Replan -> Reassign -> Verify ... | Escalate
```

The workflow is built with **LangGraph**. All agents share one state object and each appends its steps to a history, which the frontend shows as the timeline.

| # | Agent | What it does |
|---|---|---|
| 1 | Classification | Determines the category, subcategory and confidence |
| 2 | Location | Extracts the area and zone. It never invents a location: if none is found it returns `unknown` |
| 3 | History / Duplicate | Detects duplicate or related open complaints using text and location similarity |
| 4 | Priority | Assigns LOW, MEDIUM, HIGH or CRITICAL with a reason |
| 5 | Department | Routes the complaint to the responsible department |
| 6 | Assignment | Picks a team by department, zone, availability, workload and previous failures |
| 7 | Verification | Compares the team's report with independent evidence (citizen feedback) |
| - | Replan / Close / Escalate | Control steps that drive the verification loop |

**Design principle:** the language model is used only where language understanding is needed (classification, location extraction, ambiguous duplicates, adjusting priority, judging evidence). Everything that must be predictable, such as zone lookup, priority rules, department routing, team assignment and loop control, is deterministic code. If the model fails or no key is set, each agent falls back to rules and the workflow still completes.

### The verification loop

1.  The assigned team reports the fix.
2.  The Verification Agent checks it against independent evidence and decides *verified* or *failed*.
3.  On failure, the failed team is recorded and the Assignment Agent prefers a **different** team (same-zone team, then an all-zone rapid-response team, then other zones).
4.  When the number of failed verifications reaches `MAX_RETRIES` (default `3`), the complaint is **escalated** to a human. A hard recursion limit in LangGraph acts as a second safety net.

In integrated mode, a new complaint stays open ("Team assigned") until an officer verifies it from the admin page. For demos, you can also simulate outcomes automatically.

------------------------------------------------------------------------

## 🏗️ Architecture

ShehriLink has three parts that work together:

```
Frontend (React, :5173)  ->  Agent backend (LangGraph, :8000)  ->  Database backend (FastAPI + SQLite, :8001)
```

-   **Frontend** (`municipal_complaint_agent/frontend/`): the citizen and officer interface.
-   **Agent backend** (`municipal_complaint_agent/`): runs the LangGraph multi-agent workflow and exposes it through a FastAPI API.
-   **Database backend** (`backend/`): the system of record. It stores complaints, teams, technicians, assignments, verifications and activity logs, and enforces the complaint lifecycle with a state machine. A complaint can only be closed through a **passed** verification.

The agent backend talks to the database backend through `municipal_complaint_agent/app/data/http_repository.py`. Without `DB_BACKEND_URL`, the agents run alone on in-memory mock data.

------------------------------------------------------------------------

## 🧰 Technology Stack

### Frontend

-   **React 19** and **TypeScript**
-   **Vite** for development and builds
-   **Tailwind CSS** for styling
-   **React Router** for navigation
-   **Recharts** for charts
-   **lucide-react** for icons

### Agent backend

-   **Python** with **FastAPI** and **Uvicorn**
-   **LangGraph** and **LangChain Core** for the agent workflow
-   **Pydantic** for typed state and structured outputs
-   Optional LLM providers: OpenAI, Anthropic, Google Gemini, or any OpenAI-compatible endpoint

### Database backend

-   **FastAPI**, **SQLAlchemy** and **Pydantic**
-   **SQLite** by default, or **PostgreSQL** (for example Neon) through `DATABASE_URL`

### Testing and deployment

-   **pytest** for both backends
-   **Docker**, **Render**, **Netlify**, **Vercel** and **Hugging Face Spaces** configurations

------------------------------------------------------------------------

## 📁 Project Structure

```text
ShehriLink/
├── backend/                         # Database backend (FastAPI + SQLAlchemy)
│   ├── app/
│   │   ├── routers/                 # complaints, assignments, verifications, agent, dashboard, reference, demo
│   │   ├── services/                # lifecycle, workflow, workload, similarity
│   │   ├── seed/seed_data.py        # demo departments, zones, teams, technicians, complaints
│   │   ├── models.py  schemas.py  enums.py  database.py  config.py
│   │   └── main.py
│   ├── tests/
│   ├── municipal.db                 # demo data (SQLite)
│   └── requirements.txt
│
├── municipal_complaint_agent/       # Agent backend + frontend
│   ├── app/
│   │   ├── agents/                  # classification, location, history, priority, department,
│   │   │                            # assignment, verification, lifecycle
│   │   ├── graph/                   # LangGraph state, workflow and routing
│   │   ├── data/                    # mock data and the HTTP repository for the database backend
│   │   ├── models/  utils/  llm.py  config.py
│   │   └── main.py                  # FastAPI API
│   ├── frontend/                    # React + Vite + TypeScript app
│   │   └── src/
│   │       ├── pages/               # Landing, Submit, Track, and admin pages
│   │       ├── components/  lib/  data/
│   │       └── main.tsx  App.tsx
│   ├── tests/
│   ├── run.py                       # CLI: serve, demo, complaint
│   ├── requirements.txt
│   └── requirements-llm.txt         # optional LLM provider packages
│
├── deploy/                          # Hugging Face Space build files
├── Dockerfile                       # single-container build (frontend + both backends)
├── render.yaml                      # Render blueprint for the two backends
├── netlify.toml                     # Netlify config for the frontend
├── DEPLOY_VERCEL.md                 # step-by-step Vercel + Neon deployment guide
└── README.md
```

------------------------------------------------------------------------

## ⚙️ Getting Started

### Prerequisites

-   **Python** 3.11 or newer
-   **Node.js** 20 or newer, with npm

Run the three parts in separate terminals.

### 1. Database backend

```bash
cd backend
python -m venv venv
source venv/bin/activate            # Windows: venv\Scripts\activate
pip install -r requirements.txt
uvicorn app.main:app --port 8001
```

`municipal.db` already contains demo data. To reset it at any time:

```bash
python -m app.seed.seed_data --reset
```

Interactive API docs: http://localhost:8001/docs

### 2. Agent backend

```bash
cd municipal_complaint_agent
python -m venv venv
source venv/bin/activate            # Windows: venv\Scripts\activate
pip install -r requirements.txt
cp .env.example .env                # Windows: copy .env.example .env
python run.py
```

The API starts at http://localhost:8000 (docs at `/docs`). The default `.env` uses mock mode, so **no API key is needed**.

### 3. Frontend

```bash
cd municipal_complaint_agent/frontend
npm install
npm run dev
```

Open http://localhost:5173. The dev server forwards `/api/*` to the agent backend, so no CORS setup is needed. To point at a different backend, run `API_TARGET=http://localhost:9000 npm run dev`.

> The database backend must be running before you submit a complaint, otherwise the app reports that it cannot reach the database backend.

### Try the agents from the command line

```bash
cd municipal_complaint_agent
python run.py demo                                   # runs all five demo scenarios
python run.py demo failed_verification               # one scenario
python run.py complaint "Garbage not collected in Gulbahar" --scenario fail_twice
```

The built-in scenarios are `normal`, `high_priority`, `duplicate`, `failed_verification` and `max_retry`.

### Run everything in one container

```bash
docker build -t shehrilink .
docker run -p 7860:7860 shehrilink
```

Open http://localhost:7860. The container serves the built React app and the agent API together, with the database backend running inside.

------------------------------------------------------------------------

## 🔧 Configuration

### Agent backend (`municipal_complaint_agent/.env`)

| Variable | Description | Default |
|---|---|---|
| `LLM_PROVIDER` | `mock`, `openai`, `anthropic` or `google` | `mock` |
| `MODEL_NAME` | Model ID (empty uses the provider default) | empty |
| `API_KEY` | Provider API key | empty |
| `LLM_BASE_URL` | OpenAI-compatible endpoint (Groq, OpenRouter, Ollama) | empty |
| `LLM_TIMEOUT` | Seconds before an LLM call times out | `30` |
| `MAX_RETRIES` | Failed verifications before escalation | `3` |
| `VERIFICATION_SCENARIO` | `manual`, `success`, `fail_once`, `fail_twice`, `fail_<n>` or `always_fail` | `manual` |
| `DB_BACKEND_URL` | URL of the database backend. Leave empty for in-memory mock data | `http://localhost:8001` |
| `TEAM_CAPACITY` | Capacity assumed for every team when checking availability | `5` |

To use a real model, install the optional packages and set the variables:

```bash
pip install -r requirements.txt -r requirements-llm.txt
```

```env
LLM_PROVIDER=openai          # or anthropic, google
MODEL_NAME=your-model-id
API_KEY=your_api_key
```

Keep `MAX_RETRIES` equal to `MAX_REOPENS` in the database backend so both sides escalate at the same point.

### Database backend (environment variables)

| Variable | Description | Default |
|---|---|---|
| `DATABASE_URL` | SQLite file by default; set a PostgreSQL URL for hosted databases | local `municipal.db` |
| `MAX_REOPENS` | Reopens before a complaint is escalated | `3` |
| `AUTO_SEED` | Load demo data the first time an empty database is used | `0` |
| `ENABLE_DEMO_ROUTES` | Enable `/demo/*` routes, which can wipe data. Set to `0` on public deployments | `1` |

Never commit `.env` files or database connection strings.

------------------------------------------------------------------------

## 🔌 API Overview

### Agent backend (`:8000`)

| Method | Endpoint | Purpose |
|---|---|---|
| `POST` | `/api/complaints/process` | Run the full agent workflow and return the final state with its timeline |
| `POST` | `/api/complaints/{id}/verify` | Officer's manual verification (`passed: true` closes, `false` reassigns) |
| `GET` | `/api/complaints` | List processed complaints, newest first |
| `GET` | `/api/complaints/{id}` | Get one processed complaint |
| `GET` | `/api/scenarios` | Demo requests and verification scenarios |
| `GET` | `/api/health` | Active LLM provider and mode |

Example:

```bash
curl -X POST http://localhost:8000/api/complaints/process \
  -H "Content-Type: application/json" \
  -d '{
        "complaint_id": "C-1004",
        "description": "The streetlight in front of my house in Canal Town is not working since yesterday.",
        "location": "Canal Town",
        "verification_scenario": "fail_once"
      }'
```

The response contains the category, location, priority, department, assigned team, verification result, final status (`CLOSED` or `ESCALATED`) and the complete `history` used for the timeline.

### Database backend (`:8001`)

Full interactive documentation is available at `/docs`. Main groups:

-   **Complaints:** create, list, filter, update, status changes, history, related and similar complaints, reopen
-   **Agent steps:** understanding, investigation, decision, agent results and activity logs
-   **Assignments:** assign a team, update assignment status
-   **Verifications:** record a passed or failed verification (a failed one reopens the complaint)
-   **Reference data:** departments, zones, teams, technicians, users
-   **Dashboard:** statistics and recent activity
-   **Demo (optional):** seed, reset, and simulate technician work and verification

------------------------------------------------------------------------

## 🧪 Testing

```bash
cd backend && python -m pytest -q
cd municipal_complaint_agent && python -m pytest -q
```

The tests run offline and need no API key. One agent test checks that the optional LLM provider objects can be built, so it needs `requirements-llm.txt` installed.

Build the frontend for production (type-checks first, output in `dist/`):

```bash
cd municipal_complaint_agent/frontend
npm run build
```

------------------------------------------------------------------------

## 🚀 Deployment

The repository includes ready-made configurations:

-   **Docker / Hugging Face Spaces:** the root `Dockerfile` builds everything into one container on port 7860.
-   **Render + Netlify:** `render.yaml` deploys the two backends, and `netlify.toml` deploys the frontend and forwards `/api/*` to the agent backend.
-   **Vercel + Neon (free, persistent):** follow [DEPLOY_VERCEL.md](DEPLOY_VERCEL.md) to deploy the three parts as separate Vercel projects with a Neon PostgreSQL database.

------------------------------------------------------------------------

## ⚠️ Current Scope and Limitations

-   There is **no login**. The admin pages and the database API are open to anyone who knows the URL, so add authentication before any real-world use.
-   The **Teams** page and part of the admin charts use built-in sample data alongside live complaints (live ones are tagged NEW).
-   Verification evidence comes from the officer's manual decision or from simulated scenarios. A real citizen follow-up (for example by SMS) is supported by design through a `VerificationProvider` interface but is not connected yet.
-   Complaint text typed by citizens and reasons written by the agents are shown as written and are not translated to Urdu.

------------------------------------------------------------------------

**Report it. Route it. Verify it.**
