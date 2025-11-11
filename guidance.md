# AI Agents Project - Intern Guidance

## Project Overview

This project is a multi-agent AI shopping assistant built with Azure AI Foundry. It demonstrates how to build intelligent, agentic applications using Azure's AI SDKs. The application uses a **multi-agent orchestration pattern** with specialized agents handling different tasks (interior design recommendations, customer loyalty calculations, inventory management, and general chat).

## Key Technologies & SDKs

### 1. **Azure AI Agents SDK** (`azure-ai-agents==1.2.0b5`)
This is the primary SDK for creating and managing AI agents that can use tools and function calling.

**Key Usage:**
- **Creating Agents:** Agents are created in Azure AI Foundry portal and referenced by their IDs
- **Thread Management:** Each conversation session gets a thread for maintaining context
- **Tool Integration:** Agents can be equipped with Python functions as tools using `FunctionTool` and `ToolSet`
- **Running Agents:** Use `create_and_process()` to execute agent runs on threads

**Example from `agent_processor.py:52-79`:**
```python
from azure.ai.agents.models import FunctionTool, ToolSet

# Define tools for specific agent type
interior_functions = {create_image, product_recommendations}
functions = FunctionTool(interior_functions)

# Create toolset and enable auto function calls
toolset = ToolSet()
toolset.add(functions)
project_client.agents.enable_auto_function_calls(toolset)
```

### 2. **Azure AI Projects SDK** (`azure-ai-projects==1.1.0b4`)
Provides project-level orchestration and connection to Azure AI Foundry resources.

**Key Usage:**
- **AIProjectClient:** Main client for interacting with Azure AI Foundry
- **Credential Management:** Uses `DefaultAzureCredential` for authentication
- **Agent Operations:** Access agents, threads, messages through the project client

**Example from `chat_app.py:338-344`:**
```python
from azure.ai.projects import AIProjectClient
from azure.identity import DefaultAzureCredential

project_endpoint = os.environ.get("AZURE_AI_AGENT_ENDPOINT")
project_client = AIProjectClient(
    endpoint=project_endpoint,
    credential=DefaultAzureCredential(),
)
```

### 3. **Semantic Kernel** (`semantic-kernel==1.37.0`)
Note: While included in requirements, this SDK is not actively used in the current codebase. The project uses Azure AI Agents SDK instead for agent orchestration.

## Architecture Pattern: Multi-Agent with Handoff

The application uses a **handoff orchestration pattern** where:

1. **Handoff Agent** (Phi-4 model) routes user requests to specialized agents
2. **Specialized Agents** handle specific domains:
   - `interior_designer`: Product recommendations, style advice
   - `interior_designer_create_image`: Image generation/editing
   - `customer_loyalty`: Discount calculations
   - `inventory_agent`: Stock checking
   - `cora`: Greetings, cart management, general chat

### Flow (from `src/flow.md`):
```
User Message → Handoff Agent → Agent Selection → Specialized Agent → Response
```

## How to Create Agents

### Step 1: Define Agent Tools (Functions)

Create Python functions that agents can call. Each function becomes a tool.

**Example from `src/app/tools/aiSearchTools.py:31-67`:**
```python
def product_recommendations(question: str) -> list:
    """
    Search for products based on user query.

    Input:
        question (str): Natural language user query
    Output:
        list: Product information with name, price, description
    """
    search_client = SearchClient(
        endpoint=SEARCH_ENDPOINT,
        index_name=INDEX_NAME,
        credential=AzureKeyCredential(SEARCH_KEY)
    )

    results = search_client.search(
        search_text=question,
        query_type="semantic",
        top=8
    )

    return [format_product(item) for item in results]
```

**Key Points:**
- Functions must have clear docstrings (agents use these to understand when to call them)
- Type hints are important for proper function calling
- Return structured data (dicts/lists) that agents can interpret

### Step 2: Create Agent in Azure AI Foundry Portal

1. Go to Azure AI Foundry portal
2. Create a new agent with:
   - **Name** and **Instructions** (system prompt)
   - **Model selection** (e.g., gpt-4.1)
   - **Agent ID** (copy this for your code)

### Step 3: Register Agent in Code

**Example from `agent_processor.py:54-71`:**
```python
def _get_or_create_toolset(self, agent_type: str) -> ToolSet:
    if agent_type == "interior_designer":
        interior_functions = {create_image, product_recommendations}
        functions = FunctionTool(interior_functions)
    elif agent_type == "customer_loyalty":
        loyalty_functions = {calculate_discount}
        functions = FunctionTool(loyalty_functions)

    toolset = ToolSet()
    toolset.add(functions)
    self.project_client.agents.enable_auto_function_calls(toolset)
    return toolset
```

### Step 4: Use AgentProcessor to Run Conversations

**Example from `agent_processor.py:206-218`:**
```python
async def run_conversation_with_text_stream(self, input_message: str):
    """Run agent conversation asynchronously."""
    messages = await loop.run_in_executor(
        executor, self._run_conversation_sync, input_message
    )
    for msg in messages:
        yield msg
```

## Environment Configuration

### Required Environment Variables (from `src/env_sample.txt`):

```bash
# Azure AI Foundry
AZURE_OPENAI_ENDPOINT=""
AZURE_OPENAI_KEY=""
AZURE_AI_AGENT_ENDPOINT=""
AZURE_AI_AGENT_MODEL_DEPLOYMENT_NAME="gpt-4.1"

# Agent IDs (from portal)
interior_designer=""
customer_loyalty=""
inventory_agent=""
cora=""

# Additional Services
SEARCH_ENDPOINT=""
SEARCH_KEY=""
INDEX_NAME=""
APPLICATIONINSIGHTS_CONNECTION_STRING=""
```

## Key Patterns to Reuse

### 1. Agent Processor Pattern
The `AgentProcessor` class (`src/app/agents/agent_processor.py`) encapsulates agent interaction logic:
- Tool registration
- Thread management
- Message processing
- Caching for performance

**Reuse:** Copy this class and modify the `_get_or_create_toolset()` method for your agent types.

### 2. Agent Service Pattern
The `agent_service.py` provides caching to avoid re-initializing agents:

```python
_agent_processor_cache: Dict[str, AgentProcessor] = {}

def get_or_create_agent_processor(agent_id, agent_type, thread_id, project_client):
    cache_key = f"{agent_type}_{agent_id}"
    if cache_key in _agent_processor_cache:
        return _agent_processor_cache[cache_key]
    # Create new processor and cache it
```

**Reuse:** This pattern significantly improves performance in production.

### 3. Handoff Agent Pattern
Use a lightweight model (Phi-4) to route requests to specialized agents:

**From `src/prompts/handoffPrompt.txt`:**
```
Select the appropriate agent as per query.
interior_designer = product recommendations, style trends
customer_loyalty = discount calculations
cora = greetings, cart operations
```

**Implementation in `chat_app.py:212-243`:**
```python
def call_handoff(handoff_client, handoff_prompt, formatted_history, model):
    response = handoff_client.complete(
        messages=[
            SystemMessage(content=handoff_prompt),
            UserMessage(content=formatted_history),
        ],
        model=model
    )
    return response.choices[0].message.content
```

**Reuse:** This pattern allows cost-effective routing with a small model before invoking expensive specialized agents.

### 4. Tool Function Pattern
All tools follow this pattern:
- Clear function signature with type hints
- Comprehensive docstring (agents read this!)
- Return structured data (dict/list)
- Handle errors gracefully

**Example from `src/app/tools/inventoryCheck.py:10-80`:**
```python
def inventory_check(product_dict: dict) -> list:
    """
    Simulates checking for inventory details.

    Args:
        product_dict (dict): Keys are product names, values are product IDs.

    Returns:
        list: Product inventory information
    """
    # Implementation...
    return results
```

### 5. Observability Pattern
The project uses OpenTelemetry with Azure Monitor for tracing:

```python
from opentelemetry import trace
from azure.monitor.opentelemetry import configure_azure_monitor

configure_azure_monitor(connection_string=app_insights_conn_string)
tracer = trace.get_tracer(__name__)

with tracer.start_as_current_span("operation_name"):
    # Your code here
```

**Reuse:** Essential for production monitoring and debugging.

## Project Structure

```
src/
├── chat_app.py              # Main FastAPI app with WebSocket endpoint
├── app/
│   ├── agents/
│   │   └── agent_processor.py   # Core agent interaction logic
│   └── tools/               # Agent tools (functions)
│       ├── aiSearchTools.py     # Product search
│       ├── discountLogic.py     # Loyalty calculations
│       ├── inventoryCheck.py    # Stock checking
│       └── imageCreationTool.py # Image generation
├── services/
│   ├── agent_service.py     # Agent caching & management
│   ├── handoff_service.py   # Agent routing
│   └── fallback_service.py  # Fallback LLM calls
├── prompts/                 # Agent system prompts
│   ├── handoffPrompt.txt    # Routing logic
│   └── CoraPrompt.txt       # Chat agent instructions
└── utils/                   # Helper utilities
```

## Common Pitfalls to Avoid

1. **Forgetting to enable auto function calls:**
   ```python
   # Always call this after creating toolset
   project_client.agents.enable_auto_function_calls(tools=toolset)
   ```

2. **Poor function docstrings:** Agents rely on docstrings to understand when to call functions. Be descriptive!

3. **Not caching agent processors:** Creating new processors for each request is slow and expensive.

4. **Missing environment variables:** Always validate required env vars at startup.

5. **Not handling thread lifecycle:** Create one thread per conversation session, not per message.

## Getting Started Checklist

- [ ] Set up Azure AI Foundry project and get endpoint URL
- [ ] Create agents in portal and note their IDs
- [ ] Set up environment variables (use `src/env_sample.txt` as template)
- [ ] Install dependencies: `pip install -r src/requirements.txt`
- [ ] Create your agent tools (Python functions)
- [ ] Register tools in `AgentProcessor._get_or_create_toolset()`
- [ ] Configure handoff prompts for agent routing
- [ ] Test with FastAPI: `python src/chat_app.py`

## Additional Resources

- **Azure AI Agents Documentation:** https://learn.microsoft.com/azure/ai-services/agents/
- **Azure AI Projects SDK:** https://learn.microsoft.com/python/api/azure-ai-projects/
- **Project Docs:** See `docs/` folder for detailed workshop instructions
- **Flow Diagram:** `src/flow.md` for application flow visualization

## Questions?

Review the following key files:
1. `src/chat_app.py:396-467` - WebSocket session setup
2. `src/app/agents/agent_processor.py:42-84` - Agent initialization
3. `src/services/agent_service.py:6-21` - Agent caching pattern
4. `src/flow.md` - Complete application flow

Good luck with your project!
