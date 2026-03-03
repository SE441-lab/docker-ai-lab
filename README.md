# Lab: Writing a Docker Compose File for a Three-Tier AI Chatbot

## Overview

You are given a working three-service AI chatbot application with **all three Dockerfiles already written**. Your task is to author the `compose.yaml` file that wires the services together into a running system.

The application consists of three services:

| Service | Folder | Port | Role |
|---|---|---|---|
| `model` | `model/` | 11434 | Ollama-based LLM server |
| `backend` | `backend/` | 8000 | FastAPI proxy between frontend and model |
| `frontend` | `frontend/` | 3000 | Remix/React chat UI |

When fully running, the chat UI is accessible at `http://localhost:3000`.

**Do not modify any Dockerfile or source code.** Read them carefully; they contain the information you need.

---

## Prerequisites

- Docker Desktop installed and running
- At least 8 GB RAM available to Docker (16 GB recommended)
- Basic familiarity with Docker Compose concepts: services, networks, volumes, environment variables, healthchecks

---

## Step 1 — Understand the existing Dockerfiles

Before writing a single line of YAML, read each Dockerfile and answer these questions in your head (they will appear in the report):

- What port does each service expose?
- What environment variables does each service expect to receive at runtime?
- Which service needs external tools (`curl`, `jq`) and why?

---

## Step 2 — Plan shared infrastructure

Your `compose.yaml` needs two top-level resources shared across services:

**A named network** so services can reach each other by service name (Docker's internal DNS resolves service names on a shared network automatically).

**A named volume** for one of the services. Ask yourself: which service downloads a large file at first startup that you would not want to re-download every time the container restarts?

Declare both at the **top level** of `compose.yaml`, outside the `services:` block:

```yaml
volumes:
  <your-volume-name>:

networks:
  <your-network-name>:

services:
  ...
```

The names you choose here must match exactly what you reference inside each service definition.


---

## Step 3 — Define the `model` service

This is the most complex service. Work through each requirement:

**3.1 — Build context**  
Point the build at the `./model` folder.

**3.2 — Network**  
Attach to your named network.

**3.3 — Port**  
Publish the port the Ollama server listens on (check the Dockerfile `EXPOSE` instruction and the [Ollama docs](https://hub.docker.com/r/ollama/ollama)).

Compose accepts two port syntaxes — either is acceptable:
```yaml
# Short form
ports:
  - "11434:11434"

# Long form (explicit)
ports:
  - published: 11434
    target: 11434
```

**3.4 — Volume**  
Mount your named volume to `/root/.ollama` inside the container. This is where Ollama stores downloaded model weights.

Use the long-form volume mount syntax so the type is explicit:

```yaml
volumes:
  - type: volume
    source: <your-volume-name>
    target: /root/.ollama
```

> The `source` must match the volume name you declared at the top level in Step 2.

**3.5 — Environment variable**  
The startup script reads a `MODEL` environment variable to know which model to pull. Pass it in with a sensible default:

```
MODEL=${MODEL:-mistral:latest}
```

The `:-` syntax means: use the value of `$MODEL` if set in the shell, otherwise fall back to `mistral:latest`.

**3.6 — Healthcheck**  
The backend must not start until the *model* is downloaded and loaded. Write a healthcheck (https://docs.docker.com/reference/compose-file/services/#healthcheck) that:
- Calls the Ollama REST API to list loaded models
- Parses the JSON response with `jq` to check whether the model named by `$MODEL` is present
- Runs every 10 seconds, times out after 5 seconds, retries up to 50 times

The Ollama API endpoint that lists available models is:
```
GET http://localhost:11434/api/tags
```
It returns JSON in this shape:
```json
{
  "models": [
    { "name": "mistral:latest", ... },
    ...
  ]
}
```

Your `test:` command should use `curl` to call that endpoint and `jq` to check whether a model with the expected name exists. The healthcheck must be written as a `CMD-SHELL` string. Here is the structure to fill in:

```yaml
healthcheck:
  test: ["CMD-SHELL", "curl -s http://localhost:11434/api/tags | jq -e '<jq filter here>' > /dev/null"]
  interval: 10s
  timeout: 5s
  retries: 50
  start_period: 900s
```

The `jq` filter selects from the `.models[]` array and checks the `.name` field against the value of `$MODEL`. Refer to the [jq manual](https://jqlang.org/manual/) for the `select()` function.

> **Important:** Set `start_period` to at least `900s` (15 minutes). On first run, the model binary must be downloaded from the internet before the healthcheck can possibly pass. Without a long start period, Docker will mark the container unhealthy before the download completes, causing the backend to fail immediately.

**3.7 — Memory limit**  
LLMs are memory-hungry. Set a hard memory limit of `8G` on this service under `deploy.resources.limits` (https://docs.docker.com/reference/compose-file/deploy/).

---

## Step 4 — Define the `backend` service

**4.1 — Build context**  
Point the build at the `./backend` folder.

**4.2 — Network and port**  
Attach to the same named network. Publish port `8000`.

**4.3 — Environment variable**  
The backend reads `MODEL_HOST` to know where to send inference requests. Set it to the model service URL. Because both services are on the same named network, Docker resolves service names as hostnames — use the service name, not `localhost`:

```
MODEL_HOST=http://<model-service-name>:11434
```

**4.4 — Dependency with health condition**  
The backend must not start until the model service is healthy — not just started. A plain `depends_on` only waits for the container to start, not for the model to be ready. Use:

```yaml
depends_on:
  <model-service-name>:
    condition: service_healthy
```

---

## Step 5 — Define the `frontend` service

**5.1 — Build context**  
Point the build at the `./frontend` folder.

**5.2 — Network and port**  
Attach to the same named network. Publish port `3000`.

**5.3 — Environment variables**  
The frontend Dockerfile sets `PORT` and `HOST`, but these should also be passed in via Compose so they can be overridden without rebuilding the image. Pass `PORT=3000` and `HOST=0.0.0.0`.

**5.4 — Dependency**  
The frontend only needs the backend to be started (no healthcheck needed). A plain `depends_on` is sufficient here.

---

## Step 6 — Run and verify

```bash
docker compose up --build
```

On **first run**, expect to wait **5–15 minutes** while the `mistral:latest` model (~4 GB) downloads. Monitor progress with:

```bash
docker compose logs -f model
```

Once the backend and frontend services start, open:

```
http://localhost:3000
```

Send a chat message. If you get a response, all three services are communicating correctly.

To stop:

```bash
docker compose down
```

To also remove the downloaded model data (forces re-download next time):

```bash
docker compose down -v
```

---

## Deliverables

Submit one file:

1. `compose.yaml` — at the root of the repository

Include a short `REPORT.md` (max one page) answering:

1. Why is `condition: service_healthy` used on the backend's `depends_on` instead of a plain dependency?
2. Why does the model service healthcheck need a `start_period` of 900 seconds?
3. How does the backend reach the model service by hostname? What Docker feature makes this work?
4. What would happen if you removed the named volume (`docker compose down -v`) and restarted? What would be different compared to a normal restart (`docker compose down`)?

---

## Grading Rubric

| Criterion | Points |
|---|---|
| `compose.yaml` correct compose file | 10 |
| `REPORT.md` questions answered correctly | 10 |
| **Total** | **20** |

