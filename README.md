# Dockerized Vite + React Setup (macOS, No npm on Host)

This document explains a **fully working Dockerized React + Vite setup** for macOS, using Docker and Docker Compose, **without requiring npm on the host system**. It includes development with **hot reload** and production build served via Nginx.

---

## Table of Contents

1. [Project Structure](#project-structure)  
2. [React + Vite Configuration](#react--vite-configuration)  
   - `package.json`  
   - `vite.config.js`  
   - `index.html`  
   - `src` files  
3. [Docker Configuration](#docker-configuration)  
   - Dockerfile  
   - docker-compose.yml  
   - Nginx configuration  
4. [Running the Project](#running-the-project)  
5. [Explanation of Each Configuration](#explanation-of-each-configuration)  
   - Dockerfile stages  
   - Vite server configuration  
   - Docker Compose profiles  
   - Hot reload configuration  
   - Nginx in production  
6. [Troubleshooting](#troubleshooting)

---

## Project Structure

```text
simple-react-app-vite-docker/
├── Dockerfile
├── docker-compose.yml
├── index.html
├── package.json
├── vite.config.js
├── src/
│ ├── main.jsx
│ └── App.jsx
└── docker/
└── nginx.conf
```

---

## React + Vite Configuration

### 1. `package.json`

```json
{
  "name": "simple-react-app-vite-docker",
  "private": true,
  "version": "1.0.0",
  "type": "module",
  "scripts": {
    "dev": "vite --host 0.0.0.0",
    "build": "vite build"
  },
  "dependencies": {
    "react": "^18.2.0",
    "react-dom": "^18.2.0"
  },
  "devDependencies": {
    "@vitejs/plugin-react": "^4.2.0",
    "vite": "^5.4.0"
  }
}
```
## Purpose / Explanation:

- dev script: Starts Vite dev server inside Docker. --host 0.0.0.0 ensures Vite listens on all interfaces so Docker can forward ports to macOS.
- build script: Produces static files in /dist for production.
- Dependencies / devDependencies:
- react & react-dom → core React libraries
- vite → fast frontend build tool
- @vitejs/plugin-react → JSX/React support in Vite


### 2. `vite.config.js`

```js
import { defineConfig } from "vite";
import react from "@vitejs/plugin-react";

export default defineConfig({
  plugins: [react()],
  server: {
    host: "0.0.0.0",       // Bind to all interfaces for Docker
    port: 5173,            // Fixed port for consistency
    strictPort: true,      // Fail if port is already in use
    watch: {
      usePolling: true     // Required on macOS Docker for reliable file change detection
    }
  }
});
```

## Purpose / Explanation:
- host: "0.0.0.0" → ensures container listens on all network interfaces.
- port: 5173 → consistent port mapping to host.
- strictPort: true → avoids port conflicts silently starting on another port.
- watch.usePolling: true → macOS + Docker Desktop cannot reliably use native filesystem events, polling ensures HMR works.

### 3. `index.html`

```html
<!DOCTYPE html>
<html lang="en">
  <head>
    <meta charset="UTF-8" />
    <meta name="viewport" content="width=device-width, initial-scale=1.0" />
    <title>Vite React Docker</title>
  </head>
  <body>
    <div id="root"></div>
    <script type="module" src="/src/main.jsx"></script>
  </body>
</html>
```

## Explanation:
- Entry HTML file served in development and production.

```html
<div id="root"> → React mounts here.

<script type="module" src="/src/main.jsx"> → Vite will bundle React app from this entry point.
```

### 4. `src/main.jsx`

```jsx
import React from "react";
import ReactDOM from "react-dom/client";
import App from "./App";

ReactDOM.createRoot(document.getElementById("root")).render(
  <React.StrictMode>
    <App />
  </React.StrictMode>
);
```
## Explanation:
- Entry point for React application.
- Uses ReactDOM.createRoot for React 18 concurrent mode.


### 5. `src/App.jsx`

```jsx
export default function App() {
  return <h1>Docker + Vite + React (Working)</h1>;
}

```
## Explanation:

- Simple React component to confirm hot reload and build works.
- Can be expanded to full app later.

## Docker Configuration

### 1. `Dockerfile`

```dockerfile
# ---------- Base ----------
FROM node:20-alpine AS base
WORKDIR /app

# ---------- Dependencies ----------
FROM base AS deps
COPY package.json ./
RUN npm install

# ---------- Development ----------
FROM deps AS dev
COPY . .
EXPOSE 5173
CMD ["npm", "run", "dev"]

# ---------- Build ----------
FROM deps AS build
COPY . .
RUN npm run build

# ---------- Production ----------
FROM nginx:alpine AS prod
COPY docker/nginx.conf /etc/nginx/conf.d/default.conf
COPY --from=build /app/dist /usr/share/nginx/html
EXPOSE 80
CMD ["nginx", "-g", "daemon off;"]
```
## Explanation:
- Multi-stage build: separates dev, build, and prod.
- base → sets working directory and Node image.
- deps → installs dependencies once.
- dev → development server with HMR.
- build → produces optimized /dist.
- prod → Nginx serves static files; small, fast image.


### 2. `docker-compose.yml`

```yaml
version: "3.9"

services:
  frontend-dev:
    container_name: vite-react-dev
    build:
      context: .
      target: dev
    profiles: ["dev"]
    ports:
      - "5173:5173"
    volumes:
      - .:/app
      - /app/node_modules
    environment:
      - CHOKIDAR_USEPOLLING=true

  frontend-prod:
    container_name: vite-react-prod
    build:
      context: .
      target: prod
    profiles: ["prod"]
    ports:
      - "8080:80"
```
## Explanation:
- Profiles: dev and prod allow docker compose --profile dev or --profile prod.
- Volumes in dev: mount local files into container for HMR.
- CHOKIDAR_USEPOLLING → ensures live reload works on macOS.
- ports → maps container ports to host machine.

### 3. `docker/nginx.conf`  (Production Only)

```nginx
server {
  listen 80;
  server_name localhost;

  root /usr/share/nginx/html;
  index index.html;

  location / {
    try_files $uri /index.html;
  }
}
```

## Explanation:
- Nginx serves built static files from /usr/share/nginx/html.
- try_files $uri /index.html; → required for React Router; ensures deep links work.

## Running the Project
## Development (with Hot Reload)
```bash
docker compose --profile dev down -v
docker compose --profile dev build --no-cache
docker compose --profile dev up
```
- Open: http://localhost:5173
- Edit src/App.jsx → browser updates automatically
- No npm needed on host

## Production (Nginx)

```bash
docker compose --profile prod up --build
```
- Open: http://localhost:8080
- Static, optimized build
- No dev dependencies

## Explanation of each configuration

| Config                           | Purpose                                                                         |
| -------------------------------- | ------------------------------------------------------------------------------- |
| `host: "0.0.0.0"`                | Ensures Vite listens on all container interfaces so Docker can forward ports.   |
| `usePolling`                     | Fixes unreliable file watching on macOS Docker volumes.                         |
| `--host 0.0.0.0` in package.json | Redundant safety to force host binding for dev container.                       |
| Multi-stage Dockerfile           | Separates dev (Node, live reload), build (optimized), prod (small Nginx image). |
| Docker Compose profiles          | Switch between dev and prod easily.                                             |
| Volumes in dev                   | Allow live editing without rebuilding image.                                    |
| Nginx                            | Serves optimized static assets; handles deep links for React Router.            |


## Troubleshooting
- Cannot access localhost:5173
- Check docker compose --profile dev ps → container should be running.
- Check vite.config.js → host: "0.0.0.0".
- Rebuild with --no-cache.
- Hot reload not working
- Ensure CHOKIDAR_USEPOLLING=true is set.
- Ensure volumes are correctly mounted.
- Port conflicts
- Ensure no other process uses 5173 (dev) or 8080 (prod).
- Production page blank
- Ensure index.html exists in project root.
- Check Nginx config for try_files $uri /index.html;

---
