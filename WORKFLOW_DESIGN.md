# Workflow Design Documentation

## Table of Contents

1. [Overview](#overview)
2. [System Architecture](#system-architecture)
3. [Report Generation Workflow](#report-generation-workflow)
4. [Interview Workflow](#interview-workflow)
5. [State Management](#state-management)
6. [Data Flow Diagrams](#data-flow-diagrams)
7. [Performance Considerations](#performance-considerations)

---

## Overview

The Autonomous Research Report Generator uses **LangGraph**, a state machine framework, to orchestrate multi-agent workflows. Two interconnected graphs work in tandem:

1. **Report Generator Workflow** - Orchestrates the high-level report generation process
2. **Interview Workflow** - Handles deep research conversations with individual analyst personas

This document details how these workflows interact, manage state, and produce final reports.

---

## System Architecture

### High-Level Component Diagram

```
┌────────────────────────────────────────────────────────────────┐
│                      FastAPI Web Server                         │
│              (Routes, Sessions, File Management)                │
└────────────────────┬─────────────────────────────────────────┘
                     │
┌────────────────────▼─────────────────────────────────────────┐
│                   ReportService                               │
│    (Orchestrates workflows, manages thread state)             │
└────────────────────┬─────────────────────────────────────────┘
                     │
        ┌────────────┴────────────┐
        │                         │
┌───────▼──────────────┐  ┌──────▼──────────────────┐
│Report Generator      │  │ Analyst Interview       │
│Graph (LangGraph)     │  │ Graph (LangGraph)       │
│                      │  │                         │
│Nodes:               │  │Nodes:                   │
│- create_analyst     │  │- ask_question          │
│- human_feedback     │  │- search_web            │
│- conduct_interview  │  │- generate_answer       │
│- write_report       │  │- save_interview        │
│- write_intro        │  │- write_section         │
│- write_conclusion   │  │                         │
│- finalize_report    │  │                         │
└─────────────────────┘  └──────────────────────────┘
        │                         │
        └────────────┬────────────┘
                     │
        ┌────────────▼──────────────┐
        │ LLM & Research Tools      │
        │                           │
        │- OpenAI / Gemini / Groq  │
        │- Tavily Search API        │
        │- Wikipedia                │
        │- python-docx / ReportLab  │
        └──────────────────────────┘
```

---

## Report Generation Workflow

### Workflow Overview

The Report Generator is the main orchestrator. It creates analyst personas, coordinates interviews, and compiles the final report.

### Graph Structure

```
START
  ↓
[create_analyst] ← Creates diverse expert personas based on topic
  ↓
[human_feedback] ← Accepts user feedback (interrupts here for user input)
  ↓
 CONDITIONAL
  ├─→ [conduct_interview] ← Runs interview for EACH analyst in parallel
  │        ↓
  │    [Multiple parallel interview instances]
  │        ↓
  └─→ NO analysts → END
  
[conduct_interview] (all instances complete)
  ↓ (parallel edges to three nodes)
  ├→ [write_report]
  ├→ [write_introduction]
  └→ [write_conclusion]
  
  ↓ (all complete)
[finalize_report] ← Assembles intro + content + conclusion
  ↓
END
```

### Node Descriptions

#### 1. **create_analyst**
- **Purpose**: Generate diverse analyst personas based on the research topic
- **Inputs**: `topic`, `max_analysts`, `human_analyst_feedback` (optional)
- **Outputs**: `analysts` (list of Analyst objects)
- **LLM Task**: Structured output using Pydantic's `Perspectives` model
- **Key Logic**:
  ```python
  structured_llm = llm.with_structured_output(Perspectives)
  analysts = structured_llm.invoke([
      SystemMessage(content=CREATE_ANALYSTS_PROMPT),
      HumanMessage(content="Generate the set of analysts.")
  ])
  ```

**Analyst Schema**:
```python
class Analyst(BaseModel):
    affiliation: str          # e.g., "Tech Industry", "Academia"
    name: str                 # e.g., "Dr. Sarah Chen"
    role: str                 # e.g., "AI Research Director"
    description: str          # Goals, concerns, perspective
```

#### 2. **human_feedback**
- **Purpose**: Pause for human review and feedback
- **Inputs**: None (waits for external `update_state` call)
- **Outputs**: Updated `human_analyst_feedback` state
- **Interruption**: Uses `interrupt_before=["human_feedback"]` in graph compilation
- **Timing**: Occurs AFTER analysts are created but BEFORE interviews begin
- **User Interaction**: Web UI prompts user to review analysts and optionally provide feedback

#### 3. **conduct_interview** (Parallel Execution)
- **Purpose**: Run interview workflow for each analyst independently
- **Inputs**: Individual `analyst`, conversation `messages`, max turns
- **Outputs**: `sections` (compiled from all interviews), updated messages
- **Execution**: Triggered once per analyst using `Send()` in conditional edges
- **Coordination**: Uses `Annotated[list, operator.add]` to merge results from all parallel executions

**Key Pattern**:
```python
def initiate_all_interviews(state):
    analysts = state.get("analysts", [])
    return [
        Send("conduct_interview", {
            "analyst": analyst,
            "messages": [HumanMessage(...)],
            "max_num_turns": 2,
            "context": [],
            "interview": "",
            "sections": [],
        })
        for analyst in analysts
    ]
```

#### 4. **write_report**
- **Purpose**: Synthesize all interview sections into main content
- **Inputs**: `sections` (from all interviews), `topic`
- **Outputs**: `content` (main report body)
- **LLM Task**: Instruct LLM to unify and synthesize diverse perspectives
- **Processing**: Joins section array and passes to `REPORT_WRITER_INSTRUCTIONS` prompt

#### 5. **write_introduction**
- **Purpose**: Generate opening section that frames the topic
- **Inputs**: `sections`, `topic`
- **Outputs**: `introduction`
- **LLM Task**: Create engaging introduction using `INTRO_CONCLUSION_INSTRUCTIONS`
- **Context**: Uses formatted sections to ensure introduction aligns with content

#### 6. **write_conclusion**
- **Purpose**: Generate closing section summarizing key insights
- **Inputs**: `sections`, `topic`
- **Outputs**: `conclusion`
- **LLM Task**: Synthesize findings and provide forward-looking insights
- **Parallel Execution**: Runs simultaneously with `write_report` and `write_introduction`

#### 7. **finalize_report**
- **Purpose**: Assemble introduction, content, conclusion, and sources
- **Inputs**: `introduction`, `content`, `conclusion`
- **Outputs**: `final_report` (complete markdown text)
- **Processing Logic**:
  ```python
  final_report = (
      introduction + "\n\n---\n\n" +
      content + "\n\n---\n\n" +
      conclusion
  )
  if sources:
      final_report += "\n\n## Sources\n" + sources
  ```

### Edge Definitions

| From | To | Type | Condition |
|------|----|----|-----------|
| START | create_analyst | Hard | Always |
| create_analyst | human_feedback | Hard | Always |
| human_feedback | conduct_interview | Conditional | If analysts exist |
| human_feedback | END | Conditional | If no analysts |
| conduct_interview | write_report | Hard | All interviews complete |
| conduct_interview | write_introduction | Hard | All interviews complete |
| conduct_interview | write_conclusion | Hard | All interviews complete |
| write_report → write_introduction → write_conclusion | finalize_report | Hard | All complete |
| finalize_report | END | Hard | Always |

---

## Interview Workflow

### Purpose

The Interview Workflow simulates a deep research conversation between an analyst and an expert. It performs web searches, generates answers, and compiles findings into a report section.

### Workflow Structure

```
START
  ↓
[ask_question] ← Analyst generates opening question based on topic
  ↓
[search_web] ← Extract search query from conversation
  ↓           → Execute Tavily search and retrieve documents
  ↓
[generate_answer] ← Expert answers using search results as context
  ↓
[save_interview] ← Save full conversation transcript
  ↓
[write_section] ← Summarize interview into report section
  ↓
END
```

### Node Descriptions

#### 1. **ask_question**
- **Purpose**: Generate the analyst's opening question
- **Inputs**: `analyst` (Analyst object), `messages` (conversation history)
- **Outputs**: New message appended to `messages`
- **LLM Task**: Use analyst persona to generate relevant, probing questions
- **Prompt Template**: `ANALYST_ASK_QUESTIONS`
- **Output Type**: `HumanMessage` with question content

#### 2. **search_web**
- **Purpose**: Convert conversation into search queries and retrieve web sources
- **Inputs**: Current `messages` (conversation so far)
- **Outputs**: `context` (formatted search results)
- **Process**:
  1. Use LLM to extract `SearchQuery` (structured output)
  2. Call Tavily Search API with extracted query
  3. Format results as XML-like documents with URLs
  
**SearchQuery Schema**:
```python
class SearchQuery(BaseModel):
    search_query: str  # e.g., "AI impact on job market 2024"
```

**Result Format**:
```
<Document href="https://example.com/article1"/>
Content from article 1...
</Document>

<Document href="https://example.com/article2"/>
Content from article 2...
</Document>
```

#### 3. **generate_answer**
- **Purpose**: Expert generates informed response using search results
- **Inputs**: `analyst`, `messages`, `context` (search results)
- **Outputs**: New `AIMessage` with expert answer
- **LLM Task**: Use analyst persona + context to provide authoritative answer
- **Prompt Template**: `GENERATE_ANSWERS`
- **Persona Integration**: Full analyst description (name, role, affiliation, goals) injected into prompt

#### 4. **save_interview**
- **Purpose**: Preserve the full conversation transcript
- **Inputs**: `messages` (entire conversation)
- **Outputs**: `interview` (formatted string of full conversation)
- **Method**: Uses `get_buffer_string()` from LangChain to serialize messages
- **Storage**: Kept in state for logging/debugging

#### 5. **write_section**
- **Purpose**: Condense interview findings into a structured report section
- **Inputs**: `context` (search results), `analyst` (to understand perspective)
- **Outputs**: `sections` list with markdown-formatted content
- **LLM Task**: Synthesize interview and research into concise section
- **Prompt Template**: `WRITE_SECTION`
- **Format**: Markdown with headers, paragraphs, and bullet points

### Interviewing Loop

The interview is **not** a multi-turn loop within this workflow. Instead:
- One complete cycle (question → search → answer → section) occurs per interview invocation
- The `max_num_turns` parameter in `InterviewState` is prepared for future multi-turn expansion
- Currently, the workflow runs once per analyst for focused, efficient research

---

## State Management

### Three-Tier State Architecture

#### 1. **GenerateAnalystsState**
Used during analyst persona creation:
```python
class GenerateAnalystsState(TypedDict):
    topic: str                          # Research topic
    max_analysts: int                   # Number of personas to create
    human_analyst_feedback: str         # Optional user feedback
    analysts: List[Analyst]             # Output: created personas
```

#### 2. **InterviewState**
Manages individual interview conversations:
```python
class InterviewState(MessagesState):  # Inherits from LangChain MessagesState
    max_num_turns: int                  # Max interview turns (prepared for future)
    context: Annotated[list, operator.add]  # Accumulated search results
    analyst: Analyst                    # Current analyst
    interview: str                      # Conversation transcript
    sections: list                      # Generated section(s)
```

**Key**: `Annotated[list, operator.add]` means sections from parallel interviews are automatically merged (concatenated).

#### 3. **ResearchGraphState** (Main State)
Manages entire report generation process:
```python
class ResearchGraphState(TypedDict):
    topic: str                          # Research topic
    max_analysts: int                   # Number of analysts
    human_analyst_feedback: str         # User feedback
    analysts: List[Analyst]             # All analyst personas
    sections: Annotated[list, operator.add]  # All interview sections (merged)
    introduction: str                   # Intro text
    content: str                        # Main report body
    conclusion: str                     # Conclusion text
    final_report: str                   # Complete assembled report
```

### State Transitions

```
ResearchGraphState initial:
├─ topic: "AI in Healthcare"
├─ max_analysts: 3
├─ analysts: []

After create_analyst:
├─ analysts: [Analyst(...), Analyst(...), Analyst(...)]

After all conduct_interview (parallel):
├─ sections: [section1, section2, section3]

After write_* nodes:
├─ introduction: "..."
├─ content: "..."
├─ conclusion: "..."

After finalize_report:
└─ final_report: "# Introduction\n\n...\n\n# Content\n\n...\n\n# Conclusion\n\n..."
```

### Checkpointing & Memory

```python
memory = MemorySaver()  # In-memory checkpoint storage
graph = builder.compile(
    interrupt_before=["human_feedback"],
    checkpointer=memory
)
```

- **Thread-based state persistence**: Each report generation gets a unique `thread_id`
- **Interruption points**: Graph pauses at `human_feedback` for user input
- **State retrieval**: `graph.get_state(thread)` fetches current state
- **State update**: `graph.update_state(thread, updates, as_node=node_name)` modifies state at specific node

---

## Data Flow Diagrams

### End-to-End Report Generation Flow

```
User Input
    │
    ├─ Topic: "Impact of LLMs on Science"
    ├─ Max Analysts: 3
    └─ (optional) Feedback
    
    ↓
    
create_analyst
    │
    └─→ LLM generates 3 personas
        ├─ Dr. Alice (Researcher perspective)
        ├─ Prof. Bob (Industry perspective)
        └─ Dr. Carol (Ethics perspective)
    
    ↓
    
human_feedback [INTERRUPT]
    │
    ├─ User reviews analysts
    ├─ (optional) Provides feedback
    └─ Continues flow via update_state()
    
    ↓
    
conduct_interview (x3 in parallel)
    │
    ├─ Interview 1 (Dr. Alice):
    │   ├─ ask_question → "What are current LLM breakthroughs in research?"
    │   ├─ search_web → Query: "LLM applications science research 2024"
    │   │              Returns: Articles, papers, news
    │   ├─ generate_answer → Expert response using search context
    │   ├─ save_interview → Store full Q&A
    │   └─ write_section → "Alice's Perspective on LLM in Research"
    │
    ├─ Interview 2 (Prof. Bob):
    │   └─ Similar flow for industry perspective
    │
    └─ Interview 3 (Dr. Carol):
        └─ Similar flow for ethics perspective
    
    ↓ (All interviews complete, sections auto-merged)
    
write_report ────────┐
write_introduction ──┼─→ Parallel execution
write_conclusion ────┤
    
    Input: all sections + topic
    │
    ├─ write_report: Synthesize all 3 sections
    ├─ write_introduction: Frame the topic
    └─ write_conclusion: Summarize & forward-look
    
    ↓
    
finalize_report
    │
    ├─ Assemble: [Intro] + [Content] + [Conclusion]
    ├─ Add sources section
    └─ Output: Complete markdown document
    
    ↓
    
Document Generation
    │
    ├─ save_report(format="docx")
    │   └─ python-docx: Parse markdown, create DOCX
    │
    └─ save_report(format="pdf")
        └─ ReportLab: Centered formatting, multi-page PDF
    
    ↓
    
User Download
    │
    └─ Files: report.docx, report.pdf
```

### Interview Workflow Detailed Flow

```
Interview Node receives:
├─ analyst: Analyst object
├─ messages: [HumanMessage("Let's discuss about LLMs")]
├─ context: []
└─ sections: []

ask_question:
├─ Input: analyst.persona + messages
├─ LLM: "Generate a probing question from this analyst's perspective"
├─ Output: AIMessage("What are the key challenges in scaling LLM training?")
└─ State: messages.append(question)

search_web:
├─ Input: Full messages (conversation so far)
├─ LLM: Extract SearchQuery from conversation
│   └─ Output: SearchQuery(search_query="LLM training challenges scalability")
├─ Tavily API: tavily_search.invoke("LLM training challenges...")
├─ Format results: <Document href="...">content</Document>
└─ State: context = [formatted_docs]

generate_answer:
├─ Input: analyst.persona + messages + context
├─ LLM: "Using this expert knowledge, respond as this analyst"
├─ Output: AIMessage with expert answer citing sources
└─ State: messages.append(answer)

save_interview:
├─ Input: messages (full conversation)
├─ Process: get_buffer_string(messages) → markdown format
└─ State: interview = full_conversation_text

write_section:
├─ Input: context (search docs) + analyst.description
├─ LLM: "Summarize this interview into a cohesive section"
├─ Output: "## [Analyst Name]'s Perspective\n...\n### Key Findings\n..."
└─ State: sections = [markdown_section]

Output to Report Generator:
└─ sections: ["Dr. Alice's Section", "Prof. Bob's Section", "Dr. Carol's Section"]
```

---

## Performance Considerations

### Parallelization

**Interview Execution**:
- All analyst interviews run in parallel using LangGraph's `Send()` mechanism
- For 3 analysts: Single sequential time = Interview(A) + Interview(B) + Interview(C)
- Parallel execution: time ≈ max(Interview(A), Interview(B), Interview(C))
- **Speedup**: ~2.5-3x for typical cases

**Report Assembly**:
- write_report, write_introduction, write_conclusion execute simultaneously
- Total time reduced from 3x LLM call → 1x LLM call duration

### Optimization Opportunities

1. **LLM Caching**: Cache analyst persona generation for same topics
2. **Context Windowing**: Trim long search results to stay within token limits
3. **Incremental Updates**: Stream report generation progress to UI
4. **Model Selection**: Use faster models (GPT-3.5) for interviews, stronger for synthesis
5. **Temperature Tuning**: Lower temperature (0.3-0.5) for factual sections, higher (0.7+) for creative intro/conclusion

### Bottlenecks

| Stage | Typical Time | Driver |
|-------|--------------|--------|
| create_analyst | 5-10s | LLM inference + structured output |
| search_web (per analyst) | 2-5s | Tavily API latency + LLM query extraction |
| generate_answer (per analyst) | 5-15s | LLM with context window |
| write_* (3 parallel) | 10-20s | LLM synthesis |
| finalize_report | <1s | String operations |
| **Total** | **25-60s** | Interview phase (parallelized) |

Document generation (DOCX/PDF) adds 2-5s additional.

---

## Extension Points

### 1. Adding New Research Tools

**Current**: Tavily Search + Wikipedia

**To Add**: Database search, arXiv API, news APIs

```python
# In interview_workflow.py
class InterviewGraphBuilder:
    def __init__(self, llm, tavily_search, additional_tools):
        self.tavily_search = tavily_search
        self.arxiv_search = additional_tools.get("arxiv")
        self.db_search = additional_tools.get("database")
    
    def _search_web(self, state: InterviewState):
        # Use multiple tools based on context
        results_tavily = self.tavily_search.invoke(query)
        results_arxiv = self.arxiv_search.invoke(query)
        combined = results_tavily + results_arxiv
```

### 2. Multi-Turn Interviews

**Current**: Single Q&A cycle per analyst

**To Extend**: Multiple turns based on feedback

```python
def _generate_followup_questions(self, state):
    """Generate follow-up questions based on answer"""
    if len(state["messages"]) < state["max_num_turns"] * 2:
        return "ask_question"  # Loop back
    return "save_interview"    # Exit interview loop
```

Add cyclic edge: `ask_question` → `search_web` → `generate_answer` → (conditional) → repeat or `save_interview`

### 3. Custom Report Templates

**Current**: Markdown → DOCX/PDF conversion

**To Add**: Custom Jinja2 templates, branded formats

```python
def _save_as_docx_templated(self, context, template_name):
    template = jinja2.Template(open(f"templates/{template_name}.html").read())
    html = template.render(context)
    doc = Document()
    # Parse HTML to docx
```

### 4. Interactive Analyst Adjustment

**Current**: Static analyst creation + optional feedback

**To Add**: User ability to remove/add analysts mid-flow

```python
# New node after human_feedback
def refine_analysts(self, state):
    # User removes/edits analysts
    updated_analysts = state["adjusted_analysts"]
    return {"analysts": updated_analysts}

# Add edge: human_feedback → refine_analysts → conditional → interviews
```

### 5. Real-Time Progress Streaming

**Current**: Batch processing, results at end

**To Add**: Stream progress updates via WebSocket

```python
async def stream_report_generation(websocket):
    async for event in graph.astream(input_state, thread):
        await websocket.send_json({"event": event})
```

### 6. Alternative LLM Providers

**Current**: Pluggable via ModelLoader

**To Extend**: Vendor-specific optimizations

```python
class InterviewGraphBuilder:
    def __init__(self, llm, tavily_search):
        self.llm = llm
        self.is_openai = isinstance(llm, ChatOpenAI)
        self.is_groq = isinstance(llm, ChatGroq)
    
    def _search_web(self, state):
        if self.is_groq:
            # Use faster parallelized search for Groq
            results = self.tavily_search.batch(queries)
        else:
            results = self.tavily_search.invoke(query)
```

### 7. Feedback Loop Refinement

**Current**: Single feedback submission, re-runs entire pipeline

**To Add**: Incremental feedback → targeted regeneration

```python
def submit_targeted_feedback(self, thread_id, section_index, feedback):
    """Only regenerate specific sections based on feedback"""
    # Get current state
    state = graph.get_state(thread)
    # Identify affected sections
    # Conditionally re-run only those analyst interviews
```

---

## State Diagram (Complete Lifecycle)

```
┌─────────────────────────────────────────────────────────────┐
│                      START                                   │
│     {"topic": "...", "max_analysts": 3}                     │
└─────────────────────┬───────────────────────────────────────┘

┌─────────────────────▼───────────────────────────────────────┐
│                create_analyst                                │
│ Input: topic, max_analysts, human_analyst_feedback           │
│ Output: analysts = [Analyst, Analyst, Analyst]              │
└─────────────────────┬───────────────────────────────────────┘

┌─────────────────────▼───────────────────────────────────────┐
│             human_feedback [INTERRUPT]                       │
│ ⏸️  Graph pauses, user reviews analysts via web UI           │
│ User can provide feedback OR continue                        │
│ graph.update_state(thread, {"human_analyst_feedback": "..."})
└─────────────────────┬───────────────────────────────────────┘

                    ┌─┐ CONDITIONAL
              Analysts? │
                 │  │
             YES │  │ NO
                 │  └──────────────┐
    ┌────────────▼─┐               │
    │ conduct_     │         ┌─────▼──────┐
    │ interview    │         │    END      │
    │ (Analyst 1)  │         └─────────────┘
    │ [parallel]   │
    ├──────────────┤
    │ conduct_     │
    │ interview    │
    │ (Analyst 2)  │
    │ [parallel]   │
    ├──────────────┤
    │ conduct_     │
    │ interview    │
    │ (Analyst 3)  │
    │ [parallel]   │
    └───────┬──────┘
            │
     (All interviews complete, sections merged)
            │
    ┌───────┴───────────┬──────────────┬──────────────┐
    │                   │              │              │
┌───▼────────┐   ┌──────▼────┐   ┌────▼─────────┐   │
│write_report │   │write_     │   │write_        │   │
│             │   │introduction│   │conclusion    │   │
│[parallel]   │   │[parallel]  │   │[parallel]    │   │
└───┬────────┘   └──────┬────┘   └────┬─────────┘   │
    │                   │              │              │
    └───────────────────┬──────────────┘              │
                        │                             │
                ┌───────▼──────────┐                 │
                │finalize_report   │                 │
                │Assembles: intro+ │                 │
                │content+conclusion│                 │
                └───────┬──────────┘                 │
                        │                             │
                ┌───────▼──────────────────────────┐ │
                │  final_report: str (markdown)    │ │
                │  Complete report ready for       │ │
                │  document generation (DOCX/PDF)  │ │
                └───────┬──────────────────────────┘ │
                        │                             │
                        └─────────────────────────────┘

┌─────────────────────────────────────────────────────────────┐
│                        END                                   │
│                report ready to download                       │
└─────────────────────────────────────────────────────────────┘
```

---

## Configuration & Customization

### Key Configuration Points

**in `src/config/configuration.yaml`**:

```yaml
report_generation:
  max_analysts: 3              # Number of personas
  interview_depth: 5           # (reserved for multi-turn)
  parallel_interviews: true    # Enable parallel execution
  
llm:
  provider: openai             # openai | google | groq
  model: gpt-4-turbo
  temperature: 0.7
  
search:
  api: tavily                  # tavily | google_search
  max_results_per_query: 5
```

**in `src/prompt_lib/prompt_locator.py`**:

```python
CREATE_ANALYSTS_PROMPT = Template("""
You are an expert analyst generator. Create {{max_analysts}} diverse 
expert personas for the topic: {{topic}}
...
{% if human_analyst_feedback %}
Incorporate this feedback: {{human_analyst_feedback}}
{% endif %}
""")
```

---

## Summary

The Autonomous Research Report Generator combines two sophisticated LangGraph workflows:

1. **Report Generator**: Orchestrates the end-to-end process from topic input → analyst creation → interview coordination → final report assembly
2. **Interview**: Provides deep research by simulating expert conversations with web search integration

Key architectural strengths:
- ✅ **Stateful, resumable workflows** with checkpointing
- ✅ **Parallel execution** for efficiency
- ✅ **Human-in-the-loop** feedback mechanism
- ✅ **Structured outputs** via Pydantic
- ✅ **Comprehensive error handling** with logging
- ✅ **Extensible design** for new tools, models, and feedback loops

This design allows for sophisticated AI-driven research while maintaining clarity, debuggability, and user control.

---

