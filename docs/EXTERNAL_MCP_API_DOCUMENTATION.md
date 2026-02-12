# External MCP Server API Documentation

This document describes the enhanced external MCP server APIs that provide full CRUD operations for managing custom MCP server configurations.

## Overview

The external MCP server APIs allow you to:
- ✅ **CREATE** external MCP servers with custom configurations
- ✅ **UPDATE** existing MCP server configurations  
- ✅ **DELETE** MCP server files
- ✅ **REPLACE** existing files automatically (no more conflicts)

All generated files use the `external_mcp_server.py` structure with `CustomMCPToolset` from ADK patches for enhanced timeout handling and proper ADK integration.

## API Endpoints

### 1. CREATE External MCP Server

**Endpoint:** `POST /mcp-servers/external/create`

**Description:** Creates a new external MCP server file or replaces an existing one.

**Request Format:**
```json
{
  "mcp_name": "firecrawl",
  "config": {
    "command": "cmd",
    "args": ["/c", "set", "FIRECRAWL_API_KEY=fc-7177114a79604e1b95412b4a2c9362ec", "&&", "npx", "-y", "firecrawl-mcp"],
    "env": {"FIRECRAWL_API_KEY": "fc-7177114a79604e1b95412b4a2c9362ec"},
    "cwd": null
  }
}
```

**Key Features:**
- ✅ `mcp_name` appears first in the request
- ✅ `config` is properly formatted with command, args, env, and cwd
- ✅ **Automatic file replacement** - no more 409 conflicts
- ✅ Logs when replacing existing files

**Response:**
```json
{
  "success": true,
  "message": "External MCP server file 'firecrawl.py' created successfully",
  "server": {
    "id": "external_firecrawl",
    "name": "firecrawl",
    "status": "ready",
    "command": "cmd /c set FIRECRAWL_API_KEY=fc-... && npx -y firecrawl-mcp",
    "server_type": "external",
    "category": "External"
  },
  "agent_name": "firecrawl"
}
```

### 2. UPDATE External MCP Server

**Endpoint:** `PUT /mcp-servers/external/update`

**Description:** Updates an existing external MCP server configuration.

**Request Format:**
```json
{
  "mcp_name": "firecrawl",
  "config": {
    "command": "cmd",
    "args": ["/c", "set", "FIRECRAWL_API_KEY=new-api-key", "&&", "npx", "-y", "firecrawl-mcp"],
    "env": {"FIRECRAWL_API_KEY": "new-api-key"},
    "cwd": "/new/working/directory"
  }
}
```

**Key Features:**
- ✅ Updates existing MCP server files
- ✅ Returns 404 if file doesn't exist
- ✅ Completely replaces the configuration
- ✅ Maintains the same file structure

**Response:**
```json
{
  "success": true,
  "message": "External MCP server 'firecrawl' updated successfully",
  "server": {
    "id": "external_firecrawl",
    "name": "firecrawl",
    "status": "ready",
    "description": "Updated external MCP server: firecrawl"
  },
  "agent_name": "firecrawl"
}
```

### 3. DELETE External MCP Server

**Endpoint:** `DELETE /mcp-servers/external/delete`

**Description:** Deletes an external MCP server file.

**Request Format:**
```json
{
  "mcp_name": "firecrawl"
}
```

**Key Features:**
- ✅ Removes the MCP server file completely
- ✅ Returns 404 if file doesn't exist
- ✅ Provides confirmation of deletion

**Response:**
```json
{
  "success": true,
  "message": "External MCP server 'firecrawl' deleted successfully",
  "deleted_file": "/path/to/modular_agents/mcp_servers/external/firecrawl.py"
}
```

## Generated File Structure

All APIs generate files with the following structure:

```python
"""
Generated External MCP Server: firecrawl

Auto-generated from custom MCP configuration.
Generated on: 2025-08-14 16:36:30

This file uses external_mcp_server.py to provide MCP toolset functionality.
"""

from typing import Dict, Any, Union
from ..external_mcp_server import MCPServerConfig, create_mcp_toolset_from_config, CustomMCPToolset

# MCP Configuration
mcp_config = MCPServerConfig(
    name="firecrawl",
    command="cmd",
    args=['/c', 'set', 'FIRECRAWL_API_KEY=fc-...', '&&', 'npx', '-y', 'firecrawl-mcp'],
    env={'FIRECRAWL_API_KEY': 'fc-...'},
    cwd=None
)

def create_toolset() -> CustomMCPToolset:
    """Create the MCP toolset using CustomMCPToolset from ADK patches."""
    return create_mcp_toolset_from_config(mcp_config)

def get_config() -> MCPServerConfig:
    """Get the MCP server configuration."""
    return mcp_config

# Backward compatibility
server_config = {
    "name": "firecrawl",
    "command": "cmd",
    "args": ['/c', 'set', 'FIRECRAWL_API_KEY=fc-...', '&&', 'npx', '-y', 'firecrawl-mcp'],
    "env": {'FIRECRAWL_API_KEY': 'fc-...'},
    "cwd": None,
    "description": "Custom external MCP server: firecrawl",
    "category": "External",
    "tags": ["custom", "external"]
}

def get_server_config():
    """Get the server configuration for backward compatibility."""
    return server_config
```

## Key Features

### 🔧 Technical Integration
- ✅ Uses `CustomMCPToolset` from `custom_adk_patches.py`
- ✅ Provides enhanced timeout handling (180s vs 5s default)
- ✅ Full ADK integration with `StdioServerParameters`
- ✅ Proper error handling and logging

### 🎯 User Experience
- ✅ **No more conflicts** - automatic file replacement
- ✅ **Clear request format** - mcp_name first, config structured
- ✅ **Full CRUD operations** - Create, Read, Update, Delete
- ✅ **Consistent responses** - standardized success/error messages

### 📁 File Management
- ✅ Files created in `modular_agents/mcp_servers/external/` folder
- ✅ Automatic folder creation if needed
- ✅ Proper file naming: `{mcp_name}.py`
- ✅ Safe file operations with error handling

## Usage Examples

### Creating a Firecrawl MCP Server
```bash
curl -X POST "http://localhost:8000/mcp-servers/external/create" \
  -H "Content-Type: application/json" \
  -d '{
    "mcp_name": "firecrawl",
    "config": {
      "command": "cmd",
      "args": ["/c", "set", "FIRECRAWL_API_KEY=your-key", "&&", "npx", "-y", "firecrawl-mcp"],
      "env": {"FIRECRAWL_API_KEY": "your-key"},
      "cwd": null
    }
  }'
```

### Updating the Configuration
```bash
curl -X PUT "http://localhost:8000/mcp-servers/external/update" \
  -H "Content-Type: application/json" \
  -d '{
    "mcp_name": "firecrawl",
    "config": {
      "command": "cmd",
      "args": ["/c", "set", "FIRECRAWL_API_KEY=new-key", "&&", "npx", "-y", "firecrawl-mcp"],
      "env": {"FIRECRAWL_API_KEY": "new-key"},
      "cwd": "/new/path"
    }
  }'
```

### Deleting the Server
```bash
curl -X DELETE "http://localhost:8000/mcp-servers/external/delete" \
  -H "Content-Type: application/json" \
  -d '{
    "mcp_name": "firecrawl"
  }'
```

## Testing Results

✅ **All tests passed successfully:**
- ✅ CREATE API with file replacement
- ✅ UPDATE API with configuration changes  
- ✅ DELETE API with file removal
- ✅ Import functionality with CustomMCPToolset integration
- ✅ Proper error handling for missing files

The external MCP server API system is now fully functional and ready for production use!
