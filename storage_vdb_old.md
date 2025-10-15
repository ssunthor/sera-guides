# /ai/assets/docs/sera_guides/storage_vdb_old.md
# Vector Database Abstraction Overview

## Overall Purpose

This `/vdb` directory contains the necessary code to abstract interactions with the Vector Database (VDB) used by the AI Assistant (SERA/GIA). The goal is to provide a consistent interface for storing and retrieving text embeddings and associated metadata, regardless of the specific VDB implementation used (initially ChromaDB).

This allows components like `HistoryManager`, RAG tools, and Ingestion Pipelines to interact with the VDB through a defined contract (`VDBInterface`).

## File Structure & Purpose

* **`interface.py` (`VDBInterface`)**:
    * **Purpose:** Defines the Abstract Base Class (ABC) specifying the standard methods that any VDB implementation used by this project MUST provide (e.g., `add_documents`, `query`, `create_collection_if_not_exists`, `delete_documents`).
    * **K8s Equivalent:** This is purely application code, packaged within the relevant container images (Orchestrator, Tools, Ingestors).

* **`chroma_client.py` (`ChromaVDB`)**:
    * **Purpose:** The concrete implementation of `VDBInterface` specifically for interacting with a **ChromaDB server instance** using its `HttpClient`. It handles connection, collection management, and data operations (add, query, delete).
    * **K8s Equivalent:** This is application code packaged within the relevant container images. It will be configured via environment variables or `config_core.py` to connect to the internal Kubernetes service endpoint of the deployed ChromaDB server.

* **`vdb.md` (This file)**:
    * **Purpose:** Provides documentation for the `/vdb` directory, local setup, and Kubernetes deployment expectations.
    * **K8s Equivalent:** Documentation, not a deployed component.

## Local Development Setup [CRITICAL]

For local development (e.g., on the L4 dev server), the `ChromaVDB` client expects to connect to a running ChromaDB server instance. The recommended way to achieve this is:

1.  **Run ChromaDB Server via Docker:** Use the official `chromadb/chroma` image.
    ```bash
    # Example run command (adjust host path for persistence):
    docker run -d -p 8000:8000 -v /local/path/to/chroma_data:/chroma/chroma \
           --name chroma_server chromadb/chroma
    ```
2.  **Configure `ChromaVDB` Client:** Ensure the client code (likely configured via `config_core.py` or environment variables read by the Orchestrator's `_initialize_vdb` method) connects to `host="localhost"` and `port=8000`.

*(Note: Using ChromaDB's embedded `PersistentClient` directly would require modifying the `ChromaVDB` client code or creating a separate local-only implementation).*

## Kubernetes Mapping

In the target Kubernetes environment (within the `syseng-ai` namespace):

1.  **ChromaDB Server Deployment:** ChromaDB itself will be deployed as a dedicated service, typically using a `StatefulSet` to manage its pod(s) and a `PersistentVolumeClaim` (likely RWO is sufficient for ChromaDB's own data) to store its database files persistently.
2.  **ChromaDB Service:** A Kubernetes `Service` (e.g., `chromadb-service`) will be created to provide a stable internal DNS name and IP address for the ChromaDB server pod(s) within the `syseng-ai` namespace.
3.  **`ChromaVDB` Client Configuration:** The `ChromaVDB` client code running within the Orchestrator, Tools, or Ingestion Pipeline pods will be configured to connect to this internal Kubernetes service name (e.g., `host="chromadb-service.syseng-ai.svc.cluster.local"`, `port=8000`).

## Embedding Strategy Note

* The `query` method in `VDBInterface` requires the caller to provide pre-computed query embeddings.
* The `add_documents` method needs clarification: either modify it to accept pre-computed embeddings from the caller (e.g., the Ingestion Pipeline) OR configure the ChromaDB server/collection with a default embedding function accessible *by the server*. Pre-computing embeddings in the client/ingestor is often more flexible. Ensure consistency across ingestion and querying.
