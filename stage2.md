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

``` id="flow1"
Dockerfile → docker build → Image → docker run → Container
