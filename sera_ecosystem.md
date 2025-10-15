# /ai/assets/docs/sera_guides/sera_ecosystem.md
# Ecosystem Architecture & Development Guide

# 1 Introduction

This document provides the central architectural design for the **SysEng Reasoning Assistant (SERA)** ecosystem, its core components, and its underlying infrastructure. It is the primary reference for developers and architects building and integrating modules within the platform.

---

# 2 Vision & Core Principles 🎯

The SERA ecosystem is a modular, microservices-based AI platform designed to automate and assist with specialized enterprise tasks. Its architecture is built on four core principles.

## 2.1 Evolution from Sira (Gen 1) to SERA (Gen 2)

**SERA** represents the second generation of our enterprise AI platform, evolving from its predecessor, Sira (SysEng Information Retrieval Agent).

Sira (Gen 1) was architected as a highly effective but monolithic RAG application. Its core consisted of a Streamlit UI orchestrating a sophisticated, multi-stage pipeline (`workflow.py`) dedicated to a single task: providing conversational answers from Confluence documentation. While powerful for information retrieval, its monolithic design presented limitations for broader enterprise automation. Adding new, unrelated capabilities (e.g., interacting with ServiceNow or provisioning infrastructure) was architecturally complex.

**SERA** was designed to overcome these limitations. It takes the specialized RAG capability that powered Sira and refactors it into an independent, plug-and-play Tool (the RAG Confluence Retriever). The key innovation of SERA is the shift from a single application to a full ecosystem centered around an **Orchestrator**. This design allows us to combine the power of a Sira-like retriever with a diverse palette of other specialized Tools, enabling complex, multi-system automation that was beyond the scope of the first generation.

## 2.2 Core Architectural Principles

The SERA ecosystem is a modular, microservices-based AI platform designed to automate and assist with specialized enterprise tasks. Its architecture is built on five core principles.

* **Modularity**: Individual components (**Tools**) can be developed, deployed, and scaled independently. This allows for parallel development and the easy addition of new capabilities.
* **Specialization**: Each Module is an expert at a specific task, such as data ingestion (Confluence Ingestor), intelligent retrieval (RAG Confluence Retriever), or infrastructure automation (Remote Automation Framework).
* **Orchestration**: A central "brain" (**Sera Orchestrator**) intelligently decomposes user requests, calls upon the necessary specialized Tools, and synthesizes a final, coherent response.
* **Knowledge-Centric Design**: The ecosystem's effectiveness is founded on a rich knowledge base, not just raw data. The architecture mandates that specialized **Ingestor** modules are responsible for transforming raw information into a deeply understood, queryable asset.
* **Measurability & Observability**: The system is designed to be transparent, ensuring all actions are auditable and that architectural hooks are in place to support performance monitoring and quality evaluation as the platform scales.

---

# 3 High-Level Architecture

The ecosystem is designed around four primary architectural layers, each with a distinct responsibility. This separation of concerns is fundamental to the system's modularity and scalability.

* **User Entry Point**: The secure front door for all interactions. It is responsible for authenticating users and routing traffic to the correct internal services. This layer is implemented by the **Sera AI Gateway**.
* **Orchestration Layer**: The central "brain" of the system. It understands user intent, decomposes complex requests into logical plans, and manages the end-to-end workflow. This layer is embodied by the **Sera Orchestrator**.
* **Tool/Module Layer**: A collection of independent, specialized microservices that perform discrete actions or retrieve specific information when called upon by the Orchestrator. These are the **Tools** (e.g., RAG Confluence Retriever, ServiceNow Connector).
* **Shared Infrastructure Layer**: The common foundation providing stateful services like databases, vector stores, and file storage that are accessible to all other layers.

A typical user request flows through these layers in a sequence orchestrated by high-performance gRPC APIs:
1.  A user interacts with an application via the secure **Sera AI Gateway**.
2.  The Gateway forwards the request to the **Sera Orchestrator**.
3.  The Orchestrator interprets the query, retrieves conversation history, and plans a series of actions.
4.  It executes the plan by making gRPC calls to one or more specialized Tools in the Tool Layer.
5.  The Tools perform their tasks, interacting with the Shared Infrastructure as needed.
6.  The Tools return their results to the Orchestrator.
7.  The Orchestrator uses a reasoning LLM to synthesize the results into a final, coherent answer, which is sent back to the user.

---

# 4 Core Capabilities & Workflows

The SERA ecosystem is engineered to support a spectrum of tasks, from simple information retrieval to complex, multi-step automation.

---

## 4.1 Intelligent Information Retrieval (RAG)

This is the foundational capability. When a user asks a question, the Orchestrator leverages a **Retriever** module. The architecture enables an advanced, multi-stage retrieval process far beyond simple vector search:

* **Hybrid Search**: Combines semantic search on content chunks with techniques like **HyDE** (searching for similar, pre-generated questions).
* **Re-ranking**: An NVIDIA NIM reranking model re-scores the initial search results against the original query to significantly improve relevance.
* **Intelligent Filtering**: Uses AI-generated metadata to filter candidates based on the user's inferred intent (e.g., prioritizing "How-To Guides" if the user asks "How do I...").
* **Context Expansion**: For the most relevant results, the system automatically fetches adjacent "sibling" chunks from the same document section to provide broader context.

---

## 4.2 Multi-Step, Cross-System Automation

This represents the system's primary goal: executing complex business processes that span multiple systems. The architecture supports two primary execution patterns depending on the task's complexity: a direct, Orchestrator-led flow for simple tasks, and a more robust, Assistant-led delegated flow for complex automation.

### Pattern 1: Orchestrator-Led Workflow (Simple Tasks)

For straightforward requests that can be resolved by a single tool, the **Orchestrator** handles the entire process directly.

**Example: A Simple Data Lookup**

A user asks, "What is the IP address of vm-db-01?"

1.  **Orchestrator - Analysis**: The Orchestrator's LLM analyzes the query and determines it can be fulfilled by the `netbox-assistant` tool.
2.  **Orchestrator - Tool Call**: The Orchestrator makes a direct gRPC call to the **NetBox Assistants**.
3.  **Orchestrator - Synthesis**: The Connector returns the IP address, and the Orchestrator synthesizes the final answer for the user.

### Pattern 2: Assistant-Led Delegated Workflow (Complex Automation)

For complex, multi-system tasks, the Orchestrator delegates to a specialized Assistant, which plans the entire operation. The final, validated plan is then handed to the Remote Automation Framework (RAF) for secure, deterministic execution.

**Example: A Complete Provisioning Workflow**

A user requests, *"Provision a new virtual machine based on the specifications in ticket SC045."*

1.  **Orchestrator - Initial Analysis & Delegation:** The Sera Orchestrator identifies the request as a complex automation task and delegates the entire problem to the specialized Remote Automation Assistant (RAA).
2.  **RAA - Strategic Planning & Validation:** The RAA takes control and acts as the AI "strategic planner."
    * It first calls the (R) RAG Retriever to get the latest "Standard VM Provisioning Process" from Confluence.
    * It then calls the (C) ServiceNow Connector to fetch the details from ticket SC045.
    * The RAA's reasoning LLM validates the ticket data against the business rules from the retrieved documentation (e.g., "Does this production VM have a cost center?").
    * The RAA's final output is a single, structured Master Plan JSON object, which describes the entire multi-system workflow.
3.  **RAF - Secure Gatekeeping:** The RAA hands the final, validated Master Plan to the Remote Automation Framework (RAF). The RAF acts as the secure "general contractor."
    * The RAF performs its own static validation, checking the user's permissions (RBAC) and ensuring the plan's structure is sound.
4.  **RAF - Managed Workflow Execution:** The RAF receives the Master Plan from the RAA and acts as a secure, managed environment. Its primary role is to ensure the AI's plan is executed safely, transparently, and in alignment with audited standards.
    * **4a. Workflow Discovery & Audit:** The RAF's engine first checks its internal Workflow Registry (a database table) for an approved workflow that matches the `workflow_name` provided in the RAA's plan (e.g., `provision_standard_syseng_vm`).
        * **If the workflow exists and is "Approved":** The process continues to the next step.
        * **If the workflow does not exist:** The RAA's plan is treated as a request to create a new workflow. The plan, which contains the required fields and step sequence learned from RAG, is saved to the Workflow Registry with a status of "Pending Human Audit." A notification is sent to the SERA Ops team. This allows the system to learn new processes dynamically while ensuring a human must approve any new automation before it can be executed.
    * **4b. Static & Security Validation:** The RAF engine loads the approved workflow definition from its registry. This definition contains the "pre-audited required fields." The RAF performs its final gatekeeping checks:
        * **Permissions (RBAC):** Does the user have the right to execute this specific workflow?
        * **Schema:** Do the parameters provided by the RAA in the Master Plan perfectly match the audited schema of the stored workflow?
    * **4c. Orchestrated Execution:** Once all checks pass, the RAF's generic Workflow Engine (the static `workflow_engine.py` script) begins executing the steps as defined in the approved workflow data, making the gRPC calls to the NBA, CSA, and other assistants as required.
5.  **Assistants - Domain-Specific Execution:** Each Assistant performs its specialized task.
    * The NBA receives its goal from the RAF. It generates and executes its own detailed, internal plan by making a sequence of calls to the simple (C) NetBox Connector.
    * The CSA does the same for CloudStack.
6.  **Orchestrator - Final Synthesis:** The final outcome from the RAF is passed back up the main Orchestrator, which is responsible for formulating a clear and final answer for the user. This ensures the system is resilient and transparent about both success and failure.
    * **The "Happy Path" (Success):** If all steps execute successfully, the Orchestrator synthesizes a final confirmation message, completing the request (e.g., "The virtual machine vm-web-101 has been successfully provisioned with IP 10.20.30.40 based on ticket SC045.").
    * **The "Unhappy Path" (Failure):** If any step fails, the failure is not catastrophic. The specific error is caught and reported up the chain. The Orchestrator then synthesizes a helpful message explaining what went wrong and why the process was halted (e.g., "I could not complete the VM provisioning because no available IPs were found in the target subnet. Please contact the network team to expand the IP range.").

---

## 4.3 Direct Data Query & Analysis

The system can function as a natural language interface to structured data sources without requiring a RAG pipeline. This capability is enabled by specialized **Processor** modules (e.g., an Excel Analyst or Oracle Analyst) that use a safe, multi-step plan-and-execute model, rather than directly generating and running raw code like SQL. This approach ensures reliability, security, and predictability.

### Example: A user asks, "Who are the top 5 biggest spenders in the finance department this quarter?"

**Enabled Flow:**

1.  **Delegation to Processor**: The Sera Orchestrator identifies the request as a structured data query and delegates it to the appropriate specialized **Processor** module, such as the Oracle Analyst.
2.  **Plan Generation**: The Processor sends the user's query and the relevant database table schema to its reasoning LLM. The LLM's task is not to write SQL, but to return a structured JSON analysis plan. This plan is a sequence of high-level, pre-defined operations (e.g., `[{ "operation": "filter", "column": "department", "value": "finance" }, { "operation": "group_by_and_agg", ... }]`).
3.  **Safe Execution**: The Processor's internal code parses this AI-generated plan. It then iterates through the steps, calling its own pre-audited, secure functions to build and execute the actual query against the database. This critical step ensures the AI never has direct execution privileges and can only operate within the safe boundaries defined by the developer.
4.  **Synthesis**: The final, structured data result is returned to the Orchestrator, which then synthesizes it into a clear, natural-language answer for the user.
---

# 5 Core Architectural Patterns

These key design philosophies enable the complex workflows described in this document. They define how the system operates by distinguishing between two primary modes of execution: **Direct Orchestration** for straightforward tasks and **Delegated Autonomy** for complex, multi-step processes that require specialized reasoning.

## 5.1 Direct Orchestration (Iterative Execution)

This is the primary mode for most routine tasks. The **Sera Orchestrator** directly manages the workflow in an iterative loop. It breaks down a user request, calls a sequence of simple tools (like a (C) Connector or (R) Retriever), and synthesizes the results. In this mode, the Orchestrator maintains full, step-by-step control over the execution chain, feeding the output of one tool as the input to the next.

**Example**: A user asks, "What is the status of ticket `JIRA-123`?". The Orchestrator directly calls the Jira Connector tool with the parameter `ticket_id: "JIRA-123"` and formats the returned answer.

## 5.2 Delegated Autonomy (Specialized Task Chaining)

This mode is reserved for complex business processes that require their own internal logic, planning, and execution chaining. Instead of micromanaging the task, the Orchestrator's role is to recognize the request's complexity and delegate the entire sub-problem to a specialized (A) **Assistant** module, such as the Remote Automation Assistant (RAA).

This specialized assistant then becomes responsible for the entire end-to-end execution of that sub-problem:
* **Discovering Capabilities**: Querying a framework (like the RAF) to find the available atomic functions it can use.
* **Autonomous Planning**: Composing those functions into a logical, multi-step execution plan.
* **Executing the Plan**: Securely running the plan and managing its internal state.

This pattern is central to the ecosystem's power. It allows the main Orchestrator to remain a lean and efficient dispatcher while enabling highly complex, stateful automation to be encapsulated within dedicated, expert modules.

## 5.3 AI-Centric Discoverability

To support both modes of operation, tools must be discoverable and understandable by an AI agent. The ecosystem enforces this through a dual approach:

* **Formal API Definitions**: Tools expose their capabilities via strongly-typed gRPC schemas and discoverable registries (like RAF's Helper Function Registry). This provides the AI with a structured, machine-readable "spec sheet" of what a tool can do, its inputs, and its outputs.
* **LLM Instruction Guides**: Each tool can provide a supplemental Markdown file (`llm_instruction_guide.md`). The Orchestrator feeds this human-readable guide to the reasoning LLM as context. It acts as a "user manual" for the AI, explaining best practices, procedural nuances, and strategic advice on how to combine the tool's functions most effectively.

## 5.4 Bidirectional Communication (Inbound gRPC API)

Communication is not a one-way street from the Orchestrator to the tools. To handle complex, long-running tasks—especially under the Delegated Autonomy pattern—specialized tools must be able to communicate back. The architecture supports this with an inbound gRPC API on the Orchestrator. This allows an assistant to delegate a sub-task (e.g., "I need you to fetch this user's permissions from LDAP") or ask the end-user for clarification within the same session. Authorization for these inbound calls is secured by validating the `calling_tool_id` in the request against the Orchestrator's list of known, trusted tool definitions.

## 5.5 Persistent Task State Management

To make the Delegated Autonomy pattern truly robust, especially for long-running processes, the architecture mandates persistent state management. When an Orchestrator delegates a complex task to an Assistant, the state of that multi-step workflow (e.g., `current_step`, `status`, `payload`) must be saved to a durable store like the shared PostgreSQL database. This ensures that complex operations can survive service restarts and provides the foundation for future capabilities like pausing, resuming, and auditing complex automation tasks.

---

# 6 LLM Challenges & Architectural Mitigations 🧠

Building a reliable, enterprise-grade system on top of LLMs requires explicitly designing for their inherent limitations. This section outlines the primary challenges and the critical architectural principles that all modules within the SERA ecosystem must adhere to. While the Sera Orchestrator handles many of these challenges centrally, individual modules (Ingestors, Assistants, etc.) that leverage LLMs are also responsible for implementing these mitigations within their own logic.

## 6.1 Hallucinations & Factual Inaccuracy

**Challenge**: LLMs are probabilistic text generators, not databases. They can confidently invent facts, API calls, or process steps that are plausible but incorrect. For an enterprise system, acting on such a hallucination is a critical failure mode.

**Ecosystem-Wide Mitigation: Strict Grounding & Constrained Action**
This is the most critical architectural rule in the SERA ecosystem. Any module leveraging an LLM must strictly limit its role to reasoning and translation, never allowing it to be a source of truth or an unconstrained actor.

* **For Factual Questions ("What is X?")**: Any module answering a factual query is architecturally forbidden from using an LLM's internal knowledge. It must first call a definitive source of truth (a database via a Connector, a document via a Retriever) to get the data. The LLM's sole purpose is to format and present that retrieved, trusted data.
* **For Actions ("Do Y")**: An LLM is never allowed to generate or execute its own code, commands, or arbitrary API calls. For example, if asked to restart a web server, a non-grounded LLM might hallucinate a command for the wrong service (apache2 instead of nginx). To prevent this, a module developer must ensure the LLM's role is restricted to selecting from a pre-defined, human-audited list of tools and filling in the required parameters. This architectural pattern transforms the LLM from an unreliable actor into a reliable intent-to-API translator, where the underlying action is always executed by the tool's deterministic, safe code.

## 6.2 Context Window Limitations

**Challenge**: All LLMs have a finite context window, limiting the amount of information they can process at once. For complex, long-running tasks, this poses a significant risk of information loss and inconsistent outputs.

**Ecosystem-Wide Mitigation: Stateful Decomposition & Intelligent Retrieval**
No module in the ecosystem should attempt to handle tasks that exceed the context window in a single pass. The mandated architectural pattern is a deliberate, multi-step process:

* **Decomposition**: The Orchestrator or a specialized Assistant first decomposes the large task into a logical plan of smaller, manageable chunks (e.g., "Create Outline," "Write Section 1: API Endpoints," "Write Section 2: Data Schema," etc.).
* **Chunk-Based Processing**: The module should work on one chunk at a time, saving the result to a persistent store (like the PostgreSQL or Vector Database) as it completes.
* **Intelligent Context Retrieval**: When a later step needs information from a previous one, the module must not load all prior work into the prompt. Instead, it should perform a targeted semantic search or structured query over the saved chunks to retrieve and inject only the most relevant pieces of information.

For example, while the Orchestrator uses its `HistoryManager` to retrieve relevant parts of a long conversation, a specialized Documentation Creator assistant would need to implement its own similar logic to retrieve relevant, previously written sections of a document before writing a new one. This approach highlights a core design principle: complex requests do not yield instant answers. Instead, they trigger a robust, multi-step process that deliberately prioritizes coherence and accuracy over the illusion of speed.

## 6.3 Prompt Sensitivity & Non-Determinism

**Challenge**: An LLM's output is highly sensitive to its input prompt. Minor, seemingly insignificant changes can lead to vastly different results, which is a risk for predictable automation.

**Ecosystem-Wide Mitigation: Standardized Prompt Assembly & Low Temperature**
To ensure predictable and reliable behavior, any module calling an LLM for automation-related tasks must adhere to two principles:

* **Programmatic Prompt Assembly**: Prompts should be assembled programmatically from standardized templates and building blocks (e.g., system message, retrieved context, tool definitions, examples). This reduces the variability caused by ad-hoc prompt phrasing. The Orchestrator's `ContextBuilder` acts as a "prompt factory" and is the canonical implementation of this pattern.
* **Low Temperature**: API calls to LLMs for tasks requiring deterministic output (like choosing a tool or formatting a JSON object) must be made with a low "temperature" setting (e.g., $0.1$) to make the output as predictable as possible.

## 6.4 Open-Weight vs. Frontier Model Capabilities

**Challenge**: The ecosystem intentionally uses a mix of models, including locally-hosted open-weight models which may have different reasoning capabilities than cutting-edge proprietary models. Using the wrong model for a task can lead to suboptimal performance.

**Ecosystem-Wide Mitigation: Hybrid Model Strategy & Task Specialization**
The architecture promotes a "right tool for the job" approach to model selection. Module developers should not default to using the largest model for every task.
The most powerful reasoning models, like the locally-hosted `gemma3:27b` which is based on previous-generation Gemini technology, are a finite, high-demand resource. Their use should be reserved for the most complex reasoning and planning tasks, such as those performed by the Sera Orchestrator or an Automation Assistant.
Simpler, more focused tasks (e.g., classifying text, extracting structured data, summarizing a single document) within an individual module should be delegated to smaller, faster, or more specialized models (e.g., a fine-tuned `gemma:2b` for classification) to ensure efficiency and scalability across the ecosystem.

## 6.5 Computational Resource Management & Latency

**Challenge**: While the on-premise AI Factory provides significant computational power, these resources are not infinite. The core challenge is to ensure efficient resource utilization across the ecosystem to maximize system throughput and maintain low latency for all concurrent tasks. An architecture where every module defaults to using the largest available model for simple tasks creates unnecessary load, leading to processing queues and degraded performance across the entire platform.

**Ecosystem-Wide Mitigation: "Right-Sizing" Model Selection**
This principle mandates that every component in the ecosystem must be engineered to use computational resources judiciously.

* **Module-Level Responsibility**: Each module must be designed to select the most appropriate model for any given sub-task. For instance, while a module might use `gemma3:27b` as its default for complex reasoning, it must delegate simpler tasks like text classification or data extraction to smaller, faster, or more specialized models where appropriate.
* **Justification for High-End Models**: The AI Factory maintains a set of standard, approved models. The use of very large or specialized models not on this standard list (e.g., Llama3 70B or larger) requires a formal justification to the AI Factory owners. This ensures that powerful, resource-intensive models are reserved for tasks that demonstrably require their unique capabilities.

## 6.6 Internal Guardrails & Preventing Misuse

**Challenge**: In a fully internal environment, the primary security threat is not an external attacker but the potential for accidental or intentional misuse by an authenticated user. An authorized user might phrase a request ambiguously, which an overeager AI could misinterpret and apply to a production system. The challenge is to build robust guardrails that prevent the system's powerful capabilities from causing unintended consequences.

**Ecosystem-Wide Mitigation: Principle of Least Privilege & Module-Driven Confirmation**

* **Scoped Tool Access (RBAC)**: The Sera AI Gateway passes the user's authenticated Active Directory group information to the Orchestrator. The Orchestrator must use this information to enforce **Role-Based Access Control (RBAC)**, filtering the list of tools and modules available to the user for that specific session. This ensures users can only access the capabilities relevant to their role.
* **Module-Driven Confirmation Loop**: For any high-impact action, the responsibility for requiring confirmation lies with the specialized module performing the task (e.g., the Remote Automation Assistant). The module must formulate the confirmation request and send it to the Orchestrator. The Orchestrator's role is to act as a trusted intermediary, relaying this prompt to the end-user and securely passing the user's explicit consent back to the module before the final action is executed.
* **Framework-Level Parameter Validation**: The ultimate responsibility for validating the parameters of an action lies with the execution engine. For example, the Remote Automation Framework (RAF) must provide a final, deterministic safety check by validating any parameters it receives from an AI Assistant against its own internal allowlists, denylists, and business rules before executing a command.

## 6.7 API Interaction Fragility & The Syntactic Gap

**Challenge**: A significant challenge in AI-driven automation is bridging the gap between an LLM's semantic understanding and a target system's strict API requirements. A common but fragile approach is to use a simple "thin client" or direct API proxy (like a basic Model Content Protocol server). This pattern is unreliable for several reasons:

* **Schema Overload & Inefficiency**: This simple approach often compensates by "brute-forcing" context, dumping the entire API schema into the LLM's prompt for every call. This is infeasible for local models with limited context windows and is a massive waste of tokens and computational resources even for larger models.
* **The Semantic Gap**: The LLM must translate human-friendly concepts (e.g., names like "Prague") into machine-required formats (e.g., numeric IDs like `123`), often requiring complex, multi-step lookup plans for simple tasks.
* **Syntactic Brittleness**: The LLM is forced to generate a perfectly structured API payload. A single hallucinated field name or incorrect data type will cause the entire operation to fail.

**Ecosystem-Wide Mitigation: The "Smart Connector" Pattern**
The SERA architecture mitigates this by enforcing a strict separation of concerns between reasoning and execution, embodied by our **(A) Assistant** and **(C) Connector** pattern. The LLM's role is strictly limited to determining the user's *intent*. The actual API interaction is delegated to a "Smart Connector"—a deterministic, schema-aware service that acts as an intelligent translation layer.

This Smart Connector is responsible for:
* **Intelligent Schema Handling**: The Connector ingests and understands the target API schema itself. This means the LLM prompt remains small and task-focused, containing only the logical intent, not the entire API definition. This makes the process efficient and compatible with local models.
* **Bridging the Semantic Gap**: It automatically resolves human-friendly names to the required numeric IDs.
* **Payload Transformation**: It validates and transforms the logical data from the Assistant into a syntactically perfect payload that the target API will accept.
* **Encapsulating Application Logic**: It contains the hard-coded technical rules and interaction patterns of the target system's API. This includes knowing that creating a certain object requires field `X` instead of field `Y`, or that some data must be in a specific nested structure. This shields the LLM from needing to know the idiosyncratic, low-level details of a specific application's API.

This pattern transforms the LLM from a fragile code generator into a reliable intent-to-action planner, making the entire automation process more robust, efficient, and maintainable.

---

# 7 Core Components 🧠

This section describes the primary components that form the SERA ecosystem.

* **Sera AI Gateway**
    The Gateway (`https://sera-ai.gt-t.net`) is the single, secure entry point for all users. It is an intelligent reverse proxy responsible for:
    * **Authentication & Authorization**: Verifying user identity against LDAP/Active Directory and checking group memberships to control access.
    * **Application Spawning**: Dynamically launching and managing application instances (e.g., Streamlit apps) on-demand for each user.
    * **Traffic Proxying**: Securely forwarding HTTP and WebSocket traffic to the correct backend application instance.
* **Sera Orchestrator**
    The Orchestrator is the central reasoning engine. It manages the entire lifecycle of a user query:
    * **User Interaction**: Manages the chat interface and persists conversation history using PostgreSQL and ChromaDB.
    * **LLM-Powered Reasoning**: Uses a powerful generation model (e.g., `gemma3:27b`) to understand user intent, decompose complex tasks, and plan which tools to use.
    * **Tool Execution**: Dynamically calls external modules (tools) via gRPC to perform actions or gather information.
    * **Response Synthesis**: Generates a coherent, final answer from the information returned by the tools.
* **The Tool Ecosystem**
    This represents the collection of independent, specialized microservices that perform the actual work when called by the Orchestrator. While individual tools are modular, the ecosystem of tools itself—comprising **Retrievers**, **Connectors**, **Processors**, and **Assistants**—is a core component that gives SERA its diverse capabilities.
* **Shared Foundational Services**
    This is the stateful backbone supporting the entire platform. It consists of the common infrastructure that all other components rely on, including the PostgreSQL relational database, the ChromaDB vector database, and the `/assets` shared file storage.

---

# 8 Shared Infrastructure & Data Stores 🏛️

All modules operate on a common set of stateful services. This infrastructure is designed with the final Kubernetes deployment in mind, where all components will be backed by persistent storage to ensure data integrity and high availability.

## 8.1 Relational Database (PostgreSQL)

The system's memory and relational source of truth is a PostgreSQL database. Its primary role is to store structured and relational data, including conversation history, user sessions, and detailed, queryable metadata for all ingested assets.

* **Schema Management**: The database schema is not managed manually. It is defined programmatically using SQLAlchemy ORM models within the code of each component. Schema migrations and updates are handled by a dedicated tool, **Alembic**, ensuring the database structure is version-controlled and consistent with the application code.
* **Data Persistence**: In the local development environment, the database's data files are persisted on the host machine at `/ai/psql/data` via a Docker volume mount. This ensures that all data survives container restarts.

## 8.2 Vector Database (ChromaDB)

The Vector Database is the engine of conceptual understanding for the ecosystem. Its role is to store knowledge not as text, but as mathematical representations of meaning (embeddings), allowing for high-speed semantic search.

The VDB component of the SERA architecture was set up with two distinct parts: the server instance and a shared client abstraction layer.

* **The Server**: A standard ChromaDB server instance is run as a Docker container. It is responsible for the low-level storage, indexing, and querying of vector data. All modules connect to this single, central server.
* **The Shared Client Abstraction Layer**: To ensure consistency and enable future flexibility, modules do not connect to the ChromaDB server directly with their own custom code. Instead, they use a standardized, shared client library located in the `/ai/vdb/` directory. This layer consists of two key files:
    * `interface.py` (**VDBInterface**): This is the most critical piece. It's an Abstract Base Class that defines the standard contract for all vector database interactions in the ecosystem. It specifies the exact methods and signatures (e.g., `add_documents`, `query`) that any VDB client must implement. All modules are coded to use this interface, not a specific database's client.
    * `chroma_client.py` (**ChromaVDB**): This is the concrete, default implementation of the `VDBInterface`. It contains the actual logic for connecting to the ChromaDB server and translating the standard interface calls into specific ChromaDB API requests.

This two-part setup is a core architectural decision. It decouples the application logic from the specific VDB technology, meaning the backend could be swapped from ChromaDB to another system in the future by simply creating a new client that adheres to the `VDBInterface`, without needing to refactor every module in the ecosystem.

## 8.3 Shared Assets (/assets)

This storage layer acts as the physical backplane for the ecosystem. Its role is twofold: 1) To provide a durable, shared location for raw files (PDFs, images, logs) that modules need to access. 2) Critically, it houses the shared library of compiled gRPC stubs in the `/assets/protos/` directory. This directory is the lynchpin that allows the Orchestrator to dynamically communicate with any tool without requiring a static, compile-time dependency, making the ecosystem truly plug-and-play.

**Architectural Rule for Metadata**: A core data integrity rule is that any metadata stored in PostgreSQL that references a file in this directory must store the path relative to the `/assets` root (e.g., `images/my_image.png`). This ensures that references remain valid when the application moves from the local filesystem to the Kubernetes environment.

**Kubernetes Mapping**: In production, this entire directory is backed by a ReadWriteMany (RWX) Persistent Volume Claim (PVC), allowing multiple pods to mount and access the same filesystem simultaneously.

**Key Subdirectories**:
* `protos/`: Critically important. Contains compiled Python gRPC stub files (`*_pb2.py`, `*_pb2_grpc.py`). The Orchestrator's `ToolExecutor` dynamically loads these stubs at runtime to call other services.
* `docs/`: Contains ingested source documents and markdown guides that provide detailed instructions for the LLM on how to use each tool.
* `images/`, `pdfs/`, `xlsx/`: Stores ingested raw files and assets extracted from source systems.
* `cache/`, `tmp/`: For temporary and cached data that can be regenerated.
* `logs/`: Stores output logs from individual components during local development. In production, this will be handled by the cluster's logging solution.

## 8.4 Message Queue & Cache (Redis)

The ecosystem includes a Redis service that serves two critical, high-performance roles.

* **Asynchronous Task Broker**: It acts as the message broker for the background task queue (managed by Celery), enabling long-running processes like the CAP's "Re-analyze All CVs" feature to run independently of user sessions.
* **Caching Layer**: It provides a fast, in-memory cache for modules to store temporary or frequently accessed data, reducing database load and improving response times.

**Container Configuration**: The service runs using the official `redis:7` Docker image, exposing port `6379` and persisting its data to the host's `/ai/redis/data` directory.


---

# 9 LLM Services & The AI Factory 🏭

All Large Language Models (LLMs) are hosted in a dedicated, high-performance environment known as the **AI Factory**. This is a Kubernetes cluster equipped with NVIDIA H100 GPUs, optimized for inference workloads. Models are served via standardized backends, providing stable endpoints for the ecosystem to consume.

The following list describes the standard, approved models currently available. This set of services can be expanded, and new models may be deployed to the AI Factory based on a formal justification and review process, ensuring they meet the technical and resource requirements of the platform.

* `nvidia/llama-3.2-nv-embedqa-1b-v2`
    * **Backend**: NVIDIA NIM (NVIDIA Inference Microservice)
    * **Endpoint**: `https://aif-llm-nim-nv-embedqa-1b-v2.slo.gt-t.net/`
    * **Purpose**: This is the primary **Embedding Model**. Its function is to convert text (a user's query, a document chunk) into a high-dimensional numerical vector that captures its semantic meaning. This is the foundation of our **Retrieval-Augmented Generation (RAG)** pipeline, used by Ingestors to process documents and by Retrievers to find relevant context that semantically matches a user's query.
* `nvidia/llama-3.2-nv-rerankqa-1b-v2`
    * **Backend**: NVIDIA NIM
    * **Endpoint**: `https://aif-llm-nim-nv-rerankqa-1b-v2.slo.gt-t.net/`
    * **Purpose**: This is a **Reranking Model**. In a RAG pipeline, an initial vector search can return many documents that are broadly similar but not all perfectly relevant. The reranker takes the user's original query and this list of candidate documents, then re-scores each document based on how directly it answers the query. This step significantly improves the quality and precision of the context before it's sent to the reasoning model.
* `gemma3:27b`
    * **Backend**: Ollama
    * **Endpoint**: `https://aif-llm-ollama-gemma3-27b.slo.gt-t.net/`
    * **Purpose**: This is the primary **Reasoning and Generation Model**. It acts as the "brain" for the Sera Orchestrator. It excels at complex tasks like multi-step reasoning, understanding user intent, determining which tools to call (and with what parameters), and synthesizing a final, human-readable answer from the disparate data sources returned by the tools. Its use should be reserved for tasks requiring the highest level of cognitive performance.
* `gemma3:12b`
    * **Backend**: Ollama
    * **Endpoint**: `https://aif-llm-ollama-gemma3-12b.slo.gt-t.net/`
    * **Purpose**: This is the Workhorse Comprehension and Summarization Model. It offers a strong balance of performance and capability, making it ideal for tasks that require understanding and processing significant amounts of text without needing the full reasoning power of the 27b model. Its primary uses across the ecosystem include:
      * **Detailed Summarization**: Used by Ingestors and the Orchestrator's HistorySummarizer to create detailed summaries of documents or conversations.
      * **Structured Data Extraction**: Capable of parsing unstructured text and extracting information into a structured format like JSON.
      * **Plan Critique**: Acts as the secondary model in "Plan & Critique" workflows, providing a reliable sanity check for plans generated by the 27b model.
* `gemma3:4b`
    * **Backend**: Ollama
    * **Endpoint**: `https://aif-llm-ollama-gemma3-4b.slo.gt-t.net/`
    * **Purpose**: This is the primary Task Routing and Classification Model. It is optimized for speed and efficiency on less complex tasks, making it the default choice for high-throughput classification and internal routing logic. Its key roles include:
      * **Task Routing**: Used by the Orchestrator's "Task Router" to classify the complexity and intent of an incoming user query.
      * **RAG Query Expansion**: In "Ensemble RAG" workflows, this model can quickly generate alternative search queries.
      * **General Classification**: Used by any module for tasks like sentiment analysis, tone classification, or categorizing documents.
* `gemma3:1b`
    * **Backend**: Ollama
    * **Endpoint**: `https://aif-llm-ollama-gemma3-1b.slo.gt-t.net/`
    * **Purpose**: This is a High-Speed Utility Model. It is the fastest model available and is intended for simple, high-volume, and granular NLP tasks where latency is a primary concern. Its purpose is to handle simple text manipulation that is slightly beyond the scope of regular expressions. Use cases include:
      * **Keyword Extraction**: Used by Ingestors to quickly generate keyword tags for documents.
      * **Input Sanitization/Validation**: Can perform a quick first-pass check on user input to filter for PII or determine if the input is valid before it's sent to a larger model.
      * **Simple Boolean Classification**: Answering basic yes/no questions from a small piece of text, such as classifying an alert as "actionable" or "ignorable."

---

# 10 Tool Integration & Communication Standard

The ecosystem's modularity is enabled by a strict set of standards for how tools are defined and how they communicate. This replaces ad-hoc integrations with a true "plug-and-play" architecture where the Orchestrator can dynamically use any compliant module without requiring prior compilation or static dependencies.

## 10.1 gRPC as the Communication Backbone

The backbone of inter-service communication is **gRPC**. It was chosen over REST for its high performance, low latency, and use of strictly defined schemas via Protocol Buffers (`.proto` files), which guarantees efficient and type-safe communication between the Orchestrator and its tools.

## 10.2 The Tool Definition Contract (`tools.d/*.yaml`)

Every tool must declare itself to the ecosystem by providing a definition file in the Orchestrator's `config/tools.d/` directory. This YAML file is the contract that tells the Orchestrator everything it needs to know to understand and execute the tool. It includes the tool's name, a detailed description for the LLM, its input parameters (as a JSON schema), and the specific gRPC configuration required for execution. Some tools may also specify a custom prompt for the Orchestrator to use when synthesizing their results.

**Example `rag_confluence_retriever_cfr.yaml`:**
```yaml
- name: "rag_confluence_retriever"
  description: "Searches and retrieves pre-formatted context from the internal Confluence knowledge base. Can return a detailed context with full text or a brief context with only summaries."
  parameters:
    type: "object"
    properties:
      query:
        type: "string"
        description: "The user's original, verbatim query. It is critical to use the user's exact phrasing. Do not rephrase, correct, expand, or otherwise modify the user's input. If the user provides only keywords, pass those keywords directly."
      mode:
        type: "string"
        description: "The desired level of detail. Use 'DETAILED' for comprehensive answers or 'BRIEF' for summaries."
        enum: ["DETAILED", "BRIEF"]
      space_keys:
        type: "array"
        items: { type: "string" }
        description: "Optional. A list of Confluence space keys (e.g., ['SE']) to restrict the search to. Omit to search all available spaces."
      page_ids:
        type: "array"
        items: { type: "string" }
        description: "Advanced parameter for multi-step tasks. Use ONLY for follow-up actions when a previous tool result contained a link to another page and you need to retrieve information specifically from that linked page."
    required: ["query", "mode"]
  llm_instruction_guide_path: "/ai/assets/docs/llm_guides/cfr_llm_guide.md"
  synthesis_prompt_module_path: "prompts.modules.rag_confluence_retriever"
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

## 10.3 Dynamic Loading via Shared Stubs

The Orchestrator achieves modularity via **dynamic loading** at runtime. It doesn't have tool-specific client libraries built-in.

At startup, the Orchestrator reads all `*.yaml` files in `config/tools.d/`.

When the LLM decides to call a tool, the Orchestrator uses the `grpc_service_config` from the tool's YAML definition. It dynamically imports the tool's compiled gRPC stubs from the shared `/assets/protos/` directory.

> **Critical Requirement:** For this to work, every tool developer must place their compiled Python stubs (`*_pb2.py`, `*_pb2_grpc.py`) and an empty `__init__.py` file into a unique subdirectory within `/assets/protos/`.

---

## 10.4 New Module Implementation Checklist

The process for creating a new, compliant tool is standardized. Any developer (human or AI) must follow these steps:

1.  **Define API:** Create a `.proto` file defining the gRPC service and its messages.
2.  **Implement Server:** Write the Python gRPC server logic that performs the tool's function.
3.  **Generate & Place Stubs:** Compile the `.proto` file and place the generated Python stubs into a dedicated folder in the shared `/assets/protos/` directory.
4.  **Define Tool Contract:** Create the tool's `*.yaml` definition file and place it in the Orchestrator's `config/tools.d/` directory.
5.  **Provide LLM Guide (Optional):** For complex tools, create a detailed `llm_instruction_guide.md` file in `/ai/assets/docs/llm_guides/` to help the LLM use the tool effectively.
6.  **Define Custom Synthesis Prompt (Optional):** If the tool's output requires special formatting or synthesis logic (e.g., RAG results), define a custom prompt template within the Orchestrator's prompt library and reference its module path in the tool's YAML file using the `synthesis_prompt_module_path` key.

---

## 10.5 Detailed Developer Guide

While the checklist above provides a high-level overview, a comprehensive, step-by-step guide contains all the technical details, code examples, and best practices required to build a new module. This guide is essential for onboarding new developers and is also designed to be parsed and understood by an LLM, enabling the long-term vision of SERA autonomously creating new tools for itself.

**Primary Reference:** The definitive guide for module creation is located at: `/ai/assets/docs/llm_guides/llm_modules_guide.md`

---

## 10.6 Resilience and Error Handling

The system is designed to be resilient to tool failures. When the `ToolExecutor` encounters an error calling a tool (e.g., a network issue or an error returned by the tool itself), it does not crash. Instead, the error is caught, formatted, and added to the conversation context. The Orchestrator then sends this updated context back to the reasoning LLM, explicitly informing it that the tool call failed and why. This allows the LLM to re-plan, try an alternative tool, or formulate a helpful response to the user explaining the problem.

---

# 11 Ecosystem Modules & Services

The SERA platform is composed of several specialized modules that are designed to be independently developed and deployed. Full documentation for each is maintained on separate pages.

## 11.1 Core Modules

* **Sera Orchestrator:** The core reasoning engine.
* **Sera AI Gateway:** The secure user-facing proxy.
* **Confluence Ingestor (CFI):** A data pipeline that processes and stores content from Confluence into the shared data stores.
* **RAG Confluence Retriever (CFR):** A tool for intelligently searching and retrieving context from the ingested Confluence data.
* **Remote Automation Framework (RAF):** (Design) A centralized Python platform to manage and execute infrastructure automation tasks (e.g., interacting with NetBox, CloudStack).
* **Remote Automation Assistant (RAA):** (Design) A specialized LLM-driven module that translates natural language requests into actionable automation tasks for the RAF.

---

## 11.2 Module Categories

To ensure consistency and clarity across the ecosystem, each new module must be classified and named according to one, and only one, of the following categories. This is a strict definitional standard for the platform, as the category often forms part of the module's name or identifier. A module cannot belong to multiple categories.

* **(A) Assistant:**: Specialized agents that perform complex reasoning, planning, and task orchestration. An Assistant acts as the "brain" for a specific domain (like infrastructure automation). It can either dynamically compose calls to simple (C) Connectors for novel tasks or delegate the execution of complex tasks to a (F) Framework. (e.g., Remote Automation Assistant (RAA))
* **(C) Connector:** Bridges to external systems/APIs for data exchange or simple, direct actions (e.g., Jira Connector, ServiceNow Connector) -> no LLM
* **(F) Framework:** Core backend platforms that act as secure execution engines for pre-defined, multi-system workflows or reports. They are called by (A) Assistants to run complex, audited tasks. A Framework (like the RAF) contains the actual business logic for high-stakes automations, ensuring that AI agents operate within safe, pre-approved boundaries. (e.g., Remote Automation Framework (RAF))
* **(I) Ingestor:** Data ingestion pipelines that populate the shared data stores (e.g., Confluence Ingestor).
* **(P) Processor:** Backend tools for data transformation, computation, or analysis (e.g., Code Analyst).
* **(R) Retriever:** Tools dedicated specifically to knowledge retrieval for RAG (e.g., RAG Confluence Retriever).
* **(S) Strategist:** (Future) Advanced modules for complex, long-range planning.
* **(U) Utility:** A self-contained microservice that provides a specific, focused capability through a direct API endpoint (typically REST/HTTP). It does not integrate with the Orchestrator via the `tools.d/*.yaml` contract and is not discoverable by the Orchestrator's reasoning engine.

---

## 11.3 Stand-Alone Modules (Utility Services)

While the core architecture centers on the Orchestrator calling specialized Tools, the SERA ecosystem also supports **Stand-Alone Modules**. These are independent, self-contained services designed to perform specific, granular tasks that do not require the complex reasoning or multi-step planning capabilities of the Sera Orchestrator. They are classified under the **(U) Utility** category.

They function as part of the ecosystem but are intended for direct integration with other applications rather than being a "tool" in the Orchestrator's toolkit.

The primary principles governing these modules are:

* **Direct Integration:** They are designed to be called directly by other applications (e.g., a web front-end, another backend service) via a simple, well-defined API, typically REST/HTTP.
* **Self-Contained Logic:** A utility module encapsulates all the logic needed to perform its single function. It receives a request, processes it, and returns a response without needing to be part of a larger, Orchestrator-managed chain.
* **Ecosystem Citizenship:** Despite their independence, these modules are still first-class citizens of the SERA ecosystem. They must adhere to the established Ecosystem Standards (Section 12), including standardized configuration, logging, and health checks. They leverage shared resources like the AI Factory for models.
* **Resource Optimization:** These modules are ideal for using smaller, specialized, or fine-tuned models from the AI Factory. By design, they should avoid using the large, resource-intensive reasoning models like gemma3:27b, ensuring efficient resource use across the platform.

### Example in Practice: Geo-Location Utility (GLU)

A perfect use case for a Utility service is converting a textual address into precise geolocation data.

* **Module Name & Category:** Geo-Location Utility (GLU), classified as (U) Utility.
* **Purpose:** To accept a string containing a physical address and return structured JSON with the corresponding latitude, longitude, and other relevant details.
* **API Interaction:** Another application would call the GLU directly via an HTTP POST request, bypassing the Sera Orchestrator entirely.

**Example cURL request to the GLU:**

```bash
curl -X POST [https://sera-utils.gt-t.net/glu/api/v1/geocode](https://sera-utils.gt-t.net/glu/api/v1/geocode) \
-H "Content-Type: application/json" \
-d '{
  "address": "Vítězné nám. 576/1, 160 00 Praha 6-Dejvice, Czechia"
}' 
```
```json
{
  "status": "SUCCESS",
  "input_address": "Vítězné nám. 576/1, 160 00 Praha 6-Dejvice, Czechia",
  "latitude": 50.0993,
  "longitude": 14.3938,
  "confidence_score": 0.98,
  "metadata": {
    "country": "Czechia",
    "city": "Prague",
    "district": "Prague 6"
  }
}
```

# Ecosystem Standards & Operational Design

This section formally defines the operational standards that all modules must adhere to, ensuring consistency, reliability, and maintainability across the entire platform.

---

## Configuration Management

All modules must follow a standardized configuration pattern. Default settings are defined in a `config/settings.yaml` file within the module's directory. However, any setting can be overridden by environment variables. This allows for flexible deployment and secure management of secrets (like API tokens and database passwords) in both local Docker environments and production Kubernetes environments.

---

## System-Wide Logging

To ensure observability and ease of debugging, all modules must adopt the established multi-file logging standard:

- **`app.log`**: A comprehensive, verbose log with a `DEBUG` level for detailed troubleshooting.
- **`app_err.log`**: A high-level log capturing only `WARNING`, `ERROR`, and `CRITICAL` messages for quick issue identification.
- **`chat.log` / `transaction.log`**: An isolated, human-readable log that records the inputs and outputs of each core transaction (e.g., a user query, a tool execution) for auditing purposes.

---

## Module Health Checks 🩺

Every independent service (e.g., CFR, RAA) must expose a basic health check endpoint. For gRPC services, this should be a simple, standard RPC (e.g., a `Check` method in a `grpc.health.v1.Health` service) that returns a `SERVING` status. This allows automated systems like Kubernetes to perform health probes, enabling automatic restarts and robust service management.

---

## Quality Assurance & Evaluation 🧪

To manage the non-deterministic nature of LLMs, a standardized quality assurance process is required. All critical modules are accompanied by a suite of test scripts and questions. Following any change to a module's logic, prompts, or model configuration, these tests must be run to compare new outputs against "known-good" results and ensure no regressions are introduced.

---

# Deployment & CI/CD

This section outlines the current deployment model and the future, production-ready architecture.

---

## Environment Architecture

- **Current (Staging/Dev)**: Core applications run as `systemd` services on a dedicated server. This setup connects to the AI Factory Kubernetes cluster for LLM services and facilitates rapid development and iteration.
- **Future (Production)**: The final architecture will have all components containerized and deployed into the AI Factory Kubernetes cluster.
- **Data Services**: Stateful services like PostgreSQL and ChromaDB will be deployed using dedicated Kubernetes Operators or StatefulSets backed by persistent volumes.
- **Public API**: In addition to the internal gRPC network, a secure, public-facing RESTful (HTTP/JSON) API will be exposed to allow broader integration with third-party services and web clients, managed by an API Gateway.

---

## CI/CD & Source Code Management

The CI/CD process is a key area of development for moving from the current staging environment to a robust production deployment.

- **Current State**: The source code for all modules resides on the development server. Deployment is a manual process of managing the `systemd` services. For backup and versioning, the code is synced daily to a GitLab repository, but this is a one-way backup and not a true CI/CD workflow.
- **Future State (Roadmap)**: The target is to implement full, automated CI/CD pipelines within GitLab. When the ecosystem moves to Kubernetes, each module will have its own pipeline responsible for:
  - Automated testing (unit, integration).
  - Building a versioned container image.
  - Pushing the image to a container registry.
  - Automated, secure deployment to the production Kubernetes cluster.

---

# Local Development Environment

The local development environment is a standardized setup composed of containerized services and a defined filesystem structure that mirrors the production design.

---

## Relational Database (PostgreSQL)

The local environment includes a PostgreSQL service for all structured data storage.

- **Container Configuration**: The service runs using the official `postgres:16` Docker image. It is configured to expose port `5432` and persist its data to the host's `/ai/psql/data` directory. The configuration defines the default user (`ai_assistant_user`), password, and database name (`ai_assistant_db`).

```docker
# Configuration for the running sera_ai_dev_postgres container
IMAGE: postgres:16
NAME: sera_ai_dev_postgres
PORTS: 5432:5432
ENVIRONMENT:
  - POSTGRES_PASSWORD
  - POSTGRES_USER=ai_assistant_user
  - POSTGRES_DB=ai_assistant_db
VOLUMES:
  - /ai/psql/data:/var/lib/postgresql/data
RESTART_POLICY: unless-stopped
```

Connection Details: Modules connect to this service via localhost:5432 using the established credentials.

---

## Vector Database (ChromaDB)

The local environment includes a ChromaDB service for all semantic search capabilities.

- **Container Configuration**: The service runs using the official `chromadb/chroma` Docker image. It is configured to expose port `8000` and persist its vector stores to the host's `/ai/vdb/vectorstore` directory.

```docker
# Configuration for the running sera_ai_dev_chroma container
IMAGE: chromadb/chroma
NAME: sera_ai_dev_chroma
PORTS: 8000:8000
VOLUMES:
  - /ai/vdb/vectorstore:/chroma/chroma
RESTART_POLICY: unless-stopped
```

Connection Details: Client modules connect to this service via localhost:8000.

---

## Shared Asset Storage (/assets)

The local filesystem includes a standardized `/assets` directory which mirrors the structure of the production persistent volume. This directory serves as the central repository for shared files, ensuring that paths used during development are consistent with those in the production environment.