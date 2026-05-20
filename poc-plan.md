# PoC Plan: ik_llama.cpp

## Project Classification
- **Type:** model-serving
- **Key Technologies:** C/C++, CMake, llama.cpp (fork), GGUF model format, OpenAI-compatible API
- **ODH Relevance:** ik_llama.cpp provides a high-performance CPU-optimized LLM inference server with an OpenAI-compatible API. This is directly relevant to Open Data Hub's model serving capabilities — it can serve as a lightweight, CPU-friendly alternative to vLLM or TGI for quantized models, making LLM inference accessible on clusters without GPUs.

## PoC Objectives
What we want to prove:
1. **The ik_llama.cpp server can be built and containerized** — The C++ project compiles successfully in a container with all necessary build dependencies and produces working binaries.
2. **The server can load and serve a quantized GGUF model** — A small quantized model (Qwen3-0.6B IQ4_NL, ~400MB) can be downloaded and loaded at startup, demonstrating the GGUF model loading pipeline works.
3. **The OpenAI-compatible API is functional** — Chat completions, legacy completions, and model listing endpoints respond correctly, proving the server can act as a drop-in replacement for OpenAI API in downstream applications.
4. **CPU-only inference is viable on OpenShift** — The server produces coherent responses using only CPU resources, validating ik_llama.cpp's optimized CPU inference path in a containerized environment.

## Infrastructure Requirements
- **Inference Server:** Custom (llama-server binary built from source)
- **Vector Database:** none
- **Embedding Model:** none
- **GPU Required:** No (CPU-only PoC; GPU support can be added later with CUDA build)
- **Persistent Storage:** 2Gi PVC for model weights (the GGUF file is ~400MB but we allow headroom for larger models)
- **Resource Profile:** large (4Gi RAM, 2 CPU — quantized LLM inference needs decent RAM and benefits from multiple CPU cores for matrix multiplication)
- **Sidecar Containers:** none

## Test Scenarios

### Scenario 1: Health Check
- **Description:** Verify the llama-server process has started and the model is fully loaded
- **Type:** http
- **Input:** GET /health
- **Expected:** Returns HTTP 200 OK indicating server readiness
- **Timeout:** 120 seconds (model loading can take time on first startup)

### Scenario 2: Model Properties
- **Description:** Query the server's model properties endpoint to confirm model metadata is exposed
- **Type:** http
- **Input:** GET /props
- **Expected:** Returns 200 with JSON containing model info and default generation parameters
- **Timeout:** 30 seconds

### Scenario 3: Chat Completion (OpenAI-compatible)
- **Description:** Send an OpenAI-compatible chat completion request to verify the full inference pipeline
- **Type:** http
- **Input:** POST /v1/chat/completions with `{"model": "default", "messages": [{"role": "user", "content": "What is 2+2? Answer with just the number."}], "max_tokens": 32, "temperature": 0.1}`
- **Expected:** Returns 200 with JSON containing `choices` array where the assistant message includes "4"
- **Timeout:** 60 seconds

### Scenario 4: Legacy Completions
- **Description:** Test the text completion endpoint for backward compatibility
- **Type:** http
- **Input:** POST /v1/completions with `{"prompt": "The capital of France is", "max_tokens": 16, "temperature": 0.1}`
- **Expected:** Returns 200 with generated text mentioning "Paris"
- **Timeout:** 60 seconds

### Scenario 5: Model Listing
- **Description:** Verify the OpenAI-compatible model listing endpoint
- **Type:** http
- **Input:** GET /v1/models
- **Expected:** Returns 200 with JSON containing a `data` array listing the loaded model
- **Timeout:** 15 seconds

## Dockerfile Considerations

This is a C++ project that needs to be compiled from source using CMake. The Dockerfile should use a **multi-stage build**:

**Stage 1 (Builder):**
- Base image: A recent Ubuntu or Fedora with build tools
- Install build dependencies: `build-essential`, `cmake`, `git`, `libcurl4-openssl-dev`, `curl`
- Copy the source code
- Run CMake configure: `cmake -B build -DGGML_NATIVE=OFF -DLLAMA_CURL=ON` (note: `GGML_NATIVE=OFF` is critical for container portability — `GGML_NATIVE=ON` would optimize for the build machine's CPU, which may differ from the runtime CPU)
- Run CMake build: `cmake --build build --config Release -j$(nproc)`
- The key binary is `build/bin/llama-server`

**Stage 2 (Runtime):**
- Base image: Minimal Ubuntu or UBI
- Install runtime dependencies: `libcurl4`, `libgomp1` (OpenMP for parallel CPU inference)
- Copy `build/bin/llama-server` from builder stage
- Create a `/models` directory for GGUF model files
- Add a startup script that downloads the model if not present (using curl and `LLAMA_MODEL_URL` env var) and then launches llama-server
- **ENTRYPOINT** should be the startup script or directly: `/app/build/bin/llama-server --model /models/model.gguf --host 0.0.0.0 --port 8080 --ctx-size 2048`
- **Add EXPOSE 8080** — this is a long-running HTTP server
- The container listens on port 8080 and serves an OpenAI-compatible API

**Important build notes:**
- Use `-DGGML_NATIVE=OFF` to avoid CPU-specific optimizations that won't work on different cluster nodes
- Use `-DLLAMA_CURL=ON` to enable built-in model downloading via URL (the server supports `--model` with a URL)
- The project has no existing Dockerfile, but there are Containerfile templates in `docker/` directory (`ik_llama-cpu.Containerfile`) that can serve as reference

## Deployment Considerations

**Deployment Model:** Deploy as a Kubernetes **Deployment** with 1 replica. This is a long-running server process that listens on port 8080.

**Service:** Create a Kubernetes **Service** on port 8080 targeting the llama-server container. The service exposes the OpenAI-compatible API.

**Model Storage:** Use a PVC (2Gi) mounted at `/models` to persist the downloaded GGUF model file. This avoids re-downloading on pod restarts. Alternatively, an init container can download the model before the main container starts.

**Startup Time:** The server needs time to download the model (~400MB) on first run and then load it into memory. Set `initialDelaySeconds` on readiness/liveness probes to at least 90 seconds. Use the `/health` endpoint for health checks.

**Resource Requests/Limits:**
- Memory: Request 2Gi, Limit 4Gi (the quantized 0.6B model needs ~1GB RAM loaded plus overhead)
- CPU: Request 2 cores, Limit 4 cores (more cores = faster inference due to OpenMP parallelism)

**Environment Variables:**
- `LLAMA_MODEL_URL`: URL to download the GGUF model file (default: the Qwen3-0.6B IQ4_NL model from HuggingFace)

**Testing:** All tests are HTTP-based. Send requests to the Service endpoint. The server implements a standard OpenAI-compatible API at `/v1/chat/completions`, `/v1/completions`, `/v1/models`, plus custom endpoints like `/health` and `/props`.

**Scaling note:** For production, multiple replicas can be deployed behind the Service for load balancing. Each replica loads its own copy of the model into memory.