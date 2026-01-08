# NeMo Agent Toolkit - AI Copilot Instructions

This file guides AI agents working on the NeMo Agent Toolkit codebase, a framework for building agentic applications.

## Project Overview

**NeMo Agent Toolkit** is a framework-agnostic Python library for composing agents, tools, and workflows. It supports multiple agentic frameworks (LangChain, LlamaIndex, CrewAI, Google ADK, Strands) and serves as a unifying layer for agent development.

Key documentation:
- [Architecture Guide](docs/source/build-workflows/workflow-configuration.md)
- [Agent Types](docs/source/components/agents/index.md)
- [Framework Support](docs/source/components/integrations/frameworks.md)

## Architecture Essentials

### Core Components

1. **Workflows** - Entry points defined in YAML using `@register_function` decorator
   - Located in `src/nat/agent/*/register.py` (ReAct, Tool Calling, ReWOO, Reasoning agents)
   - Each workflow yields a `FunctionInfo` with an async callable
   - Config classes inherit from `FunctionBaseConfig` and agent-specific base (e.g., `AgentBaseConfig`)

2. **Builder Pattern** - Central component factory (`src/nat/builder/`)
   - `Builder.get_llm()`, `Builder.get_tools()` resolve config references to instances
   - Supports dependency injection through context and async operations
   - Context includes user management, observability hooks, and authentication

3. **Tools & Functions** - Registered components (`src/nat/tool/`)
   - Decorated with `@register_function(config_type=...)` and optional `framework_wrappers`
   - Must be async and yield `FunctionInfo` instances
   - Tool inputs/outputs are Pydantic models for type safety

4. **Function Groups** - Package related functions sharing resources
   - Defined at YAML `function_groups:` level
   - Useful for tools needing shared initialization (e.g., databases, API clients)

### Data Flow Pattern

```
YAML config → Pydantic config class → @register_function decorator
→ Builder instantiates components (LLM, tools) → Async workflow returns FunctionInfo
→ Function called with ChatRequest/str → Returns ChatResponse/str
```

### Key Agent Types

| Agent Type | Use Case | Key Config |
|---|---|---|
| **ReAct** | Iterative reasoning + tool use | `tool_names`, `llm_name`, `max_tool_calls` |
| **Tool Calling** | LLM with native function calling (OpenAI, Nim) | Requires function_calling LLM |
| **ReWOO** | Planning phase then solver phase | `include_tool_input_schema_in_tool_description` |
| **Reasoning** | Wraps agents with reasoning-capable LLMs (DeepSeek-R1) | `augmented_fn` reference |

See [agent selection guide](.cursor/rules/nat-agents/general.mdc) for decision tree.

## Developer Workflows

### Running Tests

```bash
# Full test suite with coverage
pytest tests/ --cov=src/nat

# Specific test file
pytest tests/nat/test_specific.py -v
```

### Local CI Checks

```bash
# Run all pre-commit checks (linting, formatting, copyright headers)
ci/scripts/checks.sh

# Copyright header verification only
python ci/scripts/copyright.py --verify-apache-v2

# Documentation checks (Vale style guide, link validation)
ci/scripts/documentation_checks.sh
```

### Building & Installing

```bash
# Install from source with core dependencies
pip install -e .

# Install with specific framework integrations
pip install -e ".[langchain]"  # or [crewai], [adk], etc.

# Install all optional dependencies
pip install -e ".[all]"
```

### Running Examples

Examples are installed packages in `examples/<name>/` with README, configs, and `__main__.py`.

```bash
# Execute example workflow
python -m <example_name>

# Example with config override
python -m <example_name> --config-path custom.yml
```

## Code Patterns & Conventions

### Naming

- **Package**: `nvidia-nat` (PyPI) / `nat` (imports)
- **Environment variables**: `NAT_` prefix (e.g., `NAT_API_KEY`)
- **Toolkit references**: "NeMo Agent toolkit" (not "NAT" in docs)
- **Deprecated aliases**: Never use "AgentIQ", "aiqtoolkit", "Agent Intelligence" — update if found

### Workflow Registration

Pattern from [src/nat/agent/react_agent/register.py](src/nat/agent/react_agent/register.py):

```python
from nat.builder import Builder
from nat.data_models.function import FunctionInfo, FunctionBaseConfig
from nat.registry import register_function

class MyWorkflowConfig(FunctionBaseConfig, name="my_workflow"):
    llm_name: LLMRef = Field(..., description="LLM to use")
    tool_names: list[FunctionRef] = Field(default_factory=list)

@register_function(config_type=MyWorkflowConfig, framework_wrappers=[LLMFrameworkEnum.LANGCHAIN])
async def my_workflow(config: MyWorkflowConfig, builder: Builder):
    llm = await builder.get_llm(config.llm_name, wrapper_type=LLMFrameworkEnum.LANGCHAIN)
    tools = await builder.get_tools(tool_names=config.tool_names, wrapper_type=...)

    async def _response_fn(chat_request: ChatRequestOrMessage) -> ChatResponse | str:
        # Core logic
        return response

    try:
        yield FunctionInfo.from_fn(_response_fn, description=config.description)
    except GeneratorExit:
        logger.exception("Workflow exited early!")
    finally:
        # Cleanup resources
        pass
```

### Tool Registration

From [src/nat/tool/register.py](src/nat/tool/register.py):

```python
@register_function(config_type=MyToolConfig)
async def my_tool(config: MyToolConfig, builder: Builder):
    # Initialize resources
    async def _tool_fn(input_param: str) -> str:
        # Implementation
        return result

    try:
        yield FunctionInfo.from_fn(_tool_fn)
    finally:
        # Cleanup
        pass
```

### YAML Workflow Configuration

```yaml
llms:
  my_llm:
    _type: openai
    model_name: gpt-4o
    api_key: ${OPENAI_API_KEY}

functions:
  search_tool:
    _type: document_search
    description: "Search documents"
    # Tool-specific config

function_groups:
  calculator:
    _type: calculator  # Shares state/resources

workflow:
  _type: react_agent
  llm_name: my_llm
  tool_names: [search_tool, calculator]
  verbose: true
```

### Type Safety & Validation

- Use Pydantic models for all configs and data
- Leverage `FunctionRef`, `FunctionGroupRef`, `LLMRef` for component references
- `GlobalTypeConverter.get()` handles ChatRequest ↔ str conversion
- Use `ChatResponse` with `Usage` for token tracking

## Project Structure Rules

- **Source code**: `src/nat/` (namespace-packages) and `packages/nvidia_nat_*/src/`
- **Examples**: `examples/<name>/` with `src/`, `configs/`, `scripts/`, `data/` subdirs
- **Plugins/Packages**: Each in `packages/nvidia_nat_*/` with own `pyproject.toml`
- **Tests**: `tests/nat/` mirroring source structure
- **Docs**: Sphinx + Markdown in `docs/source/` with Vale style checks

## Integration Patterns

### MCP (Model Context Protocol)

NeMo Agent Toolkit acts as both MCP client and server:
- **Client**: Tools via `nat_tools` + `mcp_tools` in agent config
- **Server**: Publish workflow tools via `nat mcp server` CLI

See [MCP documentation](docs/source/build-workflows/mcp-client.md)

### Framework Wrappers

Agents can wrap frameworks (LangChain, LlamaIndex, etc.) using:
```python
@register_function(framework_wrappers=[LLMFrameworkEnum.LANGCHAIN])
```

This exposes the workflow to framework-specific integrations (A2A server, LangServe, etc.)

### Observability & Profiling

- **Profiler**: `@track_function()` decorator captures metrics
- **Phoenix/Weave/Langfuse**: Native integrations in `src/nat/observability/`
- **OpenTelemetry**: Supported for custom observability

## Quality Assurance

### Pre-commit Checks (Required)

- **Black** formatting
- **Isort** import sorting (with `.isort.cfg` rules)
- **Ruff** linting
- **Copyright headers** (Apache-2.0) via `ci/scripts/copyright.py`
- **Path checks** (no spaces, uppercase) via `ci/scripts/path_checks.sh`

### Documentation Standards

- Use "NeMo Agent toolkit" (lowercase "toolkit")
- Vale style checks in `ci/vale/` enforce terminology
- Markdown link checking via CI
- Update [CHANGELOG.md](../CHANGELOG.md) for user-facing changes (DO NOT edit with code changes)

### Testing Conventions

- Tests in `tests/` mirror source structure
- Use pytest fixtures from `tests/conftest.py`
- Test data in `tests/test_data/` (Git LFS for large files)
- Mark integration tests with `@pytest.mark.integration`

## Critical Files to Know

| File | Purpose |
|---|---|
| [pyproject.toml](pyproject.toml) | Core + optional dependencies, package metadata |
| [.cursor/rules/](/.cursor/rules/) | Detailed Cursor/AI agent rules by domain (agents, CLI, docs, etc.) |
| [src/nat/builder/builder.py](src/nat/builder/builder.py) | Builder interface & component resolution |
| [src/nat/registry.py](src/nat/) | Function registration mechanism |
| [examples/getting_started/](examples/getting_started/) | Minimal executable examples |
| [ci/scripts/checks.sh](ci/scripts/checks.sh) | Local CI validation |

## Common Pitfalls

- ❌ Forget to `yield FunctionInfo` in workflows (breaks registration)
- ❌ Use `await builder.get_tools()` synchronously (must be in async context)
- ❌ Reference undefined `tool_names` in YAML (validation happens at runtime)
- ❌ Use "NAT" or "nat" as toolkit name in docs (update naming consistently)
- ❌ Modify CHANGELOG.md directly in PRs (maintained by maintainers)
- ✅ Always use `LLMFrameworkEnum` to match framework contexts
- ✅ Provide comprehensive docstrings for config classes (used in CLI help)
- ✅ Test workflows with multiple agent types when supporting agentic patterns
