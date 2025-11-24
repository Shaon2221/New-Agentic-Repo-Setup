```
.github/
  ├── workflows/             # GitHub Actions YAML files CI/CD (Test, Lint, Build)
  └── ISSUE_TEMPLATE/        # Standardized bug reports/feature requests
app/
  ├── main.py                # App entry point (FastAPI app initialization)
  ├── api/                   # REST API Layer (Routes only)
  │   ├── dependencies.py    # FastAPI dependencies (auth, db sessions)
  │   └── v1/                # Versioned endpoints
  ├── core/                  # Application Core (Singleton Configs & Factories)
  │   ├── config.py          # Pydantic Settings (loads .env)
  │   ├── logging.py         # JSON structure logger setup
  │   ├── telemetry.py       # Arize Phoenix / OpenTelemetry setup
  │   └── llm.py             # Shared LLM Factory (e.g. get_llm(temp=0))
  ├── agents/                # MULTI-AGENT ARCHITECTURE
  │   ├── graph.py           # The "Map": Defines nodes (agents) and edges (routes)
  │   ├── state.py           # The "Memory": Shared GraphState/Schema (TypedDict)
  │   ├── nodes/             # The "Workers": Individual agents
  │   │   ├── researcher.py  # A specific agent node
  │   │   ├── coder.py       # A specific agent node
  │   │   └── reviewer.py    # A specific agent node
  │   ├── edges.py           # The "Router": Conditional logic (e.g., "if error -> retry")
  │   ├── prompts/           # Specialized prompts per agent
  │   └── guardrails/        # Safety checks (applied at node output level)
  ├── services/              # Business Logic
  │   ├── auth_service.py
  │   ├── database_service.py
  │   ├── memory_store.py    # Persistence for agent checkpoints (Postgres/Redis)
  ├── tools/                 # Agent Tools
  │   ├── local/             # Python functions (calculation, formatting)
  │   ├── mcp/               # Model Context Protocol definitions
  │   └── http/              # API wrapper tools
  └── models/                # Pydantic Schemas (DTOs) & DB Models
ci/                          # Scripts for CI/CD
  ├── push_docker.sh
  └── run_evals.py
tests/
  ├── unit/                  # Fast, mock-heavy tests
  ├── integration/           # DB/Docker connected tests
  └── evals/                 # AI quality checks (Arize/LLM-as-a-judge)
.dockerignore                # which are not required for build
.gitignore                   # local file/scripts that we don’t want to push in git
.python-version              # Pins Python version for uv
.pre-commit-config.yaml      # enforcing code quality
.env                         # secret/API Key credentials
.env.example                 # secret/API Key credentials example
uv.lock                      # uv lock file (replaces poetry.lock/requirements.txt)
pyproject.toml               # Single source of truth for dependencies & tool config
Dockerfile                   # Multi-stage build optimized for uv
docker-compose.yml           # run and connect one or more containers together
pytest.ini                   # Test configuration
README.md                    # Overview, Architecture, Usage
```


### Mostt important things for Multi-Agent Support

**1. `agents/state.py` (The Most Important File)**
In a single-agent system, memory is just a list of messages. In a *multi-agent* system, you need a shared state that is passed between agents.

  * *What it contains:* A `TypedDict` or Pydantic model defining exactly what data exists (e.g., `messages`, `code_draft`, `review_comments`, `retry_count`).

**2. `agents/nodes/` (Instead of `domains`)**
We rename "domains" to "nodes" to match graph terminology. Each file here represents an isolated agent that takes the *State* in, does work, and updates the *State*.

  * *Example:* `nodes/researcher.py` doesn't know about the User. It only knows: "I received a topic in the State, I will write a summary, and update the State."

**3. `agents/graph.py` & `edges.py`**

  * **graph.py:** The orchestrator that wires everything together. It says: *"Start at Researcher. After Researcher, go to Reviewer."*
  * **edges.py:** The logic for branching. *"If Reviewer approves, go to End. If Reviewer rejects, go back to Coder."*

**4. `services/memory_store.py`**
Multi-agent systems often require "Time Travel" or "Human-in-the-loop". You need a specific service to save the *State* snapshot to a database (Postgres) so you can pause an agent's work and resume it later.
------------------------------------------------------------------

#### A. `uv` Specific Optimization (`.python-version` & Docker)

Since we are using **uv**, you should add a `.python-version` file to pin the exact Python version (e.g., `3.14.0`).

  * **Docker Optimization:** Your `Dockerfile` must be adapted to use `uv`'s cache mounting features. A standard `pip install` is inefficient with `uv`.
      * *Action:* Use a multi-stage Docker build where the "builder" stage uses `uv sync --frozen` with a standard cache mount (`--mount=type=cache,target=/root/.cache/uv`) to speed up builds by 10-100x.

#### B. Split Testing Strategy (`tests/evals/`)

You cannot treat AI tests like normal unit tests.

  * **Unit/Integration:** Run on every commit (Fast).
  * **Evals:** Run on `deploy` or `nightly`. These call real LLMs, cost money, and take minutes to run.
  * *Action:* Create a separate `tests/evals` folder and mark them in `pytest.ini` so they don't run by default.

#### C. Telemetry & Tracing (`telemetry.py`)

Do not scatter logging code.

  * *Action:* Create a centralized telemetry module that initializes the `OpenTelemetry` instrumentors. Call this **once** in `main.py` during startup. This ensures every agent step, tool call, and API request is traced automatically.

#### D. Strict Config Management (`app/core/config.py`)

Avoid `os.getenv` scattered throughout your code.

  * *Action:* Use `pydantic-settings`. It reads from `.env` automatically and validates types (e.g., ensuring `MAX_RETRIES` is actually an `int`, not a `string`).

<!-- end list -->

```python
# app/core/config.py
from pydantic_settings import BaseSettings

class Settings(BaseSettings):
    environment: str = "dev"  # dev, staging, prod
    openai_api_key: str
    arize_phoenix_endpoint: str | None = None

    class Config:
        env_file = ".env"

settings = Settings()
```
