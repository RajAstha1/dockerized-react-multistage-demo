# dockerized-react-multistage-demo

A comprehensive guide to Dockerizing a React application using multi-stage builds for optimized production images.

## Objective
Dockerize a React application using a multi-stage Docker build so that:
- The image is small and secure (only Nginx + static assets at runtime)
- The build toolchain (Node, npm/yarn) is kept in the builder stage
- Cached layers speed up iterative builds

## Dockerfile (Multi-stage: Node build -> Nginx serve)
```Dockerfile
# ---------- 1) Builder stage: use Node to install deps and build ----------
FROM node:20-alpine AS builder

# Set working directory
WORKDIR /app

# Install dependencies first (leverage Docker layer caching)
COPY package*.json ./
# If you use yarn or pnpm, copy respective lockfiles and adjust install cmd
# COPY yarn.lock ./
# COPY pnpm-lock.yaml ./

RUN npm ci --no-audit --no-fund

# Copy app source and build
COPY . .
RUN npm run build

# ---------- 2) Runtime stage: serve static build with Nginx ----------
FROM nginx:1.27-alpine AS runtime

# Remove default nginx static assets and copy React build
RUN rm -rf /usr/share/nginx/html/*
COPY --from=builder /app/build /usr/share/nginx/html

# Copy a basic nginx config (optional). If not provided, default works fine.
# Uncomment the next two lines if you add nginx.conf alongside Dockerfile
# COPY nginx.conf /etc/nginx/conf.d/default.conf

# Expose port and run nginx
EXPOSE 80
CMD ["nginx", "-g", "daemon off;"]
```

### Stage explanations
- Builder (node:20-alpine):
  - Installs dependencies using npm ci for clean, reproducible installs
  - Builds the production React app to /app/build
- Runtime (nginx:1.27-alpine):
  - Uses a minimal Nginx image to serve static files
  - Copies only the build output, keeping the final image small

## Project structure (example)
```
.
├─ src/
│  └─ index.js
├─ public/
│  └─ index.html
├─ package.json
├─ Dockerfile
└─ README.md
```

## Sample React entrypoint (src/index.js)
```js
import React from 'react';
import { createRoot } from 'react-dom/client';

function App() {
  return (
    <div style={{ fontFamily: 'sans-serif', padding: 24 }}>
      <h1>Dockerized React Multi-stage Build</h1>
      <p>Served by Nginx, built with Node in a separate stage.</p>
    </div>
  );
}

const root = createRoot(document.getElementById('root'));
root.render(<App />);
```

## Setup and usage

### 1) Prerequisites
- Docker Desktop or Docker Engine 20+
- A React app (Create React App, Vite, Next static export, etc.)

### 2) Build the image
```
docker build -t dockerized-react-multistage-demo .
```

Optional: use build cache mounts for faster installs (Docker BuildKit):
```
DOCKER_BUILDKIT=1 docker build -t dockerized-react-multistage-demo .
```

### 3) Run the container
```
docker run --rm -it -p 8080:80 dockerized-react-multistage-demo
```
Open http://localhost:8080

### 4) Common variations
- Using yarn:
  - Replace `npm ci` with `yarn install --frozen-lockfile`
  - Ensure yarn.lock is copied before install
- Environment variables at build time:
  - `docker build --build-arg REACT_APP_API_URL=https://api.example.com -t app .`
  - And reference process.env.REACT_APP_API_URL inside your app for CRA
- Custom Nginx config for SPA routing:
  - Add nginx.conf with a catch-all to index.html to support client-side routing

Example nginx.conf (optional):
```nginx
server {
  listen 80;
  server_name _;
  root /usr/share/nginx/html;

  location / {
    try_files $uri /index.html;
  }

  location /health {
    return 200 'ok';
    add_header Content-Type text/plain;
  }
}
```

## Extra: Useful Docker commands
- See image size and layers: `docker history dockerized-react-multistage-demo`
- Remove dangling images: `docker image prune`
- Tail logs: `docker logs -f <container>`
- Exec into container: `docker exec -it <container> sh`

## Resources
- Docker official docs: https://docs.docker.com/
- Multi-stage builds: https://docs.docker.com/build/building/multi-stage/
- Nginx image docs: https://hub.docker.com/_/nginx
- Node image docs: https://hub.docker.com/_/node
- Create React App docs: https://create-react-app.dev/
``
