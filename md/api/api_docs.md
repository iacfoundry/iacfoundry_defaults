# IaC Foundry API & MCP Configuration

This document outlines the available API endpoints for the IaC Foundry application and provides a configuration for integrating these capabilities with the Model Context Protocol (MCP).

## 1. API Endpoints

### Core Generation

#### `POST /api/generate`
Generates Terraform modules based on a provided manifest and resource selection.

**Request Body:**
```json
{
  "manifest": { ... },       // (Optional) Full manifest object
  "resources": ["azurerm_resource_group"], // (Required) List of resources to generate
  "debug": false,            // (Optional) Enable debug output
  "aiSettings": {            // (Optional) AI configuration for example generation
    "enabled": true,
    "provider": "openai",
    "apiKey": "sk-...",
    "model": "gpt-4o"
  },
  "precalculated": { ... }   // (Optional) Pre-computed data for optimization
}
```
**Response:** Returns a `.zip` file of generated modules (or JSON if debug=true).

### Provider Intelligence

#### `POST /api/provider-graph`
Generates a dependency graph for provider resources using the provider schema.

**Request Body:**
```json
{
  "manifest": { ... },       // (Required) Manifest object
  "resources": ["..."],      // (Optional) Subset of resources to analyze
  "debug": false,            // (Optional) Include debug info
  "getResourceListOnly": true // (Optional) Return only list of resources
}
```
**Response:** JSON object containing `nodes` and `edges`, or a list of resource names if `getResourceListOnly` is true.

#### `GET /api/provider-schema`
Checks if a generated provider schema exists locally.

**Query Parameters:** `?developer=hashicorp&provider=azurerm&version=3.0.0`
**Response:** `{"exists": true}` or `{"exists": false}`

#### `POST /api/provider-schema`
Generates and saves a Terraform provider schema locally by running `terraform providers schema -json`.

**Request Body:**
```json
{
  "manifest": {
    "Global": {
      "developer": "hashicorp",
      "provider": "azurerm",
      "version": "3.0.0"
    }
  }
}
```
**Response:** JSON indicating success and the path to the saved schema.

#### `POST /api/resources`
Retrieves a list of available resources for a provider from the generated local schema.

**Request Body:**
```json
{
  "manifest": { 
     "Global": { ... } 
  }
}
```
**Response:** JSON array of resource names (e.g., `["azurerm_resource_group", ...]`).

### Configuration & Management

#### `POST /api/fetch-manifests`
Retrieves a list of available manifests based on the current cloud settings (Local Defaults or GitHub).

**Request Body:**
```json
{
  "cloudSettings": {
    "mode": "defaults" // or "cloud"
    // ... validation details for cloud mode
  }
}
```
**Response:** JSON array of manifest objects.

#### `POST /api/test-ai-connection`
Tests the connection to a configured AI provider.

**Request Body:**
```json
{
  "enabled": true,
  "provider": "openai", // "openai" | "anthropic" | "gemini" | "local" | "azure-ai-foundry"
  "apiKey": "sk-...",
  "customBaseUrl": "...", // Optional
  "model": "gpt-4o"       // Optional
}
```
**Response:** JSON message confirming success or failure.

#### `POST /api/import`
Imports existing infrastructure resources.

**Request Body:**
```json
{
  "provider": "azurerm",
  "credentials": { ... } // Provider-specific credentials
}
```
**Response:** JSON object containing imported resource details.

---

## 2. MCP Configuration

To expose these API capabilities as tools within an MCP ecosystem, use the following tool definitions. This configuration maps the Foundry API endpoints to MCP Tools.

```json
{
  "tools": [
    {
      "name": "generate_modules",
      "description": "Generate Terraform modules for specified resources.",
      "inputSchema": {
        "type": "object",
        "properties": {
          "manifest": { "type": "object", "description": "The provider manifest" },
          "resources": { "type": "array", "items": { "type": "string" }, "description": "List of resources to generate" },
          "aiSettings": { "type": "object", "description": "AI configuration" }
        },
        "required": ["manifest", "resources"]
      }
    },
    {
      "name": "get_resource_graph",
      "description": "Generate a dependency graph for Terraform provider resources.",
      "inputSchema": {
        "type": "object",
        "properties": {
          "manifest": { "type": "object" },
          "resources": { "type": "array", "items": { "type": "string" } },
          "getResourceListOnly": { "type": "boolean" }
        },
        "required": ["manifest"]
      }
    },
    {
      "name": "check_provider_schema",
      "description": "Check if a local provider schema exists.",
      "inputSchema": {
        "type": "object",
        "properties": {
          "developer": { "type": "string" },
          "provider": { "type": "string" },
          "version": { "type": "string" }
        },
        "required": ["developer", "provider", "version"]
      }
    },
    {
      "name": "generate_provider_schema",
      "description": "Generate a new provider schema locally via Terraform CLI.",
      "inputSchema": {
        "type": "object",
        "properties": {
          "manifest": { "type": "object", "description": "Manifest containing Global provider settings" }
        },
        "required": ["manifest"]
      }
    },
    {
      "name": "list_resources",
      "description": "List available resources for a provider from the local schema.",
      "inputSchema": {
        "type": "object",
        "properties": {
           "manifest": { "type": "object" }
        },
        "required": ["manifest"]
      }
    },
    {
      "name": "fetch_manifests",
      "description": "List available provider manifests.",
      "inputSchema": {
        "type": "object",
        "properties": {
          "cloudSettings": { 
             "type": "object",
             "properties": {
                 "mode": { "type": "string", "enum": ["defaults", "cloud"] }
             }
          }
        }
      }
    },
    {
      "name": "test_ai_connection",
      "description": "Test connectivity with an AI provider.",
      "inputSchema": {
        "type": "object",
        "properties": {
          "provider": { "type": "string", "enum": ["openai", "anthropic", "gemini", "local", "azure-ai-foundry"] },
          "apiKey": { "type": "string" },
          "customBaseUrl": { "type": "string" },
          "model": { "type": "string" }
        },
        "required": ["provider", "apiKey"]
      }
    },
    {
      "name": "import_resources",
      "description": "Import existing infrastructure resources.",
      "inputSchema": {
        "type": "object",
        "properties": {
          "provider": { "type": "string" },
          "credentials": { "type": "object" }
        },
        "required": ["provider", "credentials"]
      }
    }
  ]
}
```

### Standalone Server Implementation

A dedicated MCP server implementation is included in the `mcp-server` directory of this project. It is pre-configured to proxy requests to the IaC Foundry API.

#### Default Configuration
- **MCP Server Port**: `9003` (SSE Endpoint: `http://localhost:9003/sse`)
- **Target API URL**: `http://localhost:9002`

#### Running the Server
1. Ensure the main Next.js application is running (default port 9002).
2. Navigate to the `mcp-server` directory.
3. Run `npm install` (first time only) and then `npm run dev`.

The server supports environment variable overrides:
```bash
PORT=9004 TARGET_API_URL=http://localhost:9002 npm run dev
```