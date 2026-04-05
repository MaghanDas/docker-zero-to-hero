# Stage 3 — Volumes & Networking
---
## Part A - Volumes: Making Data Survive 

We know that the container's writable layer is ephemeral. The moment you docker rm a container, everything it wrote to disk is gone. 
For a stateless API that's fine. But what about a database? What about user-uploaded files? 
That's where **volumes** come in.

---

![docker_volume_types (1)](https://github.com/user-attachments/assets/837b6c69-f3ec-45ab-8ad6-88ccec3a8d85)

There are **three storage types**. Each has a specific job.

## `1. Named Volumes — use for production data`
Docker creates and manages the storage location. You just give it a name. 
This is what you use for databases, persistent app data — anything that must survive container restarts.

```bash 
# Create a named volume explicitly
docker volume create pgdata

# Or let Docker create it automatically at run time
# -v <volume-name>:<path-inside-container>
docker run -d \
  --name postgres \
  -e POSTGRES_PASSWORD=secret \
  -v pgdata:/var/lib/postgresql/data \
  postgres:16

# Now stop and remove the container
docker stop postgres && docker rm postgres

# Data is still there! Run a new container with the same volume
docker run -d \
  --name postgres \
  -e POSTGRES_PASSWORD=secret \
  -v pgdata:/var/lib/postgresql/data \
  postgres:16
# Your database is intact. This is the whole point.

# Volume commands
docker volume ls                  # list all volumes
docker volume inspect pgdata      # see where Docker stored it
docker volume rm pgdata           # delete volume (data gone permanently)
docker volume prune               # remove all unused volumes
```

## `Bind Mounts - use for local development`


## Dockerfile → docker build → Image → docker run → Container
You specify an exact path on your host machine. That path is mounted directly into the container.
Changes on either side (host or container) are instantly reflected on the other side. 
This is how you get live code reloading during development — edit a file on your machine, the container sees the change immediately.

```bash
# -v <absolute-host-path>:<container-path>
# $(pwd) expands to your current directory
docker run -d \
  --name my-app-dev \
  -p 3000:3000 \
  -v $(pwd):/app \
  -v /app/node_modules \
  my-app:1.0
```
The second -v /app/node_modules is a crucial trick— it tells Docker "don't overwrite the container's node_modules with whatever is in the bind mount.
" Without it, your host's node_modules (or lack thereof) would clobber the one baked into the image.

```bash
# Modern syntax using --mount (more explicit, recommended)
docker run -d \
  --name my-app-dev \
  -p 3000:3000 \
  --mount type=bind,source=$(pwd),target=/app \
  my-app:1.0
```

## `3. tmpfs Mounts — RAM only, never hits disk`
Used for sensitive data (tokens, secrets, session data) that should never be written to disk, or for high-speed temporary caching.
```bash
docker run -d \
  --name my-app \
  --mount type=tmpfs,destination=/tmp \
  my-app:1.0
```
### Named Volume vs Bind Mount — when to use which

| Feature            | Named Volume                  | Bind Mount                          |
|--------------------|------------------------------|-------------------------------------|
| Use for            | databases, persistent data   | dev code sync, config files         |
| Path managed by    | Docker                       | you                                 |
| Performance        | excellent                    | excellent on Linux, slower on Mac   |
| Production         | yes                          | rarely                              |
| Dev workflow       | yes                          | yes — primary tool                  |

---
# Part B - Networking: How Containers Talk
---
By default, every container is isolated — it can't reach any other container. 
**Docker networking** is how you connect them. 
This is how your Node.js API container talks to your PostgreSQL container.

![docker_networking](https://github.com/user-attachments/assets/340b0d5b-1953-4668-b8ed-4dfe61cb659a)

### `The three Docker network drivers`

### `Bridge (default, most common) — `
- containers on the same bridge network can talk to each other. 
- Isolated from containers on other networks.
- The default bridge network Docker creates has no DNS.
- Custom bridge networks do — and that's why you always create your own.
  
### `Host — `
- container shares the host's network stack directly.
- No port mapping needed — if your app listens on 3000, it's available on the host's port 3000 automatically.
- Linux only (not Mac/Windows). Used for performance-critical scenarios.

### `None `
- completely isolated, no network at all.
- Used for batch jobs or security-sensitive workloads.


## `Custom bridge networks — always use these`


``` bash 
# Create a custom network
docker network create app-net

# Run postgres attached to it
docker run -d \
  --name postgres \
  --network app-net \
  -e POSTGRES_PASSWORD=secret \
  -v pgdata:/var/lib/postgresql/data \
  postgres:16

# Run API container on the same network
docker run -d \
  --name api \
  --network app-net \
  -p 3000:3000 \
  my-app:1.0
```
Now inside the api container, you can reach postgres by its container name as a hostname:

``` bash
// In your Node.js app — use container name, not localhost!
const client = new Client({
  host: 'postgres',   // Docker DNS resolves this to the postgres container's IP
  port: 5432,
  database: 'mydb',
  password: 'secret'
});
```
This is the DNS magic of custom bridge networks.
Container names become hostnames. No hardcoded IPs needed ever.

``` bash
# Network management commands
docker network ls                        # list all networks
docker network inspect app-net           # see connected containers, IPs
docker network connect app-net my-app    # attach a running container to a network
docker network disconnect app-net my-app # detach
docker network rm app-net                # delete network
docker network prune                     # remove all unused networks
```
### `Port mapping deep dive`
``` bash
# -p <host-port>:<container-port>
docker run -p 3000:3000 my-app    # host 3000 → container 3000

# Bind to a specific host interface (security — only localhost, not public)
docker run -p 127.0.0.1:3000:3000 my-app

# Map to a random available host port (Docker picks it)
docker run -p 3000 my-app
docker port my-app                 # see what port was assigned

# Multiple ports (e.g. HTTP + HTTPS)
docker run -p 80:80 -p 443:443 nginx

# Important rule: containers talk to each other via container port (no -p needed)
# -p is ONLY for exposing to the host/outside world
```

### `Putting Volumes + Networking together — a real example`
This is the closest thing to a real app you can run with raw docker run commands.
A Node API talking to PostgreSQL with persistent data:

``` bash
# 1. Create the network and volume
docker network create app-net
docker volume create pgdata

# 2. Start postgres
docker run -d \
  --name postgres \
  --network app-net \
  -e POSTGRES_DB=myapp \
  -e POSTGRES_USER=admin \
  -e POSTGRES_PASSWORD=secret \
  -v pgdata:/var/lib/postgresql/data \
  postgres:16

# 3. Start the API (connects to postgres by name)
docker run -d \
  --name api \
  --network app-net \
  -p 3000:3000 \
  -e DATABASE_URL=postgres://admin:secret@postgres:5432/myapp \
  my-app:1.0

# 4. Verify they can talk
docker exec -it api sh
# inside container:
ping postgres       # resolves by DNS
wget -qO- http://postgres:5432  # or use your app's DB connection
```

### `Stage 3 practice assignments`
```
1. Run postgres with a named volume, insert some data via docker exec -it postgres psql -U admin myapp, stop and remove the container, spin a new one with the same volume — verify your data survived
2. Create a custom network, run two containers on it, docker exec into one and ping the other by name
3. Run a container with -p 127.0.0.1:3000:3000 — verify it's only accessible from localhost, not your network IP
4. Run docker network inspect app-net after connecting containers — read the JSON and find each container's IP



