
## 📁 Project Structure

```
Research_Agent/
├── src/
│   ├── api/
│   │   ├── main.py                 # FastAPI application setup
│   │   ├── models/
│   │   │   └── request_models.py   # Pydantic request/response schemas
│   │   ├── routes/
│   │   │   └── report_routes.py    # API endpoints and handlers
│   │   ├── services/
│   │   │   └── report_service.py   # Business logic for report generation
│   │   ├── templates/              # Jinja2 HTML templates
│   │   │   ├── login.html
│   │   │   ├── signup.html
│   │   │   ├── dashboard.html
│   │   │   └── report_progress.html
│   │   └── templates/
│   ├── config/
│   │   └── configuration.yaml      # Application configuration
│   ├── database/
│   │   └── db_config.py            # SQLAlchemy models and database setup
│   ├── exception/
│   │   └── custom_exception.py     # Custom exception classes
│   ├── logger/
│   │   └── logger_config.py        # Structured logging setup
│   ├── prompt_lib/
│   │   └── prompt_locator.py       # LLM prompt templates
│   ├── schemas/
│   │   └── models.py               # Pydantic schemas for workflows
│   ├── utils/
│   │   └── model_loader.py         # LLM model initialization
│   ├── workflows/
│   │   ├── report_generator_workflow.py  # Main report generation LangGraph
│   │   └── interview_workflow.py         # Interview workflow LangGraph
│   └── notebook/                   # Jupyter notebooks for development
├── static/
│   ├── css/                        # Stylesheets
│   └── js/                         # JavaScript files
├── generated_report/               # Output directory for generated reports
├── main.py                         # CLI entry point
├── requirements.txt                # Python dependencies
├── pyproject.toml                  # Project metadata and build config
├── Dockerfile                      # Container image definition
├── Dockerfile.jenkins              # Jenkins pipeline container
├── Jenkinsfile                     # CI/CD pipeline configuration
├── azure-deploy-jenkins.sh         # AWS deployment script
├── build-and-push-docker-image.sh  # Docker build/push automation
├── setup-app-infrastructure.sh     # Infrastructure setup script
└── users.db                        # SQLite database (auto-created)
```

### Key Files Explained

| File | Purpose |
|------|---------|
| `src/api/main.py` | FastAPI app initialization, middleware setup, routes registration |
| `src/workflows/report_generator_workflow.py` | LangGraph state machine orchestrating report generation |
| `src/workflows/interview_workflow.py` | Interview flow with multi-turn agent conversations |
| `src/api/services/report_service.py` | Service layer handling report generation coordination |
| `src/database/db_config.py` | SQLAlchemy ORM models for users, reports, feedback |
| `pyproject.toml` | Project metadata, dependencies, build configuration |
