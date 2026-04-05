# Stage 1 - What is Docker and Why does it matter?

---

## The core problem Docker solves

  Before Docker, the classic developer nightmare was:  
  "It works on my machine."
---

You'd build an app, it runs fine locally, then breaks in staging or production because the OS version, library versions, environment variables, or runtime differs.

**Docker** solves this by packaging your app and everything it needs (OS libraries, runtime, dependencies, config)  
into a single portable unit called a container.

The container runs identically everywhere — your laptop, a CI server, production.

![containers_vs_vms](https://github.com/user-attachments/assets/1b42dc04-99b7-4d8c-901e-085962ee78f1)

<svg width="100%" viewBox="0 0 680 360" xmlns="http://www.w3.org/2000/svg">
  <defs>
    <marker id="arrow" viewBox="0 0 10 10" refX="8" refY="5" markerWidth="6" markerHeight="6" orient="auto-start-reverse">
      <path d="M2 1L8 5L2 9" fill="none" stroke="context-stroke" stroke-width="1.5" stroke-linecap="round" stroke-linejoin="round"/>
    </marker>
  </defs>
</svg>

### The key insight:
```
VMs virtualize hardware (each gets a full OS — GBs of overhead).
Containers virtualize the OS (they share the host kernel — just MBs of overhead).
```
### That's why containers start in seconds and VMs take minutes.
---
### Core Docker concepts
---
There are only three things you need to understand at the start:

### `Image — a read-only blueprint.`
```
Think of it like a class in OOP, or a recipe.
It contains your app code, dependencies, and configuration. Images are built once, used many times.
```
### `Container — a running instance of an image`
```
Think of it like an object instantiated from a class.
You can run dozens of containers from the same image. Containers are ephemeral — kill them, spin new ones.
```
### `Dockerfile — a text file with instructions for building an image.`
```
You write a Dockerfile, Docker reads it and produces an image.
```
### `The flow:`
```
Dockerfile → docker build → Image → docker run → Container
```
### Your first Docker commands — let's go hands-on

### `check docker is installed & running`
```bash
docker --version
docker info
```
### `Run your very first container:`
```bash 
docker run hello-world  # This pulls the hello-world image from Docker Hub (the official public registry), creates a container, runs it, and prints a message. That's the entire lifecycle in one command.

docker run -it ubuntu bash #Run an interactive Ubuntu container
You're now inside a container. It has its own filesystem. Type ls, pwd, cat /etc/os-release. Exit with exit. The container stops when you exit.
```

###  `Run an Nginx web server`
```bash
docker run -d -p 8080:80 --name my-nginx nginx
# Flags breakdown:

-d — detached mode (runs in background)
-p 8080:80 — map port 8080 on your machine to port 80 inside the container
--name my-nginx — give the container a readable name

Open http://localhost:8080 in your browser. You just ran a production-grade web server in one command.
```

###  `Useful commands`
```bash 
docker ps                    # list running containers
docker ps -a                 # list ALL containers (including stopped)
docker images                # list images on your machine
docker stop my-nginx         # stop a container gracefully
docker rm my-nginx           # delete a container
docker rmi nginx             # delete an image
docker logs my-nginx         # see container stdout/stderr
docker exec -it my-nginx bash  # shell into a running container
docker pull ubuntu           # pull an image without running it
docker inspect my-nginx      # full JSON details about a container
```

### `Clean up everything (useful to reset during learning):`
```bash
docker system prune -a       # removes all stopped containers, unused images, networks
```

### `How Docker actually works under the hood`
```
Two Linux kernel features power Docker: namespaces (isolation — each container sees only its own processes, filesystem, network) and cgroups (resource limits — CPU, memory caps per container). Docker is essentially a friendly API over these kernel primitives. On Mac and Windows, Docker runs a tiny Linux VM behind the scenes — that's Docker Desktop.
```
---
### Practice assignment for Stage 1
```
1. Run hello-world and read the output carefully — it explains what happened
2. Run an interactive ubuntu container, explore the filesystem, then exit
3. Run nginx in detached mode, visit it in browser, then docker logs it, then docker stop and docker rm it
4. Run docker run -it python:3.11 — you'll get a Python REPL inside a container with no Python installed on your machine
5. Run docker images and docker ps -a — understand what you're seeing
```


