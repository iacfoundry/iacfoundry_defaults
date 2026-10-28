# Welcome to IaC Foundry

IaC Foundry is a powerful tool designed to accelerate the creation of reusable, high-quality Terraform modules. It leverages structured markdown documentation from Terraform providers and flexible manifest files to parse arguments, infer relationships, and generate standardized module code.

## Key Concepts

### 1. Manifests

Manifests are the heart of the IaC Foundry. They are JSON files that instruct the parser on how to interpret a provider's documentation. A manifest defines:

-   **Global Settings**: Provider name, developer, version, and patterns for finding documentation.
-   **Parsing Rules**: Regular expressions and structural hints that tell the parser how to find arguments, blocks, and their attributes (e.g., required, optional).
-   **Aliasing and Dependencies**: Rules to create convenient lookup aliases (e.g., creating a `subnet_name` variable from a `subnet_id` argument) and to infer dependencies between resources.

### 2. The Generation Engine

The engine uses the manifest to read a specific resource's documentation page (in markdown format). It builds an in-memory schema of the resource, including all its arguments and nested blocks. This schema is then used as a blueprint to generate the following files:

-   `resource.tf`: The main resource block with dynamic content for arguments.
-   `variables.tf`: A strongly-typed variable definition for the module's input.
-   `outputs.tf`: Standardized outputs for the created resources.
-   `README.md`: Basic documentation for the generated module.

### 3. The User Interface

The web UI provides a dashboard to:

-   Select a provider and version.
-   Choose one or more resources to generate modules for.
-   View and interact with a dependency graph of the provider's resources.
-   Preview the generated code before downloading.
-   Customize the generation process through advanced manifest settings.

## Getting Started

1.  **Configure Manifests**: Go to **Content Settings** and either import existing manifest files or create a new one from scratch. You can also sync them from a GitHub repository.
2.  **Select a Provider**: Use the Provider Selector on the main page to choose the provider and version you want to work with.
3.  **Select Resources**: From the "Resources" dropdown, select one or more resources you wish to generate modules for.
4.  **Preview and Download**: The "Module Cast" tab will show a preview of the generated files. You can then download the complete module as a ZIP archive.
