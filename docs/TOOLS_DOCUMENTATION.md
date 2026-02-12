# Tools System Documentation

## Overview

The Tools system is one of the four main components of the NeuroStack, providing a flexible and extensible framework for creating, managing, and utilizing tools across AI agents. Tools are discrete functions that agents can use to perform specific tasks, from data acquisition to database operations.

## Architecture

### Core Components

1. **Tool Registry** (`modular_agents/core/tool_registry.py`) - Central registry for tool discovery and management
2. **Tool Functions** (`modular_agents/tools/`) - Individual tool implementations
3. **Tool Generator** (`modular_agents/core/tool_generator.py`) - Automated tool creation service
4. **Tool API** (`modular_agents/api/routers/tools.py`) - REST API for tool management

### Key Classes

#### ToolMetadata
```python
@dataclass
class ToolMetadata:
    """Metadata for a registered tool."""
    name: str
    function: Callable
    module: str
    description: str
    category: str
    tags: Set[str]
    adk_tool: Optional[FunctionTool] = None
    is_async: bool = False
```

#### ToolRegistry
The central registry that manages tool discovery, cataloging, and filtering.

## Tool Creation Process

### 1. Manual Tool Creation

#### Step 1: Create Tool File
Create a new Python file in the `modular_agents/tools/` directory:

```python
# Example: modular_agents/tools/my_custom_tools.py
"""
Custom Tools for Specific Domain

Description of what these tools do and their purpose.
"""

from ..core.tool_registry import tool_category, tool_tags
from typing import Dict, Any, List, Optional
import requests
import json

@tool_category("Custom Domain")
@tool_tags("api", "web", "data")
def fetch_web_data(url: str, timeout: int = 30) -> Dict[str, Any]:
    """
    Fetch data from a web API endpoint.
    
    Args:
        url (str): The URL to fetch data from
        timeout (int): Request timeout in seconds, defaults to 30
        
    Returns:
        Dict[str, Any]: Response data with success status
    """
    try:
        response = requests.get(url, timeout=timeout)
        response.raise_for_status()
        
        return {
            "success": True,
            "message": "Data fetched successfully",
            "data": response.json(),
            "status_code": response.status_code
        }
    except requests.exceptions.RequestException as e:
        return {
            "success": False,
            "message": f"Request failed: {str(e)}",
            "data": None
        }
    except json.JSONDecodeError as e:
        return {
            "success": False,
            "message": f"Invalid JSON response: {str(e)}",
            "data": None
        }
```

#### Step 2: Tool Decorators

Tools use decorators to specify metadata:

- `@tool_category("Category Name")`: Assigns the tool to a category
- `@tool_tags("tag1", "tag2")`: Adds tags for filtering and organization
- `@tool_metadata(key="value")`: Adds custom metadata

#### Step 3: Tool Function Requirements

All tool functions must follow these conventions:

1. **Return Format**: Return a dictionary with `success`, `message`, and relevant data
2. **Error Handling**: Include try/except blocks for robust error handling
3. **Type Hints**: Use proper type annotations for parameters and return types
4. **Docstrings**: Include comprehensive docstrings with Args and Returns sections
5. **Input Validation**: Validate inputs before processing

### 2. Automated Tool Creation

#### Using the Tool Generator API

The system provides automated tool generation using LLM capabilities:

##### Step 1: Generate Tool Plan
```python
# API Request
POST /tools/generator/plan
{
    "description": "Create tools for weather data analysis",
    "api_key": "your-llm-api-key",
    "provider": "openai",
    "model": "gpt-4"
}
```

##### Step 2: Review and Approve Plan
```python
# API Request  
POST /tools/generator/generate
{
    "request_id": "generated-request-id",
    "approved": true,
    "modifications": {
        "tools": [
            {
                "name": "analyze_weather_trends",
                "approved": true,
                "modifications": {}
            }
        ]
    }
}
```

## Tool Registration Process

### Automatic Discovery

The `ToolRegistry` automatically discovers tools through:

1. **Directory Scanning**: Scans all Python files in `modular_agents/tools/`
2. **Module Loading**: Dynamically imports each tool module
3. **Function Inspection**: Identifies tool functions using metadata attributes
4. **ADK Integration**: Creates `FunctionTool` objects for Google ADK compatibility

### Registration Flow

```python
# In ToolRegistry.__init__()
def discover_tools(self) -> None:
    """Discover and register all tools from the tools directory."""
    # 1. Scan directory for Python files
    for tool_file in self.tools_dir.glob("*.py"):
        # 2. Load module dynamically
        self._load_tools_from_file(tool_file)
        
    # 3. Register each tool function
    def _register_tool_function(self, name: str, func: Callable, module: str):
        # Extract metadata from decorators
        category = getattr(func, "__tool_category__", "General")
        tags = getattr(func, "__tool_tags__", [])
        
        # Create ToolMetadata
        tool_metadata = ToolMetadata(
            name=name,
            function=func,
            module=module,
            description=func.__doc__ or f"Tool function: {name}",
            category=category,
            tags=set(tags),
            is_async=inspect.iscoroutinefunction(func)
        )
        
        # Create ADK FunctionTool
        tool_metadata.adk_tool = FunctionTool(func=func)
        
        # Register in registry
        self._tools[name] = tool_metadata
```

### Category Inference

If no explicit category is provided, the system infers categories from:
1. File names (e.g., `database_tools.py` → "Database")
2. Function decorators
3. Module structure

## Tool Filtering and Discovery

### Filter Operations

The `ToolRegistry` provides powerful filtering capabilities:

```python
# Get all tools
all_tools = registry.get_all_tools()

# Filter by category
db_tools = registry.get_tools_by_category("Database")

# Filter by tags
finance_tools = registry.get_tools_by_tags(["finance", "stock"])

# Complex filtering
filtered_tools = registry.filter_tools(
    categories=["Database", "Finance"],
    tags=["query", "analysis"],
    exclude_names=["deprecated_tool"]
)
```

### Available Filter Methods

1. **`get_all_tools()`**: Returns all registered tools
2. **`get_tool(name)`**: Get specific tool by name
3. **`get_tools_by_category(category)`**: Filter by category
4. **`get_tools_by_tags(tags, match_all=False)`**: Filter by tags
5. **`filter_tools(**kwargs)`**: Multi-criteria filtering

## Tool API Endpoints

### GET /tools/
Get all available tools with optional filtering:

```python
# Query parameters
GET /tools/?categories=Database&tags=query&names=specific_tool

# Response
{
    "total_tools": 45,
    "filtered_tools": 12,
    "categories": [
        {
            "name": "Database",
            "count": 8,
            "tools": ["execute_query", "create_table", ...]
        }
    ],
    "all_tags": ["query", "analysis", "web", ...],
    "tools": [
        {
            "name": "execute_query",
            "description": "Execute SQL queries on database",
            "category": "Database",
            "tags": ["sql", "query"],
            "parameters": {...},
            "module_path": "modular_agents.tools.database_tools.execute_query"
        }
    ]
}
```

### GET /tools/categories
Get tool categories with counts:

```python
# Response
[
    {
        "name": "Database",
        "count": 8,
        "tools": ["execute_query", "create_table", ...]
    },
    {
        "name": "Finance",
        "count": 12,
        "tools": ["fetch_stock_data", "analyze_portfolio", ...]
    }
]
```

### POST /tools/generator/plan
Generate tool creation plan:

```python
# Request
{
    "description": "Create tools for weather data analysis",
    "api_key": "your-api-key",
    "provider": "openai",
    "model": "gpt-4"
}

# Response
{
    "request_id": "uuid-generated-id",
    "total_tools": 3,
    "estimated_dependencies": ["requests", "pandas"],
    "tools": [
        {
            "name": "fetch_weather_data",
            "description": "Fetch weather data from API",
            "category": "Weather",
            "tags": ["api", "weather", "data"],
            "parameters": {...},
            "dependencies": ["requests"]
        }
    ]
}
```

## Integration with Agents

### Tool Assignment

Tools are assigned to agents through multiple methods:

1. **Direct Assignment**: Specific tool names
2. **Category Assignment**: All tools in a category
3. **Tag-based Assignment**: Tools with specific tags
4. **Filter-based Assignment**: Complex filtering criteria

### Agent Tool Configuration

```python
# In agent creation
agent_config = {
    "tool_filter": {
        "categories": ["Database", "Finance"],
        "tags": ["query", "analysis"],
        "names": ["specific_tool_name"]
    }
}
```

### MCP Server Integration

Tools are exposed to agents through MCP (Model Context Protocol) servers:

1. **Server Creation**: Each agent gets a dedicated MCP server
2. **Tool Filtering**: Server includes only tools assigned to the agent
3. **Tool Execution**: ADK FunctionTool objects handle execution
4. **Result Handling**: Standardized response format

## File Structure

```
modular_agents/
├── tools/                          # Tool implementations
│   ├── __init__.py                 # Package initialization
│   ├── data_acquisition_tools.py   # Data fetching tools
│   ├── database_tools.py           # Database operations
│   ├── finance_tools.py            # Financial analysis tools
│   ├── weather_tools.py            # Weather data tools
│   └── [custom_tools].py           # Your custom tools
├── core/
│   ├── tool_registry.py            # Tool discovery and management
│   ├── tool_generator.py           # Automated tool creation
│   └── tool_config.py              # Tool configuration
└── api/routers/
    ├── tools.py                    # Tool API endpoints
    └── tool_generator.py           # Tool generation API
```

## Best Practices

### Tool Development

1. **Consistent Return Format**: Always return `{"success": bool, "message": str, "data": any}`
2. **Error Handling**: Use try/except blocks for all operations
3. **Input Validation**: Validate all inputs before processing
4. **Documentation**: Write comprehensive docstrings
5. **Type Hints**: Use proper type annotations
6. **Security**: Validate user inputs and sanitize outputs

### Tool Organization

1. **Logical Grouping**: Group related tools in the same file
2. **Naming Conventions**: Use descriptive, consistent names
3. **Categories**: Assign meaningful categories
4. **Tags**: Use relevant tags for filtering
5. **Dependencies**: Minimize external dependencies

### Performance Considerations

1. **Async Support**: Use async functions for I/O operations
2. **Resource Management**: Clean up resources properly
3. **Caching**: Cache expensive operations when appropriate
4. **Timeouts**: Set appropriate timeouts for external calls

## Common Tool Patterns

### API Integration Tool
```python
@tool_category("API")
@tool_tags("web", "api", "integration")
def call_external_api(endpoint: str, method: str = "GET", data: Optional[Dict] = None) -> Dict[str, Any]:
    """Generic API calling tool."""
    try:
        # Implementation
        return {"success": True, "data": result}
    except Exception as e:
        return {"success": False, "message": str(e)}
```

### Data Processing Tool
```python
@tool_category("Data Processing")
@tool_tags("data", "transform", "analysis")
def process_data(data: List[Dict], operation: str) -> Dict[str, Any]:
    """Process data with specified operation."""
    try:
        # Implementation
        return {"success": True, "processed_data": result}
    except Exception as e:
        return {"success": False, "message": str(e)}
```

### File Operations Tool
```python
@tool_category("File System")
@tool_tags("file", "io", "storage")
def read_file_contents(file_path: str, encoding: str = "utf-8") -> Dict[str, Any]:
    """Read file contents safely."""
    try:
        # Implementation with validation
        return {"success": True, "content": content}
    except Exception as e:
        return {"success": False, "message": str(e)}
```

## Troubleshooting

### Common Issues

1. **Tool Not Discovered**: Check file location and naming
2. **Import Errors**: Verify dependencies are installed
3. **Decorator Issues**: Ensure decorators are imported correctly
4. **Registration Failures**: Check function signatures and metadata
5. **ADK Integration Problems**: Verify FunctionTool compatibility

### Debugging Tips

1. **Check Logs**: Review system logs for discovery errors
2. **Test Individually**: Test tools outside the registry
3. **Validate Metadata**: Ensure decorators are applied correctly
4. **Module Imports**: Verify all imports are available
5. **Function Signatures**: Check parameter types and return types

This documentation provides a comprehensive guide to understanding and working with the tools system in the NeuroStack. Tools form the foundation of agent capabilities, making this system highly extensible and customizable for various use cases. 