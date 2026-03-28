# System Architecture

## Overview

The AI-Powered Procurement Intelligence System is a sophisticated multi-agent architecture that demonstrates advanced AI engineering principles. This document provides a deep dive into the technical design decisions, data flows, and implementation patterns.

## High-Level Architecture

```
┌─────────────────────────────────────────────────────────────────┐
│                        User Interface Layer                      │
│  (Notebook/CLI) → Input Parameters → Crew Orchestrator          │
└─────────────────────────────────────────────────────────────────┘
                              ↓
┌─────────────────────────────────────────────────────────────────┐
│                    CrewAI Orchestration Layer                    │
│  • Process: Sequential                                           │
│  • Knowledge: Company Context (StringKnowledgeSource)            │
│  • State Management: Inter-agent data passing via JSON          │
└─────────────────────────────────────────────────────────────────┘
                              ↓
        ┌─────────────────────┴─────────────────────┐
        │                                             │
┌───────▼────────┐  ┌──────────────┐  ┌─────────────▼──────┐
│  Agent Layer   │  │  Tool Layer  │  │  Model Layer       │
│                │  │              │  │                    │
│ • Query Agent  │  │ • Search API │  │ • Pydantic Schemas │
│ • Search Agent │  │ • Scrape API │  │ • Type Validation  │
│ • Scrape Agent │  │              │  │ • JSON Outputs     │
│ • Report Agent │  │              │  │                    │
└────────┬───────┘  └──────┬───────┘  └────────┬───────────┘
         │                 │                    │
         └─────────────────┴────────────────────┘
                           ↓
┌─────────────────────────────────────────────────────────────────┐
│                    External Service Layer                        │
│  • Groq LLM API    • Tavily Search    • ScrapeGraph              │
│  • AgentOps Monitoring                                          │
└─────────────────────────────────────────────────────────────────┘
                           ↓
┌─────────────────────────────────────────────────────────────────┐
│                        Output Layer                              │
│  • Structured JSON (steps 1-3)                                  │
│  • HTML Report (step 4)                                         │
└─────────────────────────────────────────────────────────────────┘
```

## Component Details

### 1. Agent Layer

#### Query Strategist Agent
**Purpose**: Generate optimized search queries

**Design Decisions**:
- Uses LLM reasoning to understand product categories
- Considers multiple search angles (brand-specific, generic, comparison)
- Filters queries to target product pages, not informational content

**Input Schema**:
```python
{
    "product_name": str,
    "websites_list": List[str],
    "no_keywords": int,
    "country_name": str,
    "language": str
}
```

**Output Schema**:
```python
class SuggestedSearchQueries(BaseModel):
    queries: List[str]  # Max 10, min 1
```

**Prompt Engineering Strategy**:
- Explicit instruction to vary query types
- Brand name suggestions
- E-commerce site targeting
- Language localization

#### Search Intelligence Agent
**Purpose**: Execute searches and aggregate results

**Design Decisions**:
- Quality filtering (score > 0.10 threshold)
- Parallel search execution via Tavily
- Result deduplication
- Content extraction for downstream processing

**Tool**: `search_engine_tool`
```python
@tool
def search_engine_tool(query: str):
    """Tavily API wrapper with error handling"""
    return search_client.search(query)
```

**Output Schema**:
```python
class AllSearchResults(BaseModel):
    results: List[SingleSearchResult]

class SingleSearchResult(BaseModel):
    title: str
    url: str
    score: float  # Quality metric
    content: str  # Page snippet
```

**Quality Assurance**:
- Score-based filtering removes low-quality results
- Suspicious URL detection
- Content relevance validation

#### Product Extraction Agent
**Purpose**: Scrape and structure product data

**Design Decisions**:
- ScrapeGraph AI for intelligent extraction
- Handles diverse e-commerce layouts
- Extracts pricing, specs, images
- Agent provides recommendation ranking

**Tool**: `scrape_website_tool`
```python
@tool
def scrape_website_tool(page_url: str, required_fields: list):
    """ScrapeGraph wrapper with structured extraction"""
    return scrape_client.smartscraper(
        website_url=page_url,
        user_prompt=f"Extract {json.dumps(required_fields)}"
    )
```

**Output Schema**:
```python
class SingleExtractedProduct(BaseModel):
    page_url: str
    product_title: str
    product_image_url: str
    product_url: str
    product_current_price: float
    product_original_price: Optional[float]
    product_discount_percentage: Optional[float]
    product_specs: List[ProductSpec]  # Top 5 specs
    agent_recommendation_rank: int  # 1-5 ranking
    agent_recommendation_notes: List[str]
```

**Intelligent Features**:
- Automatic price comparison
- Discount calculation
- Specification prioritization (top 5)
- AI-generated recommendation notes

#### Procurement Report Agent
**Purpose**: Generate professional HTML reports

**Design Decisions**:
- Bootstrap CSS for professional styling
- Structured sections (Executive Summary, Analysis, etc.)
- Company context integration
- Data visualization (tables, charts)

**Report Structure**:
1. Executive Summary
2. Introduction
3. Methodology
4. Findings (price comparison tables)
5. Analysis (trends, insights)
6. Recommendations
7. Conclusion
8. Appendices (raw data)

### 2. Data Flow

```
User Input
    ↓
[Agent 1: Query Generation]
    ↓ (JSON: queries)
[Agent 2: Web Search]
    ↓ (JSON: search results)
[Agent 3: Data Extraction]
    ↓ (JSON: structured products)
[Agent 4: Report Generation]
    ↓ (HTML: final report)
Output Files
```

**State Passing**:
- Each agent writes to a JSON file
- Sequential process ensures data availability
- File-based state enables debugging and retries
- Pydantic validation ensures schema compliance

### 3. LLM Integration

**Provider**: Groq (Llama 3.1 8B Instant)

**Configuration**:
```python
basic_llm = LLM(
    model="groq/llama-3.1-8b-instant",
    provider="groq",
    temperature=0.7  # Balanced creativity/consistency
)
```

**Why Groq?**:
- **Speed**: ~500 tokens/sec (10x faster than OpenAI)
- **Cost**: ~$0.01 per million tokens
- **Quality**: Competitive with GPT-3.5
- **Reliability**: High uptime SLA

**Prompt Engineering**:
- System prompts define agent roles/goals
- Few-shot examples (when needed)
- Structured output enforcement via Pydantic
- Temperature tuning per agent (0.7 for creativity)

### 4. Knowledge Management

**Company Context Injection**:
```python
company_context = StringKnowledgeSource(
    content="Yousef's company provides AI solutions..."
)

Crew(
    knowledge_sources=[company_context],
    # Agents can access this context
)
```

**Benefits**:
- Personalized reports
- Industry-specific recommendations
- Consistent terminology
- Brand voice preservation

### 5. Monitoring & Observability

**AgentOps Integration**:
```python
agentops.init(
    api_key="...",
    skip_auto_end_session=True,
    default_tags=['crewai']
)
```

**Tracked Metrics**:
- Agent execution time
- LLM token usage
- Tool call success/failure rates
- Error traces
- Cost per run

**Benefits**:
- Real-time debugging
- Performance optimization
- Cost tracking
- Quality assurance

## Design Patterns

### 1. Sequential Processing Pattern
```python
Crew(
    agents=[agent1, agent2, agent3, agent4],
    tasks=[task1, task2, task3, task4],
    process=Process.sequential  # Tasks execute in order
)
```

**Rationale**:
- Data dependencies (agent N needs agent N-1's output)
- Easier debugging
- Predictable execution flow
- State management simplicity

### 2. Tool Abstraction Pattern
```python
@tool
def wrapper_tool(params):
    """Abstraction over external API"""
    return external_api.call(params)
```

**Benefits**:
- Swappable implementations
- Error handling centralization
- Mocking for tests
- Rate limiting enforcement

### 3. Structured Output Pattern
```python
Task(
    output_json=PydanticModel,
    output_file="path/to/output.json"
)
```

**Benefits**:
- Type safety
- Validation at runtime
- Consistent data formats
- Easier downstream processing

### 4. Prompt Template Pattern
```python
Task(
    description="""
    Task: {action}
    Context: {context}
    Constraints: {constraints}
    Expected Output: {output_format}
    """
)
```

**Benefits**:
- Reusable prompts
- Consistent agent behavior
- Easy prompt iteration
- Clear agent instructions

## Performance Considerations

### Latency Optimization
1. **Fast LLM**: Groq's hardware acceleration
2. **Parallel Tool Calls**: Where dependencies allow
3. **Caching**: (Future) Cache search results
4. **Result Limits**: Max 10 queries, 5 specs

### Cost Optimization
1. **Token Efficiency**: Concise prompts
2. **Smart Filtering**: Remove low-quality results early
3. **Groq Pricing**: $0.01/M tokens vs $0.50/M (GPT-4)
4. **API Call Reduction**: Batch where possible

### Error Handling
```python
try:
    result = agent.execute(task)
except Exception as e:
    logger.error(f"Agent failed: {e}")
    # Graceful degradation
    return default_result
```

**Strategies**:
- Retry logic (future enhancement)
- Fallback to cached data
- Partial result returns
- User-friendly error messages

## Security Considerations

### API Key Management
- Environment variables (not hardcoded)
- `.env` file (git-ignored)
- Key rotation support
- Minimal permission scopes

### Data Privacy
- No PII storage
- Secure API communication (HTTPS)
- Local file system for outputs
- No external data logging

### Input Validation
```python
class TaskInput(BaseModel):
    product_name: str = Field(..., max_length=200)
    websites_list: List[str] = Field(..., max_items=10)
    # Pydantic enforces constraints
```

## Scalability

### Current Limitations
- Sequential processing (no parallelization)
- File-based state (not database)
- Single-threaded execution
- Local output storage

### Future Scaling Strategies
1. **Parallel Agents**: Task-level parallelism where independent
2. **Database Integration**: MongoDB/PostgreSQL for state
3. **Queue System**: Celery/RabbitMQ for async tasks
4. **Distributed Deployment**: Kubernetes cluster
5. **Caching Layer**: Redis for API responses

## Testing Strategy

### Unit Tests
```python
def test_query_generation():
    agent = QueryAgent(...)
    result = agent.generate_queries("coffee machine")
    assert len(result.queries) <= 10
    assert all(isinstance(q, str) for q in result.queries)
```

### Integration Tests
```python
def test_full_pipeline():
    crew = create_crew()
    result = crew.kickoff(test_inputs)
    assert os.path.exists("ai-agent-output/step4_*.html")
```

### Mock External APIs
```python
@patch('src.tools.search_tool.search_client')
def test_search_agent(mock_search):
    mock_search.search.return_value = mock_data
    # Test agent logic without real API calls
```

## Deployment Architecture

### Local Development
```
Jupyter Notebook → Python Runtime → Local File System
```

### Production (Proposed)
```
FastAPI Server → CrewAI → Redis Cache → PostgreSQL
     ↓
Celery Workers → External APIs → S3 Storage
     ↓
Monitoring (AgentOps, DataDog)
```

## Extensibility Points

### Adding New Agents
1. Define Pydantic output model
2. Create agent with role/goal/backstory
3. Create task with description/expected output
4. Add to crew's agent and task lists
5. Update data flow diagram

### Adding New Tools
1. Implement `@tool` decorated function
2. Add to agent's `tools` list
3. Update tool documentation
4. Add integration tests

### Changing LLM Provider
```python
# Current: Groq
llm = LLM(model="groq/llama-3.1-8b-instant")

# Switch to: OpenAI
llm = LLM(model="openai/gpt-4")

# Switch to: Anthropic
llm = LLM(model="anthropic/claude-3-sonnet")
```

## References

- [CrewAI Documentation](https://docs.crewai.com/)
- [Multi-Agent Systems (Wikipedia)](https://en.wikipedia.org/wiki/Multi-agent_system)
- [Pydantic Documentation](https://docs.pydantic.dev/)
- [Groq API Reference](https://console.groq.com/docs/)
- [AgentOps Documentation](https://docs.agentops.ai/)

---

**This architecture demonstrates production-ready AI engineering with a focus on modularity, observability, and extensibility.**
