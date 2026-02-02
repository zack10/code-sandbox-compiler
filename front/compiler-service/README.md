# Multi-Framework Compiler Service

A Docker-based microservice that compiles Angular, React, and Vue applications in isolated containers with configurable resource limits and timeouts.

## Overview

This service provides a REST API for compiling frontend framework code (Angular, React, Vue) in isolated Docker environments. Each compilation runs in a fresh container with configurable CPU and memory limits, making it safe for untrusted code execution.

## Features

- **Multi-Framework Support**: Compiles Angular, React, and Vue applications
- **Isolated Execution**: Each compilation runs in a separate Docker container
- **Resource Management**: Configurable CPU and memory limits per compilation
- **Build Caching**: Persistent caching for faster subsequent builds
- **Timeout Controls**: Configurable compilation timeouts (default: 60 seconds)
- **File Extraction**: Returns compiled JavaScript, CSS, HTML, and source maps
- **Runtime Configuration**: Update resource limits without restarting the service

## Architecture

### How It Works

1. **Request**: Client sends source code via POST to `/compile`
2. **Container Creation**: Service creates a Docker container with the appropriate framework image
3. **Code Injection**: Source code is base64-encoded and written to the container
4. **Compilation**: Framework-specific build command executes inside the container
5. **Extraction**: Compiled files are extracted from the container's dist directory
6. **Cleanup**: Container is removed after compilation (success or failure)

### Framework Configurations

Each framework has a specific configuration:

```javascript
FRAMEWORKS = {
  angular: {
    image: 'angular-compiler:latest',
    filePath: 'src/app/app.ts',
    distPath: '/workspace/template-app/dist/template-app',
    buildCmd: 'ng build --configuration production...'
  },
  react: {
    image: 'react-compiler:latest',
    filePath: 'src/App.tsx',
    distPath: '/workspace/template-app/dist',
    buildCmd: 'npm run build'
  },
  vue: {
    image: 'vue-compiler:latest',
    filePath: 'src/App.vue',
    distPath: '/workspace/template-app/dist',
    buildCmd: 'npx vite build'
  }
}
```

## API Endpoints

### Health Check

```http
GET /health
```

**Response:**
```json
{
  "status": "ok",
  "service": "multi-compiler"
}
```

### Get Configuration

```http
GET /config
```

**Response:**
```json
{
  "dockerLimits": {
    "MEMORY": 2147483648,
    "CPU_PERIOD": 100000,
    "CPU_QUOTA": 200000
  },
  "defaultCompileTimeoutMs": 60000,
  "note": "Use PUT /config to update these values at runtime"
}
```

### Update Configuration

```http
PUT /config
Content-Type: application/json

{
  "memory": 3221225472,
  "cpuPeriod": 100000,
  "cpuQuota": 300000,
  "timeout": 90000
}
```

**Parameters:**
- `memory` (optional): Memory limit in bytes
- `cpuPeriod` (optional): CPU CFS scheduler period in microseconds
- `cpuQuota` (optional): CPU CFS quota in microseconds
- `timeout` (optional): Default compilation timeout in milliseconds

**Response:**
```json
{
  "success": true,
  "message": "Configuration updated",
  "dockerLimits": { ... },
  "defaultCompileTimeoutMs": 90000
}
```

### Compile Code

```http
POST /compile
Content-Type: application/json

{
  "code": "import { Component } from '@angular/core'...",
  "framework": "angular",
  "timeout": 120000,
  "memory": 3221225472,
  "cpuPeriod": 100000,
  "cpuQuota": 250000
}
```

**Parameters:**
- `code` (required): Source code to compile
- `framework` (optional): `"angular"`, `"react"`, or `"vue"` (default: `"angular"`)
- `timeout` (optional): Compilation timeout in milliseconds (overrides global default)
- `memory` (optional): Memory limit in bytes (overrides global default)
- `cpuPeriod` (optional): CPU period (overrides global default)
- `cpuQuota` (optional): CPU quota (overrides global default)

**Success Response (200):**
```json
{
  "success": true,
  "framework": "angular",
  "files": {
    "main.js": "...",
    "styles.css": "...",
    "index.html": "...",
    "main.js.map": "..."
  },
  "compilationTime": 3542
}
```

**Error Response (400):**
```json
{
  "success": false,
  "error": "ANGULAR Build Failed",
  "logs": "Error: Module not found..."
}
```

## Framework-Specific Details

### Angular

- **Source File**: `src/app/app.ts`
- **Build Command**: `ng build --configuration production --output-hashing none --optimization true --source-map true --progress false`
- **Preprocessing**:
  - Automatically imports `CommonModule` if not present
  - Forces `selector: 'app-root'`
  - Renames class to `AppComponent`
  - Adds `standalone: true` if missing
  - Strips external `templateUrl`, `styleUrls`, and `styleUrl` references

### React

- **Source File**: `src/App.tsx`
- **Build Tool**: Vite
- **Build Command**: `npm run build`
- **Cleanup**: Removes conflicting `.css` and `App.tsx` files before compilation

### Vue

- **Source File**: `src/App.vue`
- **Build Tool**: Vite
- **Build Command**: `npx vite build`
- **Cleanup**: Removes conflicting `.vue` files in `src/components/` before compilation

## Resource Limits

### Default Configuration

- **Memory**: 2 GiB (2,147,483,648 bytes)
- **CPU Period**: 100,000 microseconds (100ms)
- **CPU Quota**: 200,000 microseconds (2 CPU cores worth)
- **Timeout**: 60,000 milliseconds (60 seconds)

### Environment Variables

Set these when starting the service:

```bash
COMPILER_MEMORY_BYTES=3221225472
COMPILER_CPU_PERIOD=100000
COMPILER_CPU_QUOTA=300000
COMPILER_TIMEOUT_MS=90000
PORT=3001
```

### Per-Request Overrides

You can override limits for individual compilations by including them in the request body:

```javascript
{
  "code": "...",
  "framework": "vue",
  "timeout": 120000,      // 2 minutes for this compilation
  "memory": 4294967296,   // 4 GiB for this compilation
  "cpuQuota": 400000      // 4 CPU cores for this compilation
}
```

## Build Caching

To improve compilation speed, each framework uses persistent caching:

- **Angular**: `/tmp/angular-cache` → `/workspace/template-app/.angular/cache`
- **React**: `/tmp/react-cache` → `/workspace/template-app/node_modules/.vite`
- **Vue**: `/tmp/vue-cache` → `/workspace/template-app/node_modules/.vite`

These cache directories are mounted as volumes to persist across container instances.

## File Extraction

After successful compilation, the service extracts files from the container's dist directory:

- **Supported Extensions**: `.js`, `.css`, `.html`, `.map`
- **Extraction Method**: TAR stream from Docker container
- **Output Format**: Object with filename keys and file content values

Only actual files (not directories) matching the supported extensions are extracted.

## Error Handling

The service handles various error conditions:

- **Invalid Framework**: Returns 400 with unsupported framework message
- **Missing Code**: Returns 400 if code field is missing or not a string
- **Invalid Resource Limits**: Returns 400 if limits are not positive numbers
- **Compilation Timeout**: Returns 400 after timeout expires
- **Build Failures**: Returns 400 with build logs
- **Container Errors**: Returns 500 with error message

All containers are forcefully removed in the `finally` block, even if errors occur.

## Installation & Setup

### Prerequisites

- Node.js 14+
- Docker installed and running
- Docker images built for each framework:
  - `angular-compiler:latest`
  - `react-compiler:latest`
  - `vue-compiler:latest`

### Install Dependencies

```bash
npm install
```

### Start the Service

```bash
node server.js
```

Or with custom configuration:

```bash
COMPILER_MEMORY_BYTES=3221225472 \
COMPILER_TIMEOUT_MS=90000 \
PORT=3001 \
node server.js
```

## Security Considerations

1. **Isolated Execution**: Each compilation runs in a separate container
2. **Resource Limits**: Prevents resource exhaustion attacks
3. **Timeout Controls**: Prevents infinite loops or hanging builds
4. **Container Cleanup**: All containers are removed after use
5. **Request Size Limits**: JSON payload limited to 10MB
6. **No Persistent State**: Containers are ephemeral and don't share state

## Example Usage

### Compile Angular Component

```bash
curl -X POST http://localhost:3001/compile \
  -H "Content-Type: application/json" \
  -d '{
    "code": "import { Component } from \"@angular/core\";\n\n@Component({\n  selector: \"app-root\",\n  template: \"<h1>Hello World</h1>\"\n})\nexport class AppComponent {}",
    "framework": "angular"
  }'
```

### Compile React Component

```bash
curl -X POST http://localhost:3001/compile \
  -H "Content-Type: application/json" \
  -d '{
    "code": "export default function App() {\n  return <h1>Hello World</h1>;\n}",
    "framework": "react",
    "timeout": 90000
  }'
```

### Compile Vue Component

```bash
curl -X POST http://localhost:3001/compile \
  -H "Content-Type: application/json" \
  -d '{
    "code": "<template>\n  <h1>Hello World</h1>\n</template>\n\n<script setup>\n</script>",
    "framework": "vue"
  }'
```

## Performance

- **First Compilation**: Slower due to cache warming (10-30 seconds)
- **Subsequent Compilations**: Faster with warm cache (2-10 seconds)
- **Concurrent Compilations**: Supported (limited by system resources)

## Logging

The service logs key events:

- Compilation start with unique ID
- Container creation
- Compilation completion with duration
- File extraction count
- Errors and cleanup issues

Example log output:
```
[a1b2c3d4] Starting angular compilation...
[a1b2c3d4] Container created (angular)
Extracted 4 files from /workspace/template-app/dist/template-app
[a1b2c3d4] ✓ Compiled in 3542ms
```
