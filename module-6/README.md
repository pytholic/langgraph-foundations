# Module 6: Deploying and Interacting with LangGraph as a Service

## Why Module 6?

In Module 5, we built the `task_mAIstro` agent and ran it **in-process** — directly in Python/Jupyter:

```python
graph = builder.compile(checkpointer=within_thread_memory, store=across_thread_memory)
graph.stream({"messages": [...]}, config, stream_mode="values")
```

This works for prototyping, but has fundamental limitations:

- **Ephemeral state** — `InMemoryStore` and `MemorySaver` live in your Python process. When it dies, everything is gone.
- **No remote access** — You can only interact with the graph from the same process that created it.
- **No concurrency handling** — No way to manage multiple users or simultaneous requests.

Module 6 solves this by turning the graph into a **deployed service** with persistent storage, HTTP access, and production-grade features.

## Module 5 vs Module 6

|                      | Module 5                              | Module 6                                    |
| -------------------- | ------------------------------------- | ------------------------------------------- |
| **Graph runs**       | In your Python process                | As a Docker service (Redis + Postgres)      |
| **State storage**    | `InMemoryStore` (ephemeral)           | Postgres (persistent)                       |
| **How you interact** | `graph.stream()` directly             | SDK client over HTTP                        |
| **New concepts**     | Memory, Trustcall, agent architecture | Deployment, SDK, assistants, double-texting |

**Module 5 = build the agent. Module 6 = deploy it and interact with it as a service.**

## Deployment Architecture

```
┌─────────────────────────────────────────┐
│           docker-compose                │
│                                         │
│  ┌──────────┐  ┌──────────┐  ┌───────┐  │
│  │ LangGraph│  │ Postgres │  │ Redis │  │
│  │  Server  │◄─┤ (state + │  │(stream│  │
│  │ (API)    │  │  store)  │  │  pub/ │  │
│  │ :8123    │  │          │  │  sub) │  │
│  └────┬─────┘  └──────────┘  └───────┘  │
└───────┼─────────────────────────────────┘
        │ HTTP/REST
        ▼
  SDK Client / Custom UI / LangSmith Studio
```

The deployment consists of three containers:

- **langgraph-api** — Serves the graph over HTTP (built via `langgraph build`)
- **langgraph-postgres** — Persists threads, checkpoints, store items, and assistants
- **langgraph-redis** — Enables streaming via pub/sub during graph execution

## What Each Notebook Covers

### 1. `creating.ipynb` — Building the Deployment

Package the graph into a Docker image and launch it:

```bash
cd module-6/deployment
langgraph build -t my-image
docker compose --env-file <PATH> up
```

This gives you a running LangGraph Server with REST API at `http://localhost:8123`.

### 2. `connecting.ipynb` — SDK and Remote Interaction

Connect to the deployed graph using the LangGraph SDK:

```python
from langgraph_sdk import get_client
client = get_client(url="http://localhost:8123")
```

Key APIs exposed by the server:

- **Runs** — Execute the graph (background, blocking, or streaming)
- **Threads** — Multi-turn conversations with persistent checkpoints
- **Store** — Direct CRUD on long-term memory (read/write todos, profiles, etc.)

Features that only exist with a deployed server:

- **Background runs** — Fire-and-forget execution
- **Streaming over HTTP** — Token-by-token responses via Redis pub/sub
- **Thread forking** — Copy a thread, edit state at any checkpoint, resume from there
- **Direct Store access** — Read/write store items without invoking the graph

### 3. `assistant.ipynb` — Versioned Configurations

Create multiple **Assistants** from the same graph with different configurations:

```python
# Personal todo assistant
personal = await client.assistants.create(
    "task_maistro",
    config={"configurable": {"todo_category": "personal", "user_id": "lance"}}
)

# Work todo assistant (different role prompt, separate namespace)
work = await client.assistants.create(
    "task_maistro",
    config={"configurable": {"todo_category": "work", "user_id": "lance"}}
)
```

Same graph, different behaviors and isolated data — all managed and versioned in Postgres.

### 4. `double-texting.ipynb` — Concurrent Message Handling

Four strategies for when a user sends a new message before the previous run completes:

| Strategy      | Behavior                                                  |
| ------------- | --------------------------------------------------------- |
| **Reject**    | Returns 409 error, previous run continues                 |
| **Enqueue**   | Queues the new run, executes after current finishes       |
| **Interrupt** | Stops current run (preserving progress), starts new one   |
| **Rollback**  | Deletes current run entirely, starts fresh with new input |

## Quick Start

```bash
# 1. Build the Docker image
cd module-6/deployment
langgraph build -t my-image

# 2. Launch (set env vars in docker-compose.yml or .env)
docker compose --env-file <PATH> up

# 3. Access
# API docs: http://localhost:8123/docs
# Studio:   https://smith.langchain.com/studio/?baseUrl=http://127.0.0.1:8123
```
