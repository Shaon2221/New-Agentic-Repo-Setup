```
.github/
  ├── workflows/             # GitHub Actions YAML files CI/CD (Test, Lint, Build)
  └── ISSUE_TEMPLATE/        # Standardized bug reports/feature requests
app/
  ├── main.py                # App entry point (FastAPI app initialization)
  ├── api/                   # REST API Layer (Routes only)
  │   ├── dependencies.py    # FastAPI dependencies (auth, db sessions)
  │   └── v1/			           # Versioned endpoints
  ├── core/                  # Application Core (Singleton Configs & Factories)
  │   ├── config.py          # Pydantic Settings (loads .env)
  │   ├── logging.py         # JSON structure logger setup
  │   ├── telemetry.py       # Arize Phoenix / OpenTelemetry setup
  │   └── llm.py             # Ex: langchain ini_chat_model
  ├── agents/                # The "Brain" (Agentic Logic)
  │   ├── orchestrator.py    # Main router/orchestrator agent
  │   ├── domains/ 		       # Specialized domain agents (e.g., billing, support)
  │   ├── prompts/			     # Prompt templates
  │   └── guardrails/		     # Safety Checks Guardrails
  ├── services/              # Business Logic
  ├── auth_service.py
  ├── database_service.py
  ├── tools/                 # Agent Tools
  │   ├── local/             # Python functions (calculation, formatting)
  │   ├── mcp/               # Model Context Protocol definitions
  │   └── http/              # API wrapper tools
  └── models/                # # Pydantic Schemas (DTOs) & DB Models
ci/                          # Scripts for CI/CD
  └── push_docker.sh
  └── run_evals.py
tests/
  ├── unit/			            # Fast, mock-heavy tests
  ├── integration/	        # DB/Docker connected tests
  └── evals/			          # AI quality checks (Arize/LLM-as-a-judge)
.dockerignore			          # which are not required for build
.gitignore					        # local file/scripts that we don’t want to push in git
.python-version             # Pins Python version for uv
.pre-commit-config.yaml     # enforcing code quality
uv.lock				              # uv lock file (replaces poetry.lock/requirements.txt)
pyproject.toml		          # Single source of truth for dependencies & tool config
Dockerfile			            # Multi-stage build optimized for uv
docker-compose.yml 	        # run and connect one or more containers together for deployment
pytest.ini				          # Test configuration
README.md			              # Contain overview. Features, Techstack, System Design, Repo Architecture, setup & usage
```
