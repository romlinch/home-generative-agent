# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Project Overview

**home-generative-agent** is a Home Assistant integration that implements a generative AI agent using LangGraph and LangChain. The agent interacts with Home Assistant to understand home context, create automations, analyze images from cameras, and maintain semantic memory of preferences. It supports multiple LLM providers (OpenAI, Ollama, Google Gemini) for different purposes with edge-based inference where possible.

## Development Commands

### Setup & Installation
```bash
scripts/setup          # Install Python dependencies from requirements.txt
```

### Linting & Code Quality
```bash
scripts/lint           # Format code with ruff and fix linting issues
                       # Runs: ruff format . && ruff check . --fix
```

### Running Development Environment
```bash
scripts/develop        # Start Home Assistant in debug mode with custom component
                       # Configures PYTHONPATH to load from custom_components/
                       # Config and cache stored in local ./config directory
```

### Running Tests
This project does not have a test suite setup. Contributions should be validated against Home Assistant's core using the devcontainer environment.

To test changes:
1. Use `scripts/develop` to run a local HA instance
2. Configure the integration in the UI (Settings > Devices & Services)
3. Test manually through conversation or automation creation

### Manual Code Checks (CI Pipeline)
```bash
python3 -m ruff check .          # Lint with strict rules (from .ruff.toml)
python3 -m ruff format . --check # Check formatting without changes
```

## Code Architecture

### Directory Structure

```
custom_components/home_generative_agent/
├── __init__.py                 # Integration initialization, async setup
├── conversation.py             # Conversation agent implementation
├── config_flow.py             # UI configuration flow
├── const.py                   # Configuration constants & defaults
├── manifest.json              # Integration metadata
├── services.yaml              # Service definitions
├── image.py                   # Image entity platform
├── sensor.py                  # Sensor entity platform
├── agent/                     # LangGraph agent orchestration
│   ├── graph.py              # State machine & workflow definition
│   ├── tools.py              # LangChain tools (camera, memory, automation)
│   └── token_counter.py       # LLM token counting utilities
├── core/                      # Core functionality
│   ├── runtime.py             # Runtime data structures
│   ├── video_analyzer.py      # Video analysis orchestration
│   ├── video_helpers.py       # Video processing utilities
│   ├── image_entity.py        # Image entity helpers
│   ├── recognized_sensor.py   # Recognized people sensor
│   ├── person_gallery.py      # Face recognition integration
│   ├── utils.py               # Shared utilities
│   └── migrations.py          # Database migrations
config/                        # Development configuration
└── blueprints/                # Home Assistant automation blueprints
    ├── hga_scene_analysis.yaml
    └── hga_summary.yaml
```

### Key Architectural Concepts

#### LangGraph Workflow
The agent uses a state machine with 5 nodes:
- **`__start__` / `__end__`**: Graph entry/exit points
- **`agent`**: Primary LLM decision-making node; calls `llm.invoke()`
- **`action`**: Tool execution node; runs selected tools and returns results
- **`summarize_and_remove_messages`**: Context management; trims old messages if needed

Edges between nodes form conditional paths:
- Solid edges: unconditional transitions
- Dashed edges: conditional based on agent decision (has tool to call or needs summarization)

#### State Management
`State` class in `agent/graph.py` extends `MessagesState`:
- `messages`: Conversation history (LangChain message objects)
- `summary`: Compressed context from trimmed messages
- `chat_model_usage_metadata`: Token tracking across providers
- `messages_to_remove`: Messages marked for deletion during context cleanup

#### Context Length Management
LLM context is managed via configurable limits in `const.py`:
- `CONTEXT_MAX_MESSAGES`: Number of messages before trimming (default: 80)
- `CONTEXT_MAX_TOKENS`: Token limit before trimming (default: 25600)
- `CONTEXT_MANAGE_USE_TOKENS`: Use token-based vs message-based management (default: True)

When limits exceeded, old messages are summarized and removed, with summary prepended to system message.

#### Multi-Model Architecture
Different models handle different tasks:
- **Chat Model** (OpenAI GPT-4o / Ollama Qwen3): High-level reasoning
- **Vision Model** (Ollama Qwen2.5VL): Camera image analysis
- **Summarization Model** (Ollama Qwen3:1.7b): Context summarization
- **Embedding Model** (Ollama mxbai-embed-large): Semantic search vectors

Models can be cloud-based or edge-based (running locally via Ollama).

#### Database & Memory
- **PostgreSQL + pgvector**: Persistent storage for conversation history and embeddings
- **AsyncPostgresSaver**: LangGraph checkpoint storage for workflow state
- **AsyncPostgresStore**: Vector store for semantic memory and video analysis results

#### Tools Available to Agent
Tools are defined in `agent/tools.py` using LangChain's `@tool` decorator:
- `get_and_analyze_camera_image`: Analyzes camera feeds with vision model
- `upsert_memory`: Stores/updates semantic memories with vector embedding
- `add_automation`: Creates Home Assistant automations from agent reasoning
- `get_entity_history`: Queries HA database for entity state history

Home Assistant's native intents (e.g., `HassTurnOn`, `HassTurnOff`) are also available via the LLM API.

### Module Dependencies

**Key External Libraries:**
- `langchain` / `langgraph`: Agent framework and orchestration
- `langchain-openai` / `langchain-ollama` / `langchain-google-genai`: Model providers
- `psycopg`: PostgreSQL async driver with pgvector support
- `homeassistant`: Core HA framework
- `aiofiles`: Async file operations
- `transformers`: ML model utilities
- `ollama`: Local inference runtime client

**Home Assistant Integration Points:**
- `conversation` platform: User interaction through HA UI
- `recorder`: Database access for entity history
- `camera`: Camera image capture
- `automation`: Automation creation and management
- `LLM API`: Native tool definitions and context

## Configuration Flow

Configuration happens in `config_flow.py`:
1. User provides API keys (OpenAI, Gemini) and Ollama URL during setup
2. Database URI for PostgreSQL is configured
3. Optional face recognition service URL
4. Model selection with recommended defaults
5. Database is bootstrapped on first run (migrations applied)

Configuration is stored in Home Assistant's config entries and accessed via `hass.data[DOMAIN]`.

## Common Development Patterns

### Adding a New Tool
1. Create async function in `agent/tools.py` with `@tool` decorator
2. Include docstring describing parameters (used for LLM context)
3. Use `InjectedToolArg` for injected dependencies (hass, config, store)
4. Return `str` output (LLM-readable results)
5. Tool automatically registered when graph is initialized

### Modifying LLM Prompts
System prompts, summarization templates, and tool instructions are in `const.py`:
- `SUMMARIZATION_SYSTEM_PROMPT`: System message for summarization model
- `SUMMARIZATION_PROMPT_TEMPLATE`: Template for messages to summarize
- `VLM_SYSTEM_PROMPT`: Vision model system instruction
- Individual tool docstrings define tool usage instructions

### Working with Camera Analysis
Vision model analysis happens in `agent/tools.py`:
1. `_get_camera_image()` fetches latest snapshot from camera entity
2. Image encoded as base64 and sent to vision model
3. Prompt instructs model to analyze scene and summarize findings
4. Results stored in memory via `upsert_memory()` tool

Video analyzer runs async in `core/video_analyzer.py`, triggered by motion events.

## Code Quality Standards

### Linting Rules
- **Target Python**: 3.13+
- **Formatter**: ruff (opinionated, auto-fixes most issues)
- **Max Complexity**: 25 (McCabe complexity)
- **Ignored Rules**: D203, D212, COM812, ISC001, ANN401 (ruff incompatibilities)

Run `scripts/lint` before committing code.

### Naming Conventions
- **Constants**: `UPPERCASE_WITH_UNDERSCORES`
- **Functions/Methods**: `snake_case`
- **Async functions**: Prefix with `async def`, suffix with `_async` if disambiguation needed
- **Private functions**: Prefix with `_`
- **Module variables**: Descriptive `snake_case`

### Type Hints
- Use type hints throughout (required by Home Assistant standards)
- Use `from __future__ import annotations` at top of modules
- Use `TYPE_CHECKING` block for forward references avoiding circular imports
- Specify return types and parameter types

### Error Handling
- Catch specific exceptions, avoid bare `except:`
- Log exceptions with context before re-raising
- Use `HomeAssistantError` for HA-specific errors
- Tool execution errors are caught in `action` node and agent retries with error context

## Testing Approach

No unit test framework is configured. Instead:

1. **Manual Integration Testing**: Use `scripts/develop` to run HA instance with component
2. **Configuration Testing**: Test config flow with various provider/model combinations
3. **Tool Testing**: Invoke agent through conversation UI to test tool execution
4. **Linting Verification**: `scripts/lint` ensures code quality before commit

Home Assistant community expects HACS-compliant code following core standards (checked in CI).

## Database Considerations

PostgreSQL with pgvector extension is required:
- Vector embeddings stored for semantic search
- Conversation history and workflow state persisted
- Migrations auto-run on startup (`core/migrations.py`)
- Connection pooling via `AsyncConnectionPool` for concurrent access

For development, install the PostgreSQL addon from: https://github.com/goruck/addon-postgres-pgvector

## Important Notes

- **Home Assistant Compliance**: Code must follow [Home Assistant Core standards](https://developers.home-assistant.io/docs/development_index/)
- **HACS Requirements**: Integration is published to HACS; maintain compatibility
- **Async-First**: All I/O operations must be async (LangGraph requires async context)
- **Context Preservation**: Prompts and examples are positioned upfront to enable LLM prompt caching
- **Token Efficiency**: Agent uses token counting to balance cost, accuracy, and latency
- **Edge vs Cloud**: Design allows flexible model provider selection; defaults balance performance/cost

## Key Files to Know

- **`const.py`**: Configuration constants and defaults—start here for tuning
- **`agent/graph.py`**: Core workflow state machine—understand this for agent behavior
- **`agent/tools.py`**: Tool implementations—extend here for new capabilities
- **`conversation.py`**: Integration with HA conversation platform—entry point for user interactions
- **`__init__.py`**: Setup logic, model initialization, database connection pooling
