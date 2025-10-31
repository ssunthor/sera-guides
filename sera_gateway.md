# /ai/assets/docs/sera_guides/sera_gateway.md
# Sera AI Gateway Documentation

**Version:** 1.2
**Date:** 2025-11-01
**Author:** Martin Kecka & Gemini AI

---

## 1. Overview 📜

The Sera AI Gateway is a secure, centralized web portal for accessing various internal AI and Machine Learning applications. It acts as an intelligent reverse proxy that handles user authentication, authorization, and on-demand spawning of application instances.

The core purpose is to provide a single, secure entry point (**https://sera-ai.gt-t.net**) for users. It abstracts away the complexity of accessing different backend services, whether they are simple on-demand Streamlit scripts or complex, persistent applications built with a modern React/FastAPI stack.

---

## 2. Core Components 🏗️

The system is built around a few key files that work together.

* **`config.py`**: The central configuration file. It defines everything from authentication methods and LDAP server details to the list of available applications and session settings. This is the primary file to modify when adding new apps or changing environment settings.
* **`main.py`**: The heart of the gateway. It's a **FastAPI** application responsible for:
    * Serving login, launcher, and spawning pages.
    * Handling user authentication (login/logout) and session management.
    * Proxying traffic to the correct backend, whether it's a *dynamically spawned instance* or a *persistent, long-running service*.
    * Providing API endpoints to check application status (for spawned instances).
    * Injecting user authentication data into HTTP headers for proxied services.
* **`pages/`**: This directory contains the HTML templates for the user interface (`login.html`, `launcher.html`, and `spawning.html`).
* **`tmp/sera_ai_instances.db`**: A local **SQLite** database that tracks the state (`spawning`, `ready`, `failed`), PID, and port of active **on-demand** application instances.
    * **Note:** This is *not* used for `launch_type: "service"` applications (like CAP/CAA) which run persistently.

---

## 3. Authentication & Authorization 🔐

The gateway uses a robust, two-layered approach to control access.

### Authentication

Authentication is the process of verifying a user's identity. The gateway supports two methods, controlled by the `AUTH_METHOD` variable in `config.py`:

* **LDAP (Primary)**: When `AUTH_METHOD` is set to `"ldap"`, the gateway connects to the configured Active Directory servers. It first verifies the user's credentials (`sAMAccountName` and password). It then performs a critical check: the user must be a member (directly or through nesting) of the main **`SERA-Role-GeneralUsers`** group or the **`SERA-Role-Admins`** group. If both checks pass, the user is authenticated.
* **Dictionary (Fallback)**: If `ldap3` isn't installed or `AUTH_METHOD` is set to `"dict"`, the gateway uses the `VALID_USERS` dictionary in `config.py` for basic username/password validation. This is primarily for development and testing.

Session data (username, display name, group memberships) is stored in a secure, signed, and timed browser cookie. The session cookie is refreshed on every request, implementing a "sliding" inactivity timeout defined by `SESSION_INACTIVITY_TIMEOUT_SECONDS`.

### Authorization

Authorization is the process of determining what an authenticated user is allowed to do.

* Access to each application is controlled by the `allowed_groups` list within its definition in the `APPLICATIONS` dictionary in `config.py`.
* The gateway compares the user's LDAP groups (stored in their session cookie) against the application's `allowed_groups`.
* Users in the **`SERA-Role-Admins`** group are super-users and have access to all applications, bypassing individual checks.

### Passing Identity to Backend Services

For persistent services (like React/FastAPI applications), the gateway's primary role is authentication. After validating the user's session, it injects their identity into the proxied HTTP request using the following headers:

* **`X-Forwarded-User`**: The user's `sAMAccountName` (e.g., `jdoe`).
* **`X-Forwarded-Display-Name`**: The user's display name (e.g., `John Doe`).
* **`X-Forwarded-Groups`**: A comma-separated list of the user's relevant SERA group CNs (e.g., `SERA-Role-Admins,SERA-App-CAP-Users`).

The backend service (e.g., CAP) is then responsible for its own internal authorization based on these trusted headers.

---

## 4. Application Proxy Models ⚙️

The gateway supports two distinct models for managing and proxying applications, controlled by the `launch_type` in `config.py`.

### Model 1: On-Demand Instance Spawning (e.g., Streamlit)

This model is for simple, single-user applications (like Streamlit scripts or basic Python apps) that are not designed to run persistently.

* **Config:** `launch_type: "streamlit"` or `launch_type: "python"`
* **Process:**
    1.  **User Request**: A user clicks "Launch" on an application (e.g., `sira`).
    2.  **Check & Spawn**: The gateway checks the `sera_ai_instances.db`. If no 'ready' instance is found, it marks the instance as **`spawning`** and launches a **background thread**.
    3.  **Status Page**: The user is immediately redirected to the dynamic status page (`/spawning/sira`). This page polls the `/status/sira` API endpoint.
    4.  **Port Probing**: The background thread spawns the process (e.g., `streamlit run ...`) and **actively probes the new port** until it gets a successful connection.
    5.  **Redirection**: Once the port is live, the thread updates the database to **`ready`**. The user's browser polls this new status and redirects to the application's proxy URL (`/app/sira/`).
* **Proxy Logic**: The gateway proxies all HTTP and WebSocket traffic (e.g., `/app/sira/_stcore/stream`) to the unique port (e.g., `127.0.0.1:8502`) assigned to that user's spawned instance.
* **Cleanup**: A background task (`cleanup_inactive_instances`) finds and terminates these spawned processes after a period of inactivity or on user logout.

### Model 2: Persistent Service Proxy (e.g., React/FastAPI)

This model is for modern, complex applications (like CAP or CAA) that run as persistent, long-running services and often have their own React-based UIs.

* **Config:** `launch_type: "service"` and a `service_url` (e.g., `http://127.0.0.1:7888`).
* **Process:**
    1.  **User Request**: A user clicks "Launch" on the application (e.g., `cv_analysis_processor`).
    2.  **Authentication**: The gateway validates the user's session cookie.
    3.  **Direct Proxy**: The gateway immediately proxies the request. This model **does not use the SQLite database or spawning logic**.
* **Proxy Logic**: The gateway's role is twofold:
    1.  **UI Serving**: It serves the static React UI assets (e.g., `/app/cv_analysis_processor/index.html`, `/app/cv_analysis_processor/assets/main.js`).
    2.  **API Proxying**: It proxies all API calls (e.g., `/api/share/`, `/app/cv_analysis_processor/api/v1/jobs`) to the backend `service_url`, **injecting the user's authentication headers** (`X-Forwarded-User`, etc.) into the request.
* **Cleanup**: None. The gateway is stateless in this model. The persistent service is managed externally (e.g., by `systemd` or Kubernetes).

This dual-model architecture allows the gateway to provide a seamless single sign-on (SSO) experience for all applications in the ecosystem, regardless of their underlying technology.

---

## 5. Future Enhancements & Roadmap 🚀

While the core functionality is robust, several enhancements could be considered for future versions.

* **Modernized User Interface (UI):** The gateway's own `launcher.html` page is functional but could be modernized to improve aesthetics, perhaps matching the look and feel of the newer React applications.
* **Centralized Observability:** While logs are captured, the gateway is in a prime position to log standardized metrics (like request counts and timings per app) to a central dashboard (e.g., Grafana/Prometheus) to provide a high-level overview of ecosystem health and application usage.