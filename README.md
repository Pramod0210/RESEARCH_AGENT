# Autonomous Research Report Generator

A sophisticated multi-agent research system that generates comprehensive, AI-driven reports by simulating diverse expert perspectives. This project combines LLM-powered research workflows, interactive interviewing agents, and modern document generation to produce publication-quality reports on virtually any topic.

![Python 3.11+](https://img.shields.io/badge/python-3.11%2B-blue)
![FastAPI](https://img.shields.io/badge/FastAPI-0.120.0-green)
![LangGraph](https://img.shields.io/badge/LangGraph-0.6.8-orange)

---

## 📋 Table of Contents

- [Overview](#overview)
- [Features](#features)
- [Architecture](#architecture)
- [Installation](#installation)
- [Configuration](#configuration)
- [Usage](#usage)
- [API Endpoints](#api-endpoints)
- [Deployment](#deployment)


---

## 🎯 Overview

The **Autonomous Research Report Generator** automates the research and report-writing process by:

1. **Generating Expert Personas**: Dynamically creates multiple analyst personas with different perspectives based on the topic
2. **Interactive Interviews**: Conducts automated interviews with each persona to gather diverse viewpoints
3. **Multi-Source Research**: Integrates web search (Tavily API) and Wikipedia for real-time information gathering
4. **Intelligent Report Writing**: Synthesizes all perspectives into a cohesive, well-structured report
5. **Multi-Format Output**: Generates both DOCX and PDF documents with professional formatting
6. **User Feedback Integration**: Allows users to provide feedback that influences report regeneration

### Use Cases

- Academic research paper generation
- Industry analysis and market reports
- Policy impact assessments
- Technical documentation
- Competitive intelligence
- Content creation and journalism

---

## ✨ Features

### Core Features

- **Multi-Agent Orchestration**: Uses LangGraph to coordinate multiple AI agents in a stateful workflow
- **Adaptive Analyst Personas**: Creates custom expert roles based on topic and user feedback
- **Real-time Web Search**: Integrates Tavily Search API for current information
- **Knowledge Base Integration**: Wikipedia integration for factual verification
- **Structured Output**: Leverages Pydantic models for validated data flow
- **Conversation Memory**: Maintains state across multi-turn agent interactions

### User Interface

- **Web Dashboard**: FastAPI-powered responsive UI for report generation
- **User Authentication**: Secure login/signup with bcrypt password hashing
- **Progress Tracking**: Real-time feedback mechanism during report generation
- **File Management**: Direct download of generated DOCX and PDF reports
- **Session Management**: Cookie-based session tracking

### Document Generation

- **DOCX Generation**: Uses `python-docx` for Word document creation
- **PDF Generation**: ReportLab-based PDF formatting with proper typography
- **Template-Based Rendering**: Jinja2 templates for consistent formatting
- **Markdown Support**: Supports markdown-to-document conversion

### Deployment & Operations

- **Docker Support**: Includes Dockerfiles for containerization
- **Jenkins Pipeline**: Automated CI/CD with Jenkinsfile
- **AWS Deployment**: Infrastructure-as-code scripts for AWS setup
- **Health Checks**: Built-in health check endpoints for orchestration
- **Logging**: Structured logging with `structlog`

---

## 🏗️ Architecture

```
┌─────────────────────────────────────────────────────────────┐
│                     FastAPI Web Server                       │
│  (Authentication, Dashboard, Report Submission/Download)     │
└──────────────────────┬──────────────────────────────────────┘
                       │
┌──────────────────────▼──────────────────────────────────────┐
│              Report Generation Service                       │
│     (Orchestrates workflow, manages state, stores results)   │
└──────────────────────┬──────────────────────────────────────┘
                       │
        ┌──────────────┼──────────────┐
        │              │              │
┌───────▼──────┐ ┌────▼────┐ ┌──────▼─────────┐
│ Report Gen   │ │Interview │ │ Document Gen   │
│ Workflow     │ │ Workflow │ │ (DOCX/PDF)     │
│ (LangGraph)  │ │(LangGraph)│ │                │
└───────┬──────┘ └────┬────┘ └──────┬─────────┘
        │             │             │
        └─────────┬───┴──────┬──────┘
                  │          │
        ┌─────────▼────┐  ┌──▼──────────────┐
        │ LLM Providers│  │ Research Tools  │
        │ (OpenAI,     │  │ - Tavily Search │
        │  Google,     │  │ - Wikipedia     │
        │  Groq)       │  │                 │
        └──────────────┘  └─────────────────┘
```

### Key Components

| Component | Purpose |
|-----------|---------|
| **FastAPI Server** | Handles HTTP requests, user sessions, file downloads |
| **Report Generator Workflow** | LangGraph state machine for orchestrating report generation |
| **Interview Workflow** | Multi-turn agent conversation for research depth |
| **LLM Integration** | Pluggable LLM providers (OpenAI, Google Gemini, Groq) |
| **Document Generation** | Converts report content to DOCX/PDF formats |
| **Database** | SQLAlchemy ORM with SQLite for user management |
| **Templates** | Jinja2 templates for HTML UI and document formatting |

---

## 🚀 Installation

### Prerequisites

- **Python 3.11+**
- **pip** or **uv** (package manager)
- API Keys:
  - Tavily Search API key ([https://tavily.com](https://tavily.com))
  - LLM provider API key (OpenAI, Google Gemini, or Groq)

### Step 1: Clone the Repository

```bash
git clone <repository-url>
cd Research_Agent
```

### Step 2: Create Virtual Environment

```bash
# Using Python's venv
python3.11 -m venv venv
source venv/bin/activate

# OR using uv (faster alternative)
uv venv
source .venv/bin/activate
```

### Step 3: Install Dependencies

```bash
# Using pip
pip install -r requirements.txt

# OR using uv
uv pip install -r requirements.txt
```

### Step 4: Install Package in Development Mode

```bash
pip install -e .
```

---

## ⚙️ Configuration

### Environment Variables

Create a `.env` file in the project root:

```env
# LLM Provider Configuration
OPENAI_API_KEY=your_openai_api_key
GOOGLE_API_KEY=your_google_api_key
GROQ_API_KEY=your_groq_api_key

# Search API Configuration
TAVILY_API_KEY=your_tavily_api_key

# FastAPI Configuration
HOST=0.0.0.0
PORT=8000
DEBUG=true

# Database Configuration
DATABASE_URL=sqlite:///./users.db

# Logging Configuration
LOG_LEVEL=INFO
```

---

## 💻 Usage

### Running Locally

#### Option 1: Start FastAPI Server

```bash
uvicorn src.api.main:app --reload --host 0.0.0.0 --port 8000
```

Navigate to http://localhost:8000 in your browser.

#### Option 2: Run CLI Script

```bash
python main.py
```

### Web Interface Workflow

1. **Sign Up**: Create a new user account
2. **Login**: Use credentials to access the dashboard
3. **Enter Topic**: Submit a research topic in the dashboard
4. **Review Progress**: Monitor report generation in real-time
5. **Provide Feedback**: (Optional) Submit feedback to refine the report
6. **Download Report**: Download the generated DOCX or PDF file

### Programmatic Usage

```python
from src.api.services.report_service import ReportService

# Initialize service
service = ReportService()

# Start report generation
result = service.start_report_generation(
    topic="Impact of AI on Healthcare",
    max_analysts=3
)
thread_id = result["thread_id"]

# Submit user feedback (optional)
service.submit_feedback(
    thread_id=thread_id,
    feedback="Focus more on regulatory impacts"
)

# Check status
status = service.get_report_status(thread_id)
print(f"Document path: {status['docx_path']}")
print(f"PDF path: {status['pdf_path']}")

# Download the report
file_response = service.download_file("report_filename.docx")
```

### API Endpoints

#### Authentication

| Method | Endpoint | Description |
|--------|----------|-------------|
| GET | `/` | Show login page |
| POST | `/login` | User login |
| GET | `/signup` | Show signup page |
| POST | `/signup` | User registration |

#### Report Management

| Method | Endpoint | Description |
|--------|----------|-------------|
| GET | `/dashboard` | User dashboard (requires session) |
| POST | `/generate_report` | Start new report generation |
| POST | `/submit_feedback` | Submit feedback on report |
| GET | `/download/{file_name}` | Download generated report |
| GET | `/health` | Health check endpoint |

#### Request Models

**ReportRequest**:
```python
{
    "topic": "str - Topic for report generation",
    "max_analysts": "int - Number of analyst personas (default: 3)",
    "feedback": "str (optional) - User feedback to refine report"
}
```

**FeedbackRequest**:
```python
{
    "thread_id": "str - Thread identifier for feedback",
    "feedback": "str - Feedback content"
}
```

---


## 🐳 Deployment

### Docker Deployment

#### Build Docker Image

```bash
./build-and-push-docker-image.sh
```

#### Run Docker Container

```bash
docker build -f Dockerfile -t research-agent:latest .
docker run -p 8000:8000 \
  -e OPENAI_API_KEY=your_key \
  -e TAVILY_API_KEY=your_key \
  research-agent:latest
```

### AWS Deployment

#### Using Setup Script

```bash
chmod +x setup-app-infrastructure.sh
./setup-app-infrastructure.sh
```

#### Manual Deployment

```bash
chmod +x azure-deploy-jenkins.sh
./azure-deploy-jenkins.sh
```

### Jenkins Pipeline

The `Jenkinsfile` defines:
- Build stage: Install dependencies, run tests
- Test stage: Run unit tests and linting
- Build Docker: Create and tag container image
- Deploy: Push to registry and deploy to cluster

---

## 📊 Generated Report Example

Reports include:

- **Executive Summary**: High-level overview and key findings
- **Multiple Perspectives**: Detailed analysis from each analyst persona
- **Research Sources**: Citations and references to web sources and Wikipedia
- **Conclusion**: Synthesis of all perspectives with recommendations
- **Professional Formatting**: Page breaks, headers, footers, and consistent styling

Output formats:
- `.docx` - Microsoft Word format (editable, formatted)
- `.pdf` - PDF format (read-only, portable)

---

## 🔐 Security Considerations

- **Password Security**: Uses bcrypt hashing (passlib) for password storage
- **Session Management**: Cookie-based sessions (production should use JWT)
- **Input Validation**: Pydantic validation on all API inputs
- **SQL Injection**: SQLAlchemy ORM prevents SQL injection
- **CORS**: Configured for cross-origin requests (customize for production)

**Production Recommendations:**

```python
# Use environment-based secrets management
# Implement JWT tokens instead of simple session cookies
# Add rate limiting and request throttling
# Enable HTTPS/TLS in production
# Implement audit logging for sensitive operations
```

---


**Last Updated**: March 2026
**Version**: 0.1.0
