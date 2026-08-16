# Autonomous Research Report Generator

Give it a topic. It invents a panel of domain analysts, sends each one off to interview an expert with live web search behind them, and merges what they bring back into a single sourced report — delivered as DOCX and PDF.

The interesting part is not that an LLM writes a report. It is that the pipeline **stops halfway and waits for you.** After the analyst panel is drafted and before any research spend, the graph interrupts and holds its checkpoint. You redirect the panel in plain English — *"drop the economist, add a labour-law perspective"* — and it resumes from exactly where it paused.

Built on LangGraph with a two-level graph, map-reduce fan-out, and durable checkpointing.

![Python 3.11+](https://img.shields.io/badge/python-3.11%2B-blue)
![FastAPI](https://img.shields.io/badge/FastAPI-0.120.0-green)
![LangGraph](https://img.shields.io/badge/LangGraph-0.6.8-orange)

---

## Why this project

Most "AI writes a report" demos are one prompt in a loop. The failure mode is a confident monolithic voice with no sourcing and no way to steer it. This one is built around three ideas:

- **A panel beats a prompt.** Instead of one model writing everything, the system generates *N* analyst personas — each with a name, role, affiliation, and a distinct axis of concern — and researches the topic once per persona. Disagreement between sections is a feature, not a bug.
- **The human steers before the money is spent.** The graph is compiled with `interrupt_before=["human_feedback"]`. It halts *after* drafting the panel and *before* fanning out into interviews, which is the expensive step. Feedback regenerates the panel; nothing is wasted.
- **Fan-out is real parallelism, not a for-loop.** Interviews are dispatched with LangGraph's `Send`, so each analyst runs as an independent subgraph instance. Their sections merge back through a state reducer rather than overwriting each other.

---

## Architecture

Two graphs, nested. The outer graph orchestrates the panel; the inner graph is one analyst's research cycle, instantiated once per analyst.

```
                             ┌──────────────────┐
                    START ──▶│  create_analyst  │  structured output → Perspectives
                             └────────┬─────────┘
                                      ▼
                             ┌──────────────────┐
                             │  human_feedback  │  ⏸ interrupt_before — graph halts,
                             └────────┬─────────┘     checkpoint persisted by thread_id
                                      │
                          Send(...) fan-out, one per analyst
                    ┌─────────────────┼─────────────────┐
                    ▼                 ▼                 ▼
             ┌────────────┐    ┌────────────┐    ┌────────────┐
             │ interview  │    │ interview  │    │ interview  │   ← subgraph, runs in parallel
             │  analyst 1 │    │  analyst 2 │    │  analyst 3 │
             └──────┬─────┘    └──────┬─────┘    └──────┬─────┘
                    └─────────────────┼─────────────────┘
                                      │  sections merge via operator.add reducer
                    ┌─────────────────┼─────────────────┐
                    ▼                 ▼                 ▼
            ┌──────────────┐  ┌──────────────┐  ┌──────────────┐
            │ write_intro  │  │ write_report │  │write_conclus.│   ← also parallel
            └──────┬───────┘  └──────┬───────┘  └──────┬───────┘
                   └─────────────────┼─────────────────┘
                                     ▼
                            ┌─────────────────┐
                            │ finalize_report │ ──▶ END ──▶ DOCX + PDF
                            └─────────────────┘
```

**The interview subgraph** — one analyst's full research cycle:

```
START ─▶ ask_question ─▶ search_web ─▶ generate_answer ─▶ save_interview ─▶ write_section ─▶ END
         persona-driven   Tavily,        grounded in         transcript        markdown
         question         structured     retrieved docs      buffered          section
                          SearchQuery    only
```

### How the pieces hold together

| Mechanism | Where | What it buys |
|---|---|---|
| `interrupt_before=["human_feedback"]` | [report_generator_workflow.py](src/workflows/report_generator_workflow.py) | Graph pauses mid-run; human redirects the panel before research spend |
| `MemorySaver` checkpointer + `thread_id` | [report_service.py](src/api/services/report_service.py) | Paused runs are addressable and resumable across HTTP requests |
| `Send(...)` conditional fan-out | [report_generator_workflow.py](src/workflows/report_generator_workflow.py) | Dynamic parallelism — panel size decides how many subgraphs spawn |
| `Annotated[list, operator.add]` | [models.py](src/schemas/models.py) | Parallel branches append to shared state instead of clobbering it |
| `with_structured_output(...)` | analyst + query generation | Pydantic-validated `Perspectives` / `SearchQuery`, no output parsing |
| Jinja2 prompt templates | [prompt_locator.py](src/prompt_lib/prompt_locator.py) | Prompts are data with `{% if %}` fallbacks, not f-strings in business logic |
| `ModelLoader` + YAML | [model_loader.py](src/utils/model_loader.py) | Swap OpenAI ↔ Gemini ↔ Groq with one env var |

### Resuming a paused run

The pause is the pivot the whole design turns on, so it is worth showing concretely:

```python
# 1. Runs until the interrupt, then returns — analysts drafted, nothing researched yet
result    = service.start_report_generation("Impact of LLMs on the future of jobs", 3)
thread_id = result["thread_id"]

# 2. Feedback is written into state *as if* the paused node produced it, then the graph resumes
service.submit_feedback(thread_id, "Add a labour-economics perspective; drop the VC angle")

# 3. Fan-out, synthesis, and both file renders have now happened
status = service.get_report_status(thread_id)
# {'status': 'completed', 'docx_path': '...', 'pdf_path': '...'}
```

Step 2 works because `update_state(..., as_node="human_feedback")` attributes the write to the interrupted node, so LangGraph continues down the edges leaving it rather than restarting.

---

## Stack

| Layer | Choice | Why |
|---|---|---|
| Orchestration | LangGraph | Stateful graphs with interrupts, checkpointing, and `Send` fan-out |
| LLM | OpenAI `gpt-4o` (default) | Swappable for Gemini 2.0 Flash or Groq DeepSeek-R1-70B via `LLM_PROVIDER` |
| Web search | Tavily | Search API that returns clean extracted content, not raw HTML |
| Schema | Pydantic | Structured LLM output validated at the boundary |
| API | FastAPI + Jinja2 | Server-rendered UI and JSON health endpoint in one app |
| Auth | passlib + bcrypt | Salted password hashing over SQLAlchemy/SQLite |
| Documents | python-docx, ReportLab | Heading-aware DOCX; wrapped, paginated, footered PDF |
| Logging | structlog | Bound structured context (`module=`, `topic=`) instead of string logs |
| CI/CD | Jenkins → ACR → Azure Container Apps | Multi-stage Docker build, scripted infra |

---

## Running it

**Prerequisites** — Python 3.11+, a [Tavily](https://tavily.com) key, and one LLM provider key.

```bash
git clone https://github.com/Pramod0210/RESEARCH_AGENT.git
cd RESEARCH_AGENT

python3.11 -m venv venv && source venv/bin/activate
pip install -r requirements.txt          # includes -e . for the src package

cp .env.example .env                     # then fill in your keys
```

Start the server:

```bash
uvicorn src.api.main:app --reload --port 8000
```

Open <http://localhost:8000>, sign up, and submit a topic from the dashboard.

**Or skip the web layer entirely** — `report_generator_workflow.py` has a `__main__` block that runs the same graph on the command line and prompts for feedback at the interrupt:

```bash
python -m src.workflows.report_generator_workflow
```

Reports land in `generated_report/<topic>_<timestamp>/`, one folder per run, containing both `.docx` and `.pdf`.

### Configuration

Model choice lives in [src/config/configuration.yaml](src/config/configuration.yaml); `LLM_PROVIDER` in `.env` picks which block is loaded.

```yaml
llm:
  openai:
    provider: "openai"
    model_name: "gpt-4o"
    temperature: 0
  google:
    provider: "google"
    model_name: "gemini-2.0-flash"
  groq:
    provider: "groq"
    model_name: "deepseek-r1-distill-llama-70b"
```

---

## HTTP surface

The UI is server-rendered, so these are form-encoded HTML routes rather than a JSON API.

| Method | Route | Purpose |
|---|---|---|
| `GET` | `/` | Login page |
| `POST` | `/login` | Authenticate, set session cookie |
| `GET` `POST` | `/signup` | Registration |
| `GET` | `/dashboard` | Topic submission (session required) |
| `POST` | `/generate_report` | Runs the graph up to the interrupt, returns a `thread_id` |
| `POST` | `/submit_feedback` | Injects feedback, resumes the graph, renders download links |
| `GET` | `/download/{file_name}` | Serves a generated DOCX/PDF |
| `GET` | `/health` | JSON liveness probe — wired to the Docker `HEALTHCHECK` |

---

## Deployment

Multi-stage Dockerfile — dependencies compile in a builder stage, only the installed packages carry into the runtime image.

```bash
docker build -t research-agent:latest .
docker run -p 8000:8000 --env-file .env research-agent:latest
```

**Azure**, driven by Jenkins:

```bash
./setup-app-infrastructure.sh    # Resource group, ACR, Container Apps env, Azure Files share
./azure-deploy-jenkins.sh        # Jenkins on ACI, preloaded with Python 3.11 + Azure CLI
```

The [Jenkinsfile](Jenkinsfile) then runs checkout → venv → dependencies → tests → ACR tag resolution → `az containerapp` deploy, pulling Azure credentials and API keys from Jenkins credential bindings. Generated reports persist to a mounted Azure Files share so they survive container restarts.

---

## Known limitations

Honest notes on where this is a portfolio project rather than a production service:

- **Sessions are in-process and unsigned.** `SESSIONS` is a module-level dict and the cookie value is a predictable `{username}_session`. Fine for a single-instance demo; it needs signed tokens and shared storage before it goes anywhere real.
- **Checkpoints are in-memory.** `MemorySaver` means paused runs are lost on restart. Swapping in LangGraph's SQLite or Postgres checkpointer is the natural next step.
- **Generation is synchronous.** A report runs inside the HTTP request, which takes minutes. It belongs on a task queue with the client polling `thread_id`.
- **Interviews are single-turn.** Each analyst asks one question, runs one search, and writes one section. `max_num_turns` is carried in state but the shipped subgraph is linear — the multi-turn loop exists only in the [notebook prototype](src/notebook/).
- **Panel size is fixed at 3** in the web route, though the graph and service accept any value.

---

## Repository layout

```
src/
├── workflows/          LangGraph graphs — outer report graph + interview subgraph
├── schemas/            Pydantic models and TypedDict graph state
├── prompt_lib/         Jinja2 prompt templates
├── utils/              ModelLoader (provider abstraction), YAML config loader
├── api/                FastAPI app, routes, service layer, Jinja2 templates
├── database/           SQLAlchemy models, bcrypt helpers
├── logger/             structlog setup
├── exception/          Custom exception type
└── notebook/           Exploratory prototypes (not part of the running app)
```

[WORKFLOW_DESIGN.md](WORKFLOW_DESIGN.md) documents the graph design in more depth.

---

Built by **Pramod Kumar** — [GitHub](https://github.com/Pramod0210)
