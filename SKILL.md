---
name: a2ui-adk
description: Build agents that generate rich, interactive UIs using Google's A2UI (Agent-to-User Interface) format with Google ADK (Agent Development Kit). Use when building ADK agents that need to render dynamic UI components, forms, cards, lists, or interactive elements via the A2UI declarative JSON format. Covers A2UI schema management, catalog configuration, A2A transport integration, response parsing/validation, and the orchestrator pattern for multi-agent UI routing.
---

# A2UI + ADK Skill

Build Google ADK agents that generate rich, interactive UIs using the A2UI declarative JSON format.

## What is A2UI?

**A2UI (Agent-to-User Interface)** is an open standard (Apache 2.0, github.com/google/A2UI) that lets agents "speak UI." Agents send declarative JSON describing UI intent; client apps render it using native components (Flutter, Angular, Lit, React, etc.).

Core principles:
- **Security first**: Declarative data format, not executable code. Clients maintain a catalog of trusted, pre-approved UI components.
- **LLM-friendly**: Flat list of components with ID references, easy for LLMs to generate incrementally.
- **Framework-agnostic**: Same JSON payload renders on any supported client framework.
- **Currently v0.8 (Public Preview)**, v0.9 also available.

## Architecture Overview

```
Agent (ADK + A2UI SDK) → A2UI JSON Response → Transport (A2A/AG UI) → Client Renderer (Lit/Flutter/etc.)
```

1. **Generation**: Agent uses LLM to generate A2UI JSON payload
2. **Transport**: Sent via A2A protocol or AG UI
3. **Resolution**: Client's A2UI Renderer parses the JSON
4. **Rendering**: Maps abstract components to native widgets

## Key Dependencies

```toml
[project]
dependencies = [
    "a2a-sdk>=0.3.0",
    "google-adk>=1.8.0",
    "google-genai>=1.27.0",
    "litellm",
    "jsonschema>=4.0.0",
    "a2ui-agent",           # The A2UI agent SDK (from agent_sdks/python in the A2UI repo)
    "python-dotenv>=1.1.0",
    "click>=8.1.8",
]
```

The `a2ui-agent` package is sourced from the A2UI repo at `agent_sdks/python` (use `uv` workspace source):
```toml
[tool.uv.sources]
a2ui-agent = { path = "../../../agent_sdks/python", editable = true }
```

## Core A2UI SDK Imports

```python
# Schema management
from a2ui.core.schema.constants import VERSION_0_8, VERSION_0_9, A2UI_OPEN_TAG, A2UI_CLOSE_TAG
from a2ui.core.schema.manager import A2uiSchemaManager
from a2ui.core.schema.common_modifiers import remove_strict_validation

# Catalog (built-in component catalog)
from a2ui.basic_catalog.provider import BasicCatalog

# Response parsing
from a2ui.core.parser.parser import parse_response, ResponsePart

# A2A integration helpers
from a2ui.a2a import (
    create_a2ui_part,        # Wrap A2UI JSON as A2A DataPart
    is_a2ui_part,            # Check if an A2A Part contains A2UI data
    get_a2ui_agent_extension,# Create AgentExtension for agent card
    parse_response_to_parts, # Parse LLM response → list of A2A Parts
    try_activate_a2ui_extension, # Activate A2UI extension from request context
    A2UI_EXTENSION_URI,
)
```

## Pattern 1: Single Agent with A2UI (e.g., Restaurant Finder)

### Step 1: Schema Manager Setup

```python
from a2ui.core.schema.manager import A2uiSchemaManager
from a2ui.core.schema.constants import VERSION_0_8
from a2ui.basic_catalog.provider import BasicCatalog
from a2ui.core.schema.common_modifiers import remove_strict_validation

schema_manager = A2uiSchemaManager(
    VERSION_0_8,
    catalogs=[
        BasicCatalog.get_config(version=VERSION_0_8, examples_path="examples")
    ],
    schema_modifiers=[remove_strict_validation],
)
```

- `examples_path` points to a directory of JSON example files used for few-shot prompting
- `remove_strict_validation` strips `additionalProperties: false` to make LLM generation easier

### Step 2: Build the ADK LlmAgent

```python
from google.adk.agents.llm_agent import LlmAgent
from google.adk.models.lite_llm import LiteLlm

ROLE_DESCRIPTION = "You are a helpful assistant. Your final output MUST be an A2UI UI JSON response."
UI_DESCRIPTION = "Describe when to use which template/layout..."

instruction = schema_manager.generate_system_prompt(
    role_description=ROLE_DESCRIPTION,
    ui_description=UI_DESCRIPTION,
    include_schema=True,       # Injects the JSON schema into the prompt
    include_examples=True,     # Injects example A2UI payloads
    validate_examples=True,    # Validates examples against schema at startup
)

agent = LlmAgent(
    model=LiteLlm(model="gemini/gemini-2.5-flash"),
    name="my_agent",
    description="Agent description",
    instruction=instruction,
    tools=[my_tool_function],
)
```

### Step 3: Agent Class with Streaming + Validation

```python
from google.adk.runners import Runner
from google.adk.artifacts import InMemoryArtifactService
from google.adk.memory.in_memory_memory_service import InMemoryMemoryService
from google.adk.sessions import InMemorySessionService
from google.genai import types
from a2ui.core.parser.parser import parse_response
from a2ui.a2a import parse_response_to_parts

class MyAgent:
    def __init__(self, base_url: str, use_ui: bool = False):
        self.use_ui = use_ui
        self._schema_manager = A2uiSchemaManager(...) if use_ui else None
        self._agent = self._build_agent(use_ui)
        self._runner = Runner(
            app_name=self._agent.name,
            agent=self._agent,
            artifact_service=InMemoryArtifactService(),
            session_service=InMemorySessionService(),
            memory_service=InMemoryMemoryService(),
        )

    async def stream(self, query, session_id):
        # Create/get session
        session = await self._runner.session_service.get_session(...)
        if session is None:
            session = await self._runner.session_service.create_session(...)

        # Run agent with retry on validation failure
        max_retries = 1
        for attempt in range(max_retries + 1):
            message = types.Content(role="user", parts=[types.Part.from_text(text=query)])
            final_response = None

            async for event in self._runner.run_async(
                user_id=self._user_id, session_id=session.id, new_message=message
            ):
                if event.is_final_response():
                    final_response = "\n".join(
                        [p.text for p in event.content.parts if p.text]
                    )
                    break
                else:
                    yield {"is_task_complete": False, "updates": "Processing..."}

            # Validate A2UI JSON if using UI
            if self.use_ui and final_response:
                try:
                    response_parts = parse_response(final_response)
                    for part in response_parts:
                        if part.a2ui_json:
                            selected_catalog = self._schema_manager.get_selected_catalog()
                            selected_catalog.validator.validate(part.a2ui_json)
                    # Valid! Send final response
                    yield {
                        "is_task_complete": True,
                        "parts": parse_response_to_parts(final_response, fallback_text="OK."),
                    }
                    return
                except Exception as e:
                    if attempt < max_retries:
                        query = f"Previous response invalid: {e}. Retry with valid A2UI JSON."
                        continue
                    # Exhausted retries, send error
                    yield {"is_task_complete": True, "parts": [Part(root=TextPart(text="Error generating UI."))]}
                    return
            else:
                yield {"is_task_complete": True, "parts": parse_response_to_parts(final_response)}
                return
```

### Step 4: Agent Executor (A2A Server)

The executor bridges A2A protocol to your agent. Key pattern: maintain both a UI agent and a text-only agent, selecting based on whether the A2UI extension is active.

```python
from a2a.server.agent_execution import AgentExecutor, RequestContext
from a2a.server.events import EventQueue
from a2a.server.tasks import TaskUpdater
from a2a.types import DataPart, Part, TaskState, TextPart
from a2a.utils import new_agent_parts_message, new_agent_text_message, new_task
from a2ui.a2a import try_activate_a2ui_extension

class MyAgentExecutor(AgentExecutor):
    def __init__(self, ui_agent, text_agent):
        self.ui_agent = ui_agent
        self.text_agent = text_agent

    async def execute(self, context: RequestContext, event_queue: EventQueue):
        use_ui = try_activate_a2ui_extension(context)
        agent = self.ui_agent if use_ui else self.text_agent

        # Extract query from message parts
        query = context.get_user_input()

        # Handle A2UI user actions (button clicks, form submissions)
        for part in (context.message.parts or []):
            if isinstance(part.root, DataPart) and "userAction" in part.root.data:
                ui_event = part.root.data["userAction"]
                action = ui_event.get("actionName")
                ctx = ui_event.get("context", {})
                query = f"User action: {action} with data: {ctx}"
                break

        task = context.current_task or new_task(context.message)
        if not context.current_task:
            await event_queue.enqueue_event(task)
        updater = TaskUpdater(event_queue, task.id, task.context_id)

        async for item in agent.stream(query, task.context_id):
            if not item["is_task_complete"]:
                await updater.update_status(
                    TaskState.working,
                    new_agent_text_message(item["updates"], task.context_id, task.id),
                )
            else:
                await updater.update_status(
                    TaskState.input_required,  # or TaskState.completed
                    new_agent_parts_message(item["parts"], task.context_id, task.id),
                    final=True,
                )
                break
```

### Step 5: Server Entry Point (__main__.py)

```python
import click, os, logging, uvicorn
from a2a.server.apps import A2AStarletteApplication
from a2a.server.request_handlers import DefaultRequestHandler
from a2a.server.tasks import InMemoryTaskStore
from dotenv import load_dotenv
from starlette.middleware.cors import CORSMiddleware

load_dotenv()

@click.command()
@click.option("--host", default="localhost")
@click.option("--port", default=10002)
def main(host, port):
    base_url = f"http://{host}:{port}"
    ui_agent = MyAgent(base_url=base_url, use_ui=True)
    text_agent = MyAgent(base_url=base_url, use_ui=False)

    executor = MyAgentExecutor(ui_agent, text_agent)
    handler = DefaultRequestHandler(agent_executor=executor, task_store=InMemoryTaskStore())
    server = A2AStarletteApplication(agent_card=ui_agent.get_agent_card(), http_handler=handler)

    app = server.build()
    app.add_middleware(CORSMiddleware, allow_origin_regex=r"http://localhost:\d+",
                       allow_credentials=True, allow_methods=["*"], allow_headers=["*"])
    uvicorn.run(app, host=host, port=port)

if __name__ == "__main__":
    main()
```

### Step 6: Agent Card with A2UI Extension

```python
from a2a.types import AgentCapabilities, AgentCard, AgentSkill
from a2ui.a2a import get_a2ui_agent_extension

def get_agent_card(self) -> AgentCard:
    return AgentCard(
        name="My Agent",
        description="Agent description",
        url=self.base_url,
        version="1.0.0",
        default_input_modes=["text", "text/plain"],
        default_output_modes=["text", "text/plain"],
        capabilities=AgentCapabilities(
            streaming=True,
            extensions=[
                get_a2ui_agent_extension(
                    self._schema_manager.accepts_inline_catalogs,
                    self._schema_manager.supported_catalog_ids,
                )
            ],
        ),
        skills=[AgentSkill(id="my_skill", name="My Skill", description="...", tags=["tag"], examples=["example query"])],
    )
```

## Pattern 2: Orchestrator with Multi-Agent UI Routing

For orchestrating multiple sub-agents that each produce A2UI, use:

### Remote Sub-Agent Discovery

```python
from a2a.client import A2ACardResolver
from google.adk.agents.remote_a2a_agent import RemoteA2aAgent
from google.adk.agents.llm_agent import LlmAgent
from google.adk.planners.built_in_planner import BuiltInPlanner

# Discover sub-agents via A2A card resolution
async with httpx.AsyncClient() as client:
    resolver = A2ACardResolver(httpx_client=client, base_url=subagent_url)
    subagent_card = await resolver.get_agent_card()

# Create remote agent wrapper
remote_agent = RemoteA2aAgent(
    name, subagent_card,
    description=json.dumps({...}),
    a2a_part_converter=convert_a2a_part_to_genai_part,
    genai_part_converter=convert_genai_part_to_a2a_part,
    a2a_client_factory=A2AClientFactoryWithA2UIMetadata(...)
)

# Build orchestrator
orchestrator = LlmAgent(
    model=LiteLlm(model="gemini/gemini-2.5-flash"),
    name="orchestrator_agent",
    instruction="Route tasks to the appropriate subagent.",
    sub_agents=[remote_agent_1, remote_agent_2],
    planner=BuiltInPlanner(thinking_config=genai_types.ThinkingConfig(include_thoughts=True)),
    before_model_callback=programmatic_route_user_action,  # Route UI actions to correct sub-agent
)
```

### Part Converters (A2A <-> GenAI)

When A2UI parts pass through the orchestrator, they need conversion between A2A and GenAI formats:

```python
from a2ui.a2a import is_a2ui_part

def convert_a2a_part_to_genai_part(a2a_part):
    """Serialize A2UI DataParts to text for LLM context."""
    if is_a2ui_part(a2a_part):
        return genai_types.Part(text=a2a_part.model_dump_json())
    return part_converter.convert_a2a_part_to_genai_part(a2a_part)

def convert_genai_part_to_a2a_part(part):
    """Deserialize A2UI text back to DataParts."""
    if part.text:
        try:
            a2a_part = a2a_types.Part.model_validate_json(part.text)
            if is_a2ui_part(a2a_part):
                return a2a_part
        except pydantic.ValidationError:
            pass
    return part_converter.convert_genai_part_to_a2a_part(part)
```

### Surface-to-Subagent Routing

Route user actions to the correct sub-agent based on `surfaceId`:

```python
# When a sub-agent sends a beginRendering with a surfaceId, save the mapping
if begin_rendering := a2ui_data.get("beginRendering"):
    surface_id = begin_rendering.get("surfaceId")
    SubagentRouteManager.set_route_to_subagent_name(surface_id, agent_name, ...)

# When a userAction arrives, look up which sub-agent owns that surface
if user_action := a2ui_data.get("userAction"):
    surface_id = user_action.get("surfaceId")
    target_agent = SubagentRouteManager.get_route_to_subagent_name(surface_id, state)
    # Programmatically route via transfer_to_agent function call
```

### A2UI Metadata Interceptor

Pass A2UI extension headers and client capabilities to remote agents:

```python
from a2a.client.middleware import ClientCallInterceptor
from a2a.extensions.common import HTTP_EXTENSION_HEADER

class A2UIMetadataInterceptor(ClientCallInterceptor):
    async def intercept(self, method_name, request_payload, http_kwargs, agent_card, context):
        if context and context.state and context.state.get("use_ui"):
            http_kwargs["headers"] = {HTTP_EXTENSION_HEADER: A2UI_EXTENSION_URI}
            # Add client capabilities to message metadata
            if (params := request_payload.get("params")) and (message := params.get("message")):
                message.setdefault("metadata", {})[A2UI_CLIENT_CAPABILITIES_KEY] = context.state.get("client_capabilities")
        return request_payload, http_kwargs
```

## A2UI JSON Format Quick Reference

A2UI responses are wrapped in `<a2ui-json>` and `</a2ui-json>` tags within LLM output. The JSON follows the A2UI schema with these key message types:

- **`beginRendering`**: Start a new UI surface (with `surfaceId`)
- **`surfaceUpdate`**: Update components within a surface
- **`dataModelUpdate`**: Update data bindings
- **`endRendering`**: Signal rendering is complete

Components use the BasicCatalog types: `Text`, `Card`, `Button`, `TextField`, `Image`, `Column`, `Row`, `Grid`, etc.

## A2UI Data Part Format (A2A Transport)

A2UI data in A2A uses `DataPart` with metadata:
```python
Part(root=DataPart(
    data={"surfaceUpdate": {...}},
    metadata={"mimeType": "application/json+a2ui"}
))
```

The `mimeType: "application/json+a2ui"` is the discriminator for identifying A2UI parts.

## User Actions (Client Events)

When users interact with rendered UI (button clicks, form submissions), the client sends back:
```json
{
  "userAction": {
    "actionName": "book_restaurant",
    "surfaceId": "main-surface",
    "context": {
      "restaurantName": "Example Restaurant",
      "address": "123 Main St"
    }
  }
}
```

The executor extracts these from `DataPart.data["userAction"]` and converts them into natural language queries for the LLM.

## Running Samples

```bash
# Clone A2UI repo
git clone https://github.com/google/A2UI.git && cd A2UI

# Set API key
export GEMINI_API_KEY="your_key"

# Run any ADK sample
cd samples/agent/adk/restaurant_finder  # or contact_lookup, rizzcharts, etc.
uv run .

# Run the web client (separate terminal)
cd renderers/markdown/markdown-it && npm install && npm run build
cd ../../web_core && npm install && npm run build
cd ../lit && npm install && npm run build
cd ../../samples/client/lit/shell && npm install && npm run dev
```

## File Structure for a New A2UI ADK Agent

```
my_agent/
  __init__.py
  __main__.py          # Server entry point (click CLI, uvicorn)
  agent.py             # Agent class with schema manager, LLM agent, stream method
  agent_executor.py    # A2A AgentExecutor bridging protocol to agent
  tools.py             # ADK tool functions
  prompt_builder.py    # Role/UI description constants, prompt generation
  pyproject.toml       # Dependencies including a2ui-agent
  examples/            # A2UI JSON example files for few-shot prompting
  .env.example         # GEMINI_API_KEY template
```
