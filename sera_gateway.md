# /ai/assets/docs/sera_guides/sera_gateway.md
# Sera AI Gateway Documentation

**Version:** 1.0  
**Date:** 2025-08-08  
**Author:** Martin Kecka & Gemini AI  

---

## 1. Overview 📜

The Sera AI Gateway is a secure, centralized web portal for accessing various internal AI and Machine Learning applications. It acts as an intelligent reverse proxy that handles user authentication, authorization, and on-demand spawning of application instances.

The core purpose is to provide a single, secure entry point ([https://sera-ai.gt-t.net]) for users, abstracting away the complexity of managing and accessing different backend services, particularly **Streamlit** and standard Python web apps.

---

## 2. Core Components 🏗️

The system is built around a few key files that work together.

* **`config.py`**: The central configuration file. It defines everything from authentication methods and LDAP server details to the list of available applications and session settings. This is the primary file to modify when adding new apps or changing environment settings.
* **`main.py`**: The heart of the gateway. It's a **FastAPI** application responsible for:
    * Serving login and launcher pages.
    * Handling user authentication (login/logout).
    * Spawning/terminating application processes.
    * Proxying HTTP and WebSocket traffic to the correct backend application instance.
* **`pages/`**: This directory contains the HTML templates for the user interface (`login.html` and `launcher.html`).
* **`sera-ai-gateway.service`**: The `systemd` unit file that manages the gateway, ensuring it runs as a persistent background service on the server.

---

## 3. Authentication & Authorization 🔐

The gateway uses a robust, two-layered approach to control access.

### Authentication

Authentication is the process of verifying a user's identity. The gateway supports two methods, controlled by the `AUTH_METHOD` variable in `config.py`:

* **LDAP (Primary)**: When `AUTH_METHOD` is set to `"ldap"`, the gateway connects to the configured Active Directory servers. It first verifies the user's credentials (`sAMAccountName` and password). It then checks if the user is a member (directly or through nesting) of the main `Sera AI - Users` group. If both checks pass, the user is authenticated.
* **Dictionary (Fallback)**: If `ldap3` isn't installed or `AUTH_METHOD` is set to `"dict"`, the gateway uses the `VALID_USERS` dictionary in `config.py` for basic username/password validation. This is primarily for development and testing.

Session data (username, display name, group memberships) is stored in a secure, signed, and timed browser cookie using the **ItsDangerous** library.

### Authorization

Authorization is the process of determining what an authenticated user is allowed to do.

* Access to each application is controlled by the `allowed_groups` list within its definition in the `APPLICATIONS` dictionary in `config.py`.
* The gateway compares the user's LDAP groups (stored in their session cookie) against the application's `allowed_groups`.
* Users in the `Sera AI - Admin` group are super-users and have access to all applications, bypassing individual checks.

---

## 4. Process & Instance Management ⚙️

A key feature of the gateway is its ability to manage application instances dynamically.

* **User Request**: When a user clicks "Launch" on an application, the gateway receives the request.
* **Check for Existing Instance**: It queries a local SQLite database (`sera_ai_instances.db`) to see if an instance is already running for that specific user and application (e.g., `user1_excel_analyst`).
* **Spawn if Needed**: If no active instance is found, the gateway:
    1.  Finds an available network port between `STREAMLIT_PORT_START` and `STREAMLIT_PORT_END`.
    2.  Constructs a shell command to launch the application (e.g., `streamlit run ...` or `python3 ...`) with user-specific parameters.
    3.  Spawns the application as a new subprocess using `subprocess.Popen`.
    4.  Records the new instance's PID and port in the SQLite database.
* **Proxy Traffic**: The gateway then begins proxying the user's requests to the newly spawned or existing process.
* **Cleanup**: A background task (`cleanup_inactive_instances`) runs periodically to:
    * Terminate and remove instances that have been inactive for too long (`APP_INSTANCE_MAX_INACTIVE_SECONDS`).
    * Remove instances linked to dead PIDs.
    * Terminate all of a user's running applications upon logout.

---

## 5. Reverse Proxy Logic 🌐

The gateway intelligently forwards client traffic to the correct backend application instance.

### HTTP Requests
All standard web requests (e.g., `/app/sira/`, `/app/excel_analyst/static/some.css`) are captured by the `@app.api_route("/{app_id}/{path:path}")` endpoint. It uses the **httpx** library to stream the request and response between the user and the backend application, ensuring compatibility with large file uploads (up to 250MB).

### WebSocket Proxy
**Streamlit** relies heavily on WebSockets for its interactive elements. The gateway has a dedicated endpoint (`@app.websocket("/app/{app_id}/_stcore/stream")`) that establishes a WebSocket connection with the user and a corresponding connection with the backend Streamlit app. It then transparently shuttles messages back and forth, making the proxy invisible to both the client and the application.

### **6. Future Enhancements & Roadmap** 🚀

While the gateway is fully functional, several enhancements could be considered for future versions to improve reliability, user experience, and maintainability.

* **Reliable Process Spawning:** The current `time.sleep(8)` method to wait for an application to start can be fragile. A more robust approach would be to implement an **active port probe**. After launching a process, the gateway would repeatedly try to connect to the application's port. This ensures the user is connected the moment the app is ready, making the system more efficient and resilient to varying server loads.

* **Interactive Application Launch:** To improve the user experience, the "hang" while an application spawns could be replaced with a dynamic status page. When a user clicks "Launch," they would be redirected to a temporary "Spawning your application..." page. This page could use JavaScript to poll the gateway for a status update and automatically redirect the user to the application once it's ready. This provides clear feedback and makes the gateway feel more responsive.
    

* **Modernized User Interface (UI):** The current HTML interface is clean and functional. It could be modernized using a lightweight CSS framework or custom styles to improve aesthetics. Adding small animations, better-styled cards for the applications, and a more polished login screen would enhance the overall user impression.

* **Lightweight Observability:** Without needing a full monitoring stack, the gateway's value for troubleshooting can be increased by adding more detailed logs for key events. For example, logging specific app spawn times, user launch requests, and authorization failures would provide valuable insight for administrators. Example logs:
    * `INFO: User 'jdoe' launched app 'excel_analyst'.`
    * `INFO: App 'excel_analyst' for user 'jdoe' spawned successfully in 4.72 seconds.`
    * `WARNING: Authorization failed: User 'bsmith' attempted to launch unauthorized app 'oracle_analyst'.`