# Grist MCP Server Setup for Claude Code Router (CCR) with Ollama/Devstral

This guide explains how to set up the Grist MCP server to work with Claude Code Router (CCR) pointing to an Ollama model (devstral).

## Table of Contents

1. [Overview](#overview)
2. [Prerequisites](#prerequisites)
3. [Grist MCP Server Installation](#grist-mcp-server-installation)
4. [MCP Server Configuration](#mcp-server-configuration)
5. [CCR Configuration](#ccr-configuration)
6. [Testing the Setup](#testing-the-setup)
7. [Available Tools Reference](#available-tools-reference)
8. [Troubleshooting](#troubleshooting)

---

## Overview

### What is MCP (Model Context Protocol)?

MCP is a standardized protocol that allows language models to interact with external tools and data sources. The Grist MCP server implements this protocol to enable LLMs to read and manipulate Grist spreadsheet data.

### Architecture

```
┌─────────────────┐     ┌─────────────────┐     ┌─────────────────┐
│  Claude Code    │     │    MCP Server   │     │   Grist API     │
│  Router (CCR)   │◄───►│  (Python/stdio) │◄───►│   (REST API)    │
│  + Ollama       │     │                 │     │                 │
└─────────────────┘     └─────────────────┘     └─────────────────┘
```

The MCP server acts as a bridge:
- Receives tool calls from CCR/Ollama via stdio (stdin/stdout)
- Translates them to Grist API requests
- Returns results back to the model

---

## Prerequisites

### System Requirements

- Python 3.8 or higher
- pip (Python package manager)
- Ollama with devstral model installed
- Claude Code Router (CCR) installed

### Required Accounts

- **Grist Account**: Get your API key from [Grist Settings](https://docs.getgrist.com/) → Profile Settings → API

---

## Grist MCP Server Installation

### Step 1: Clone/Copy the Server Files

Create a directory for the MCP server:

```bash
mkdir -p ~/mcp-servers/grist-mcp
cd ~/mcp-servers/grist-mcp
```

### Step 2: Create the Server Script

Create `grist_mcp_server.py` with the following content:

```python
#!/usr/bin/env python3
"""
Grist MCP Server - Provides MCP tools for interacting with Grist API

This server implements the Model Context Protocol (MCP) for Grist,
enabling language models to interact with Grist spreadsheets.
"""

import json
import os
import logging
import sys
from typing import List, Dict, Any, Optional, Union

import httpx
from pydantic import BaseModel, Field, AnyHttpUrl
from dotenv import load_dotenv

try:
    from mcp.server.fastmcp import FastMCP, Context
except ImportError:
    print("Error: fastmcp package not found. Please install it with: pip install fastmcp")
    sys.exit(1)

# Version
__version__ = "0.1.0"

# Configure logging
log_level = os.environ.get("LOG_LEVEL", "INFO").upper()
log_levels = {
    "DEBUG": logging.DEBUG,
    "INFO": logging.INFO,
    "WARNING": logging.WARNING,
    "ERROR": logging.ERROR,
    "CRITICAL": logging.CRITICAL
}
level = log_levels.get(log_level, logging.INFO)

logging.basicConfig(
    level=level,
    format='%(asctime)s - %(name)s - %(levelname)s - %(message)s',
    handlers=[
        logging.FileHandler("grist_mcp_server.log", mode='w'),
        logging.StreamHandler(sys.stderr)  # Use stderr for logs, stdout for MCP
    ]
)
logger = logging.getLogger("grist_mcp_server")

# Load environment variables from .env file
load_dotenv()

# Mask sensitive information
def mask_api_key(api_key: str) -> str:
    """Mask the API key for logging purposes"""
    if len(api_key) > 10:
        return f"{api_key[:5]}...{api_key[-5:]}"
    return "[SET]"

# Create the MCP server with explicit name and instructions
mcp = FastMCP(
    name="Grist API Client",
    instructions="""
    You are an assistant specialized in interacting with the Grist API.
    You can use the following tools to interact with Grist data:

    1. Listing Tools:
       - list_organizations: Lists all accessible Grist organizations
       - list_workspaces: Lists workspaces within an organization
       - list_documents: Lists documents within a workspace
       - list_tables: Lists tables within a document
       - list_columns: Lists columns within a table
       - list_records: Lists records within a table

    2. Record Management Tools:
       - add_grist_records: Adds records to a table
       - update_grist_records: Updates existing records
       - delete_grist_records: Deletes records

    3. SQL Tools for Filtering:
       - filter_sql_query: SQL query optimized for filtering
       - execute_sql_query: General SQL query for complex queries
    """
)

# Models
class GristOrg(BaseModel):
    """Grist organization model"""
    id: int
    name: str
    domain: Optional[str] = None

class GristWorkspace(BaseModel):
    """Grist workspace model"""
    id: int
    name: str

class GristDocument(BaseModel):
    """Grist document model"""
    id: str
    name: str

class GristTable(BaseModel):
    """Grist table model"""
    id: str

class GristColumn(BaseModel):
    """Grist column model"""
    id: str
    fields: Dict[str, Any]

class GristRecord(BaseModel):
    """Grist record model"""
    id: int
    fields: Dict[str, Any]

# Client
class GristClient:
    """Client for the Grist API"""

    def __init__(self, api_key: str, api_url: str):
        self.api_key = api_key
        self.api_url = api_url
        self.headers = {
            "Authorization": f"Bearer {api_key}",
            "Content-Type": "application/json",
            "Accept": "application/json"
        }
        logger.debug(f"GristClient initialized with API URL: {api_url}")

    async def _request(self,
                      method: str,
                      endpoint: str,
                      json_data: Optional[Dict[str, Any]] = None,
                      params: Optional[Dict[str, Any]] = None) -> Any:
        """Make a request to the Grist API"""
        if not endpoint.startswith('/'):
            endpoint = '/' + endpoint
        api_url = self.api_url.rstrip('/')
        url = api_url + endpoint

        logger.debug(f"Making {method} request to {url}")

        try:
            async with httpx.AsyncClient(timeout=30.0) as client:
                response = await client.request(
                    method=method,
                    url=url,
                    headers=self.headers,
                    json=json_data,
                    params=params
                )

                if response.status_code >= 400:
                    logger.error(f"Error response: {response.text}")

                response.raise_for_status()
                return response.json()
        except httpx.HTTPStatusError as e:
            logger.error(f"HTTP error: {e}")
            raise ValueError(f"HTTP error: {e.response.status_code} - {e.response.text}")
        except httpx.RequestError as e:
            logger.error(f"Request error: {e}")
            raise ValueError(f"Request error: {str(e)}")

    async def list_orgs(self) -> List[GristOrg]:
        """List all organizations the user has access to"""
        data = await self._request("GET", "/orgs")
        if not isinstance(data, list):
            return []
        return [GristOrg(**org) for org in data]

    async def list_workspaces(self, org_id: Union[int, str]) -> List[GristWorkspace]:
        """List all workspaces in an organization"""
        data = await self._request("GET", f"/orgs/{org_id}/workspaces")
        if not isinstance(data, list):
            return []
        return [GristWorkspace(**workspace) for workspace in data]

    async def list_documents(self, workspace_id: int) -> List[GristDocument]:
        """List all documents in a workspace"""
        data = await self._request("GET", f"/workspaces/{workspace_id}")
        docs = data.get("docs", [])
        return [GristDocument(**doc) for doc in docs]

    async def list_tables(self, doc_id: str) -> List[GristTable]:
        """List all tables in a document"""
        data = await self._request("GET", f"/docs/{doc_id}/tables")
        return [GristTable(**table) for table in data.get("tables", [])]

    async def list_columns(self, doc_id: str, table_id: str) -> List[GristColumn]:
        """List all columns in a table"""
        data = await self._request("GET", f"/docs/{doc_id}/tables/{table_id}/columns")
        return [GristColumn(**column) for column in data.get("columns", [])]

    async def list_records(self, doc_id: str, table_id: str,
                        sort: Optional[str] = None,
                        limit: Optional[int] = None) -> List[GristRecord]:
        """List records in a table with optional sorting and limiting"""
        params = {}
        if sort:
            params["sort"] = sort
        if limit and limit > 0:
            params["limit"] = limit

        data = await self._request(
            "GET",
            f"/docs/{doc_id}/tables/{table_id}/records",
            params=params
        )
        return [GristRecord(**record) for record in data.get("records", [])]

    async def add_records(self, doc_id: str, table_id: str,
                       records: List[Dict[str, Any]]) -> List[int]:
        """Add records to a table"""
        if all("fields" in record for record in records):
            formatted_records = {"records": records}
        else:
            formatted_records = {"records": [{"fields": record} for record in records]}

        data = await self._request(
            "POST",
            f"/docs/{doc_id}/tables/{table_id}/records",
            json_data=formatted_records
        )
        return [record["id"] for record in data.get("records", [])]

    async def update_records(self, doc_id: str, table_id: str,
                          records: List[Dict[str, Any]]) -> List[int]:
        """Update records in a table"""
        if all(isinstance(record, dict) and "id" in record and "fields" in record for record in records):
            formatted_records = {"records": records}
        else:
            formatted_records = {"records": []}
            for record in records:
                if "id" not in record:
                    raise ValueError(f"Each record must contain an 'id' field: {record}")
                record_id = record.pop("id")
                formatted_records["records"].append({
                    "id": record_id,
                    "fields": record
                })

        data = await self._request(
            "PATCH",
            f"/docs/{doc_id}/tables/{table_id}/records",
            json_data=formatted_records
        )

        if data is None:
            return [record["id"] for record in formatted_records["records"]]
        elif "records" in data and isinstance(data["records"], list):
            return [record["id"] for record in data["records"]]
        else:
            return [record["id"] for record in formatted_records["records"]]

    async def delete_records(self, doc_id: str, table_id: str, record_ids: List[int]) -> None:
        """Delete records from a table"""
        await self._request(
            "POST",
            f"/docs/{doc_id}/tables/{table_id}/data/delete",
            json_data=record_ids
        )


# Get configuration from environment variables
def get_client(ctx: Optional[Context] = None) -> GristClient:
    """Get a configured Grist client"""
    api_key = os.environ.get("GRIST_API_KEY", "")
    api_url = os.environ.get("GRIST_API_URL", os.environ.get("GRIST_API_HOST", "https://docs.getgrist.com/api"))

    if not api_key:
        raise ValueError("GRIST_API_KEY environment variable is not set")

    if not api_url.startswith("http"):
        api_url = "https://" + api_url

    return GristClient(api_key=api_key, api_url=api_url)


# MCP Tools
@mcp.tool()
async def list_organizations(ctx: Context) -> List[Dict[str, Any]]:
    """
    List all Grist organizations that the user has access to.

    Returns a list of organizations with their IDs, names, and domains.
    """
    logger.info("Tool called: list_organizations")
    client = get_client(ctx)
    orgs = await client.list_orgs()
    return [org.dict() for org in orgs]


@mcp.tool()
async def list_workspaces(org_id: Union[int, str], ctx: Context) -> List[Dict[str, Any]]:
    """
    List all workspaces in a Grist organization.

    Args:
        org_id: The ID of the organization to list workspaces for

    Returns:
        A list of workspaces with their IDs and names
    """
    logger.info(f"Tool called: list_workspaces with org_id: {org_id}")
    client = get_client(ctx)
    workspaces = await client.list_workspaces(org_id)
    return [workspace.dict() for workspace in workspaces]


@mcp.tool()
async def list_documents(workspace_id: int, ctx: Context) -> List[Dict[str, Any]]:
    """
    List all documents in a Grist workspace.

    Args:
        workspace_id: The ID of the workspace to list documents for

    Returns:
        A list of documents with their IDs and names
    """
    logger.info(f"Tool called: list_documents with workspace_id: {workspace_id}")
    client = get_client(ctx)
    documents = await client.list_documents(workspace_id)
    return [document.dict() for document in documents]


@mcp.tool()
async def list_tables(doc_id: str, ctx: Context) -> List[Dict[str, Any]]:
    """
    List all tables in a Grist document.

    Args:
        doc_id: The ID of the document to list tables for

    Returns:
        A list of tables with their IDs
    """
    logger.info(f"Tool called: list_tables with doc_id: {doc_id}")
    client = get_client(ctx)
    tables = await client.list_tables(doc_id)
    return [table.dict() for table in tables]


@mcp.tool()
async def list_columns(doc_id: str, table_id: str, ctx: Context) -> List[Dict[str, Any]]:
    """
    List all columns in a Grist table.

    Args:
        doc_id: The ID of the document containing the table
        table_id: The ID of the table to list columns for

    Returns:
        A list of columns with their IDs and field data
    """
    logger.info(f"Tool called: list_columns with doc_id: {doc_id}, table_id: {table_id}")
    client = get_client(ctx)
    columns = await client.list_columns(doc_id, table_id)
    return [column.dict() for column in columns]


@mcp.tool()
async def list_records(
    doc_id: str,
    table_id: str,
    sort: Optional[str] = None,
    limit: Optional[int] = None,
    ctx: Context = None
) -> List[Dict[str, Any]]:
    """
    Lists records from a Grist table with optional sorting and limiting.

    Args:
        doc_id: The ID of the document containing the table
        table_id: The ID of the table to query
        sort: Optional column to sort by (prefix with '-' for descending)
        limit: Optional maximum number of records to return

    Returns:
        A list of records with their IDs and data
    """
    logger.info(f"Tool call: list_records with doc_id: {doc_id}, table_id: {table_id}")
    client = get_client(ctx)

    records = await client.list_records(
        doc_id=doc_id,
        table_id=table_id,
        sort=sort,
        limit=limit
    )

    return [record.dict() for record in records]


@mcp.tool()
async def add_grist_records(
    doc_id: str,
    table_id: str,
    records: List[Dict[str, Any]],
    ctx: Context
) -> List[int]:
    """
    Adds records to a Grist table.

    Args:
        doc_id: The ID of the Grist document
        table_id: The ID of the table
        records: List of records to add. Each record is a dictionary
                where keys are column names and values are the data.
                Example: [{"name": "Dupont", "first_name": "Jean", "age": 35}]

    Returns:
        List of IDs of the created records
    """
    logger.info(f"Tool called: add_grist_records for doc_id: {doc_id}, table_id: {table_id}")
    client = get_client(ctx)
    return await client.add_records(doc_id, table_id, records)


@mcp.tool()
async def update_grist_records(
    doc_id: str,
    table_id: str,
    records: List[Dict[str, Any]],
    ctx: Context
) -> List[int]:
    """
    Updates records in a Grist table.

    Args:
        doc_id: The ID of the Grist document
        table_id: The ID of the table
        records: List of records to update. Each record must contain
                an 'id' field and the fields to update.
                Example: [{"id": 1, "name": "Durand", "age": 36}]

    Returns:
        List of IDs of the updated records
    """
    logger.info(f"Tool called: update_grist_records for doc_id: {doc_id}, table_id: {table_id}")
    client = get_client(ctx)
    return await client.update_records(doc_id, table_id, records)


@mcp.tool()
async def delete_grist_records(
    doc_id: str,
    table_id: str,
    record_ids: List[int],
    ctx: Context
) -> Dict[str, Any]:
    """
    Delete records from a Grist table.

    Args:
        doc_id: The ID of the Grist document
        table_id: The ID of the table
        record_ids: List of record IDs to delete

    Returns:
        A dictionary containing the operation status and a message
    """
    logger.info(f"Tool called: delete_grist_records for doc_id: {doc_id}, table_id: {table_id}")
    client = get_client(ctx)

    if not record_ids:
        return {"success": False, "message": "No record IDs provided"}

    if not all(isinstance(record_id, int) for record_id in record_ids):
        return {"success": False, "message": "All record IDs must be integers"}

    try:
        await client.delete_records(doc_id, table_id, record_ids)
        return {"success": True, "message": f"{len(record_ids)} records deleted", "deleted_ids": record_ids}
    except Exception as e:
        return {"success": False, "message": f"Error: {str(e)}"}


@mcp.tool()
async def filter_sql_query(
    doc_id: str,
    table_id: str,
    columns: Optional[List[str]] = None,
    where_conditions: Optional[Dict[str, Any]] = None,
    order_by: Optional[str] = None,
    limit: Optional[int] = None,
    ctx: Context = None
) -> Dict[str, Any]:
    """
    Executes a filtering SQL query on a Grist table.

    Args:
        doc_id: ID of the document
        table_id: ID of the table to filter
        columns: List of columns to select (all by default)
        where_conditions: Filtering conditions as a dictionary
                         Example: {"organization": "OPSIA", "status": "active"}
        order_by: Column for sorting (optional)
        limit: Maximum number of results (optional)

    Returns:
        Results of the filtering query
    """
    client = get_client(ctx)

    columns_str = "*" if not columns else ", ".join(columns)
    sql_query = f"SELECT {columns_str} FROM {table_id}"

    if where_conditions:
        conditions = []
        for column, value in where_conditions.items():
            if isinstance(value, str):
                conditions.append(f"{column} = '{value}'")
            else:
                conditions.append(f"{column} = {value}")
        sql_query += " WHERE " + " AND ".join(conditions)

    if order_by:
        sql_query += f" ORDER BY {order_by}"

    if limit:
        sql_query += f" LIMIT {limit}"

    result = await client._request(
        "POST",
        f"/docs/{doc_id}/sql",
        json_data={"sql": sql_query}
    )

    return {
        "success": True,
        "query": sql_query,
        "record_count": len(result.get("records", [])),
        "records": result.get("records", [])
    }


@mcp.tool()
async def execute_sql_query(
    doc_id: str,
    sql_query: str,
    parameters: Optional[List[Any]] = None,
    timeout_ms: Optional[int] = 1000,
    ctx: Context = None
) -> Dict[str, Any]:
    """
    Executes a general SQL query on a Grist document.

    Args:
        doc_id: ID of the document
        sql_query: SQL query to execute (SELECT only)
        parameters: Parameters for the SQL query (optional)
        timeout_ms: Timeout in milliseconds (default 1000)

    Returns:
        Results of the SQL query
    """
    client = get_client(ctx)

    sql_query = sql_query.strip()
    if sql_query.endswith(";"):
        sql_query = sql_query[:-1]

    if not sql_query.lower().startswith(("select", "with")):
        return {"success": False, "message": "Only SELECT queries are allowed"}

    query_params = {"sql": sql_query}
    if parameters:
        query_params["args"] = parameters
    if timeout_ms:
        query_params["timeout"] = timeout_ms

    result = await client._request(
        "POST",
        f"/docs/{doc_id}/sql",
        json_data=query_params
    )

    return {
        "success": True,
        "query": sql_query,
        "record_count": len(result.get("records", [])),
        "records": result.get("records", [])
    }


def main():
    """Run the MCP server"""
    logger.info(f"Starting Grist MCP Server v{__version__}")

    # Log environment configuration
    env_vars = {
        "GRIST_API_KEY": mask_api_key(os.environ.get("GRIST_API_KEY", "")),
        "GRIST_API_URL": os.environ.get("GRIST_API_URL", ""),
        "LOG_LEVEL": os.environ.get("LOG_LEVEL", "INFO")
    }
    logger.info(f"Environment configuration: {env_vars}")

    # Test connection
    try:
        client = get_client()
        logger.info(f"Successfully initialized Grist client with URL: {client.api_url}")
    except Exception as e:
        logger.error(f"Failed to initialize Grist client: {e}")

    mcp.run()


if __name__ == "__main__":
    main()
```

### Step 3: Create requirements.txt

```bash
cat > requirements.txt << 'EOF'
fastmcp>=0.3.0
httpx>=0.24.0
pydantic>=2.0.0
python-dotenv>=1.0.0
EOF
```

### Step 4: Install Dependencies

```bash
pip install -r requirements.txt
```

### Step 5: Create Environment File

```bash
cat > .env << 'EOF'
# Grist API Key (required)
GRIST_API_KEY=your_api_key_here

# Grist API URL (default: docs.getgrist.com/api)
GRIST_API_URL=https://docs.getgrist.com/api

# Log level (optional)
LOG_LEVEL=INFO
EOF
```

Replace `your_api_key_here` with your actual Grist API key.

---

## MCP Server Configuration

### How MCP Communication Works

The MCP server communicates via stdio (standard input/output):
- **stdin**: Receives JSON-RPC requests from the client (CCR)
- **stdout**: Sends JSON-RPC responses back to the client
- **stderr**: Used for logging (doesn't interfere with MCP protocol)

This is critical: The server must not print anything to stdout except valid MCP protocol messages.

### Verifying the Server Works

Test the server manually:

```bash
# Set environment variables
export GRIST_API_KEY="your_api_key"
export GRIST_API_URL="https://docs.getgrist.com/api"

# Run the server (it will wait for input)
python3 grist_mcp_server.py
```

The server should start without errors. You can send it a test message (but for real testing, use CCR).

---

## CCR Configuration

### Understanding CCR MCP Configuration

Claude Code Router (CCR) uses a configuration file to define MCP servers. The configuration specifies how to launch and communicate with each MCP server.

### Configuration File Location

CCR typically reads MCP configuration from one of these locations:

1. **Project-level**: `.claude/settings.json` in your project directory
2. **User-level**: `~/.config/claude-code/settings.json`
3. **Global settings**: `~/.claude.json`

### Configuration Format

Add the Grist MCP server to your CCR configuration:

```json
{
  "mcpServers": {
    "grist": {
      "type": "stdio",
      "command": "python3",
      "args": [
        "/path/to/your/grist_mcp_server.py"
      ],
      "env": {
        "GRIST_API_KEY": "your_grist_api_key_here",
        "GRIST_API_URL": "https://docs.getgrist.com/api",
        "LOG_LEVEL": "INFO"
      }
    }
  }
}
```

### Key Configuration Options

| Option | Description | Example |
|--------|-------------|---------|
| `type` | Communication method | `"stdio"` (standard for MCP) |
| `command` | Executable to run | `"python3"` or full path |
| `args` | Arguments to pass | Path to the server script |
| `env` | Environment variables | API keys, URLs, etc. |

### Complete Example Configuration for CCR + Ollama

Here's a complete configuration example:

```json
{
  "mcpServers": {
    "grist": {
      "type": "stdio",
      "command": "python3",
      "args": [
        "/home/user/mcp-servers/grist-mcp/grist_mcp_server.py"
      ],
      "env": {
        "GRIST_API_KEY": "f00f47e21bf31c92cc431b90026676677960b9ea",
        "GRIST_API_URL": "https://docs.getgrist.com/api",
        "LOG_LEVEL": "INFO"
      }
    }
  }
}
```

### Using with Ollama/Devstral

When using CCR with Ollama (devstral model), the MCP server works the same way. The key considerations:

1. **Tool Support**: Devstral must support tool/function calling for MCP to work effectively
2. **Tool Schema**: The MCP server exposes tools that CCR translates to the model's tool format
3. **Model Capabilities**: Ensure devstral is capable of understanding and using the tool schema

#### Checking Devstral Tool Support

Verify that devstral supports function calling:

```bash
# Check model capabilities
ollama show devstral --json | jq '.capabilities'
```

If devstral supports tool use, CCR will automatically:
1. Discover available tools from the MCP server
2. Present them to the model in the appropriate format
3. Execute tool calls and return results

---

## Testing the Setup

### Step 1: Verify Server Starts

```bash
cd ~/mcp-servers/grist-mcp
export GRIST_API_KEY="your_key"
export GRIST_API_URL="https://docs.getgrist.com/api"
python3 grist_mcp_server.py
```

Check `grist_mcp_server.log` for startup messages.

### Step 2: Test with CCR

After configuring CCR, start a conversation and ask about Grist data:

```
"List my Grist organizations"
"What tables are in document hcmQR7zX3i5HqNwzCbi1jn?"
"Show me the columns in the Contractors table"
```

### Step 3: Check Logs

Monitor the log file for tool calls and responses:

```bash
tail -f ~/mcp-servers/grist-mcp/grist_mcp_server.log
```

---

## Available Tools Reference

### Organization & Navigation Tools

| Tool | Description | Parameters |
|------|-------------|------------|
| `list_organizations` | List all accessible organizations | None |
| `list_workspaces` | List workspaces in an org | `org_id` |
| `list_documents` | List documents in a workspace | `workspace_id` |
| `list_tables` | List tables in a document | `doc_id` |
| `list_columns` | List columns in a table | `doc_id`, `table_id` |

### Data Reading Tools

| Tool | Description | Parameters |
|------|-------------|------------|
| `list_records` | Get records from a table | `doc_id`, `table_id`, `sort?`, `limit?` |
| `filter_sql_query` | Simple SQL filtering | `doc_id`, `table_id`, `columns?`, `where_conditions?`, `order_by?`, `limit?` |
| `execute_sql_query` | Complex SQL queries | `doc_id`, `sql_query`, `parameters?`, `timeout_ms?` |

### Data Modification Tools

| Tool | Description | Parameters |
|------|-------------|------------|
| `add_grist_records` | Add new records | `doc_id`, `table_id`, `records` |
| `update_grist_records` | Update existing records | `doc_id`, `table_id`, `records` (with `id` field) |
| `delete_grist_records` | Delete records | `doc_id`, `table_id`, `record_ids` |

### Known Document IDs (BookKeeping Project)

| Document | Doc ID |
|----------|--------|
| Headwaters Book Keeping | `hcmQR7zX3i5HqNwzCbi1jn` |
| Christ the Anchor Book Keeping | `kEfMiMXXMwg5YGYfhrAnvo` |

---

## Troubleshooting

### Common Issues

#### 1. "GRIST_API_KEY environment variable is not set"

**Solution**: Ensure the API key is set in the `env` section of your CCR config:

```json
"env": {
  "GRIST_API_KEY": "your_actual_key_here"
}
```

#### 2. "fastmcp package not found"

**Solution**: Install the required packages:

```bash
pip install fastmcp httpx pydantic python-dotenv
```

#### 3. Server not responding

**Cause**: stdout/stderr confusion

**Solution**: Ensure logging goes to stderr, not stdout. The server code already handles this, but verify no print statements are going to stdout.

#### 4. Tool calls not working with devstral

**Cause**: Model may not support function calling

**Solution**:
- Verify devstral supports tool use
- Try a different model that supports function calling
- Check CCR logs for tool translation errors

#### 5. HTTP 401 Unauthorized

**Cause**: Invalid API key

**Solution**:
- Regenerate your API key in Grist settings
- Verify the key is correctly set in the configuration
- Check there are no extra spaces or quotes around the key

#### 6. Connection timeouts

**Cause**: Network issues or wrong API URL

**Solution**:
- Verify you can reach the Grist API: `curl -H "Authorization: Bearer YOUR_KEY" https://docs.getgrist.com/api/orgs`
- Check firewall settings
- Verify the API URL is correct

### Debug Mode

Enable debug logging for more information:

```json
"env": {
  "LOG_LEVEL": "DEBUG"
}
```

Then check `grist_mcp_server.log` for detailed request/response information.

### Verifying MCP Protocol

To verify the server speaks MCP correctly, you can use the MCP inspector:

```bash
# Install MCP inspector
npm install -g @modelcontextprotocol/inspector

# Run inspector
mcp-inspector python3 /path/to/grist_mcp_server.py
```

---

## Additional Resources

- [MCP Protocol Specification](https://modelcontextprotocol.io/)
- [Grist API Documentation](https://support.getgrist.com/api/)
- [FastMCP Documentation](https://github.com/jlowin/fastmcp)
- [Ollama Documentation](https://ollama.ai/docs)

---

## Version History

- **v1.0** - Initial setup guide for CCR + Ollama/devstral integration
