# 📦 Stage 2 — Dockerfiles & Image Building

---

## 🧠 Overview

This is where Docker becomes practical.

Everything you ship starts with a **Dockerfile** — the blueprint for your container.

---

## 📌 What is a Dockerfile?

A **Dockerfile** is a plain text file containing instructions to build a Docker image.

- Executed **top → bottom**
- Each instruction creates a **layer**
- Layers are **cached** → makes builds fast

---

## 🔄 Build Flow

![dockerfile_to_container_flow](https://github.com/user-attachments/assets/56317ab9-7b0a-426a-8b45-224a2bd9b5e1)

## Dockerfile → docker build → Image → docker run → Container

## 🧱 Example App (Node.js)

### `app.js`
```js
const express = require('express');
const app = express();

app.get('/', (req, res) => res.send('Hello from Docker! 🐳'));
app.get('/health', (req, res) => res.json({ status: 'ok' }));

app.listen(3000, () => console.log('Server running on port 3000'));
```
### `package.json`
```json
{
  "name": "my-app",
  "version": "1.0.0",
  "dependencies": {
    "express": "^4.18.2"
  }
}
```
### `Dockerfile`
```Dockerfile
# FROM — every Dockerfile starts here. This is your base image.
# You're building ON TOP of an existing image (Node.js 20 on Alpine Linux).
# Alpine is a 5MB minimal Linux distro — much smaller than ubuntu (77MB).
FROM node:20-alpine

# LABEL — optional metadata about your image (author, version, etc.)
LABEL maintainer="you@example.com"

# WORKDIR — sets the working directory inside the container.
# All subsequent commands run from here. Creates the dir if it doesn't exist.
WORKDIR /app

# COPY — copies files from your machine INTO the image.
# Syntax: COPY <source-on-host> <destination-in-image>
# We copy package files FIRST (before app code) — this is critical for caching.
COPY package*.json ./

# RUN — executes a shell command during the BUILD phase (not at runtime).
# This installs dependencies. --omit=dev skips devDependencies in production.
RUN npm install --omit=dev

# COPY the rest of your app code AFTER installing dependencies.
# This ordering makes layer caching work properly (explained below).
COPY . .

# EXPOSE — documents which port the app listens on.
# This is documentation only — it does NOT actually publish the port.
# You still need -p flag in docker run to publish it.
EXPOSE 3000

# CMD — the default command to run when the container starts.
# Use JSON array format (exec form) — NOT shell string form.
# Exec form: ["node", "app.js"]    ← correct, PID 1, handles signals properly
# Shell form: node app.js          ← wraps in /bin/sh, signal handling breaks
CMD ["node", "app.js"]
```
### `Build and run it`
```bash
# Build: -t tags the image with a name, . means "use current directory"
docker build -t my-app:1.0 .

# Run: map port 3000 on host to 3000 in container
docker run -d -p 3000:3000 --name my-app my-app:1.0

# Test it
curl http://localhost:3000
curl http://localhost:3000/health

# See the logs
docker logs my-app
```
### `The layer system- Most important in image building`
```
Every instruction in a Dockerfile creates a layer. Layers are cached.

This is what makes Docker fast in CI/CD — if a layer hasn't changed, Docker reuses the cached version and skips rebuilding it.
```
![docker_layer_caching](https://github.com/user-attachments/assets/c87380fa-88dd-49a2-8619-a6794358c5dc)

### `See caching in action`
```bash
docker build -t my-app:1.0 .   # first build — all layers run
docker build -t my-app:1.0 .   # second build — watch "CACHED" appear on every layer

# Now change app.js (edit the response string), then rebuild:
docker build -t my-app:1.0 .   # only the COPY . . and CMD layers re-run
                                # npm install is still cached — instant!
```

### `The .dockerignore file (always create this) `
```
Just like .gitignore, this tells Docker what NOT to copy into the image. Without it, you're shipping node_modules (hundreds of MB), .git history, local .env files, and test files into your image.
Create .dockerignore in the same directory as your Dockerfile:
```
```bash
node_modules
.git
.gitignore
*.md
.env
.env.*
coverage
.nyc_output
*.log
dist
.DS_Store
```

### key Dockerfile instruction reference

| Instruction  | When it runs      | Purpose |
|-------------|-------------------|--------|
| FROM        | build             | base image to start from |
| RUN         | build             | execute commands, install packages |
| COPY        | build             | copy files from host into image |
| ADD         | build             | like COPY but also handles URLs and tar extraction |
| WORKDIR     | build             | set working directory |
| ENV         | build + runtime   | set environment variables |
| ARG         | build only        | build-time variables (not in final image) |
| EXPOSE      | documentation     | declare port (doesn't publish it) |
| CMD         | runtime           | default command when container starts |
| ENTRYPOINT  | runtime           | fixed command, CMD becomes its arguments |
| HEALTHCHECK | runtime           | tells Docker how to test container health |
| USER        | build + runtime   | switch to non-root user (security!) |


### `CMD vs ENTRYPOINT`
```bash
# CMD alone — fully overridable
CMD ["node", "app.js"]
# docker run my-app              → runs: node app.js
# docker run my-app node other.js → runs: node other.js  (CMD overridden)

# ENTRYPOINT alone — command is fixed
ENTRYPOINT ["node"]
# docker run my-app app.js      → runs: node app.js
# docker run my-app             → runs: node  (no args, probably errors)

# ENTRYPOINT + CMD together — best pattern for production
ENTRYPOINT ["node"]
CMD ["app.js"]
# docker run my-app             → runs: node app.js  (CMD is default arg)
# docker run my-app other.js    → runs: node other.js (CMD overridden, ENTRYPOINT fixed)
```

### `ENG vs ARG - both set variables, very different scopes`
```bash
# ARG — only exists during build. Not in the running container.
ARG NODE_ENV=production
ARG BUILD_VERSION=1.0.0

# ENV — exists during build AND in the running container.
ENV NODE_ENV=production
ENV PORT=3000

# Common pattern: promote ARG to ENV
ARG APP_VERSION
ENV APP_VERSION=$APP_VERSION

Pass ARGs at build time:
docker build --build-arg APP_VERSION=2.1.0 -t my-app:2.1.0 .
```

### `Production-grade Dockerfile`
```Dockerfile
FROM node:20-alpine

# Install security updates
RUN apk update && apk upgrade && apk add --no-cache dumb-init

# Use non-root user (security best practice — Stage 6 goes deeper on this)
RUN addgroup -S appgroup && adduser -S appuser -G appgroup

WORKDIR /app

# Copy and install deps first (cache optimization)
COPY package*.json ./
RUN npm ci --omit=dev && npm cache clean --force

# Copy app source
COPY --chown=appuser:appgroup . .

# Switch to non-root user
USER appuser

EXPOSE 3000

# dumb-init handles signals properly (PID 1 problem)
ENTRYPOINT ["dumb-init", "--"]
CMD ["node", "app.js"]
```

### `inspect your built image`
```bash
docker build -t my-app:prod .

# See all layers and their sizes
docker history my-app:prod

# Full image details
docker inspect my-app:prod

# Check the final image size
docker images my-app
```

### `Practice Assignments for stage:2`
```
1. Write the Dockerfile above from scratch, build it, run it, hit localhost:3000
2. Deliberately put COPY . . before npm install, rebuild twice — see the cache miss
3. Fix the order, rebuild — see the cache hit
4. Add a .dockerignore, check the build context size difference with docker build output
5. Try docker history my-app:prod — see every layer, its size, and the command that made it
6. Change just app.js and rebuild — confirm only the last 2 layers rebuild


