# Agents System Documentation

## Overview

The Agents system is the third core component of the NeuroStack, responsible for creating and managing AI agents that utilize tools through MCP servers. Agents are the end-user interface that orchestrates tools and MCP servers to accomplish complex tasks through natural language interaction.

## Architecture

### Core Components

1. **NeuroStack** (`modular_agents/agents/agent_factory.py`) - Creates agents from configurations
2. **Agent Registry** (`modular_agents/core/agent_registry.py`) - Manages agent discovery and metadata
3. **Unified Factory** (`modular_agents/core/factory.py`) - Orchestrates the entire system
4. **Configuration Manager** (`modular_agents/core/config_manager.py`) - Handles agent configurations
5. **Agent API** (`modular_agents/api/routers/agents.py`) - REST API for agent management

### Key Classes

#### AgentFactory
```python
class AgentFactory:
    """
    Factory for creating AI agents with configurable toolsets.
    
    Features:
    - Create agents from configuration files
    - Dynamic tool filtering and loading
    - Automatic MCP server management
    - Agent lifecycle management
    """
    
    def __init__(
        self,
        tool_registry: Optional[ToolRegistry] = None,
        config_manager: Optional[ConfigManager] = None
    ):
        # Implementation details...
```

#### AgentRegistry
```python
class AgentRegistry:
    """
    Registry for managing and filtering agents across the modular agent system.
    
    Features:
    - Automatic agent discovery from agents directory
    - Agent filtering by category, tags, or names
    - Metadata management and caching
    - Tool assignment and MCP server integration
    - Dynamic agent loading/reloading
    """
```

## Agent Creation Process

### 1. Agent Creation from MCP Servers

#### Automatic Agent Creation

When an agent is created, it automatically gets connected to MCP servers:

```python
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

#### MCP Server Integration

Each agent gets its own MCP server that filters tools based on the agent's configuration:

```python
def _create_server_script(self, agent_name: str, server: BaseMCPServer) -> Path:
    """Create a Python script that runs the MCP server for this agent."""
    
    script_content = f'''
import asyncio
import sys
from pathlib import Path

sys.path.insert(0, str(Path(__file__).parent.parent.parent))

from modular_agents.mcp_servers.base_server import BaseMCPServer
from modular_agents.core.tool_registry import ToolRegistry
from modular_agents.core.logger import setup_logging

async def main():
    """Run the MCP server for agent: {agent_name}"""
    setup_logging()
    
    tool_registry = ToolRegistry()
    
    server = BaseMCPServer(
        server_name="{server.server_name}",
        tool_registry=tool_registry,
        tool_filter={server.tool_filter},
        log_level="INFO"
    )
    
    await server.run_stdio_server()

if __name__ == "__main__":
    asyncio.run(main())
'''
```

### 2. Agent Creation Methods

#### Method 1: Configuration-Based Creation

Create agents from YAML configuration files:

```yaml
# agents/configs/data_acquisition_agent.yaml
name: "Data Acquisition Agent"
description: "Specialized agent for fetching financial data"
version: "1.0.0"
category: "Financial"

agent_config:
  model: "gemini-2.0-flash"
  temperature: 0.1
  max_tokens: 4000
  system_prompt: |
    You are a specialized Data Acquisition Agent for financial markets.
    Your primary role is to fetch historical stock data from multiple sources.

tool_filters:
  categories: ["Data Acquisition"]
  tags: ["finance", "stock", "historical"]
  
required_tools:
  - "fetch_yfinance_data"
  - "validate_ticker_symbols"
  - "batch_ticker_fetch"
```

```python
# Create agent from configuration
factory = AgentFactory()
agent = factory.create_agent_from_config("data_acquisition_agent")
```

#### Method 2: Programmatic Creation

Create agents programmatically with specific tool filters:

```python
from modular_agents.agents.agent_factory import AgentFactory
from modular_agents.core.config_manager import AgentConfig

# Create NeuroStack
factory = AgentFactory()

# Create agent with specific tool filter
agent = factory.create_agent_with_tools(
    name="database_specialist",
    model="gemini-2.0-flash",
    instruction="You are a database specialist with access to SQL tools.",
    tool_filter={
        "categories": ["Database"],
        "tags": ["sql", "query"]
    }
)
```

#### Method 3: API-Based Creation

Create agents through REST API:

```python
# API Request
POST /agents/
{
    "name": "Financial Analyst",
    "description": "Agent for financial analysis and reporting",
    "instructions": "You are a financial analyst specializing in market data analysis.",
    "model_settings": {
        "name": "gemini-2.0-flash",
        "temperature": 0.1,
        "max_tokens": 4000
    },
    "mcp_servers": [
        {
            "name": "financial_data_server",
            "port": 8080,
            "tool_filter": {
                "categories": ["Finance", "Data Acquisition"],
                "tags": ["stock", "market", "analysis"]
            }
        }
    ],
    "tags": ["finance", "analysis", "market"]
}
```

#### Method 4: Direct Tools Creation

Create agents with direct tool access (bypassing MCP servers):

```python
# API Request
POST /agents/direct-tools
{
    "name": "Data Processor",
    "description": "Agent with direct access to data processing tools",
    "instructions": "You process data using direct tool access.",
    "model_settings": {
        "name": "gemini-2.0-flash",
        "temperature": 0.0
    },
    "direct_tools": {
        "tool_names": [
            "fetch_yfinance_data",
            "validate_ticker_symbols",
            "batch_ticker_fetch"
        ]
    }
}
```

### 3. File-Based Agent Creation

#### Agent Directory Structure

The system creates physical agent files in the `agents/` directory:

```
agents/
├── database_agent/
│   ├── __init__.py
│   ├── agent.py          # Main agent definition
│   └── mcp_server.py     # Dedicated MCP server
├── finance_agent/
│   ├── __init__.py
│   ├── agent.py
│   └── mcp_server.py
└── configs/
    ├── database_agent.yaml
    └── finance_agent.yaml
```

#### Generated Agent File

```python
# agents/database_agent/agent.py
"""
Database Agent

Specialized agent for database operations and queries.
"""

from google.adk.agents import LlmAgent
from google.adk.tools.mcp_tool.mcp_toolset import MCPToolset, StdioConnectionParams
from pathlib import Path
import sys

# Add the modular_agents package to the path
sys.path.insert(0, str(Path(__file__).parent.parent.parent))

# Create database_agent agent
root_agent = LlmAgent(
    name="database_agent",
    model="gemini-2.0-flash",
    description="Specialized agent for database operations",
    instruction="""You are a database specialist with access to SQL tools.
    
Your available tools include: execute_sql_query, fetch_data_from_table, update_database_record

Key responsibilities:
- Execute SQL queries safely and efficiently
- Fetch data from database tables
- Update database records as needed
- Provide clear explanations of database operations

Always validate queries before execution and handle errors gracefully.""",
    tools=[
        MCPToolset(
            connection_params=StdioConnectionParams(
                server_params={
                    "command": "python",
                    "args": [str(Path(__file__).parent / "mcp_server.py")],
                }
            )
        )
    ],
)
```

#### Generated MCP Server File

```python
# agents/database_agent/mcp_server.py
"""
Database Agent MCP Server

This script is auto-generated for the database_agent agent.
"""

import asyncio
import sys
from pathlib import Path

sys.path.insert(0, str(Path(__file__).parent.parent.parent))

from modular_agents.mcp_servers.base_server import BaseMCPServer
from modular_agents.core.tool_registry import ToolRegistry
from modular_agents.core.logger import setup_logging

async def main():
    """Run the MCP server for database_agent agent"""
    setup_logging()
    
    tool_registry = ToolRegistry()
    
    server = BaseMCPServer(
        server_name="database_agent_mcp_server",
        tool_registry=tool_registry,
        tool_filter={
            "categories": ["Database"],
            "tags": ["sql", "query"]
        },
        log_level="INFO"
    )
    
    await server.run_stdio_server()

if __name__ == "__main__":
    try:
        asyncio.run(main())
    except KeyboardInterrupt:
        pass
    except Exception as e:
        import logging
        logging.error(f"database_agent agent MCP server error: {e}", exc_info=True)
        sys.exit(1)
```

## Multi-Server Agent Integration

### Connecting to Multiple MCP Servers

Agents can connect to multiple MCP servers for diverse capabilities:

```python
# Agent with multiple MCP servers
agent = LlmAgent(
    model="gemini-2.0-flash",
    name="multi_capability_agent",
    instruction="You are a versatile assistant with access to multiple services.",
    tools=[
        # Internal MCP server for database operations
        MCPToolset(
            connection_params=StdioServerParameters(
                command="python3",
                args=["/path/to/database_server.py"],
            )
        ),
        # Internal MCP server for file operations
        MCPToolset(
            connection_params=StdioServerParameters(
                command="python3",
                args=["/path/to/file_server.py"],
            )
        ),
        # External MCP server for GitHub integration
        MCPToolset(
            connection_params=StdioServerParameters(
                command="npx",
                args=["@modelcontextprotocol/server-github"],
            )
        )
    ],
)
```

### External Service Integration

Agents can integrate with external services through external MCP servers:

```python
# Agent configuration with external services
agent_config = {
    "name": "productivity_agent",
    "mcp_servers": [
        {
            "name": "notion_server",
            "type": "external",
            "command": "npx @notionhq/notion-mcp-server",
            "env": {
                "NOTION_TOKEN": "${NOTION_TOKEN}"
            },
            "tools": [
                "notion_create_page",
                "notion_get_page",
                "notion_update_page",
                "notion_search"
            ]
        },
        {
            "name": "github_server", 
            "type": "external",
            "command": "npx @modelcontextprotocol/server-github",
            "env": {
                "GITHUB_TOKEN": "${GITHUB_TOKEN}"
            },
            "tools": [
                "github_create_repo",
                "github_get_repo",
                "github_create_issue"
            ]
        }
    ]
}
```

## Agent Registry and Discovery

### Agent Discovery Process

The `AgentRegistry` automatically discovers agents from the agents directory:

```python
class AgentRegistry:
    def discover_agents(self) -> None:
        """Discover and register all agents from the agents directory."""
        
        # Scan all agent directories
        for agent_dir in self.agents_dir.iterdir():
            if not agent_dir.is_dir():
                continue
                
            try:
                self._load_agent_from_directory(agent_dir)
            except Exception as e:
                self.logger.error(f"Failed to load agent from {agent_dir}: {e}")
    
    def _load_agent_from_directory(self, agent_dir: Path) -> None:
        """Load agent from a specific directory."""
        agent_name = agent_dir.name
        
        # Load agent.py module
        agent_file = agent_dir / "agent.py"
        if not agent_file.exists():
            return
            
        # Import the agent module
        module = importlib.util.module_from_spec(spec)
        spec.loader.exec_module(module)
        
        # Extract agent metadata
        if hasattr(module, 'root_agent'):
            root_agent = getattr(module, 'root_agent')
            # Register agent with extracted metadata
```

### Agent Filtering and Management

```python
# Get all agents
all_agents = factory.get_all_agents()

# Filter agents by category
finance_agents = factory.get_agents_by_category("Finance")

# Filter agents by tags
analysis_agents = factory.get_agents_by_tags(["analysis", "data"])

# Complex filtering
filtered_agents = factory.filter_agents(
    categories=["Database", "Finance"],
    tags=["query", "analysis"],
    active_only=True
)
```

## Agent API Endpoints

### GET /agents/
List all available agents:

```python
# Response
{
    "agents": [
        {
            "name": "database_agent",
            "status": "active",
            "created_at": "2025-01-20T10:00:00Z",
            "model_name": "gemini-2.0-flash",
            "total_tool_count": 3,
            "all_tools": ["execute_sql_query", "fetch_data_from_table", "update_database_record"],
            "mcp_servers": [
                {
                    "name": "database_agent_mcp_server",
                    "status": "available",
                    "tool_count": 3,
                    "tools": ["execute_sql_query", "fetch_data_from_table", "update_database_record"],
                    "script_path": "/path/to/mcp_server.py"
                }
            ],
            "instructions": "You are a database specialist...",
            "description": "Specialized agent for database operations",
            "tags": ["database", "sql", "query"]
        }
    ]
}
```

### POST /agents/
Create a new agent:

```python
# Request
{
    "name": "Financial Analyst",
    "description": "Agent for financial analysis and reporting",
    "instructions": "You are a financial analyst specializing in market data analysis.",
    "model_settings": {
        "name": "gemini-2.0-flash",
        "temperature": 0.1,
        "max_tokens": 4000
    },
    "mcp_servers": [
        {
            "name": "financial_data_server",
            "port": 8080,
            "tool_filter": {
                "categories": ["Finance", "Data Acquisition"],
                "tags": ["stock", "market", "analysis"]
            }
        }
    ],
    "tags": ["finance", "analysis", "market"]
}

# Response
{
    "success": true,
    "message": "Agent 'Financial_Analyst' configured successfully with 12 tools across 1 MCP servers",
    "agent": {
        "name": "Financial_Analyst",
        "status": "active",
        "created_at": "2025-01-20T10:00:00Z",
        "model_name": "gemini-2.0-flash",
        "model_settings": {
            "name": "gemini-2.0-flash",
            "temperature": 0.1,
            "max_tokens": 4000,
            "top_p": 0.9,
            "top_k": 40
        },
        "total_tool_count": 12,
        "all_tools": ["fetch_yfinance_data", "analyze_portfolio", ...],
        "mcp_servers": [
            {
                "name": "financial_data_server",
                "status": "configured",
                "tool_count": 12,
                "tools": ["fetch_yfinance_data", "analyze_portfolio", ...],
                "port": 8080
            }
        ],
        "instructions": "You are a financial analyst specializing in market data analysis.",
        "description": "Agent for financial analysis and reporting",
        "tags": ["finance", "analysis", "market"]
    }
}
```

### POST /agents/direct-tools
Create agent with direct tool access:

```python
# Request
{
    "name": "Data Processor",
    "description": "Agent with direct access to data processing tools",
    "instructions": "You process data using direct tool access.",
    "model_settings": {
        "name": "gemini-2.0-flash",
        "temperature": 0.0
    },
    "direct_tools": {
        "tool_names": [
            "fetch_yfinance_data",
            "validate_ticker_symbols",
            "batch_ticker_fetch"
        ]
    }
}

# Response
{
    "success": true,
    "message": "Direct tools agent 'Data_Processor' created successfully with 3 tools",
    "agent": {
        "name": "Data_Processor",
        "status": "active",
        "total_tool_count": 3,
        "all_tools": ["fetch_yfinance_data", "validate_ticker_symbols", "batch_ticker_fetch"],
        "mcp_servers": []  # No MCP servers for direct tools agents
    }
}
```

### POST /agents/{agent_name}/chat
Chat with an agent:

```python
# Request
{
    "message": "Fetch the latest stock data for AAPL",
    "conversation_id": "conv_123",
    "context": {
        "user_id": "user_456",
        "session_id": "session_789"
    }
}

# Response
{
    "success": true,
    "response": "I'll fetch the latest stock data for AAPL using the Yahoo Finance API...",
    "conversation_id": "conv_123",
    "agent_name": "financial_analyst",
    "timestamp": "2025-01-20T10:30:00Z",
    "tool_calls": [
        {
            "tool_name": "fetch_yfinance_data",
            "arguments": {"ticker": "AAPL", "period": "1d"},
            "result": {"success": true, "data": {...}}
        }
    ]
}
```

## Agent Tool Assignment

### Tool Assignment Methods

1. **Category-Based Assignment**
```python
# Assign all tools from specific categories
tool_filter = {
    "categories": ["Database", "File System"]
}
```

2. **Tag-Based Assignment**
```python
# Assign tools with specific tags
tool_filter = {
    "tags": ["query", "analysis", "data"],
    "match_all_tags": False  # Match any tag
}
```

3. **Specific Tool Assignment**
```python
# Assign specific tools by name
tool_filter = {
    "names": ["fetch_yfinance_data", "execute_sql_query", "read_text_file"]
}
```

4. **Complex Filtering**
```python
# Combine multiple criteria
tool_filter = {
    "categories": ["Database", "Finance"],
    "tags": ["query", "analysis"],
    "exclude_names": ["dangerous_tool"]
}
```

### Dynamic Tool Assignment

```python
# Auto-assign tools based on agent category
success = factory.auto_assign_tools("financial_agent", "category")

# Manually assign specific tools
success = factory.assign_tools_to_agent(
    "database_agent", 
    ["execute_sql_query", "fetch_data_from_table"]
)

# Get tools assigned to an agent
agent_tools = factory.get_agent_tools("database_agent")
```

## Agent Lifecycle Management

### Agent States

Agents can be in various states:
- **Active**: Agent is running and available
- **Inactive**: Agent is stopped or disabled
- **Starting**: Agent is being initialized
- **Error**: Agent encountered an error

### Agent Operations

```python
# Activate/Deactivate agents
factory.activate_agent("database_agent")
factory.deactivate_agent("database_agent")

# Delete agents
factory.delete_agent("database_agent")  # Removes files and registry entry

# Reload agent registry
factory.reload_agents()  # Rediscover agents from filesystem
```

## Agent Configuration Patterns

### Database Agent Pattern
```python
agent_config = {
    "name": "database_specialist",
    "model": "gemini-2.0-flash",
    "instruction": "You are a database specialist with SQL expertise.",
    "tool_filter": {
        "categories": ["Database"],
        "tags": ["sql", "query", "database"]
    }
}
```

### API Integration Agent Pattern
```python
agent_config = {
    "name": "api_integrator",
    "model": "gemini-2.0-flash",
    "instruction": "You integrate with external APIs and services.",
    "tool_filter": {
        "categories": ["API", "Web"],
        "tags": ["http", "rest", "api"]
    }
}
```

### Data Analysis Agent Pattern
```python
agent_config = {
    "name": "data_analyst",
    "model": "gemini-2.0-flash",
    "instruction": "You analyze data and generate insights.",
    "tool_filter": {
        "categories": ["Data Processing", "Analysis"],
        "tags": ["analysis", "statistics", "visualization"]
    }
}
```

### Multi-Domain Agent Pattern
```python
agent_config = {
    "name": "versatile_assistant",
    "model": "gemini-2.0-flash",
    "instruction": "You are a versatile assistant with diverse capabilities.",
    "mcp_servers": [
        {
            "name": "database_server",
            "tool_filter": {"categories": ["Database"]}
        },
        {
            "name": "file_server", 
            "tool_filter": {"categories": ["File System"]}
        },
        {
            "name": "api_server",
            "tool_filter": {"categories": ["API"]}
        }
    ]
}
```

## File Structure

```
modular_agents/
├── agents/                         # Agent implementations
│   ├── __init__.py                 # Package initialization
│   ├── agent_factory.py           # Agent creation factory
│   ├── database_agent/             # Example agent directory
│   │   ├── __init__.py
│   │   ├── agent.py                # Agent definition
│   │   └── mcp_server.py           # Dedicated MCP server
│   ├── finance_agent/              # Another example agent
│   │   ├── __init__.py
│   │   ├── agent.py
│   │   └── mcp_server.py
│   └── configs/                    # Agent configurations
│       ├── database_agent.yaml
│       └── finance_agent.yaml
├── core/
│   ├── agent_registry.py           # Agent discovery and management
│   ├── factory.py                  # Unified factory system
│   └── config_manager.py           # Configuration management
├── api/routers/
│   └── agents.py                   # Agent API endpoints
└── data/
    └── agents.json                 # Persistent agent data
```

## Best Practices

### Agent Development

1. **Clear Instructions**: Write specific, actionable instructions for agents
2. **Tool Selection**: Choose appropriate tools for agent capabilities
3. **Error Handling**: Implement robust error handling in agent logic
4. **Resource Management**: Monitor agent resource usage
5. **Security**: Validate inputs and sanitize outputs

### Agent Organization

1. **Logical Grouping**: Group related agents by domain or function
2. **Naming Conventions**: Use descriptive, consistent agent names
3. **Documentation**: Document agent capabilities and limitations
4. **Version Control**: Track agent configurations and changes
5. **Testing**: Test agents with various scenarios and edge cases

### Performance Considerations

1. **Tool Optimization**: Optimize tool selection for agent performance
2. **Memory Management**: Monitor agent memory usage
3. **Connection Pooling**: Reuse MCP server connections when possible
4. **Caching**: Cache expensive operations and results
5. **Monitoring**: Monitor agent performance and health

## Common Agent Patterns

### Specialist Agent Pattern
```python
# Agent specialized for a specific domain
agent = LlmAgent(
    name="financial_specialist",
    model="gemini-2.0-flash",
    instruction="You are a financial analysis specialist.",
    tools=[
        MCPToolset(
            connection_params=StdioServerParameters(
                command="python3",
                args=["/path/to/finance_server.py"],
            )
        )
    ]
)
```

### Generalist Agent Pattern
```python
# Agent with broad capabilities
agent = LlmAgent(
    name="general_assistant",
    model="gemini-2.0-flash",
    instruction="You are a versatile assistant.",
    tools=[
        MCPToolset(connection_params=StdioServerParameters(
            command="python3", args=["/path/to/database_server.py"]
        )),
        MCPToolset(connection_params=StdioServerParameters(
            command="python3", args=["/path/to/file_server.py"]
        )),
        MCPToolset(connection_params=StdioServerParameters(
            command="python3", args=["/path/to/api_server.py"]
        ))
    ]
)
```

### Workflow Agent Pattern
```python
# Agent designed for workflow orchestration
agent = LlmAgent(
    name="workflow_orchestrator",
    model="gemini-2.0-flash",
    instruction="You orchestrate complex workflows.",
    tools=[
        MCPToolset(connection_params=StdioServerParameters(
            command="python3", args=["/path/to/workflow_server.py"]
        ))
    ]
)
```

## Troubleshooting

### Common Issues

1. **Agent Not Starting**: Check MCP server configuration and tool availability
2. **Tool Not Available**: Verify tool filter configuration and tool registry
3. **Connection Failures**: Check MCP server status and network connectivity
4. **Performance Issues**: Monitor tool execution times and resource usage
5. **Configuration Errors**: Validate agent configuration syntax and values

### Debugging Tips

1. **Check Logs**: Review agent and server logs for error messages
2. **Test Tools**: Test individual tools outside of agent context
3. **Validate Configuration**: Ensure agent configurations are correct
4. **Monitor Resources**: Check memory and CPU usage
5. **Network Diagnostics**: Verify connectivity for external services

This documentation provides a comprehensive guide to understanding and working with the Agents system in the NeuroStack. Agents serve as the intelligent interface that orchestrates tools and MCP servers to accomplish complex tasks through natural language interaction. 