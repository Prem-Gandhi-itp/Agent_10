# MCP Servers System Documentation

## Overview

The MCP (Model Context Protocol) Servers system is the second core component of the NeuroStack, serving as the bridge between tools and agents. MCP servers package tools into standardized interfaces that agents can communicate with, enabling modular and scalable agent architectures.

## Architecture

### Core Components

1. **Base MCP Server** (`modular_agents/mcp_servers/base_server.py`) - Core server implementation
2. **Dynamic MCP Server** (`modular_agents/mcp_servers/dynamic_server.py`) - Configurable server
3. **External MCP Server** (`modular_agents/mcp_servers/external_server.py`) - External service integration
4. **Server Registry** (`modular_agents/mcp_servers/server_registry.py`) - Server management and discovery
5. **Server API** (`modular_agents/api/routers/servers.py`) - REST API for server management

### Key Classes

#### BaseMCPServer
```python
class BaseMCPServer:
    """
    Base MCP Server that can be configured with any set of tools.
    
    Features:
    - Dynamic tool loading from ToolRegistry
    - Configurable tool filtering
    - Comprehensive logging
    - Error handling and recovery
    - Async/await support
    """
    
    def __init__(
        self,
        server_name: str,
        tool_registry: Optional[ToolRegistry] = None,
        tool_filter: Optional[Dict] = None,
        log_level: str = "INFO"
    ):
        # Implementation details...
```

#### ExternalMCPServer
```python
class ExternalMCPServer:
    """
    External MCP Server that connects to remote MCP servers.
    
    Features:
    - Connect to external MCP servers via stdio, TCP, or WebSocket
    - Automatic server lifecycle management
    - Tool discovery and proxying
    - Health monitoring
    - Error handling and recovery
    """
```

## MCP Server Creation Process

### 1. Automatic MCP Server Creation from Tools

#### Agent-Based Server Creation

When an agent is created, the system automatically generates an MCP server:

```python
# In AgentFactory.create_agent()
def create_agent(self, config: AgentConfig) -> LlmAgent:
    """Create an agent from an AgentConfig object."""
    
    # Create MCP server for this agent
    server = BaseMCPServer(
        server_name=f"{config.name}_mcp_server",
        tool_registry=self.tool_registry,
        tool_filter=config.tool_filter
    )
    
    # Create a temporary server script for this agent
    server_script_path = self._create_server_script(config.name, server)
    
    # Create the agent with MCP toolset
    agent = LlmAgent(
        model=config.model,
        name=config.name,
        instruction=config.instruction,
        tools=[
            MCPToolset(
                connection_params=StdioServerParameters(
                    command="python3",
                    args=[str(server_script_path)],
                )
            )
        ],
    )
```

#### Server Script Generation

The system generates Python scripts for each MCP server:

```python
def _create_server_script(self, agent_name: str, server: BaseMCPServer) -> Path:
    """Create a Python script that runs the MCP server for this agent."""
    
    script_content = f'''"""
Generated MCP Server Script for Agent: {agent_name}
"""

import asyncio
import sys
from pathlib import Path

# Add the modular_agents package to the path
sys.path.insert(0, str(Path(__file__).parent.parent.parent))

from modular_agents.mcp_servers.base_server import BaseMCPServer
from modular_agents.core.tool_registry import ToolRegistry
from modular_agents.core.logger import setup_logging

async def main():
    """Run the MCP server for agent: {agent_name}"""
    setup_logging()
    
    # Create tool registry and server with the same configuration
    tool_registry = ToolRegistry()
    
    server = BaseMCPServer(
        server_name="{server.server_name}",
        tool_registry=tool_registry,
        tool_filter={server.tool_filter},
        log_level="INFO"
    )
    
    # Run the server
    await server.run_stdio_server()

if __name__ == "__main__":
    try:
        asyncio.run(main())
    except KeyboardInterrupt:
        pass
    except Exception as e:
        import logging
        logging.error(f"Server error: {e}", exc_info=True)
        sys.exit(1)
'''
```

### 2. Manual MCP Server Creation

#### Using the Dynamic Server

Create servers programmatically:

```python
from modular_agents.mcp_servers.dynamic_server import DynamicMCPServer
from modular_agents.core.tool_registry import ToolRegistry

# Create tool registry
tool_registry = ToolRegistry()

# Create server with specific tool filter
server = DynamicMCPServer(
    server_name="my_custom_server",
    tool_registry=tool_registry,
    tool_filter={
        "categories": ["Database", "File System"],
        "tags": ["query", "read"]
    }
)

# Run the server
await server.run_stdio_server()
```

#### Command Line Server Creation

```bash
# Create server with specific categories
python -m modular_agents.mcp_servers.dynamic_server \
    --name "database_server" \
    --categories "Database" \
    --log-level "INFO"

# Create server with specific tools
python -m modular_agents.mcp_servers.dynamic_server \
    --name "file_server" \
    --tools "read_text_file" "write_text_file" "list_directory"

# Create server with tag filtering
python -m modular_agents.mcp_servers.dynamic_server \
    --name "analysis_server" \
    --tags "analysis" "data" \
    --match-all-tags
```

### 3. File-Based Server Definitions

#### Internal Server Files

Create server definitions in `modular_agents/mcp_servers/internal/`:

```python
# modular_agents/mcp_servers/internal/weather_server.py
"""
Weather MCP Server

Provides weather-related tools for agents.
"""

from typing import Dict, Any
from datetime import datetime

# Server configuration
server_config = {
    "id": "weather_server",
    "name": "weather_server",
    "server_name": "weather_server",
    "agent_name": "Weather Agent",
    "selected_tools": [
        "get_current_weather",
        "get_weather_forecast",
        "get_weather_alerts"
    ],
    "tools": [
        "get_current_weather",
        "get_weather_forecast", 
        "get_weather_alerts"
    ],
    "tool_count": 3,
    "description": "MCP server for weather operations",
    "port": 8080,
    "status": "ready",
    "server_type": "internal",
    "category": "Weather",
    "tags": ["weather", "forecast", "alerts"]
}

# Server metadata
server_metadata = {
    "name": "weather_server",
    "type": "internal",
    "generated_at": datetime.now().isoformat(),
    "tools": server_config["tools"],
    "tool_count": len(server_config["tools"]),
    "description": server_config["description"]
}

def get_server_config() -> Dict[str, Any]:
    """Get the server configuration."""
    return server_config

def get_server_metadata() -> Dict[str, Any]:
    """Get the server metadata."""
    return server_metadata
```

## External MCP Server Creation

### 1. External Server Configuration

#### Basic External Server Setup

```python
# modular_agents/mcp_servers/external/github_server.py
"""
GitHub External MCP Server

Connects to the official GitHub MCP server.
"""

from typing import Dict, Any
from datetime import datetime

# Server configuration
server_config = {
    "id": "github_server",
    "name": "github_server",
    "server_name": "github_server",
    "agent_name": "GitHub Agent",
    "config_name": "github",
    "description": "External MCP server for GitHub operations",
    "command": "npx @modelcontextprotocol/server-github",
    "status": "stopped",
    "process_id": None,
    "port": None,
    "tools": [
        "github_create_repo",
        "github_get_repo",
        "github_list_repos",
        "github_create_issue",
        "github_get_issue",
        "github_list_issues",
        "github_create_pull_request",
        "github_get_pull_request",
        "github_list_pull_requests"
    ],
    "tool_count": 9,
    "server_type": "external",
    "category": "External",
    "tags": ["github", "git", "repository", "version-control"],
    "custom_env": {
        "GITHUB_TOKEN": "${GITHUB_TOKEN}"  # Use environment variable
    }
}

# Server metadata
server_metadata = {
    "name": "github_server",
    "type": "external",
    "generated_at": datetime.now().isoformat(),
    "config_name": "github",
    "command": "npx @modelcontextprotocol/server-github",
    "description": "External MCP server for GitHub operations",
    "required_env": ["GITHUB_TOKEN"]
}

def get_server_config() -> Dict[str, Any]:
    """Get the server configuration."""
    return server_config

def get_server_metadata() -> Dict[str, Any]:
    """Get the server metadata."""
    return server_metadata
```

### 2. External Server Types

#### Popular External MCP Servers

1. **GitHub Server**
```python
server_config = {
    "command": "npx @modelcontextprotocol/server-github",
    "custom_env": {
        "GITHUB_TOKEN": "${GITHUB_TOKEN}"
    },
    "tools": [
        "github_create_repo",
        "github_get_repo",
        "github_list_repos",
        "github_create_issue",
        "github_get_issue",
        "github_list_issues"
    ]
}
```

2. **Notion Server**
```python
server_config = {
    "command": "npx @modelcontextprotocol/server-notion",
    "custom_env": {
        "NOTION_TOKEN": "${NOTION_TOKEN}"
    },
    "tools": [
        "notion_create_page",
        "notion_get_page",
        "notion_update_page",
        "notion_search"
    ]
}
```

3. **Slack Server**
```python
server_config = {
    "command": "npx @modelcontextprotocol/server-slack",
    "custom_env": {
        "SLACK_BOT_TOKEN": "${SLACK_BOT_TOKEN}",
        "SLACK_SIGNING_SECRET": "${SLACK_SIGNING_SECRET}"
    },
    "tools": [
        "slack_send_message",
        "slack_get_channel_history",
        "slack_create_channel",
        "slack_invite_user"
    ]
}
```

4. **Database Server**
```python
server_config = {
    "command": "npx @modelcontextprotocol/server-postgres",
    "custom_env": {
        "DATABASE_URL": "${DATABASE_URL}"
    },
    "tools": [
        "postgres_query",
        "postgres_execute",
        "postgres_describe_table",
        "postgres_list_tables"
    ]
}
```

### 3. Custom External Server Creation

#### Creating Your Own External MCP Server

```python
# modular_agents/mcp_servers/external/custom_api_server.py
"""
Custom API External MCP Server

Connects to a custom API service.
"""

from typing import Dict, Any
from datetime import datetime

# Server configuration
server_config = {
    "id": "custom_api_server",
    "name": "custom_api_server",
    "server_name": "custom_api_server",
    "agent_name": "Custom API Agent",
    "config_name": "custom_api",
    "description": "External MCP server for custom API operations",
    "command": "python",
    "args": ["/path/to/your/custom_mcp_server.py"],
    "status": "stopped",
    "process_id": None,
    "port": None,
    "connection_type": "stdio",  # or "tcp", "websocket"
    "tools": [
        "custom_api_get_data",
        "custom_api_post_data",
        "custom_api_update_data",
        "custom_api_delete_data"
    ],
    "tool_count": 4,
    "server_type": "external",
    "category": "External",
    "tags": ["api", "custom", "http"],
    "custom_env": {
        "API_KEY": "${CUSTOM_API_KEY}",
        "API_BASE_URL": "${CUSTOM_API_BASE_URL}"
    },
    "timeout": 30,
    "auto_start": True
}

def get_server_config() -> Dict[str, Any]:
    """Get the server configuration."""
    return server_config
```

## Server Registry and Management

### Server Registry System

The `MCPServerRegistry` manages all server definitions:

```python
class MCPServerRegistry:
    """Registry for managing MCP servers defined in individual files."""
    
    def __init__(self):
        self.internal_servers: Dict[str, Dict[str, Any]] = {}
        self.external_servers: Dict[str, Dict[str, Any]] = {}
        
    def load_servers(self):
        """Load all MCP servers from internal and external folders."""
        self._load_internal_servers()
        self._load_external_servers()
        
    def get_all_servers(self) -> Dict[str, Dict[str, Any]]:
        """Get all registered servers."""
        return {**self.internal_servers, **self.external_servers}
```

### Server Discovery Process

1. **Directory Scanning**: Scans `internal/` and `external/` directories
2. **Module Loading**: Dynamically imports server definition files
3. **Configuration Extraction**: Extracts `server_config` from each module
4. **Registry Population**: Stores server configurations in registry

## MCP Server API Endpoints

### GET /servers/
List all available MCP servers:

```python
# Response
{
    "internal_servers": [
        {
            "id": "weather_server",
            "name": "weather_server",
            "description": "MCP server for weather operations",
            "tool_count": 3,
            "status": "ready",
            "tools": ["get_current_weather", "get_weather_forecast", "get_weather_alerts"]
        }
    ],
    "external_servers": [
        {
            "id": "github_server",
            "name": "github_server",
            "description": "External MCP server for GitHub operations",
            "tool_count": 9,
            "status": "stopped",
            "command": "npx @modelcontextprotocol/server-github"
        }
    ]
}
```

### POST /servers/create
Create a new MCP server:

```python
# Request
{
    "server_name": "custom_server",
    "agent_name": "Custom Agent",
    "selected_tools": ["tool1", "tool2", "tool3"],
    "description": "Custom MCP server for specific tasks",
    "port": 8080
}

# Response
{
    "success": true,
    "message": "MCP server 'custom_server' created successfully",
    "server": {
        "id": "custom_server",
        "name": "custom_server",
        "status": "ready",
        "tool_count": 3,
        "script_path": "/tmp/mcp_servers/custom_server/custom_server_server.py",
        "command": "python \"/tmp/mcp_servers/custom_server/custom_server_server.py\""
    }
}
```

### POST /servers/{server_id}/start
Start an MCP server:

```python
# Response
{
    "success": true,
    "message": "Server 'weather_server' started successfully",
    "server": {
        "id": "weather_server",
        "status": "running",
        "process_id": 12345,
        "port": 8080,
        "uptime": "5m"
    }
}
```

### POST /servers/{server_id}/stop
Stop an MCP server:

```python
# Response
{
    "success": true,
    "message": "Server 'weather_server' stopped successfully",
    "server": {
        "id": "weather_server",
        "status": "stopped",
        "process_id": null,
        "uptime": "0m"
    }
}
```

## Tool-to-Server Mapping

### Tool Filtering in MCP Servers

MCP servers use tool filters to determine which tools to include:

```python
# Filter by categories
tool_filter = {
    "categories": ["Database", "File System"]
}

# Filter by tags
tool_filter = {
    "tags": ["query", "read", "write"],
    "match_all_tags": False  # Match any tag
}

# Filter by specific tool names
tool_filter = {
    "names": ["read_text_file", "write_text_file", "execute_sql_query"]
}

# Complex filtering
tool_filter = {
    "categories": ["Database"],
    "tags": ["query"],
    "exclude_names": ["dangerous_tool"]
}
```

### Tool Loading Process

1. **Registry Query**: Server queries ToolRegistry with filter
2. **Tool Extraction**: Extracts matching tools and their metadata
3. **ADK Conversion**: Converts tools to ADK FunctionTool objects
4. **MCP Integration**: Exposes tools through MCP protocol

## Integration with Agents

### Agent-Server Connection

Agents connect to MCP servers through toolsets:

```python
# In agent creation
agent = LlmAgent(
    model="gemini-2.0-flash",
    name="my_agent",
    instruction="You are a helpful assistant",
    tools=[
        MCPToolset(
            connection_params=StdioServerParameters(
                command="python3",
                args=["/path/to/server_script.py"],
            )
        )
    ],
)
```

### Multiple Server Support

Agents can connect to multiple MCP servers:

```python
tools = [
    MCPToolset(
        connection_params=StdioServerParameters(
            command="python3",
            args=["/path/to/database_server.py"],
        )
    ),
    MCPToolset(
        connection_params=StdioServerParameters(
            command="python3",
            args=["/path/to/file_server.py"],
        )
    )
]
```

## File Structure

```
modular_agents/
├── mcp_servers/                    # MCP server implementations
│   ├── __init__.py                 # Package initialization
│   ├── base_server.py              # Base MCP server class
│   ├── dynamic_server.py           # Configurable MCP server
│   ├── external_server.py          # External server wrapper
│   ├── server_registry.py          # Server discovery and management
│   ├── internal/                   # Internal server definitions
│   │   ├── __init__.py
│   │   ├── example_weather_server.py
│   │   └── [custom_servers].py
│   ├── external/                   # External server definitions
│   │   ├── __init__.py
│   │   ├── example_github_server.py
│   │   └── [custom_external_servers].py
│   └── README.md                   # Documentation
├── api/routers/
│   └── servers.py                  # Server API endpoints
└── core/
    └── tool_registry.py            # Tool registry integration
```

## Best Practices

### Internal Server Development

1. **Consistent Configuration**: Use standardized server_config format
2. **Tool Organization**: Group related tools in the same server
3. **Error Handling**: Implement robust error handling and logging
4. **Resource Management**: Clean up resources properly
5. **Documentation**: Document server purpose and tool capabilities

### External Server Integration

1. **Environment Variables**: Use environment variables for sensitive data
2. **Health Monitoring**: Implement health checks for external services
3. **Retry Logic**: Add retry mechanisms for network failures
4. **Timeout Configuration**: Set appropriate timeouts for external calls
5. **Security**: Validate and sanitize external data

### Performance Considerations

1. **Connection Pooling**: Reuse connections when possible
2. **Caching**: Cache expensive operations
3. **Async Operations**: Use async/await for I/O operations
4. **Resource Limits**: Set appropriate resource limits
5. **Monitoring**: Monitor server performance and health

## Common Server Patterns

### Database Server Pattern
```python
server_config = {
    "name": "database_server",
    "tools": [
        "execute_query",
        "create_table",
        "insert_data",
        "update_data",
        "delete_data"
    ],
    "category": "Database",
    "tags": ["sql", "database", "query"]
}
```

### API Integration Server Pattern
```python
server_config = {
    "name": "api_server",
    "tools": [
        "get_api_data",
        "post_api_data",
        "put_api_data",
        "delete_api_data"
    ],
    "category": "API",
    "tags": ["api", "http", "rest"]
}
```

### File Operations Server Pattern
```python
server_config = {
    "name": "file_server",
    "tools": [
        "read_file",
        "write_file",
        "list_directory",
        "create_directory",
        "delete_file"
    ],
    "category": "File System",
    "tags": ["file", "filesystem", "io"]
}
```

## Troubleshooting

### Common Issues

1. **Server Not Starting**: Check script paths and permissions
2. **Tool Not Available**: Verify tool filter configuration
3. **Connection Failures**: Check server status and network connectivity
4. **External Server Issues**: Verify external service availability and credentials
5. **Performance Problems**: Monitor resource usage and optimize filters

### Debugging Tips

1. **Check Logs**: Review server logs for error messages
2. **Test Individually**: Test servers outside of agent context
3. **Validate Configuration**: Ensure server configurations are correct
4. **Network Diagnostics**: Check network connectivity for external servers
5. **Resource Monitoring**: Monitor memory and CPU usage

This documentation provides a comprehensive guide to understanding and working with the MCP servers system in the NeuroStack. MCP servers serve as the crucial bridge between tools and agents, enabling flexible and scalable agent architectures. 