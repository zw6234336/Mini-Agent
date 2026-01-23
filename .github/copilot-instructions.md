# Mini Agent - AI Coding Agent Instructions

## Project Overview

Mini Agent is a production-ready AI agent framework showcasing best practices for building agents with MiniMax M2.1 model. It features a complete agent execution loop with persistent memory, intelligent context management, Claude Skills integration, and native MCP (Model Context Protocol) support.

**Key Components:**
- [mini_agent/agent.py](../mini_agent/agent.py) - Core agent loop with token management and context summarization
- [mini_agent/llm/llm_wrapper.py](../mini_agent/llm/llm_wrapper.py) - Unified LLM interface supporting both Anthropic and OpenAI providers
- [mini_agent/tools/](../mini_agent/tools/) - Tool implementations (file ops, bash, MCP, skills)
- [mini_agent/config.py](../mini_agent/config.py) - YAML-based configuration with Pydantic validation

## Development Workflow

### Setup & Dependencies
```bash
# Install uv package manager first (required)
curl -LsSf https://astral.sh/uv/install.sh | sh

# Sync all dependencies
uv sync

# Initialize Claude Skills submodule (optional)
git submodule update --init --recursive

# Copy and configure
cp mini_agent/config/config-example.yaml mini_agent/config/config.yaml
# Edit config.yaml with your API key and base URL
```

### Running the Agent
```bash
# Interactive CLI (primary interface)
uv run python -m mini_agent.cli

# Development mode with custom workspace
uv run python -m mini_agent.cli --workspace /path/to/workspace

# As installed tool (after `uv tool install`)
mini-agent
```

### Testing
```bash
# Run all tests
pytest tests/ -v

# Run specific tests
pytest tests/test_agent.py tests/test_tools.py -v

# With coverage
pytest tests/ -v --cov=mini_agent --cov-report=html
```

## Architecture Patterns

### Agent Execution Loop
The agent follows a strict execution pattern in [agent.py](../mini_agent/agent.py):
1. Send message history to LLM
2. Check for cancellation signals (`_check_cancelled()`)
3. Handle tool calls via async execution
4. Perform automatic context summarization when token limit exceeded
5. Repeat until task completion or max_steps reached

**Token Management:** Agent automatically summarizes conversation history when `api_total_tokens > token_limit` (default 80K) to enable infinite-length tasks. See `_generate_summary()` and `_apply_summary()` in [agent.py](../mini_agent/agent.py:350-450).

### Tool System
All tools inherit from [tools/base.py](../mini_agent/tools/base.py)::Tool with required interface:
```python
@property
def name(self) -> str
@property  
def description(self) -> str
@property
def parameters(self) -> dict[str, Any]  # JSON Schema format
async def execute(self, **kwargs) -> ToolResult
```

Tools auto-convert to both Anthropic (`to_schema()`) and OpenAI (`to_openai_schema()`) formats.

**Important:** Tool execution is asynchronous. When adding custom tools, implement `async def execute()`.

### MCP Integration
MCP tools are dynamically loaded from [mini_agent/config/mcp.json](../mini_agent/config/mcp.json). See [tools/mcp_loader.py](../mini_agent/tools/mcp_loader.py) for implementation.

**Connection Types:** Supports `stdio`, `sse`, and `streamable_http`. Timeout configuration is critical:
- `connect_timeout`: 10s (MCP server startup)
- `execute_timeout`: 60s (tool execution)  
- `sse_read_timeout`: 120s (SSE stream reading)

Set via `set_mcp_timeout_config()` in [mcp_loader.py](../mini_agent/tools/mcp_loader.py:30-50).

**Cleanup:** Always call `cleanup_mcp_connections()` on shutdown to properly close sessions.

**MCP Configuration Examples:**

STDIO connection (local MCP server):
```json
{
  "mcpServers": {
    "memory": {
      "command": "npx",
      "args": ["-y", "@modelcontextprotocol/server-memory"],
      "connect_timeout": 10.0,
      "execute_timeout": 60.0
    }
  }
}
```

URL-based connection (remote MCP server):
```json
{
  "mcpServers": {
    "search": {
      "url": "https://api.example.com/mcp",
      "type": "streamable_http",
      "headers": {
        "Authorization": "Bearer YOUR_TOKEN"
      },
      "connect_timeout": 15.0,
      "sse_read_timeout": 180.0
    }
  }
}
```

Git-based MCP server:
```json
{
  "mcpServers": {
    "custom": {
      "command": "npx",
      "args": ["-y", "git+https://github.com/user/mcp-server.git"],
      "env": {
        "API_KEY": "your_key_here"
      }
    }
  }
}
```

**Connection Flow:**
1. `load_mcp_tools_async()` reads mcp.json
2. For each enabled server, creates `MCPServerConnection`
3. Async connects with timeout protection
4. Lists available tools from server
5. Wraps each tool as `MCPTool` instance
6. Returns unified tool list

**Debugging MCP:**
- Check logs in `~/.mini-agent/log/` for connection errors
- Test with `disabled: true` to isolate problematic servers
- Increase timeouts for slow servers via per-server config
- Use MCP server's `--debug` flag if available

### Skills System
Claude Skills are loaded from [mini_agent/skills/](../mini_agent/skills/) directory (git submodule). Each skill has:
- `SKILL.md` with YAML frontmatter (name, description, allowedTools)
- Optional scripts in subdirectories
- Optional LICENSE.txt

Loader at [tools/skill_loader.py](../mini_agent/tools/skill_loader.py) processes relative paths and injects skill root directory into prompts. Skills are converted to special "skill" tool calls that append instructions to system prompt.

**Progressive Disclosure:** Skills use 3-level loading:
1. **Level 1 (Metadata):** Agent sees skill names/descriptions in system prompt
2. **Level 2 (Full Content):** Load full skill via `get_skill(skill_name)` tool
3. **Level 3+ (Resources):** Skills reference scripts/files, agent accesses via read_file/bash

**Creating Custom Skills:**
```yaml
---
name: "my-custom-skill"
description: "Brief description for agent"
### Test Structure
- All async tests use `@pytest.mark.asyncio` decorator
- Test workspace created with `tempfile.TemporaryDirectory()`
- Mock LLM responses for unit tests (see [tests/test_agent.py](../tests/test_agent.py))
- Integration tests in [tests/test_integration.py](../tests/test_integration.py) require real API key
- MCP tests in [tests/test_mcp.py](../tests/test_mcp.py) use mock servers

### Testing Patterns

**Agent Testing Pattern:**
```python
@pytest.mark.asyncio
async def test_agent_behavior():
    with tempfile.TemporaryDirectory() as workspace_dir:
        config = Config.from_yaml("mini_agent/config/config.yaml")
        llm_client = LLMClient(
            api_key=config.llm.api_key,
            api_base=config.llm.api_base,
            model=config.llm.model
        )
        tools = [ReadTool(workspace_dir), WriteTool(workspace_dir)]
        agent = Agent(llm_client, system_prompt, tools, 
                     max_steps=10, workspace_dir=workspace_dir)
        agent.add_user_message("task description")
        result = await agent.run()
        # Assert on result or workspace state
```

**Tool Testing Pattern:**
```python
@pytest.mark.asyncio
async def test_custom_tool():
    tool = MyCustomTool()
    result = await tool.execute(param1="value", param2=123)
    assert result.success
    assert "expected" in result.content
```

**MCP Testing with Timeout:**
```python
@pytest.mark.asyncio
async def test_mcp_connection():
    original = get_mcp_timeout_config()
    try:
        set_mcp_timeout_config(connect_timeout=2.0)
        tools = await load_mcp_tools_async("test-mcp.json")
        assert len(tools) > 0
    finally:
        set_mcp_timeout_config(connect_timeout=original.connect_timeout)
        await cleanup_mcp_connections()
```

### Running Tests

```bash
# Fast unit tests (no API calls)
pytest tests/test_tools.py tests/test_skill_loader.py -v

# Integration tests (requires API key)
pytest tests/test_integration.py -v

# Specific test
pytest tests/test_agent.py::test_agent_simple_task -v

# With coverage
pytest tests/ --cov=mini_agent --cov-report=term-missing

# Parallel execution
pytest tests/ -n auto
```

### Test Isolation
- Each test uses fresh workspace via `tempfile.TemporaryDirectory()`
- MCP connections cleaned up in `finally` blocks
- Config loaded per-test, not globally
- Cache directory: `workspace/.pytest_cache` (gitignored)
See reference.md for details.
```

Skills directory structure:
```
mini_agent/skills/
  my-skill/
    SKILL.md          # Required: skill definition
    scripts/          # Optional: helper scripts
    reference/        # Optional: reference docs
    LICENSE.txt       # Optional: license
```

**Path Processing:** Skill loader auto-converts relative paths to absolute:
- `scripts/file.py` → `/full/path/to/skills/my-skill/scripts/file.py`
- `[doc](reference.md)` → Link with absolute path + "use read_file" hint

### LLM Provider Abstraction
[llm/llm_wrapper.py](../mini_agent/llm/llm_wrapper.py) provides unified interface for multiple providers:
- Auto-appends `/anthropic` or `/v1` suffix for MiniMax API domains
- Uses third-party API endpoints as-is
- Delegates to provider-specific clients: [anthropic_client.py](../mini_agent/llm/anthropic_client.py) and [openai_client.py](../mini_agent/llm/openai_client.py)

**Message Format:** Uses unified [schema/schema.py](../mini_agent/schema/schema.py)::Message with optional `thinking` field for extended reasoning (M2.1 feature).

## Project-Specific Conventions

### Configuration
- **Never hardcode credentials** - all config via [mini_agent/config/config.yaml](../mini_agent/config/config.yaml)
- Config schema defined in [config.py](../mini_agent/config.py) with Pydantic validation
- System prompt is separate markdown file at [mini_agent/config/system_prompt.md](../mini_agent/config/system_prompt.md)

### Workspace Management
- Default workspace: `./workspace` (configurable)
- Agent auto-injects workspace path into system prompt
- All file tools (Read/Write/Edit) operate relative to workspace root
- Workspace path is resolved to absolute in [agent.py](../mini_agent/agent.py:65-70)

### Logging
- Comprehensive logging via [logger.py](../mini_agent/logger.py) (AgentLogger)
- Logs saved to `~/.mini-agent/log/` by default
- Each session creates timestamped log file
- Includes full request/response payloads for debugging

### Terminal Output
- Colored output using ANSI codes (Colors class in [agent.py](../mini_agent/agent.py:20-40))
- Width-aware formatting via [utils/terminal_utils.py](../mini_agent/utils/terminal_utils.py)
- Pretty-printed JSON with syntax highlighting for tool calls
- Thinking blocks displayed with special formatting

### Error Handling
- Retry logic in [retry.py](../mini_agent/retry.py) for transient LLM API failures
- Graceful degradation for MCP connection timeouts
- Cancellation support via asyncio.Event (Ctrl+C / Esc key)

## Testing Guidelines

- All async tests use `@pytest.mark.asyncio` decorator
- Test workspace created with `tempfile.TemporaryDirectory()`
- Mock LLM responses for unit tests (see [tests/test_agent.py](../tests/test_agent.py))
- Integration tests in [tests/test_integration.py](../tests/test_integration.py) require real API key
- MCP tests in [tests/test_mcp.py](../tests/test_mcp.py) use mock servers

## Common Pitfalls

1. **MCP Timeout Issues:** Default timeouts may be too short for slow MCP servers. Always configure via `set_mcp_timeout_config()` before loading tools.

2. **Submodule Not Initialized:** Skills won't load if submodule isn't initialized. Run `git submodule update --init --recursive`.

3. **Tool Schema Format:** Anthropic and OpenAI schemas differ. Use `to_schema()` and `to_openai_schema()` methods, don't construct manually.

4. **Workspace Path Resolution:** File tools expect workspace-relative paths. Don't use absolute paths in user messages - agent handles resolution.

5. **Message History Token Count:** Use tiktoken for accurate counting (`_estimate_tokens()` in agent.py), not simple character count.

## Key Files Reference

- Entry point: [mini_agent/cli.py](../mini_agent/cli.py) - CLI with prompt-toolkit
- Agent core: [mini_agent/agent.py](../mini_agent/agent.py) - Main execution loop
- Config: [mini_agent/config.py](../mini_agent/config.py) - Pydantic config models
- Schema: [mini_agent/schema/schema.py](../mini_agent/schema/schema.py) - Message/Response types
- Tools: [mini_agent/tools/](../mini_agent/tools/) - All tool implementations
- Examples: [examples/](../examples/) - Usage patterns and demos
