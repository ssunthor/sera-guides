# Sera Router - Design Document

**Version:** 0.1
**Date:** 2025-11-01
**Author:** Martin Kecka & Gemini AI

---

## 1. Introduction & Purpose

The Sera Router is a high-performance, centralized gRPC service that acts as the single, intelligent entry point for all Large Language Model (LLM) calls across the entire SERA ecosystem.

It replaces the local HAProxy instance and all module-specific `LLMInterface` classes. Its purpose is to completely abstract away the complexity of the backend LLM infrastructure (Ollama, NIM, VLLM, etc.) and provide all modules with a single, unified, and powerful API for text generation, vision analysis, and tool-use.

The Sera Router is the key enabler for the `sera-common` library, allowing all modules (Orchestrator, CAP, RMI, etc.) to use one shared client, managed from one central location.

### Core Goals:

* **Centralization:** Provide one gRPC service for all modules to call.
* **Abstraction:** Hide all backend complexity. Modules request `gemma3:27b`; they don't know or care if it's served by Ollama, VLLM, or NIM.
* **Smart Task Routing:** Intelligently route requests based on `task_type` (e.g., "summarization") to the most cost-effective and appropriate model (e.g., `gemma3:12b`).
* **Smart Load Balancing:** Replace the local HAProxy by managing a pool of backend endpoints (e.g., `slo`, `nyh`, `prg`) and distributing load using a least-connections-style logic.
* **Dynamic Parameter Injection:** Automatically modify request payloads with logic that is too complex for a simple proxy (e.g., "This prompt has 20k tokens, I will auto-set `num_ctx: 24000`").
* **Centralized Management:** All routing, model, and endpoint logic is defined in a single `settings.yaml` file, designed to be managed by the Sera Admin Suite (SAS).

---

## 2. Architecture & Integration

The Sera Router is a **Core Foundational Service** that sits at the network network edge of the Sera Ecosystem on `ukslo-crp-ai01`. It replaces the local HAProxy and becomes the new gateway to the remote AI Factory.

### Data Flow:

[Module (e.g., CAP, RMI, Orchestrator)]
        |
        | 1. Imports `SeraLLMClient` from `sera-common`
        |    Makes gRPC call to Sera Router (e.g., "grpc://localhost:50055")
        v
[Sera Router ] <-- (This new gRPC service)
        |
        | 2. Reads `settings.yaml` (managed by SAS)
        | 3. Performs Smart Task Routing (finds model)
        | 4. Performs Load Balancing (finds endpoint)
        | 5. Performs Payload Translation (builds Ollama/NIM JSON)
        | 6. Performs Parameter Injection (e.g., auto-sets `num_ctx`)
        |
        | 7. Makes direct HTTP request to the chosen endpoint
        v
[HAProxy-Remote (in K8s AI Factory)]
        |
        | 8. (e.g., [https://aif-llm-ollama-gemma3-27b.slo.gt-t.net](https://aif-llm-ollama-gemma3-27b.slo.gt-t.net))
        |    Balances load across the 24xH100 GPU pods
        v
[LLM GPU Pods (Ollama, NIM, VLLM)]

### Interactions:

* **Consumers (All Modules):** The Sera Orchestrator, CAP, RMI, CFI, CAA, etc., will all be updated to use the new `sera-common` library, which contains the gRPC client for the Sera Router.
* **Dependencies (Remote AI Factory):** The Sera Router makes outgoing HTTP/S requests to the remote HAProxy endpoints in the K8s cluster (e.g., `aif-llm-ollama-gemma3-27b.slo.gt-t.net`).
* **Manager (SAS):** The SAS-WebUI and CU (Configuration Utility) will be used to read, edit, and reload this module's `settings.yaml` file.

---

## 3. Core Functionality & Process Flow

The Sera Router's logic for handling a single gRPC call is a multi-step pipeline:

1.  **Receive Request:** A module (e.g., CAP) calls the `GenerateContent` gRPC method with a `GenerateContentRequest`. This request includes the `trace_id`, `task_type` (e.g., "summarization"), `model_preference` (e.g., "auto"), and the `Content` payload.
2.  **Smart Model Selection (Task Routing):**
    * The Sera Router reads the `model_preference`.
    * If `model_preference` is `"auto"` (or null): It looks up the `task_type` in its `settings.yaml` `task_routing_rules` to find the target model.
        * `"summarization"` -> `model_name: "gemma3:12b"`
        * `"classification"` -> `model_name: "gemma3:4b"`
        * `"general_reasoning"` -> `model_name: "gemma3:27b"`
    * If `model_preference` is specific (e.g., `"gemma3:27b"`): It bypasses the task router and uses the requested model directly. `model_name: "gemma3:27b"`.
3.  **Endpoint Selection (Load Balancing):**
    * The Sera Router now has a `model_name` (e.g., `"gemma3:27b"`). It looks this up in its `settings.yaml` `model_definitions`.
    * It finds the list of available endpoints: `[slo, nyh, prg]`.
    * It uses an internal, in-memory counter (e.g., round-robin or basic least-connections) to select the best endpoint URL (e.g., `https://aif-llm-ollama-gemma3-27b.nyh.gt-t.net`).
    * It also retrieves the `backend_type` (e.g., `"ollama"`) for this model.
4.  **Payload Translation (Abstraction):**
    * The Sera Router knows the target backend is `"ollama"`.
    * It takes the generic gRPC `Content` payload and translates it into the specific JSON format that the Ollama `/api/chat` endpoint expects.
    * (If the backend were `"nim"`, it would translate the same gRPC payload into the NVIDIA NIM JSON format).
5.  **Parameter Injection (Smart Logic):**
    * Before sending, the Sera Router performs its final smart logic.
    * It reads its `parameter_injection_rules` from `settings.yaml`.
    * **Example 1 (Tokens):** It calculates the prompt token count. If `token_count > 16000`, it injects/overwrites `options: { num_ctx: 24000 }` in the Ollama JSON payload.
    * **Example 2 (Vision):** If it detects an `image` Part in the request, it validates that the chosen model (`gemma3:27b`) actually supports vision. If not, it can either error gracefully or (in a future version) auto-route to a vision-capable model.
6.  **Execute & Stream Response:**
    * The Sera Router, now acting as an HTTP client, makes the `requests.post()` call to the selected remote HAProxy (e.g., `...nyh.gt-t.net`) with the fully prepared payload.
    * It streams the response back to the original module (CAP) through the open gRPC stream, translating the backend's JSON chunks into the gRPC `GenerateContentResponse` format.

---

## 4. API Design (gRPC)

The Sera Router will expose a single, powerful gRPC service. The `sera-common` library will provide a simple Python client for this service.

**File:** `/ai/assets/protos/sera_router/generated/sera_router.proto`

[proto]
syntax = "proto3";

package sera.router.v1;

import "google/protobuf/struct.proto";

// The Sera Router Service is the single point of entry for all
// LLM calls in the SERA ecosystem.
service SeraRouter {
  // GenerateContent provides a unified, bi-directional streaming API for
  // text, vision, and tool-use generation.
  rpc GenerateContent(stream GenerateContentRequest)
      returns (stream GenerateContentResponse);
}

// The request message from a client module (e.g., CAP, Orchestrator).
message GenerateContentRequest {
  // A unique ID for end-to-end logging.
  // Propagated from the Orchestrator or generated by the module.
  string trace_id = 1;

  // The ID of the calling module (e.g., "CAP", "RMI", "Orchestrator").
  string calling_module_id = 2;

  // The "intent" of the call. Used for "auto" model routing.
  // (e.g., "summarization", "classification", "general_reasoning", "code_gen")
  string task_type = 3;

  // The model preference.
  // "auto": Lets the Sera Router pick the best model based on task_type.
  // "gemma3:27b": Forces the Sera Router to use this specific model.
  string model_preference = 4;

  // The core content payload (e.g., the prompt).
  Content content = 5;

  // A list of tools the LLM can use, in OpenAI format.
  repeated Tool tools = 6;

  // Optional generation config (temp, max_tokens).
  GenerationConfig generation_config = 7;
}

// The response message streamed back from the Sera Router.
message GenerateContentResponse {
  // A chunk of the generated content from the LLM.
  Content content_chunk = 1;

  // Metadata sent with the *final* response chunk.
  UsageMetadata metadata = 2;
}

// A generic content block, compatible with Gemini/OpenAI.
message Content {
  string role = 1; // "user", "assistant", "system"
  repeated Part parts = 2;
}

// A part of a content block, supporting multi-modal requests.
message Part {
  oneof data {
    string text = 1;
    InlineData image_data = 2;
    ToolCall tool_call = 3;
    ToolResponse tool_response = 4;
  }
}

// Represents an image.
message InlineData {
  string mime_type = 1; // e.g., "image/jpeg", "image/png"
  bytes data = 2;       // The raw image bytes
}

// --- Standard Tool Call Structures ---
message Tool {
  google.protobuf.Struct function_declaration = 1; // OpenAI format
}

message ToolCall {
  string id = 1;
  string name = 2;
  string args_json = 3; // Arguments as a JSON string
}

message ToolResponse {
  string id = 1;
  string name = 2;
  string response_json = 3; // The tool's output as a JSON string
}

// --- Standard Config Structures ---
message GenerationConfig {
  optional float temperature = 1;
  optional int32 max_output_tokens = 2;
}

// Final metadata message.
message UsageMetadata {
  int32 prompt_token_count = 1;
  int32 candidates_token_count = 2;
  int32 total_token_count = 3;
  string model_used = 4;          // The *actual* model that was chosen
  string endpoint_hit = 5;      // The endpoint that served the request (e.g., "slo")
  string error_message = 6;     // If any error occurred
}
[/proto]

---

## 5. Configuration (settings.yaml)

The Sera Router's "brain" is its `settings.yaml` file, which will be managed by the SAS-WebUI.

**File:** `/ai/sera_router/config/settings.yaml`

[yaml]
# 1. Smart Routing Logic: Maps a task to a model.
# The "model_preference" in a gRPC request overrides this.
task_routing_rules:
  summarization: "gemma3:12b"
  classification: "gemma3:4b"
  code_gen: "gemma3:27b"
  general_reasoning: "gemma3:27b"
  default: "gemma3:27b" # Fallback if task_type is unknown

# 2. Model Definitions: The core abstraction map.
# This links a model name to its backend type and available endpoints.
model_definitions:
  "gemma3:27b":
    backend_type: "ollama" # "ollama", "nim", "vllm", "openai"
    ssl_verify: false
    endpoints:
      - "[https://aif-llm-ollama-gemma3-27b.slo.gt-t.net](https://aif-llm-ollama-gemma3-27b.slo.gt-t.net)"
      - "[https://aif-llm-ollama-gemma3-27b.nyh.gt-t.net](https://aif-llm-ollama-gemma3-27b.nyh.gt-t.net)"
      - "[https://aif-llm-ollama-gemma3-27b.prg.gt-t.net](https://aif-llm-ollama-gemma3-27b.prg.gt-t.net)"
  "gemma3:12b":
    backend_type: "ollama"
    ssl_verify: false
    endpoints:
      - "[https://aif-llm-ollama-gemma3-12b.slo.gt-t.net](https://aif-llm-ollama-gemma3-12b.slo.gt-t.net)"
      - "[https://aif-llm-ollama-gemma3-12b.nyh.gt-t.net](https://aif-llm-ollama-gemma3-12b.nyh.gt-t.net)"
      - ...
  "gemma3:4b":
    backend_type: "ollama"
    ssl_verify: false
    endpoints:
      - "[https://aif-llm-ollama-gemma3-4b.slo.gt-t.net](https://aif-llm-ollama-gemma3-4b.slo.gt-t.net)"
      - ...
  "nim-embedqa":
    backend_type: "nim"
    api_key_env_var: "NVIDIA_API_KEY"
    endpoints:
      - "[https://aif-llm-nim-nv-embedqa-1b-v2.slo.gt-t.net](https://aif-llm-nim-nv-embedqa-1b-v2.slo.gt-t.net)"
      - ...
  "nim-rerank":
    backend_type: "nim"
    api_key_env_var: "NVIDIA_API_KEY"
    endpoints:
      - "[https://aif-llm-nim-nv-rerankqa-1b-v2.slo.gt-t.net](https://aif-llm-nim-nv-rerankqa-1b-v2.slo.gt-t.net)"
      - ...

# 3. Parameter Injection Rules: Smart, logic-based overrides.
parameter_injection_rules:
  - backend_type: "ollama"
    token_threshold: 16000 # If prompt > 16k tokens
    payload_injection:
      options:
        num_ctx: 24000 # Auto-extend context window
  - backend_type: "ollama"
    token_threshold: 30000
    payload_injection:
      options:
        num_ctx: 32768
[/yaml]

---

## 6. The sera-common Library

This Sera Router enables the creation of a new `sera-common` Python library. This library will be added as a `requirements.txt` dependency for all other modules.

The `sera-common` library will contain:

* **gRPC Stubs:** The compiled Python files from `sera_router.proto`.
* **SeraLLMClient:** A new, simple `llm_interface.py`.

### Example of the new sera_common.SeraLLMClient:

[python]
# In sera-common/llm_client.py
import grpc
from .generated import sera_router_pb2, sera_router_pb2_grpc

class SeraLLMClient:
    def __init__(self, sera_router_service_url: str):
        # sera_router_service_url = "grpc://localhost:50055" (from the module's config)
        self.channel = grpc.insecure_channel(sera_router_service_url)
        self.stub = sera_router_pb2_grpc.SeraRouterStub(self.channel)
        print(f"SeraLLMClient connected to Sera Router at {sera_router_service_url}")

    def generate_content(
        self,
        messages: list,
        task_type: str,
        model_preference: str = "auto",
        tools: list = None,
        trace_id: str = "unknown"
    ):
        # 1. This client's ONLY job is to convert Python dicts
        #    into the gRPC request objects.
        request_pb = sera_router_pb2.GenerateContentRequest(
            trace_id=trace_id,
            calling_module_id="my_module_name", # This would be set in __init__
            task_type=task_type,
            model_preference=model_preference,
            # ... (logic to convert `messages` into `Content` pb)
            # ... (logic to convert `tools` into `Tool` pb)
        )

        # 2. Make the single, unified gRPC call
        try:
            responses = self.stub.GenerateContent(iter([request_pb]))
            for response in responses:
                # 3. Yield back the results, translated from gRPC to Python
                if response.HasField("content_chunk"):
                    yield {"type": "chunk", "data": ...} # logic to convert pb to dict
                if response.HasField("metadata"):
                    yield {"type": "metadata", "data": ...} # logic to convert pb to dict

        except grpc.RpcError as e:
            print(f"Error calling Sera Router: {e}")
            yield {"type": "error", "message": str(e)}

    def close(self):
        self.channel.close()
[/python]