# Stage 4 — Docker Compose
---

Until now we've been running containers with long docker run commands. In a real app we have a frontend, a backend, 
a database, maybe a cache, maybe a message queue — that's 5 separate docker run commands with networks, volumes, env vars,
port mappings all typed manually every time. One typo and something breaks.
Docker Compose solves this. **One YAML file describes your entire application stack**. One command brings it all up. One command tears it all down.

![docker_compose_overview](https://github.com/user-attachments/assets/d0d454e9-bdd5-4579-a290-2a8effdcacae)

###  `The anatomy of a docker-compose.yml`
Let's build a real full-stack compose file from scratch — a Node.js API, PostgreSQL database, 
and Redis cache. Every line explained:

``` yaml
# Version is optional in modern Docker Compose (v2+) but good to know about
# Modern Compose (v2) doesn't need this — it's auto-detected
# version: "3.9"

services:         # Every container you want to run is a "service"

  # ── SERVICE 1: Node.js API ─────────────────────────────────
  api:
    build:
      context: ./api        # path to the folder containing Dockerfile
      dockerfile: Dockerfile # optional if it's named "Dockerfile"
    container_name: myapp-api
    ports:
      - "3000:3000"         # host:container
    environment:            # env vars injected at runtime
      NODE_ENV: production
      PORT: 3000
      DATABASE_URL: postgres://admin:secret@postgres:5432/myapp
      REDIS_URL: redis://redis:6379
    depends_on:             # start order — api waits for these to START
      postgres:             # (not "be ready" — see healthcheck section below)
        condition: service_healthy
      redis:
        condition: service_started
    networks:
      - app-net
    restart: unless-stopped # restart policy (always / on-failure / unless-stopped)
    volumes:
      - ./api/logs:/app/logs  # bind mount for logs

  # ── SERVICE 2: PostgreSQL ──────────────────────────────────
  postgres:
    image: postgres:16-alpine  # use an existing image, no build needed
    container_name: myapp-postgres
    environment:
      POSTGRES_DB: myapp
      POSTGRES_USER: admin
      POSTGRES_PASSWORD: secret
    volumes:
      - pgdata:/var/lib/postgresql/data        # named volume for persistence
      - ./db/init.sql:/docker-entrypoint-initdb.d/init.sql  # auto-run on first start
    ports:
      - "5432:5432"   # expose to host (for local dev tools like TablePlus)
    networks:
      - app-net
    healthcheck:             # Compose waits for THIS before starting dependents
      test: ["CMD-SHELL", "pg_isready -U admin -d myapp"]
      interval: 5s           # check every 5 seconds
      timeout: 5s            # fail if no response in 5s
      retries: 5             # try 5 times before marking unhealthy
      start_period: 10s      # grace period before first check

  # ── SERVICE 3: Redis ──────────────────────────────────────
  redis:
    image: redis:7-alpine
    container_name: myapp-redis
    ports:
      - "6379:6379"
    networks:
      - app-net
    volumes:
      - redisdata:/data
    command: redis-server --appendonly yes   # override default CMD

# ── NETWORKS ──────────────────────────────────────────────
networks:
  app-net:
    driver: bridge    # explicit is better than implicit

# ── VOLUMES ───────────────────────────────────────────────
volumes:
  pgdata:             # Docker manages the actual storage location
  redisdata:
```

### `The Compose commands we often use.
``` bash
# Bring everything up (builds images if needed, creates network+volumes, starts all services)
docker compose up

# Up in detached mode (background) — this is what you use normally
docker compose up -d

# Build images first, then bring up (use when you've changed a Dockerfile)
docker compose up -d --build

# Bring everything down (stops + removes containers and networks, keeps volumes)
docker compose down

# Down AND remove volumes — careful, this deletes your database data
docker compose down -v

# See all running services and their status
docker compose ps

# Follow logs for all services
docker compose logs -f

# Follow logs for a specific service only
docker compose logs -f api

# Run a one-off command in a service (starts a temporary container)
docker compose run --rm api node migrate.js

# Execute a command in an already-running service container
docker compose exec api sh
docker compose exec postgres psql -U admin myapp

# Restart a single service (without touching others)
docker compose restart api

# Scale a service to multiple instances
docker compose up -d --scale api=3

# Pull latest versions of all images
docker compose pull

# See what Compose would create (dry run)
docker compose config
```

### `depends_on - the most misunderstood feature`
this is where most beginners get burned. depends_on controls start order, not readiness.
``` yaml
# This means: start postgres before api
# It does NOT mean: wait until postgres is accepting connections
depends_on:
  - postgres
```
PostgreSQL takes a few seconds to initialize. If your API tries to connect during that window — connection refused. The fix is healthcheck + condition: service_healthy:

``` yaml
# In postgres service — define what "healthy" means
healthcheck:
  test: ["CMD-SHELL", "pg_isready -U admin -d myapp"]
  interval: 5s
  timeout: 5s
  retries: 5

# In api service — wait until postgres is actually healthy
depends_on:
  postgres:
    condition: service_healthy
```
Now Compose will wait for postgres to pass its healthcheck before starting the api. This is production-correct behavior.

### `Environment variables - the right way`
Never hardcode secrets in docker-compose.yml. Use a .env file instead. Compose automatically reads .env from the same directory:
.env (never commit this to git)
``` bash
POSTGRES_PASSWORD=passwordany
POSTGRES_USER=usenay
POSTGRES_DB=myappdb
API_SECRET_KEY=abc123xuz
NODE_ENV=production/dev
```
docker-compose.yml — reference with ${VARIABLE} syntax:
``` yaml
services:
  postgres:
    environment:
      POSTGRES_PASSWORD: ${POSTGRES_PASSWORD}
      POSTGRES_USER: ${POSTGRES_USER}
      POSTGRES_DB: ${POSTGRES_DB}

  api:
    environment:
      DATABASE_URL: postgres://${POSTGRES_USER}:${POSTGRES_PASSWORD}@postgres:5432/${POSTGRES_DB}
      SECRET_KEY: ${API_SECRET_KEY}
```
we can also use multiple env files:
``` yaml
services:
  api:
    env_file:
      - .env
      - .env.local    # overrides .env, for local developer tweaks
```

### `Override files — dev vs production configs`
This is a pro pattern. You have a base compose file and environment-specific overrides:
docker-compose.yml — base (shared config)
``` yaml
services:
  api:
    build: ./api
    networks:
      - app-net
  postgres:
    image: postgres:16-alpine
    networks:
      - app-net
networks:
  app-net:
```
docker-compose.override.yml — dev overrides (auto-loaded by Compose)

``` yaml
services:
  api:
    volumes:
      - ./api:/app              # live code sync in dev
      - /app/node_modules
    environment:
      NODE_ENV: development
    ports:
      - "3000:3000"
    command: npm run dev        # nodemon for hot reload
```

docker-compose.prod.yml — production overrides

``` yaml
services:
  api:
    image: myregistry/myapp-api:${TAG}  # use pre-built image, not build
    environment:
      NODE_ENV: production
    restart: unless-stopped
    deploy:
      resources:
        limits:
          cpus: '0.5'
          memory: 512M
```
``` bash
# Dev — auto-loads docker-compose.yml + docker-compose.override.yml
docker compose up -d

# Production — explicitly specify files
docker compose -f docker-compose.yml -f docker-compose.prod.yml up -d
```
```
A complete working example — try this right now
Create this folder structure:
myapp/
├── docker-compose.yml
├── .env
└── api/
    ├── Dockerfile
    ├── package.json
    └── app.js
    
```
app.js
``` js
const express = require('express');
const { Pool } = require('pg');

const app = express();
const pool = new Pool({ connectionString: process.env.DATABASE_URL });

app.get('/', async (req, res) => {
  const result = await pool.query('SELECT NOW() as time');
  res.json({ message: 'Hello from Docker Compose!', db_time: result.rows[0].time });
});

app.get('/health', (req, res) => res.json({ status: 'ok' }));

app.listen(process.env.PORT || 3000, () =>
  console.log(`Running on port ${process.env.PORT || 3000}`)
);
```
Now run it
``` bash
cd myapp
docker compose up -d --build

# Watch it come up
docker compose logs -f

# Hit the API
curl http://localhost:3000
# {"message":"Hello from Docker Compose!","db_time":"2024-..."}

# Shell into the database
docker compose exec postgres psql -U admin myapp

# Tear down (keeps volumes)
docker compose down

# Tear down + delete database data
docker compose down -v
```

## `Stage-4: practice assignments.`
```
1. Write the full compose file above, bring it up, hit the API — confirm it queries the database
2. Deliberately remove the healthcheck and condition: service_healthy — watch the race condition break your API on startup
3. Add them back, bring it up again — watch Compose wait for postgres
4. docker compose logs -f while bringing up — read the startup sequence
5. docker compose exec postgres psql -U admin myapp — run some SQL, then docker compose down -v and back up — confirm data is gone without the volume




