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

#### `POST /api/generate/required-providers`
Builds `terraform.required_providers.tf` for the providers in use. Same core as the CLI's `--required-providers` and
the MCP tool `generate_required_providers`.

**Request Body:**
```json
{
  "providers": [
    "hashicorp/azurerm@v4.81.0",                                 // version -> "~>4.0"
    "microsoft/fabric@~>1.14",                                   // explicit constraint
    "hashicorp/azuread",                                         // latest manifest's version
    { "source": "databricks/databricks", "version": "v1.115.0", "name": "databricks" }
  ],
  "existing": "terraform { required_providers { ... } }",       // (Optional) current file, merged
  "mode": "merge"                                                // (Optional) "merge" (default) or "replace"
}
```
**Response:**
```json
{
  "path": "terraform.required_providers.tf",
  "content": "terraform {\n  required_providers {\n ...",
  "providers": [{ "name": "azurerm", "source": "hashicorp/azurerm", "constraint": "~>4.0" }],
  "warnings": []
}
```
Merging replaces listed providers by local name, keeps other entries, and never lowers a constraint unless one is
given explicitly.

#### `POST /api/generate/provider-files`
Rebuilds a provider's shared files from the modules a repo actually has: `<provider>.modules.tf`,
`<provider>.variables.tf`, tests, the defaults module, `<resource>.module.tf` and the defaults registry. No module
is generated. Same core as the CLI's `--provider-files` and the MCP tool `generate_provider_files`.

**Request Body:**
```json
{
  "manifest": { ... },          // Provider manifest (or developer/provider/version + cloudSettings)
  "files": [                    // The repo's .tf files, repo-relative: module folders and existing shared files
    { "path": "hashicorp/azurerm/key_vault/key_vault.variables.tf", "content": "..." }
  ],
  "modules": ["storage_account"],  // (Optional) In use but not in `files` yet: wired, not generated
  "cloudSettings": { ... }      // (Optional) Settings repo lookup exceptions in cloud mode
}
```
**Response:**
```json
{
  "files": [{ "path": "hashicorp/azurerm/azurerm.modules.tf", "content": "...", "status": "changed" }],
  "inventory": [{ "resource": "key_vault", "source": "repo", "dependencies": ["azuread_group_output"], ... }],
  "skipped": [{ "folder": "hashicorp/azurerm/key_vault_secret_random", "reason": "no azurerm_key_vault_secret_random_data variable (custom or composite module)" }],
  "presentModules": ["azuread_group", "azurerm_key_vault", ...],
  "warnings": []
}
```
`status` is `added`, `changed` or `unchanged` against the files sent. Each module is wired from its own files
(its `*_output` variables and `*_output_names` output); existing shared files are merged by block, so
hand-written blocks are kept.

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
      "name": "generate_required_providers",
      "description": "Build terraform.required_providers.tf for the providers in use, merged with a local repo's file.",
      "inputSchema": {
        "type": "object",
        "properties": {
          "providers": { "type": "array", "items": { "type": "string" }, "description": "e.g. [\"hashicorp/azurerm@v4.81.0\"]" },
          "repoDir": { "type": "string", "description": "Optional local repo checkout whose file is merged" },
          "mode": { "type": "string", "enum": ["merge", "replace"] },
          "write": { "type": "boolean", "description": "Write the file into repoDir" }
        },
        "required": ["providers"]
      }
    },
    {
      "name": "generate_provider_files",
      "description": "Rebuild a provider's shared files from the modules a local repo already has (no module folder is changed).",
      "inputSchema": {
        "type": "object",
        "properties": {
          "manifest": { "type": "object", "description": "The provider manifest" },
          "repoDir": { "type": "string", "description": "Absolute path of the local module repo checkout" },
          "modules": { "type": "array", "items": { "type": "string" }, "description": "In use but not in the repo yet" },
          "write": { "type": "boolean", "description": "Write the added and changed files into repoDir" }
        },
        "required": ["manifest", "repoDir"]
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

### Naming Scheme Generator

#### `GET /api/naming-schema`
Retrieves the naming scheme data for a given provider.

**Query Parameters:** `?developer=hashicorp&provider=azurerm`

**Response:** JSON object containing `definitions`, `schemes`, `policies`, and `globalPolicy`.

#### `POST /api/naming-schema`
Saves naming scheme data for a provider to the local filesystem.

**Request Body:**
```json
{
  "developer": "hashicorp",
  "provider": "azurerm",
  "definitions": [ ... ],
  "schemes": { "azurerm_storage_account": { ... } },
  "policies": { "azurerm_storage_account": { ... } },
  "globalPolicy": { ... }
}
```
**Response:** JSON indicating success and the path where the file was saved:
`public/defaults/naming/[developer]/[provider].json`

#### `POST /api/naming-schema/ai`
Uses the configured AI provider to generate or audit a naming scheme.

**Request Body:**
```json
{
  "action": "generate",  // or "audit"
  "prompt": "Include project prefix, environment, and resource code separated by hyphens",
  "developer": "hashicorp",
  "provider": "azurerm",
  "resource": "azurerm_storage_account",
  "aiSettings": { ... }
}
```
**Response (generate):** JSON object containing `components`, `separator`, and `explanation`.
**Response (audit):** JSON object containing `markdown` audit report.

---

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