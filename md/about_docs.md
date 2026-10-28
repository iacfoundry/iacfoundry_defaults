# Welcome to IaC Foundry

IaC Foundry is a powerful tool designed to accelerate the creation of reusable, high-quality Terraform modules. It leverages structured markdown documentation from Terraform providers and flexible manifest files to parse arguments, infer relationships, and generate standardized module code.

## Key Concepts

### 1. Manifests

Manifests are the heart of the IaC Foundry. They are JSON files that instruct the parser on how to interpret a provider's documentation. A manifest defines:

-   **Global Settings**: Provider name, developer, version, and patterns for finding documentation.
-   **Schema Configuration**: Utilizes the official Terraform Provider Schema (via `terraform providers schema -json`) for accurate definitions.
-   **Aliasing and Dependencies**: Rules to create convenient lookup aliases (e.g., creating a `subnet_name` variable from a `subnet_id` argument) and to infer dependencies between resources.

### 2. The Generation Engine

The engine utilizes the Provider Schema (or parses documentation markup) to build a complete model of the resource. This blueprint is used to generate a suite of files:

-   `resource.tf`: The main resource block.
-   `variables.tf`: Strongly-typed variable definitions.
-   `outputs.tf`: Standardized outputs.
-   `examples.tfvars`: Example variable values (optionally enriched by AI).
-   `README.md`: Module documentation.

Additionally, the engine generates **Provider Files** (`modules.tf`, `provider.tf`, `variables.tf`) at the root level to help you instantiate the modules immediately.

**Output Options:**
You can interact with these generated files in multiple ways:
-   **View**: Inspect the code directly in the browser's editor tabs.
-   **Copy**: Copy individual file contents to your clipboard.
-   **Download**: Export the entire set of modules and provider files as a structured ZIP archive.

### 3. The User Interface

The web UI provides a dashboard to:

-   Select a provider and version.
-   Choose one or more resources to generate modules for.
-   View and interact with a dependency graph of the provider's resources.
-   Preview the generated code before downloading.
-   Customize the generation process through advanced manifest settings.

### 4. AI Integration

IaC Foundry integrates with AI providers (OpenAI, Azure AI Foundry, Anthropic, Gemini, or Local LLMs) to supercharge your workflow.

-   **Intelligent Variable Examples**: The AI can analyze your generated variables and create realistic `terraform.tfvars` examples, complete with valid values and comments.
-   **Smart Documentation**: Enhance resource documentation with AI-generated summaries and usage tips.

### 5. Advanced Editors

-   **Provider Files Editor**: Directly edit and customize the `modules.tf`, `provider.tf`, and `variables.tf` files that wrap your generated modules.
-   **Import Workflow**: Import existing Azure resources (and more coming soon) to jumpstart your configuration.

## Getting Started

1.  **Configure Manifests**: Go to **Content Settings** and either import existing manifest files or create a new one from scratch. You can also sync them from a GitHub repository.
2.  **Select a Provider**: Use the Provider Selector on the main page to choose the provider and version you want to work with.
3.  **Select Resources**: 
    -   Use the dropdown to select resources one by one.
    -   Use the **Paste List** button to bulk-add a list of resources.
    -   Use the **Graph** view to select resources and their dependencies visually.
4.  **Preview and Download**: The "Module Generator" tab will show a preview of the generated files. You can then download the complete module as a ZIP archive.
