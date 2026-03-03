# /ai/assets/docs/sera_guides/sera_orchestrator.md
# Orchestrator Architecture & Development Guide

**Version:** 1.0 (Updated based on current codebase and settings)  
**Date:** 2025-08-08  
**Author:** Martin Kecka & Gemini AI  
**AI Name:** SERA: SysEng Reasoning AI (or SysEng Reasoning Assistant) -> current name for Orchestrator (for SysEng team only)  
**AI Name 2:** GIA: GTT Intelligent Assistant (Simple and direct) -> future name of global Orchestrator

This document describes the core orchestrator for the Modular AI Assistant. It is designed to leverage Large Language Models (LLMs) and interact with a suite of independent, external tool services/applications via APIs. This guide details the orchestrator's architecture, its method for configuring and calling external tools, its data store strategies, and the development plan.

## 1. Architecture Overview

**Primary Environment:** Local Development Server (e.g., `ukslo-crp-ai01` with L4 GPU), aiming for functional equivalence with a future Kubernetes deployment.

**Orchestrator's LLM Options (for its own reasoning, planning, and synthesis):**

- **Flexible LLM Integration:** The `LLMInterface` manages communication with various LLM providers. The primary reasoning model is `gemma3:27b`, configured via the `chat_endpoints` list in `config/settings.yaml`, which allows for multiple, selectable chat models in the UI. For semantic search, the system uses a dedicated embedding model (e.g., `nvidia/llama-3.2-nv-embedqa-1b-v2` via NVIDIA NIM) to create vector representations of conversational history.

**Core Orchestrator Components:**

- **Streamlit UI (`app_streamlit_core.py`):** Frontend for user interaction, providing a dynamic chat interface with features like:
  - Conversation history loading, pinning, and deletion from a persistent PostgreSQL database.
  - Dynamic persona and LLM model selection per chat.
  - Real-time display of LLM responses, tool calls, tool results, and errors.
  - Automatic chat title generation.
- **Main Orchestrator Script (`sera_orchestrator.py`):** Main application entry point, handles CLI arguments (including `--reset` for databases), starts UI or gRPC server, and initializes the `Orchestrator` instance.
- **Core Orchestrator (`orchestrator.py`):** Initializes all components, handles the main `process_query` flow, manages an in-memory session state cache (turn index, working memory, confusion notes), coordinates sub-modules, and triggers asynchronous history summarization. Implements correction/confusion detection based on keywords.
- **Orchestration Sub-Modules (`/orchestration/`):**
  - **`config_loader.py`:** Loads `config/settings.yaml`, `config/personas.yaml`, tool definitions from `config/tools.d/*.yaml`, and `config/vdb_schema.yaml`. Manages environment variable overrides and derives combined configurations (e.g., `AVAILABLE_ORCHESTRATOR_LLM_MODELS_FOR_UI`).
  - **`llm_interface.py`:** Manages clients for various LLM providers (NVIDIA NIM, Ollama) based on the `chat_endpoints` and embedding model configurations. Each configured endpoint is treated as a distinct client.
  - **`tool_executor.py`:** Dispatches requests to external gRPC tool services. Features dynamic loading of compiled gRPC stubs at runtime based on tool definitions, allowing for modular tool integration.
  - **`history_manager.py`:** Manages saving/retrieving conversational history using `VDBHandler` (for raw turn embeddings in `history_store` and summary embeddings in `summary_store`) and `SQLHandler` (for structured turn data and summaries in PostgreSQL).
  - **`history_summarizer.py`:** Generates brief and detailed summaries of conversation turns using the LLM. Stores summaries in PostgreSQL and their embeddings (for brief summaries) in a dedicated VDB collection. Triggered asynchronously by the Orchestrator.
  - **`context_builder.py`:** Assembles LLM prompts, incorporating system prompts (from persona configurations), conversation history (raw and semantic), current working memory, the current query, tool definitions, and fetched tool usage guides. Manages token limits, generates/updates working memory summaries, incorporates "confusion notes" from previous turns, and includes specific handling for Gemma-style tool instructions.
  - **`token_utils.py`:** Provides utility functions for counting tokens using `tiktoken`.
  - **`io_handlers/`:**
    - **`vdb_handler.py` (`VDBHandler` class):** Encapsulates low-level ChromaDB client interactions for specified collections.
    - **`sql_handler.py` (`SQLHandler` class):** Manages PostgreSQL interactions using SQLAlchemy for `ConversationSession`, `HistoryTurn`, and `HistoryTurnSummary` models.
- **Orchestrator Inbound gRPC API Server (`grpc_api/server.py`, `grpc_api/services.py`):**
  - Exposes gRPC services (e.g., `OrchestratorInteractionService` with `ProcessInstruction` method) allowing authenticated external tools to request complex processing, leveraging the Orchestrator's LLM and tool integration capabilities.
- **External Tool Services/Applications:** Independent applications/services with their own gRPC APIs. The Orchestrator interacts with them via the `ToolExecutor`.
- **Shared Database Models (`/database/models.py`):** Central SQLAlchemy ORM models for PostgreSQL: `ConversationSession`, `HistoryTurn`, `HistoryTurnSummary`. Note: `AuditLog` model mentioned in earlier design phases is not currently implemented.
- **Relational Database (PostgreSQL):** Used for structured conversation history, turn summaries, and session management.
- **Vector Database Service (ChromaDB):** Used by `HistoryManager` for the `history_store` collection (raw turn embeddings) and the `summary_store` collection (summary embeddings).
- **Shared Asset Storage:** Local filesystem directory used by `ContextBuilder` for caching fetched tool guides.

**Simplified Flow (Iterative):** UI (`app_streamlit_core.py`) -> Orchestrator (`process_query`) -> `HistoryManager` (retrieve history from VDBs & PSQL) -> `ContextBuilder` (assemble context including system prompt, working memory, confusion notes, tool definitions, and relevant history) -> `LLMInterface` (Orchestrator's LLM plans/selects external tool or generates response) -> Orchestrator (`ToolExecutor` makes gRPC call to external tool service if needed) -> External Tool Service (executes, returns gRPC response) -> Orchestrator (receives response, updates context) -> `LLMInterface` (Orchestrator's LLM synthesizes final response or plans further, iterative tool use up to `MAX_TOOL_ITERATIONS` from config) -> UI. `HistoryManager` saves turn to VDB and PSQL. `HistorySummarizer` is triggered asynchronously for the turn.

---

## 2. Orchestrator Project Structure

/sera_orchestrator/
├── sera_orchestrator.py         # Main application entry point, CLI, starts UI or gRPC server
├── app_streamlit_core.py        # Streamlit UI logic
├── orchestrator.py              # Core Orchestrator class
├── bootstrap_helper.py          # (Simple script for venv creation and pip install -r requirements.txt)
├── requirements.txt
│
├── grpc_api/                    # For inbound gRPC API
│   ├── __init__.py
│   ├── protos/                  # .proto Protocol Buffer definitions (not in provided files, but implied)
│   │   └── orchestrator_service.proto # (Assumed based on generated code)
│   ├── generated/               # Python code generated by protoc
│   │   ├── __init__.py
│   │   └── orchestrator_service_pb2.py
│   │   └── orchestrator_service_pb2_grpc.py
│   ├── services.py              # Python implementation of the gRPC services
│   └── server.py                # Logic to start the gRPC server
│
├── config/
│   ├── settings.yaml            # Main configuration
│   ├── personas.yaml
│   ├── vdb_schema.yaml          # For orchestrator's VDB collections
│   └── tools.d/                 # Modular tool definitions (*.yaml) for EXTERNAL tools
│       ├── retriever_cfr.yaml   # RAG Confluence Retriever (CFR)
│       └── ...
│
├── orchestration/
│   ├── config_loader.py
│   ├── llm_interface.py
│   ├── tool_executor.py         # Makes API calls to external tools
│   ├── history_manager.py
│   ├── history_summarizer.py
│   ├── context_builder.py
│   ├── token_utils.py
│   └── io_handlers/
│       ├── vdb_handler.py
│       └── sql_handler.py
│
├── database/
│   └── models.py                 # SQLAlchemy ORM models (ConversationSession, HistoryTurn, HistoryTurnSummary)
│
├── docs/
│   ├── sera_design.md            # This file
│   └── llm-modules_guide.md      # Documentation for LLM How to make module to interact with Sera
│
├── prompts/
│   ├── system_prompts/
│   │   ├── general_assistant.py
│   │   ├── pro_assistant.py
│   │   ├── flash_assistant.py
│   │   ├── task_planner.py
│   │   └── critique_analyst.py
│   └── working_memory/
│   │   └── default_wm_generator.py # Prompt for working memory
│   └── modules/
│       └── rag_confluence_retriever.py # Prompt CFR module
│
└── utils/
    ├── logging_setup.py
    └── chat_logger.py



---

## 3. Orchestrator Responsibilities

* **Initialization:** `Orchestrator` class initializes its sub-modules, loads configuration via `ConfigLoader`, and sets up clients.
* **User Input:** Via `process_query` from Streamlit UI or `ProcessInstruction` gRPC call.
* **History Management (`HistoryManager`):** Uses `LLMInterface` for embeddings and `VDBHandler` for VDB storage (raw turns, summary embeddings), and `SQLHandler` for PostgreSQL storage (structured turns, text summaries).
* **History Summarization (`HistorySummarizer`):** Asynchronously generates and stores summaries of conversation turns.
* **Context Building (`ContextBuilder`):** Assembles LLM messages including system prompts, history (raw, semantic, working memory), current query, tool definitions, and fetched tool guides. Manages token limits and working memory generation/updates. Includes confusion notes in context.
* **LLM Interaction (`LLMInterface`):** Manages calls to Ollama/NIM for chat and embeddings.
* **Tool Definition Loading (`ConfigLoader`):** Loads definitions of external tools from `config/tools.d/`.
* **Persona Management:** `ContextBuilder` loads persona system prompts; `Orchestrator` uses persona configuration for model/parameter selection.
* **Function Calling & Execution (`Orchestrator.process_query` & `ToolExecutor`):** Iterative loop in `process_query` up to `MAX_TOOL_ITERATIONS`. `ToolExecutor` makes API calls to external tool services (expects gRPC for internal tools).
* **State Management:**
    * Conversational history in VDB and PostgreSQL (via `HistoryManager`).
    * In-memory session state (turn index, working memory summary, confusion note) in `Orchestrator` instance.
* **Response Synthesis:** Primarily by the Orchestrator's LLM.
* **Serving gRPC API Requests (`grpc_api/server.py`, `grpc_api/services.py`):** Exposes gRPC services for authorized external tools.
* **Correction/Confusion Detection (`Orchestrator`):** Uses keyword matching to detect potential user corrections or LLM self-corrections, influencing context for subsequent turns.

## 3.1. Future Design: State Management for Delegated & Long-Running Tasks

While the current state management effectively handles conversational history, a more advanced mechanism is required to support complex, long-running tasks delegated to specialized **(A) Assistant** modules. The current error-handling model is synchronous, but to achieve true resilience and enable asynchronous operations, a persistent task state manager is planned.

**Architectural Goal:**
When the Orchestrator delegates a task (e.g., "Provision a VM"), it will create a persistent record for that task in the PostgreSQL database. This record will track the entire lifecycle of the sub-task managed by the Assistant.

**Proposed Database Model (`TaskState`):**
A new SQLAlchemy model will be introduced to store:
* `task_id`: A unique identifier for the delegated task.
* `session_id`: The parent conversation session this task belongs to.
* `delegated_to_tool_id`: The name of the Assistant handling the task (e.g., `remote_automation_assistant`).
* `status`: The current state of the task (e.g., `PENDING`, `RUNNING`, `SUCCESS`, `FAILED`, `AWAITING_USER_INPUT`).
* `state_payload`: A JSONB field containing the task's internal state, such as the current step, intermediate results, or error information.
* `created_at` / `updated_at`: Timestamps for tracking and potential timeouts.

This pattern makes the ecosystem resilient to service restarts and lays the groundwork for advanced features like pausing/resuming complex workflows and providing users with a status dashboard for their background tasks.

---

## 4. Orchestrator Inbound gRPC API

To enable tightly coupled and efficient communication from internal tool services back to the Orchestrator, it exposes a gRPC API. This allows trusted tools to request advanced processing and leverage the Orchestrator's core LLM reasoning.

### 4.1. Purpose

- Allow internal tool services to make specific, structured requests to the Orchestrator.
- Enable tools to offload complex natural language understanding, response generation, planning, or multi-tool coordination tasks.
- Provide a formal, high-performance interface for programmatic interaction.

### 4.2. Technology Stack

- **Framework:** gRPC with Protocol Buffers.
- **Language:** Python for the server-side implementation.

### 4.3. Service Identification & Authorization

This gRPC API is designed for internal communication within a trusted, isolated network.

- **Method:** A calling tool is authorized if the `calling_tool_id` in its gRPC request matches the name of a tool defined in a `*.yaml` file within the `config/tools.d/` directory.
- **Verification Process:**
  1. At startup, `ConfigLoader` caches all tool names from `config/tools.d/`.
  2. On receiving a gRPC request, the Orchestrator checks if the request's `calling_tool_id` is in the cached list.
  3. If a match is found, the call is authorized. Otherwise, it is rejected with a `PERMISSION_DENIED` status.
- **Rationale:** This leverages the existing tool configuration directory as a central registry, simplifying management for internal services.

### 4.4. Core gRPC API Components Structure (within `/grpc_api/`)

- **`protos/orchestrator_service.proto`:** Protocol Buffer file defining services, RPCs, and messages.
- **`generated/`:** Python code (`*_pb2.py`, `*_pb2_grpc.py`) generated by `protoc`.
- **`services.py`:** Python classes implementing the server-side logic for the RPCs.
- **`server.py`:** Script to configure and start the gRPC server.

### 4.5. Example Service Definition (orchestrator_service.proto)

This definition focuses on providing a way for internal modules to submit a "prompt" or "instruction" along with relevant context, similar to how a user interacts with the Orchestrator via a chat interface.

```proto
syntax = "proto3";

package orchestrator.v1;

import "google/protobuf/struct.proto"; // For flexible, JSON-like context data

// Allows internal modules to leverage the Orchestrator's core processing.
service OrchestratorInteractionService {
  // Enables a module to send a request, similar to a user query.
  // The response is streamed for progressive updates.
  rpc ProcessInstruction(InstructionRequest) returns (stream InstructionResponse) {}
}

message InstructionRequest {
  // Identifier of the calling module (e.g., "email_service_alpha").
  string calling_tool_id = 1;

  // The primary instruction or query for the Orchestrator.
  string user_prompt_equivalent = 2;

  // Structured data providing rich context for the instruction.
  google.protobuf.Struct rich_context_data = 3;

  // Optional: Link to an existing Orchestrator session to use its history.
  optional string orchestrator_session_id = 4;

  // Optional: Hint for which Orchestrator persona to use.
  optional string target_orchestrator_persona_key = 5;
}

// Streamed response from the Orchestrator.
message InstructionResponse {
  enum MessageType {
    MESSAGE_TYPE_UNSPECIFIED = 0;
    PROGRESS_UPDATE = 1;      // e.g., "Calling tool: JIRA_Retriever"
    TEXT_CHUNK = 2;           // A piece of a generated textual response
    TOOL_CALL_INFO = 3;       // Info about a tool the Orchestrator called
    TOOL_RESULT_INFO = 4;     // Info about a result from a tool
    FINAL_OUTPUT = 5;         // Contains the final structured result
    ERROR_INFO = 6;           // An error occurred.
  }

  MessageType message_type = 1;
  string text_content = 2;
  google.protobuf.Struct structured_payload = 3;
}
```

### 4.6. Internal Processing Flow for Inbound gRPC Requests

1.  **Request Reception:** The gRPC server receives a call to `ProcessInstruction`.
2.  **Service Identification:** The `calling_tool_id` is validated against the list of known tools.
3.  **Adaptation:** The `InstructionRequest` is adapted into inputs for the core `Orchestrator.process_query` method (or a similar internal variant). A session ID is either used from the request or created for a new "headless" interaction.
4.  **Core Execution:** The Orchestrator retrieves history, builds context (including `rich_context_data`), interacts with the LLM, and executes any necessary tool calls via the `ToolExecutor`.
5.  **Streaming Response:** The gRPC service method yields `InstructionResponse` messages back to the caller as processing occurs, providing progressive updates.
6.  **Persistence:** The interaction is saved to the history database under the relevant session ID.

### 4.7. Future API Considerations

While the gRPC API is ideal for high-performance, internal, type-safe communication between trusted services, future requirements may necessitate a public-facing API for broader integration. A RESTful (HTTP/JSON) API could be exposed to allow external applications, third-party services, or user-facing web clients (not built in Streamlit) to interact with the Orchestrator in a more standardized, firewall-friendly manner. This would be a separate interface from the internal gRPC services and would require its own robust authentication and authorization mechanism (e.g., OAuth2, API Keys).

## 5. Data Stores Strategy

### 5.1. Relational Database (PostgreSQL)

- **Schema:** Defined in `/database/models.py` (`ConversationSession`, `HistoryTurn`, `HistoryTurnSummary`). Managed by `SQLAlchemy's Base.metadata.create_all(engine)`. Note: Alembic for migrations is not currently part of the provided scripts.
- **Interaction:** `orchestration/io_handlers/sql_handler.py` (`SQLHandler`) encapsulates SQLAlchemy session management and CRUD operations.

### 5.2. Vector Database (VDB - ChromaDB)

- **Interaction:** `HistoryManager` uses `VDBHandler` for two primary collections configured in `settings.yaml`:
  - **`history_store`:** Stores embeddings of the full, raw content of each conversation turn for semantic search over recent dialogue.
  - **`summary_store`:** Stores embeddings of the concise, LLM-generated summaries of each turn for more abstract semantic searches over the conversation's key points.

### 5.3. Shared Asset Storage

- **Local:** A local filesystem directory (e.g., `/ai/assets/`).
- **Interaction:** `ContextBuilder` uses a subdirectory (`.../cache/orchestrator/tool_guides/`) for caching fetched tool guides.

## 6. Data Ingestion Pipelines (for External Tools)

- Data Ingestion Pipelines (e.g., the Confluence Ingestor - CFI) are independent Python applications.
- Their primary role is to populate data stores (such as VDB collections or PostgreSQL tables) with data required by external tool services (e.g., `rag_confluence_retriever`).
- The Orchestrator does not directly interact with or depend on these ingestion pipelines.

## 7. Tool Definition (`config/tools.d/`)

Tool definitions in `config/tools.d/*.yaml` files describe external gRPC services and how the Orchestrator should call their APIs.

- **Format:** A list of tool objects, each with:
  - `name`, `description`, `parameters`: An OpenAI-compatible function schema.
  - `llm_instruction_guide_path` (Optional): Path (local filesystem or URL) to a Markdown guide.
  - `execution` block:
    - `type`: Must be `"api"`.
    - `base_api_url`: The gRPC URL for the tool (e.g., `grpc://localhost:50077`).
    - `grpc_service_config` (Required):
      - `stub_directory_path_from_assets`: Path relative to `/ai/assets/protos/` where the tool's compiled Python gRPC stubs are located.
      - `stub_module_name`: The Python filename prefix of the generated stubs (e.g., `cfr_service`).
      - `service_name_in_proto`: The service name from the `.proto` file (e.g., `RAGConfluenceRetrieverService`).
      - `rpc_method_name`: The specific RPC method to call (e.g., `GetFormattedContext`).
      - `request_message_name`: The name of the request message protobuf (e.g., `GetFormattedContextRequest`).
  - `version` (Optional): The version of the external tool.

**Example (`config/tools.d/rag_confluence_retriever_cfr.yaml`):**
```yaml
- name: "rag_confluence_retriever"
  description: "Searches and retrieves pre-formatted context from Confluence."
  parameters:
    type: "object"
    properties:
      query:
        type: "string"
        description: "The search query for retrieving relevant documents."
    required: ["query"]
  llm_instruction_guide_path: "/ai/assets/docs/llm_guides/cfr_llm_guide.md"
  execution:
    type: "api"
    base_api_url: "grpc://localhost:50077"
    grpc_service_config:
      stub_directory_path_from_assets: "rag_confluence_retriever/generated"
      stub_module_name: "cfr_service"
      service_name_in_proto: "RAGConfluenceRetrieverService"
      rpc_method_name: "GetFormattedContext"
      request_message_name: "GetFormattedContextRequest"
  version: "1.0"
```

## 8. Tool Implementation and Error Handling

- **Input (to External Tools):** The `ToolExecutor` serializes parameters into the Protobuf message format required by the external tool's gRPC service.
- **Output (from External Tools):** External services must return a gRPC response. The `ToolExecutor` expects a JSON-serializable dictionary or string within this response.
- **Error Handling:**
  - Network or gRPC call errors are caught by the `ToolExecutor`.
  - The error message is passed back through the Orchestrator and displayed in the UI.
  - The error details (tool name, parameters, error message) are added to the conversation context.
  - The LLM is informed of the tool failure, allowing it to re-plan, use an alternative tool, or synthesize a user-friendly message explaining the issue.

## 9. Interaction Flow (Iterative)

1.  **Input & History:** A query from the UI is sent to the Orchestrator. `HistoryManager` retrieves relevant history from PostgreSQL and ChromaDB.
2.  **Context Assembly:** `ContextBuilder` creates the full prompt for the Orchestrator's LLM, including system prompts, history, tool definitions, and fetched tool guides.
3.  **LLM Planning:** `LLMInterface` sends the context to the LLM, which responds with either a direct answer or a `tool_calls` request.
4.  **Decision:** If the LLM provides an answer, the flow proceeds to step 7. If it requests a tool call, the flow continues to step 5.
5.  **Tool Execution Loop:**
    a. The Orchestrator parses the `tool_calls` request.
    b. For each call, `ToolExecutor` makes a gRPC request to the configured external tool service.
    c. The Orchestrator collects the gRPC responses or errors from the tool.
    d. The tool call details and its result (or error) are appended to the context.
    e. **LLM Synthesis:** The updated context is sent back to the LLM to synthesize a final answer or plan the next tool call.
    f. The loop continues until a final answer is generated or the `MAX_TOOL_ITERATIONS` limit is reached.
6.  **Final Response:** The Orchestrator receives the final synthesized output from the LLM.
7.  **History Saving:** `HistoryManager` saves the complete turn (query, tool calls, results, final answer) to the databases.
8.  **Output to UI:** The final response is streamed back to the Streamlit UI.

## 10. Logging Strategy

The orchestrator uses a multi-file logging system configured by `utils/logging_setup.py` for diagnostics and auditing.

- **Setup:** Logging is configured at startup for both console and file output. The console level is set via `config/settings.yaml`.
- **Log Files:** When file logging is enabled, three rotating log files are created:
  - `app.log`: A comprehensive log for detailed debugging.
  - `app_err.log`: A high-level log for `WARNING` and `ERROR` messages only.
  - `chat.log`: An isolated log containing only the chat transaction history for easy review and auditing.
- **Configuration:** The system is configured to reduce noise from third-party libraries, keeping the main logs focused on application-specific events. The standard log format includes timestamps and module details for clear traceability.

## 11. Future Kubernetes Deployment Considerations

- Containerize the Orchestrator, UI, and all external tool services.
- Deploy PostgreSQL and ChromaDB as stateful services.
- Use a ReadWriteMany (RWX) PersistentVolumeClaim for Shared Asset Storage.
- Manage configurations and secrets via K8s ConfigMaps and Secrets.
- Use K8s Services for internal networking and discovery.

## 12. Local Development Environment Setup

- **Orchestrator Code:** Clone the repository and set up a Python venv using `bootstrap_helper.py`.
- **Databases (Docker):** Run PostgreSQL and ChromaDB using their respective Docker configurations.
- **Shared Asset Storage:** Ensure the local filesystem path (e.g., `/ai/assets/`) exists and is correctly configured.
- **LLM Configuration:** Configure LLM endpoints (Ollama, NVIDIA NIM) in `config/settings.yaml`.
- **External Tool Services (CRITICAL):**
  - Each external tool (CFR, RAA, etc.) must be run as a separate application from its own source directory.
  - Each tool must expose a gRPC API.
  - The `base_api_url` for each service must be correctly defined in its corresponding `config/tools.d/*.yaml` file.
- **Orchestrator gRPC Server:** The Orchestrator's inbound gRPC server must be running (e.g., via `python sera_orchestrator.py --run-grpc-server`) to accept calls from other tools.

## 13. High-Level Plan & TODOs

- **Phase 0: Foundations**
  - [x] Local PostgreSQL & ChromaDB Docker setups usable.
  - [x] Shared Asset Storage path defined.
  - [x] Project Scaffolding created.
  - [x] `bootstrap_helper.py` and `requirements.txt` are functional.
  - [x] All configuration is managed through YAML files.
  - [x] Logging is set up via `utils/logging_setup.py`.
- **Phase 1: Core Orchestrator Logic**
  - [x] All `orchestration/` sub-modules are implemented (`VDBHandler`, `SQLHandler`, `ConfigLoader`, `LLMInterface`, `HistoryManager`, `HistorySummarizer`, `ContextBuilder`).
  - [x] `database/models.py`: All required SQLAlchemy ORM models are defined.
  - [x] `orchestrator.py`: Core `process_query` loop and state management are implemented.
  - [x] `orchestration/tool_executor.py`: Implemented with dynamic gRPC stub loading.
  - [x] `app_streamlit_core.py`: A functional chat interface is implemented.
- **Phase 1.B: Orchestrator Inbound gRPC API Implementation**
  - [x] Design & Protos: Proto definitions are created and located in `grpc_api/protos/`.
  - [x] Code Generation: Generated stub files are present in `grpc_api/generated/`.
  - [x] Server-Side Implementation: `grpc_api/services.py` and `grpc_api/server.py` implement the server logic and service authorization.
  - [ ] Client-Side Testing: Requires an external tool to be updated to act as a client to this API for end-to-end testing.
  - [ ] Documentation: Finalize API documentation for developers of other tool services.
- **Phase 2: Refinement & Expansion (Ongoing)**
  - **High Priority:**
    - [x] **Fix SERA Chat Continuation (RAG):** Resolved bug with Gemma tool-use JSON parsing and improved conversational follow-up logic.
    - [ ] **Develop a "Plan & Critique" Workflow for Tool Use**
      - **Goal:** To significantly increase the reliability and accuracy of automated tasks by introducing an automated AI-powered review step before execution.
      - **Description:** To prevent flawed or inefficient tool usage, this workflow uses a second AI as a "sanity check." The primary model generates a plan, and a secondary model critiques it. This leverages your parallel processing capability to build a more robust and self-correcting system.
      - **Implementation Plan:**
        - In `Orchestrator.process_query`, after the primary model (`gemma3:27b`) returns a `tool_calls` object, intercept the execution.
        - Spawn a parallel, non-blocking request to a second model (e.g., `gemma3:12b`) using the `critique_analyst_persona`. This request will contain the original user query and the `tool_calls` JSON generated by the first model.
        - The Orchestrator will receive the critique. Implement logic to handle the outcome:
          - If the critique is positive ("Plan is sound"), proceed with execution via `ToolExecutor`.
          - If the critique is negative ("Step 2 is redundant..."), loop back to the primary model with the original context plus the critique, and ask it to generate a revised plan.
      - **Dependencies/Notes:** This creates a more complex, multi-step reasoning loop within the Orchestrator.
    - [ ] **Implement Ensemble RAG for Complex Queries**
      - **Goal:** To maximize the relevance and coverage of RAG results by generating and executing multiple query variations in parallel, solving the problem of query phrasing uncertainty.
      - **Description:** Users often struggle to phrase the perfect search query. This approach eliminates the guesswork by using multiple AI models to generate several different phrasings of the user's question, alongside the user's original text. All queries are run against the RAG tool simultaneously, and the best results are synthesized for the final answer.
      - **Implementation Plan:**
        - In the Orchestrator, when a `rag_confluence_retriever` call is decided upon, trigger a "query expansion" step instead of calling the tool directly.
        - Make parallel calls to 2-3 different models (e.g., `gemma3:27b`, `gemma3:12b`) with a prompt like: "Rephrase the following user query to be optimal for a vector database search: '[user query]'".
        - Create a final list of queries containing the AI-generated variations plus the user's original, verbatim query.
        - Make parallel gRPC calls to the `rag_confluence_retriever` (CFR) tool, one for each query in the list.
        - The combined, de-duplicated results from the CFR are then passed to the main `gemma3:27b` model for the final synthesis step.
      - **Dependencies/Notes:**
        - The RAG Confluence Retriever (CFR) tool must be updated to efficiently handle multiple concurrent requests.
        - The CFR must be responsible for de-duplicating the search results from the various queries before returning a single, consolidated context to the Orchestrator.
    - [ ] **Enhance History Manager:** Implement specific handling of links from RAG tool results to enable direct follow-up queries on returned sources.
    - [ ] **Implement User Feedback Queue:** Create a UI mechanism for users to flag unhelpful or incorrect answers, which are then queued for review. This is essential for continuous model and prompt improvement.
    - [ ] **Implement Structured History Storage:** Allow the Orchestrator to store specific items from conversation history (e.g., links, generated documents, code snippets) in structured PostgreSQL tables for more reliable and precise context retrieval in long conversations.
  - **Medium Priority:**
    - [ ] **Implement a Pre-Processing "Task Router"**
      - **Goal:** To route incoming queries to the most appropriately sized LLM, optimizing resource utilization and speeding up response times for simple requests.
      - **Description:** Not all user queries require the full reasoning power of the largest model. This task involves creating a lightweight, initial step where a very fast, small model classifies the user's query by complexity and intent. Based on this classification, the Orchestrator will dynamically select the right-sized model (4b, 12b, or 27b) for the main processing and response generation.
      - **Implementation Plan:**
        - Define a set of task classifications (e.g., `simple_qa`, `complex_reasoning`, `chitchat`, `summarization`).
        - In `Orchestrator.process_query`, before building the main context, add a call to a small model endpoint (e.g., `gemma3:1b` or `gemma3:4b`) with a prompt designed to classify the user's query into one of the predefined categories.
        - Implement logic in the Orchestrator to read the classification and set the `target_model_id` for the rest of the turn. For example, `chitchat` could be handled by `gemma3:4b`, while `complex_reasoning` would be routed to `gemma3:27b`.
      - **Dependencies/Notes:** This is largely self-contained within the Orchestrator's logic.
    - [ ] **Offload Ancillary Tasks to Smaller Models**
      - **Goal:** To free up the primary reasoning models by delegating routine, non-reasoning tasks (like summarization or title generation) to smaller, dedicated models.
      - **Description:** Certain internal tasks performed by the orchestrator, such as generating a conversation summary in the `HistorySummarizer` or creating a new chat title, are computationally simple. These tasks should not consume the resources of a 27b model.
      - **Implementation Plan:**
        - Identify all internal, system-initiated LLM calls within the codebase.
        - Modify the `HistorySummarizer` to use a hardcoded, smaller model endpoint (e.g., `gemma3:4b`) for its summarization calls, rather than inheriting the model from the main chat.
        - Refactor any other ancillary functions (like chat title generation) to use a specific, small model endpoint for their needs.
      - **Dependencies/Notes:** May require passing the `LLMInterface` to modules that don't currently have it.
    - [ ] **Create a "Consensus Answer" Mode for High-Stakes Q&A**
      - **Goal:** To improve the factual accuracy and reliability of critical answers by generating responses from multiple models and synthesizing a final answer based on their consensus.
      - **Description:** For questions where a correct answer is critical, this workflow leverages multiple powerful models as a panel of experts. By comparing their outputs and forming a consensus, the system can produce a final answer with a much higher confidence level than any single model could achieve alone.
      - **Implementation Plan:**
        - Create a new mechanism to trigger this mode, likely via a dedicated persona (`high_accuracy_persona`).
        - When this mode is active, the Orchestrator will send the same final prompt to 2-3 of the largest available models in parallel (e.g., two different `gemma3:27b` instances).
        - The Orchestrator will collect all generated text-based answers.
        - A final "meta-synthesis" call will be made to a single `gemma3:27b` model. The prompt will include the original question and all the generated answers, instructing the AI to: "Review the following answers from different AI assistants. Synthesize a single, final, high-confidence answer based on the points of consensus. Explicitly mention any significant disagreements between them."
      - **Dependencies/Notes:** This is computationally intensive but provides a powerful mechanism for quality assurance on critical queries.
    - [ ] **Implement Persistent Task State Management:** As detailed in Section 3.1, develop the `TaskState` database model and the Orchestrator logic to create, update, and query the state of long-running, delegated tasks.
    - [ ] **Implement Reranking for History Retrieval:** Integrate a reranking model (e.g., `nvidia/llama-3.2-nv-rerankqa-1b-v2`) to work in conjunction with the embedding model to significantly improve the relevance of semantically retrieved conversation history.
    - [ ] **Implement Onboarding & "Cookbook" Assistant:** Create a specialized persona or mode that can answer "What can you do?" and provide users with a "cookbook" of example prompts and common workflows to accelerate user adoption.
    - [ ] **UI Enhancement - Rich Data Visualization:** Instead of only returning markdown tables, enhance the UI to render interactive charts, graphs, and diagrams based on structured data returned from tools.
    - [ ] **Implement "Self-Service" Prompt Editor:** Create a user-friendly UI for subject matter experts to create, test, and manage prompts for specialized assistant personas without developer intervention.
    - [ ] **Refactor SERA UI:** Plan and execute the migration from the Streamlit UI to a fully custom frontend to support more advanced features like file uploads, interactive canvases, and richer data visualization.
  - **Low Priority:**
    - [ ] **Design & Implement Public REST API:** Plan and develop a secure, public-facing RESTful API to allow external, non-internal services to interact with the Orchestrator, complementing the internal gRPC interface.
    - [ ] **Centralize & Scope Tool Definitions:** Move the `tools.d/` directory to a shared registry (e.g., `/assets/registry/tools.d/`). Implement a `metadata.scope` tag in each tool's YAML definition to enable granular, role-based loading for different modules.
  - **General & Ongoing:**
    - [ ] Continue to refine error handling, token management (`ContextBuilder`), LLM interaction logic (`LLMInterface`), and structured logging for enhanced observability.
    - [ ] CRITICAL Fix BUG when switching personas doesn't create new clean chat, this is leading to incorrect answers/confusion.
    - [ ] ADD speach to text.


## Family naming convention (draft)

The "IRA" Naming Convention: A Cohesive Family
Tier 1 (SysEng / Development)
Name: SIRA

Acronym: SysEng Intelligent Reasoning Assistant

Rationale: A direct and powerful homage to your Gen1 project. It shows a clear line of evolution and honors the system's roots. It's a fantastic internal name with history.

Tier 2 (Internal GTT / Company-Wide)
Name: GIRA

Acronym: GTT Intelligent Reasoning Assistant

Rationale: Pronounced "Gee-ra." It's unique, short, easy to remember, and clearly designates the company-wide (GTT) scope. It feels both technical and approachable.

Tier 3 (Customer-Facing / Ethervision)
Name: EIRA

Acronym: Ethervision Intelligent Reasoning Assistant

Rationale: Pronounced "Era" or "Eye-ra." This is an elegant, real name (like the Welsh name Eira, meaning "snow") that sounds sophisticated, modern, and trustworthy for a customer-facing product.

Option 2: The "Synthesis" Convention
Universal Descriptor: ISA = Intelligent Synthesis Assistant

SISA (SysEng Intelligent Synthesis Assistant) - The creator's edition, with a personal touch.

GISA (GTT Intelligent Synthesis Assistant) - The internal enterprise powerhouse.

EISA (Ethervision Intelligent Synthesis Assistant) - The elegant, customer-facing expert.

The name SISA can be re-acronymed to Sisel Intelligent Synthesis Assistant. (Sisel is my nickname = creator name also in czech Sisa is short version is sisel)

