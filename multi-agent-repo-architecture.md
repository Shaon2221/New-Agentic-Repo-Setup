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
uv.lock                      # uv lock file (replaces poetry.lock/requirements.txt)
pyproject.toml               # Single source of truth for dependencies & tool config
Dockerfile                   # Multi-stage build optimized for uv
docker-compose.yml           # run and connect one or more containers together
pytest.ini                   # Test configuration
README.md                    # Overview, Architecture, Usage
```
