# YAYO Frontend

Standalone Vue frontend project that can connect to both the YAYO Python and YAYO Java versions.

Project directory:

```text
/Users/xiao_xiong/Desktop/code/YAYOFrontend
```

## Features

- Switch between Java and Python backends in the UI.
- Normalize `/chat` response fields:
  - Python: `conv_id`, `agent_type`, `latency_ms`
  - Java: `conversation_id`, `agent_type`, `latency_ms`
- Supports chat debugging, health checks, monitoring summaries, knowledge base search, knowledge document import, and file uploads.
- Supports Docker + Nginx deployment.

## Default Backend URLs

| Backend | Default URL |
|------|----------|
| Python | `http://localhost:8000` |
| Java | `http://localhost:8080` |

In development mode, Vite proxies:

| Frontend Path | Proxies To |
|----------|--------|
| `/api/python` | `http://localhost:8000` |
| `/api/java` | `http://localhost:8080` |

In Docker mode, Nginx accesses the Python and Java services on the host machine through `host.docker.internal`.

## Local Development

Install dependencies:

```bash
npm install
```

Start the dev server:

```bash
npm run dev
```

Open:

```text
http://localhost:5173
```

If the backend ports are not using the defaults, override them when starting:

```bash
VITE_PYTHON_API_URL=http://localhost:8000 \
VITE_JAVA_API_URL=http://localhost:8080 \
npm run dev
```

## Docker Deployment

Build the frontend static files:

```bash
npm run build
```

Build and start the container:

```bash
docker compose up -d --build
```

Open:

```text
http://localhost:5174
```

Stop:

```bash
docker compose down
```

## Backend Startup Reference

Python default:

```text
http://localhost:8000
```

Java default:

```text
http://localhost:8080
```

The two backends do not need to run at the same time. Select the backend you want to debug from the frontend page.
