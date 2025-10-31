# /ai/assets/docs/sera_guides/sera_gateway.md
# Sera AI Gateway Documentation

**Version:** 1.1
**Date:** 2025-11-01
**Author:** Martin Kecka & Gemini AI

---

## 1. Overview 📜

The Sera AI Gateway is a secure, centralized web portal (`https://sera-ai.gt-t.net`) that serves as the single entry point for all applications within the SERA AI Ecosystem.

It has evolved into a sophisticated, intelligent reverse proxy with two primary responsibilities:

1.  **On-Demand Instance Management:** For applications like Streamlit or simple Python scripts, the gateway dynamically **spawns, manages, and terminates** individual application instances for each user.
2.  **Authenticating Proxy for Services:** For modern, persistent applications (like React frontends with FastAPI backends, e.g., CAP/CAA), the gateway acts as a secure **authenticating reverse proxy**. It validates the user's session and forwards requests to the correct persistent backend service, injecting the user's identity as HTTP headers.

It handles all user authentication, authorization, session management, and traffic proxying, providing a seamless and secure experience while abstracting the backend complexity from the end-user.

---

## 2. Core Components 🏗️

The system is built around a few key files that work together:

* **`main.py`**: The heart of the gateway. It's a **FastAPI** application responsible for:
    * Serving the login, launcher, and spawning pages.
    * Handling user authentication (login/logout) via LDAP or a fallback dictionary.
    * Managing the application lifecycle (spawning, proxying, and cleanup).
    * Proxying HTTP, API, and WebSocket traffic to the correct backend.
* **`config.py`**: The central configuration file. It defines:
    * Authentication methods (LDAP/dict) and server details.
    * Session and security settings (`SECRET_KEY`, `INTERNAL_API_KEY`).
    * The **`APPLICATIONS` dictionary**, which is the "service registry" that tells the gateway *how* to handle each application (e.g., spawn a process vs. proxy to a service).
* **`pages/`**: Contains the static HTML templates for the user interface (`login.html`, `launcher.html`, `spawning.html`).
* **`tmp/sera_ai_instances.db`**: A lightweight **SQLite database** that acts as the state store for on-demand instances. It tracks the `key` (user + app_id), `pid`, `port`, `status` (`spawning`, `ready`, `failed`), and `last_accessed` time for each running process.

---

## 3. Authentication & Authorization 🔐

The gateway uses a robust, two-layered approach to control access.

### Authentication

Authentication verifies a user's identity. The gateway supports two methods, controlled by `AUTH_METHOD` in `config.py`:

* **LDAP (Primary)**: When set to `"ldap"`, the gateway connects to Active Directory.
    1.  It first validates the user's `sAMAccountName` and password.
    2.  It then performs a crucial check: the user **must** be a member (directly or nested) of the primary login group (`LDAP_ALLOWED_GROUP_CN`, e.g., "SERA-Role-GeneralUsers") or the admin group (`LDAP_ADMIN_GROUP_CN`).
    3.  If both checks pass, an authenticated session is created.
* **Dictionary (Fallback)**: If `AUTH_METHOD` is `"dict"`, it uses the `VALID_USERS` dictionary for development and testing.

### Session Management

Session data (username, display name, group memberships) is stored in a secure, signed, and timed browser cookie using the **ItsDangerous** library.

The gateway features a **sliding session timeout**. A middleware (`refresh_session_middleware`) re-issues the session cookie on *every* request, renewing its expiration time. This means the user is only logged out after a true period of inactivity (defined by `SESSION_INACTIVITY_TIMEOUT_SECONDS`), not after a fixed time from login.

### Authorization

Authorization determines what an authenticated user is allowed to do.

* Access to each application is controlled by the `allowed_groups` list in its `config.py:APPLICATIONS` definition.
* The gateway compares the user's LDAP groups (from their session cookie) against the application's `allowed_groups`.
* Users in the `LDAP_ADMIN_GROUP_CN` ("SERA-Role-Admins") bypass this check and have access to all applications.

---

## 4. Application Launch & Proxy Architecture ⚙️

This is the core logic of the gateway. The routing and launch behavior is determined entirely by the `launch_type` specified for each app in the `config.py:APPLICATIONS` dictionary.

### Type 1: On-Demand Instances (`launch_type: "streamlit"` or `"python"`)

This flow is used for applications that are spawned on-demand for a single user.

**1. Spawning Flow (Async with Polling):**
This process is now fully asynchronous to prevent the user from seeing a "hung" browser.
1.  User clicks "Launch" for an app (e.g., "SIRA"). The gateway receives the request at `/launch/sira`.
2.  The `get_or_spawn_instance` function is called. It writes a "spawning" status to the `sera_ai_instances.db` for the `username_sira` key.
3.  The gateway immediately redirects the user to `/spawning/sira`.
4.  The `spawning.html` page is served. This page contains JavaScript that polls the `/status/sira` endpoint every 2 seconds.
5.  In the background, a new thread runs `_spawn_process_and_update_db`. This function:
    * Finds a free port (e.g., `8502`).
    * Constructs the `subprocess.Popen` command (e.g., `streamlit run ... --server.port 8502`).
    * **Launches the process and actively probes the port** in a loop until it receives a successful connection or the `APP_SPAWN_TIMEOUT_SECONDS` is reached.
    * Once the app is confirmed to be running, it updates the database row to `status: "ready"`, `pid: 12345`, `port: 8502`.
6.  The user's browser, still polling `/status/sira`, receives the `{"status": "ready"}` JSON response.
7.  The JavaScript immediately redirects the user to the application's URL: `/app/sira/`.

**2. Proxy Flow:**
* **HTTP:** All requests to `/app/sira/{path:path}` are handled by the `reverse_proxy_route`. It looks up the `username_sira` key in the database, finds the port (`8502`), and proxies the request to `http://127.0.0.1:8502/{path:path}`.
* **WebSocket:** For Streamlit apps, the dedicated `@app.websocket("/app/{app_id}/_stcore/stream")` endpoint captures the WebSocket connection, looks up the correct port, and establishes a new connection to `ws://127.0.0.1:8502/_stcore/stream`, shuttling messages back and forth.

### Type 2: Persistent Services (`launch_type: "service"`)

This flow is used for modern, persistent applications like the **CV Analysis Processor (CAP)**, which run their own React frontend and FastAPI backend.

**1. "Launch" Flow:**
1.  User clicks "Launch" for "CAP" (`/launch/cv_analysis_processor`).
2.  The gateway sees `launch_type: "service"`.
3.  **No spawning occurs.**
4.  The user is immediately redirected to the application's root URL: `/app/cv_analysis_processor/`.

**2. Proxy Flow (Frontend & Backend):**
This is where the gateway acts as an *authenticating proxy*.
* **Frontend (React App):** The user's request to `/app/cv_analysis_processor/` (and for all its static assets like `.js` and `.css` files) is handled by the `reverse_proxy_route`.
    * It checks the user's session and authorization.
    * It looks up the `service_url` from `config.py` (e.g., `http://127.0.0.1:7888`).
    * It proxies the request to `http://127.0.0.1:7888/`.
* **Backend (FastAPI API):** The React application makes API calls to its *own* backend, but it makes them to the gateway's path (e.g., `/api/positions`).
    * A dedicated route, `@app.api_route("/api/{path:path}")`, captures these.
    * The gateway authenticates the user's session cookie.
    * It proxies the request to the backend service's API path (e.g., `http://127.0.0.1:7888/api/{path:path}`).
    * **Crucially, it injects the user's identity as HTTP headers** that the backend service (CAP) can trust:
        * `X-Forwarded-User`: The user's `sAMAccountName`.
        * `X-Forwarded-Display-Name`: The user's `givenName` from LDAP.
        * `X-Forwarded-Groups`: A comma-separated list of the user's relevant AD groups (e.g., `SERA-App-CAP-Admins,SERA-Role-Admins`).

---

## 5. Instance & Session Management 🧹

### Instance Cleanup
A background task (`cleanup_inactive_instances`) runs periodically (e.g., every 5 minutes) to maintain hygiene for **on-demand instances**:
* **Dead PIDs:** It finds instances in the DB where the `pid` is no longer running on the server and removes them.
* **Inactivity:** It finds instances where `(current_time - last_accessed) > APP_INSTANCE_MAX_INACTIVE_SECONDS` (e.g., 6 hours). It terminates the process (`os.kill`) and removes the DB entry.
* **Failed Spawns:** It cleans up any entries stuck in the "spawning" or "failed" state for too long.

### Logout
When a user visits `/logout`:
1.  All **on-demand instances** associated with that user are terminated, and their DB entries are deleted.
2.  The `session` cookie is cleared.
3.  The user is redirected to the login page.
(Note: This does *not* affect persistent `service` applications, only the spawned processes.)

---

## 6. Internal API Endpoints 🔌

To support a microservice-based ecosystem, the gateway exposes internal API endpoints secured by the `INTERNAL_API_KEY`.

* **`GET /api/internal/users?group_cn=...`**:
    * This allows other trusted backend services (like CAP) to query the gateway for a list of users in a specific Active Directory group.
    * This is critical for features like allowing a CAP Admin to see a list of all `SERA-App-CAP-Users` without the CAP service needing its own LDAP credentials.
    * The gateway acts as the central, secure broker for LDAP queries.

---

## 7. Roadmap & Future Enhancements 🚀

* **SAML/SSO Integration:** The current username/password form will be supplemented or replaced with a full SAML 2.0 / ADFS integration to provide a true single sign-on (SSO) experience.
* **Containerization:** The gateway (and its `systemd` service) will be containerized for deployment in Kubernetes, aligning with the rest of the SERA ecosystem. The SQLite database will be replaced with a Persistent Volume Claim (PVC) or a shared database.
* **Dynamic App Configuration:** The `APPLICATIONS` dictionary, currently hardcoded in `config.py`, could be moved into the database to allow for dynamic registration of new applications without restarting the gateway.