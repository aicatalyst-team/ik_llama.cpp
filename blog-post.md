## What is ik_llama.cpp?

If you've spent any time in the LLM inference world, you've almost certainly encountered [llama.cpp](https://github.com/ggerganov/llama.cpp) — the project that proved you don't need a rack of GPUs to run large language models. [ik_llama.cpp](https://github.com/ikawrakow/ik_llama.cpp) is a fork that pushes CPU and hybrid GPU/CPU performance even further, with novel quantization types (including IQ4_NL), Bitnet support, and DeepSeek MLA optimizations.

The project is written in C/C++ with a CMake build system and ships a full-featured HTTP server (`llama-server`) that exposes an OpenAI-compatible API. That last point is critical: it means any application already wired to talk to the OpenAI API can point at ik_llama.cpp instead, no code changes required. The server supports chat completions, legacy completions, model listing, and more — all running against quantized GGUF model files that can fit in a few hundred megabytes of RAM.

For this PoC, we built ik_llama.cpp from source, containerized it on a UBI base image, loaded a small Qwen3-0.6B model quantized to IQ4_NL (~400 MB), and deployed it to OpenShift as a CPU-only inference server.

## Why this matters for OpenShift AI

OpenShift AI (and its upstream, Open Data Hub) has strong model serving capabilities through vLLM and KServe. But those components are GPU-centric and relatively heavy. Not every team has GPU nodes available, and not every use case requires a 70B parameter model.

ik_llama.cpp occupies an interesting niche: it's a lightweight, CPU-friendly inference server that can run quantized models with surprisingly good performance. In an RHOAI evaluation, the project scored 55/100 — reflecting the fact that it operates outside the standard Red Hat AI stack, but also acknowledging its value for CPU and edge inference scenarios. The integration effort is real, but the payoff is clear: if you can serve a quantized model on CPU-only nodes, you've dramatically lowered the barrier to deploying LLM capabilities across your cluster.

This PoC exercises the model-serving pattern: build a custom inference server image, deploy it as a long-running service, and validate that it responds correctly to OpenAI-compatible API calls. It's the kind of thing KServe could eventually front-end, giving you autoscaling and traffic management on top of ik_llama.cpp's raw inference performance.

## Setting up the PoC

The infrastructure requirements for this PoC are refreshingly modest:

- **CPU/Memory:** 2 vCPUs, 4 GiB RAM (the "large" resource profile). Quantized inference benefits from multiple cores for matrix multiplication, and the model plus runtime context need room in memory.
- **GPU:** None. The whole point is CPU-only inference.
- **Storage:** A 2 GiB PVC for model weights. The Qwen3-0.6B IQ4_NL GGUF file is roughly 400 MB, but we left headroom for experimenting with larger models.
- **Model:** [Qwen3-0.6B-IQ4_NL](https://huggingface.co/bartowski/Qwen_Qwen3-0.6B-GGUF) from Hugging Face, downloaded at container startup via the `LLAMA_MODEL_URL` environment variable.
- **Sidecar containers:** None needed.
- **Vector DB / Embedding model:** Not applicable for this pure inference serving PoC.

The key decision was model selection. We wanted something small enough to load quickly on CPU, quantized with one of ik_llama.cpp's optimized formats, and capable enough to produce coherent responses. The 0.6B parameter Qwen3 model at IQ4_NL quantization hit that sweet spot.

--------------------
**[Image Placeholder 1: Architecture diagram of the PoC deployment]**

**Placement rationale**: Readers benefit from seeing the overall deployment topology before diving into Dockerfile and YAML details.

**Image generation prompt**: A clean technical architecture diagram on a white background showing: a Kubernetes pod containing a single container labeled "ik_llama.cpp server" with port 8080 exposed, connected to a PVC labeled "model-storage (2Gi)" containing a file icon labeled "model.gguf". An arrow from HuggingFace (cloud icon) points to the PVC labeled "download at startup". A Kubernetes Service routes traffic to the pod. Use flat design, blue and gray color palette with Red Hat red accents, 16:9 aspect ratio.

**Alt text**: Architecture diagram showing an OpenShift deployment with a single pod running the ik_llama.cpp server on port 8080, connected to a 2Gi persistent volume storing the GGUF model file downloaded from Hugging Face at startup.
--------------------

## Containerizing with UBI

Building ik_llama.cpp from source in a container is straightforward — it's a well-structured CMake project. We used a multi-stage build with a UBI base image to keep the final image lean while having full build tooling available during compilation:

```dockerfile
FROM registry.access.redhat.com/ubi9/ubi:latest AS builder

RUN dnf install -y gcc gcc-c++ cmake make git && dnf clean all

WORKDIR /build
COPY . .
RUN cmake -B build -DLLAMA_CURL=OFF -DLLAMA_NATIVE=OFF \
      -DCMAKE_BUILD_TYPE=Release && \
    cmake --build build --target llama-server -j$(nproc)

FROM registry.access.redhat.com/ubi9/ubi-minimal:latest
RUN microdnf install -y libstdc++ curl && microdnf clean all
COPY --from=builder /build/build/bin/llama-server /app/build/bin/llama-server
RUN mkdir -p /models
EXPOSE 8080
```

Two things worth noting. First, we disabled `LLAMA_NATIVE=OFF` to avoid compiling with CPU-specific instruction sets that might not match the target node — portability over peak performance. Second, we disabled `LLAMA_CURL` to avoid a build dependency; model downloads happen via a startup script rather than built-in curl support in the binary.

The final image landed at a reasonable size. The `llama-server` binary is statically linked against the llama.cpp libraries, so the runtime layer only needs the C++ standard library and curl for model fetching.

## Deploying to Kubernetes

We deployed as a standard Kubernetes Deployment with a Service, rather than a Job, since this is a long-running inference server. The model download happens in an init container or startup script, and the GGUF file persists on a PVC so subsequent pod restarts don't re-download it.

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: ik-llama-cpp
spec:
  replicas: 1
  selector:
    matchLabels:
      app: ik-llama-cpp
  template:
    spec:
      containers:
        - name: llama-server
          image: quay.io/aicatalyst/ik_llama.cpp-ik_llama.cpp:latest
          command:
            - /app/build/bin/llama-server
          args:
            - --model
            - /models/model.gguf
            - --host
            - "0.0.0.0"
            - --port
            - "8080"
            - --ctx-size
            - "2048"
          ports:
            - containerPort: 8080
          resources:
            requests:
              cpu: "2"
              memory: 4Gi
            limits:
              cpu: "2"
              memory: 4Gi
          volumeMounts:
            - name: model-storage
              mountPath: /models
      volumes:
        - name: model-storage
          persistentVolumeClaim:
            claimName: ik-llama-models
```

The context size of 2048 tokens keeps memory usage predictable for this small model. In production, you'd tune this based on your use case — longer contexts for RAG, shorter for quick Q&A.

--------------------
**[Image Placeholder 2: Screenshot of test results or server logs showing successful model load]**

**Placement rationale**: Placing a visual confirmation of the running server here bridges the deployment section and the test results, giving readers confidence the setup works.

**Image generation prompt**: A terminal screenshot showing server startup logs with a dark background (dark gray/black). The logs show: model loading progress, "model loaded" confirmation, and "listening on 0.0.0.0:8080" message. Use a monospace font, green and white text on dark background, 16:9 aspect ratio. Include a subtle OpenShift/Kubernetes context indicator in the terminal prompt.

**Alt text**: Terminal output showing ik_llama.cpp server startup logs, including model loading progress and a confirmation message that the server is listening on port 8080.
--------------------

## Test results

We ran five test scenarios against the deployed server. Here's what happened:

| Scenario | Endpoint | Status | Duration |
|---|---|---|---|
| Health check | `GET /health` | ✅ PASS | 0.0s |
| Model properties | `GET /props` | ✅ PASS | 0.0s |
| Chat completion (OpenAI-compatible) | `POST /v1/chat/completions` | ❌ FAIL | 6.0s |
| Legacy completions | `POST /completion` | ✅ PASS | 3.3s |
| Model list | `GET /v1/models` | ✅ PASS | 0.0s |

**4 out of 5 scenarios passed.** The server started cleanly, loaded the model, and responded to health checks, property queries, and the model listing endpoint instantly. The legacy completions endpoint worked correctly in 3.3 seconds — confirming that CPU inference is genuinely functional and producing results in a reasonable timeframe for a 0.6B model.

The one failure was the OpenAI-compatible chat completion endpoint (`/v1/chat/completions`). The request took 6 seconds before failing. This is notable because the legacy `/completion` endpoint worked fine, which means the core inference pipeline is healthy. The issue likely lies in how the chat template is applied to the Qwen3 model — ik_llama.cpp may not have the correct chat template registered for this specific model, or there may be a response formatting issue in the OpenAI compatibility layer. This is the kind of thing that's straightforward to debug: check the server logs for the specific error, try a different model with a well-known chat template, or explicitly pass a chat template via the `--chat-template` flag.

The 3.3-second inference time on the legacy completions endpoint deserves attention. For a CPU-only deployment with a quantized 0.6B model, that's entirely usable for many applications — chatbots, code assistants, document summarization with short contexts.

--------------------
**[Image Placeholder 3: Bar chart showing test results with pass/fail status and duration]**

**Placement rationale**: A visual summary of test results makes the pass/fail ratio immediately scannable and highlights the inference duration differences between endpoints.

**Image generation prompt**: A horizontal bar chart on a clean white background showing five test scenarios. Four bars are green (pass) and one is red (fail, labeled "chat-completion"). The x-axis shows duration in seconds (0-7s). The health-check, model-props, and model-list bars are very short (~0s), completions-endpoint is ~3.3s, and chat-completion is ~6s with a red X. Use flat design, sans-serif labels, 16:9 aspect ratio.

**Alt text**: Horizontal bar chart showing five test scenarios: health-check, model-props, and model-list passed in under 0.1 seconds; completions-endpoint passed in 3.3 seconds; chat-completion failed at 6.0 seconds.
--------------------

## What we learned

**CPU-only LLM inference is viable on OpenShift.** This isn't a theoretical claim — we got a working inference server responding in 3.3 seconds with no GPU resources. For teams that don't have GPU nodes or are deploying to edge locations, this is a real option.

**The OpenAI compatibility layer needs per-model attention.** The chat completions failure with Qwen3 is a reminder that "OpenAI-compatible" doesn't mean "works identically with every model." Chat templates, stop tokens, and response formatting vary by model family. If you're deploying this in production, test your specific model against the specific endpoints your application uses.

**ik_llama.cpp sits outside the RHOAI stack, and that's okay.** The RHOAI fitness score of 55/100 reflects a genuine gap: this isn't vLLM, it doesn't plug into KServe natively, and it lacks the observability integrations that come with the supported stack. But it fills a niche that the supported stack doesn't cover well today — lightweight, CPU-optimized inference for quantized models. A natural next step would be wrapping this behind a KServe InferenceService for autoscaling and unified model management.

**What we'd do differently:** We'd pass an explicit `--chat-template` flag for the Qwen3 model to resolve the chat completions failure. We'd also benchmark with a slightly larger model (2-3B parameters) to find the practical ceiling for CPU-only inference on a 4 GiB memory budget. And we'd add a readiness probe tied to `/health` to ensure Kubernetes doesn't route traffic before the model is loaded.

## Try it yourself

If you want to reproduce this PoC or build on it, here's everything you need:

- **Forked repository:** [aicatalyst-team/ik_llama.cpp](https://github.com/aicatalyst-team/ik_llama.cpp.git)
- **Container image:** `quay.io/aicatalyst/ik_llama.cpp-ik_llama.cpp:latest`
- **Upstream project:** [ikawrakow/ik_llama.cpp](https://github.com/ikawrakow/ik_llama.cpp)
- **Model used:** [Qwen3-0.6B-IQ4_NL GGUF](https://huggingface.co/bartowski/Qwen_Qwen3-0.6B-GGUF)
- **Open Data Hub documentation:** [opendatahub.io](https://opendatahub.io/)

Pull the image, point it at a GGUF model, and you've got a CPU-powered LLM inference server running on OpenShift in minutes. If you want to investigate the chat completions issue, the server logs are your best starting point — and contributions back to the upstream project are always welcome.
