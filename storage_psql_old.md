# /ai/assets/docs/sera_guides/storage_psql_old.md
# PostgreSQL Local Development Setup

## Overall Purpose

This `/ai/psql` directory is designated for PostgreSQL-related files during local development of the AI Assistant (SERA/GIA). Its primary purpose is to house the persisted data directory for the PostgreSQL instance running locally via Docker.

## File Structure

* **`/ai/psql/data/`**: This subdirectory is mounted as a Docker volume into the PostgreSQL container (`/var/lib/postgresql/data`). It stores all the actual database files (tables, indexes, data) ensuring persistence across container restarts.
* **`/ai/psql/psql.md` (This file)**: Documentation for the local PostgreSQL setup.

## Local Development Setup (Docker) [CRITICAL]

For local development (e.g., on the L4 dev server), PostgreSQL should be run using its official Docker container to maintain consistency with the target Kubernetes environment.

1.  **Ensure Directory Exists:** `mkdir -p /ai/psql/data`
2.  **Run Command:**
    ```bash
    # Replace passwords with strong, secure values!
    docker run -d \
      --name sera_ai_dev_postgres \
      -p 5432:5432 \
      -e POSTGRES_PASSWORD="7pDcu95Ws0YJ" \
      -e POSTGRES_USER=ai_assistant_user \
      -e POSTGRES_DB=ai_assistant_db \
      -v /ai/psql/data:/var/lib/postgresql/data \
      postgres:16 
    ```
3.  **Connection:** The database will be accessible locally via:
    * Host: `localhost`
    * Port: `5432`
    * User: `ai_assistant_user`
    * Password: `7pDcu95Ws0YJ`.
    * Database: `ai_assistant_db`
    * DSN Example: `postgresql://ai_assistant_user:7pDcu95Ws0YJ@localhost:5432/ai_assistant_db`

## Kubernetes Mapping

In the target Kubernetes environment (within the `syseng-ai` namespace):

1.  **PostgreSQL Deployment:** PostgreSQL will be deployed using a dedicated Kubernetes Operator (like Zalando PGO or Crunchy Data PGO) or a `StatefulSet`.
2.  **Persistence:** It will use a dedicated `PersistentVolumeClaim` (likely RWO is sufficient) for its data directory (`/var/lib/postgresql/data` inside the container), fulfilled by NetApp Trident. The local `/ai/psql/data` directory *is not* directly transferred; the K8s instance will have its own fresh data volume initially.
3.  **Service:** A Kubernetes `Service` (e.g., `ai-postgres-postgresql`) will provide the stable internal DNS name for connection within the `syseng-ai` namespace.
4.  **Connection:** Application components (Orchestrator, Tools, Ingestors) running in K8s will connect using the internal service name (e.g., `ai-postgres-postgresql.syseng-ai.svc.cluster.local`) and credentials managed via Kubernetes Secrets.

## Schema Management Note

The actual database **schema** (table structures, relationships, etc.) is **not defined or stored** in this `/ai/psql` directory. The schema is defined by **SQLAlchemy models** located in `/ai/app/database/models/` and managed/applied using **Alembic migrations**. This `/ai/psql/data` directory only holds the live data files for the *local development instance*.