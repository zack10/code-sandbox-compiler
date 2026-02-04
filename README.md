# Code Sandbox Compiler

A comprehensive code execution and compilation platform that provides secure code execution and multi-framework frontend compilation capabilities. This project combines **Judge0** for backend code execution and a **multi-framework compiler service** for frontend builds.

## Overview

This project consists of two main components:

1. **Judge0** - A secure online code execution system that supports 70+ programming languages
2. **Front Compiler Service** - A multi-framework compiler that builds Angular, React, and Vue applications

Together, these services enable you to:
- Execute code in multiple programming languages securely
- Compile frontend frameworks (Angular, React, Vue) from source code
- Build online coding platforms, technical interview systems, or learning environments

## Project Structure

```
code-sandbox-compiler/
├── .github/
│   └── workflows/
│       └── main.yml              # CI/CD pipeline to build & push images to Docker Hub
│
├── front/                        # Muti-framework Compiler Service
│   ├── compiler-service/         # Node.js Express API
│   │   ├── server.js             # Main API that handles /compile and orchestrates Docker builds
│   │   ├── package.json          # Dependencies (dockerode, express, etc.)
│   │   └── README.md             # Documentation for the API endpoints
│   │
│   ├── Dockerfile.compiler-service # Builds the API service container
│   ├── Dockerfile.angular-compiler # Components necessary to build Angular apps
│   ├── Dockerfile.react-compiler   # Components necessary to build React apps
│   ├── Dockerfile.vue-compiler     # Components necessary to build Vue apps
│   ├── docker-compose.yml          # Defines the compiler service and builder profiles
│   └── README.md                   # Documentation specific to the front compiler
│
├── judge0/                       # Judge0 Backend (Code Execution)
│   ├── docker-compose.yml        # Official Judge0 configuration (Server, Worker, DB, Redis)
│   └── README.md                 # Setup guide (URGENT: Read this for Ubuntu Kernel fixes)
│
├── docker-compose.yml            # ROOT Orchestrator: Starts EVERYTHING (Judge0 + Compiler)
└── README.md                     # Main documentation
```

## Features

### Judge0 (Backend Code Execution)
- ✅ **70+ Programming Languages** - C, C++, Java, Python, JavaScript, TypeScript, Go, Rust, PHP, Ruby, C#, Kotlin, Swift, Bash, and more
- ✅ **Secure Sandboxing** - Uses Docker + Isolate sandbox for complete isolation
- ✅ **Resource Limits** - CPU, memory, and process limits enforced via cgroups
- ✅ **RESTful API** - Simple HTTP API for code submissions and results
- ✅ **Production Ready** - Includes PostgreSQL and Redis for persistence and queuing

### Front Compiler Service
- ✅ **Multi-Framework Support** - Angular, React, and Vue compilation
- ✅ **Isolated Builds** - Each compilation runs in its own Docker container
- ✅ **Cached Builds** - Host-mounted caches for faster subsequent compiles
- ✅ **Single API Endpoint** - Unified `/compile` endpoint for all frameworks
- ✅ **Source Maps** - Includes source maps for debugging
- ✅ **Health Monitoring** - Built-in health check endpoint

## Quick Start

### Prerequisites

- **Docker** and **Docker Compose** installed
- **Ubuntu Server 24.04.3 LTS** (Recommended for Server Deployment)
- At least **4 GB RAM** recommended
- **20 GB free disk space** minimum

### Running both Judge0 and Compiler Service (unified)

From the **project root**:

```bash
# Start all services (Judge0 + Compiler Service)
docker compose up --build
```

- **Judge0 API:** http://localhost:2358  
- **Compiler API:** http://localhost:3001  

Stop everything:

```bash
docker compose down
```

See **judge0/README.md** for kernel/cgroup setup on Ubuntu before running Judge0.

### Running Judge0 only

1. Navigate to the judge0 directory:
   ```bash
   cd judge0
   ```

2. Follow the detailed installation guide:
   ```bash
   cat README.md
   ```

3. Key steps:
   - Fix kernel cgroup configuration (critical for Judge0)
   - Install Docker and Docker Compose
   - Deploy using docker-compose

4. Start Judge0:
   ```bash
   docker-compose up -d
   ```

5. Test the service:
   ```bash
   curl http://localhost:2358/languages
   ```

**Note:** See `judge0/README.md` for complete installation instructions, including the critical cgroup v2 fix.

### Running Front Compiler Service

1. Navigate to the front directory:
   ```bash
   cd front
   ```

2. Build all compiler images:
   ```bash
   docker compose build
   ```

3. Start the compiler service:
   ```bash
   docker compose up -d compiler-service
   ```

4. Verify it's running:
   ```bash
   curl http://localhost:3001/health
   ```

5. Compile a sample Angular component:
   ```bash
   curl -X POST http://localhost:3001/compile \
     -H "Content-Type: application/json" \
     -d '{
       "code": "import { Component } from \"@angular/core\"; @Component({ selector: \"app-root\", template: \"<h1>Hello World</h1>\" }) export class AppComponent {}",
       "framework": "angular"
     }'
   ```

**Note:** See `front/README.md` for detailed API documentation and framework-specific behavior.

## API Endpoints

### Judge0 API

- **Base URL:** `http://localhost:2358`
- **List Languages:** `GET /languages`
- **Submit Code:** `POST /submissions?wait=true`
- **Get Submission:** `GET /submissions/{token}`

### Front Compiler API

- **Base URL:** `http://localhost:3001`
- **Health Check:** `GET /health`
- **Compile Code:** `POST /compile`

## Architecture

### Judge0 Architecture
```
┌─────────────┐
│   Client    │
└──────┬──────┘
       │ HTTP/REST
       ▼
┌─────────────┐     ┌─────────────┐     ┌─────────────┐
│   Server    │────▶│   Worker    │────▶│   Isolate   │
│  (Port 2358)│     │  (Processes)│     │  (Sandbox)  │
└──────┬──────┘     └──────┬──────┘     └─────────────┘
       │                   │
       ▼                   ▼
┌─────────────┐     ┌─────────────┐
│ PostgreSQL  │     │    Redis    │
└─────────────┘     └─────────────┘
```

### Front Compiler Architecture
```
┌─────────────┐
│   Client    │
└──────┬──────┘
       │ HTTP POST /compile
       ▼
┌──────────────────┐
│ Compiler Service │
│   (Express API)  │
└──────┬───────────┘
       │ Docker API
       ▼
┌─────────────────────────────────┐
│  Framework-Specific Containers  │
│  ┌──────────┐ ┌──────────┐      │
│  │ Angular  │ │  React   │      │
│  └──────────┘ └──────────┘      │
│  ┌──────────┐                   │
│  │   Vue    │                   │
│  └──────────┘                   │
└─────────────────────────────────┘
```

## Security Considerations

### Judge0 Security
- Code executes in isolated Docker containers
- Isolate sandbox enforces strict resource limits
- cgroups v1 controls CPU, memory, and process limits
- No access to host system or other executions
- File system restrictions prevent unauthorized access

### Front Compiler Security
- Each compilation runs in a separate container
- Containers are ephemeral (created and destroyed per request)
- No persistent storage between compilations
- Docker socket access required (consider security implications)

## Development

### Judge0 Development
- See `judge0/README.md` for development setup
- Judge0 runs as a Docker service
- Configuration via environment variables in `docker-compose.yml`

### Front Compiler Development
- Main server: `front/compiler-service/server.js`
- Port configurable via `PORT` environment variable (default: 3001)
- Local development requires Node.js and Docker access
- Recommended: Use Docker Compose for consistent runtime

## Troubleshooting

### Judge0 Issues
- **Containers restarting:** Check cgroup configuration (see `judge0/README.md`)
- **Port 2358 unreachable:** Check firewall rules
- **Database errors:** Wait 30 seconds after first start, then retry

### Front Compiler Issues
- **Build failures:** Check Docker logs: `docker compose logs compiler-service`
- **Timeout errors:** Increase timeout in request or server configuration
- **Cache issues:** Clear mounted cache directories if needed

## Documentation

- **[Judge0 Installation Guide](judge0/README.md)** - Complete setup instructions