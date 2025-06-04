# LLM Subsystem Documentation

## 1. Overview

This document provides an overview of the Language Model (LLM) subsystem within the OpenCode application. This subsystem is responsible for all interactions with large language models, including managing different model providers, constructing prompts, defining agentic behavior, and providing tools for the agents to use.

## 2. Directory Structure

The LLM code is organized within the `internal/llm/` directory. Key subdirectories include:

- **`agent/`**: Contains the logic for AI agents, defining how they process information, interact with users (via the main application), and utilize available tools.
- **`models/`**: Defines metadata and properties for various supported LLM models (e.g., context window size, pricing, provider-specific identifiers).
- **`prompt/`**: Handles the construction and templating of system prompts and user-facing prompts sent to the LLMs for different tasks (e.g., coding, summarization, title generation).
- **`provider/`**: Manages communication with different LLM providers (e.g., OpenAI, Anthropic, Gemini, Azure), abstracting their specific APIs behind a common interface.
- **`tools/`**: Contains implementations of various tools that LLM agents can use to interact with the local environment (e.g., file system operations, command execution, code searching).

## 3. Core Components and Flow

### 3.1. Agent (`internal/llm/agent/`)

- **Purpose**: Orchestrates the interaction between the user, the LLM, and the available tools.
- **Key Files**:
    - `agent.go`: Defines the main `Agent` struct and `Service` interface. Handles the lifecycle of a request, including message processing, tool execution, and streaming responses.
    - `tools.go`: Registers and provides sets of tools for different agent types (e.g., `CoderAgentTools`, `TaskAgentTools`).
    - `agent-tool.go`: Implements a special "agent" tool that allows one agent to invoke another, more specialized agent (a form of recursive agent interaction).
    - `mcp-tools.go`: Implements integration with external tools via the Meta Call Protocol (MCP).
- **Functionality**:
    - Receives user requests and conversation history.
    - Selects an appropriate LLM provider and model.
    - Constructs prompts using context from `internal/llm/prompt/`.
    - Streams responses from the LLM, handling text deltas and tool usage requests.
    - Executes tools (from `internal/llm/tools/`) when requested by the LLM.
    - Manages session state, including token usage and costs.
    - Supports cancellation of ongoing requests.
    - Can summarize conversations.

### 3.2. Models (`internal/llm/models/`)

- **Purpose**: Provides a centralized registry and definition for all supported LLM models.
- **Key Files**:
    - `models.go`: Defines the `Model` struct (containing ID, name, provider, API model name, costs, context window, capabilities like `CanReason`, `SupportsAttachments`) and the global `SupportedModels` map.
    - Provider-specific files (e.g., `openai.go`, `anthropic.go`): Define model constants and populate `Model` structs for specific providers.
    - `local.go`: Handles dynamic discovery and registration of locally hosted LLMs.
- **Functionality**:
    - Allows the system to be aware of various models and their properties.
    - Used by the `agent` and `provider` to select and configure models for API calls.

### 3.3. Prompt (`internal/llm/prompt/`)

- **Purpose**: Constructs and manages the system prompts and contextual information provided to the LLMs.
- **Key Files**:
    - `prompt.go`: Contains `GetAgentPrompt` which assembles the final system prompt by combining a base prompt with dynamic context.
    - `coder.go`, `summarizer.go`, `task.go`, `title.go`: Define the base system prompts for different agent types and tasks, often tailored for specific LLM providers (e.g., OpenAI vs. Anthropic).
- **Functionality**:
    - `GetAgentPrompt` dynamically fetches project-specific context from files defined in `config.ContextPaths` (e.g., `OpenCode.md`, `.cursorrules`).
    - Includes environment information (OS, CWD, git status, file listing) in prompts for coding agents.
    - Provides detailed instructions to the LLM on its role, expected behavior, tool usage, and output format.

### 3.4. Provider (`internal/llm/provider/`)

- **Purpose**: Abstracts the communication details for various LLM providers.
- **Key Files**:
    - `provider.go`: Defines the `Provider` interface (with `SendMessages` and `StreamResponse` methods), `ProviderEvent` types, and a factory `NewProvider` to instantiate specific provider clients.
    - Provider-specific files (e.g., `openai.go`, `anthropic.go`, `gemini.go`): Implement the client logic for each provider, handling API requests, authentication, message/tool format conversion, streaming, and error handling (including retries).
    - Wrapper files (e.g., `azure.go`, `bedrock.go`, `vertexai.go`): Adapt specific provider SDKs (often reusing `openaiClient` or `geminiClient` if APIs are compatible) for services like Azure OpenAI, AWS Bedrock, and Google Vertex AI.
- **Functionality**:
    - Provides a consistent interface for the `agent` to send requests and receive responses/streams.
    - Handles provider-specific authentication and request/response formats.
    - Implements retry logic for transient errors (e.g., rate limits).
    - Converts between the internal message/tool format and the provider's format.

### 3.5. Tools (`internal/llm/tools/`)

- **Purpose**: Defines a collection of functions that the LLM agent can request to be executed to interact with the environment or fetch information.
- **Key Files**:
    - `tools.go`: Defines the `BaseTool` interface (`Info` and `Run` methods), `ToolInfo`, `ToolCall`, and `ToolResponse` structs.
    - Individual tool files (e.g., `bash.go`, `edit.go`, `view.go`, `glob.go`, `grep.go`, `fetch.go`, `sourcegraph.go`, `diagnostics.go`): Implement specific tools.
    - `shell/shell.go`: Implements a persistent shell for the `bashTool`.
    - `file.go`: Utility to track file read/write times for safety checks in tools like `editTool`.
- **Functionality**:
    - Each tool provides:
        - `Info()`: Metadata (name, description, JSON schema for parameters) for the LLM to understand its capabilities.
        - `Run()`: The actual Go code to execute the tool's action.
    - Tools often interact with `permission.Service` to request user approval for sensitive operations.
    - Tools like `editTool`, `patchTool`, and `writeTool` interact with `history.Service` to record file changes.
    - Tools like `viewTool`, `editTool`, `diagnosticsTool` can interact with LSP clients for enhanced functionality.

## 4. How to Add New Features

### 4.1. Adding a New LLM Model

1.  **Define in `internal/llm/models/`**:
    *   If the model is from an existing provider (e.g., a new OpenAI model), add its `ModelID` constant and `Model` struct definition to the provider's file (e.g., `openai.go`).
    *   If it's a new provider, create a new file (e.g., `newprovider.go`) and define its models there.
2.  **Update `SupportedModels`**: Ensure your new model map is merged into the global `SupportedModels` map in `internal/llm/models/models.go`'s `init()` function.
3.  **Provider Implementation**: If it's a new provider, you'll also need to implement it in `internal/llm/provider/` (see next section).
4.  **Configuration**: Users will then be able to select this model in the application configuration or through the UI if the provider is correctly set up.

### 4.2. Adding a New LLM Provider

1.  **Create Provider File**: In `internal/llm/provider/`, create `newprovider.go`.
2.  **Implement `ProviderClient`**: Define a struct (e.g., `newProviderClient`) and implement the `send()` and `stream()` methods to interact with the new provider's API. This will involve:
    *   Using the provider's SDK or making HTTP requests.
    *   Converting messages and tool definitions to the provider's format.
    *   Handling streaming events and converting them to `ProviderEvent`s.
    *   Mapping finish reasons and extracting token usage.
3.  **Update `NewProvider`**: In `internal/llm/provider/provider.go`, add a case for your new `models.ModelProvider` constant in the `NewProvider` factory function to return an instance of your new client.
4.  **Model Definitions**: Add model definitions for this provider in `internal/llm/models/` as described above.
5.  **Configuration**: Add configuration options for the new provider in `internal/config/config.go` (e.g., API key, endpoint).

### 4.3. Adding a New Tool

1.  **Create Tool File**: In `internal/llm/tools/`, create `mytool.go`.
2.  **Define Tool Struct**: Create a struct for your tool.
3.  **Implement `BaseTool` Interface**:
    *   `Info() ToolInfo`: Return the tool's name, a detailed description for the LLM on how and when to use it, and a JSON schema for its parameters.
    *   `Run(ctx context.Context, call ToolCall) (ToolResponse, error)`: Implement the tool's logic. Parse input from `call.Input` (JSON string), perform the action, and return a `ToolResponse`.
4.  **Register Tool**: In `internal/llm/agent/tools.go`, add an instance of your new tool to the appropriate tool list (e.g., `CoderAgentTools`).
5.  **Permissions**: If the tool performs sensitive actions, integrate with `permission.Service` within its `Run` method.
6.  **History**: If the tool modifies files, consider integrating with `history.Service`.

```
