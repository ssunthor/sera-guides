# /ai/assets/docs/sera_guides/storage_redis_old.md
# Redis Local Development Setup

## Overall Purpose

This `/ai/redis` directory is designated for Redis-related files during the local development of the SERA ecosystem. Its primary purpose is to house the persisted data directory for the Redis instance running locally via Docker, which serves as the primary message broker for asynchronous task queues and as a general-purpose caching layer.

## File Structure

* **`/ai/redis/data/`**: This subdirectory is mounted as a Docker volume into the Redis container (`/data`). It stores the Redis database files (`.rdb`), ensuring persistence across container restarts.
* **`/ai/redis/redis.md` (This file)**: Documentation for the local Redis setup.

## Local Development Setup (Docker) [CRITICAL]

For local development, Redis should be run using its official Docker container to maintain consistency with the target Kubernetes environment.

1.  **Ensure Directory Exists:** `mkdir -p /ai/redis/data`
2.  **Run Command:**
    ```bash
    # Replace 'YourSecurePasswordHere' with a strong, secure value!
    docker run -d \
      --name sera_ai_dev_redis \
      -p 6379:6379 \
      -v /ai/redis/data:/data \
      redis:7 --requirepass "xJRa4q3Yhs77"
    ```
3.  **Connection:** The Redis server will be accessible locally via:
    * **Host:** `localhost`
    * **Port:** `6379`
    * **Password:** `xJRa4q3Yhs77`
    * **Connection URI Example:** `redis://:xJRa4q3Yhs77@localhost:6379/0`

## Kubernetes Mapping

In the target Kubernetes environment (within the `syseng-ai` namespace):

1.  **Redis Deployment:** Redis will be deployed using a `StatefulSet` to ensure stable network identity.
2.  **Persistence:** It will use a dedicated `PersistentVolumeClaim` for its data directory, fulfilled by NetApp Trident.
3.  **Service:** A Kubernetes `Service` (e.g., `ai-redis-master`) will provide the stable internal DNS name for connection.
4.  **Connection:** Application components will connect using the internal service name and credentials managed via Kubernetes Secrets.

## Usage Note

Redis serves a different purpose than PostgreSQL. Its primary roles in the SERA ecosystem are:

* **Task Queue Broker:** To manage the queue of background jobs (e.g., re-analyzing CVs) for workers like Celery.
* **Caching Layer:** To store frequently accessed, non-critical data to improve application performance.

It should **not** be used as a primary store for structured, relational data; that remains the responsibility of PostgreSQL.