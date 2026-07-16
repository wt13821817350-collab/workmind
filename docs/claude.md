
# WorkMind Development Guide

## Project Vision

WorkMind is an Enterprise AI Workspace.

The goal is NOT to build another chatbot.

The goal is to build AI Employees that help enterprises complete repetitive work.

---

## Product Principles

Always follow these principles.

1. Every feature must solve a real enterprise problem.

2. Avoid demo-only features.

3. Every module should be reusable.

4. Follow enterprise software architecture.

5. Code quality is more important than speed.

---

## Tech Stack

Frontend

Vue3
TypeScript
Vite
Element Plus
Pinia

Backend

FastAPI
SQLAlchemy
Alembic
MySQL
Redis

AI

LangGraph
RAG
OpenAI Compatible API

Deployment

Docker
Nginx

---

## Architecture

Frontend

↓

Backend API

↓

Service Layer

↓

AI Layer

↓

Storage

---

## Coding Rules

Always

Use async.

Use dependency injection.

Use typing.

Use RESTful APIs.

Never

Generate duplicated code.

Never hardcode configuration.

Never mix business logic with controllers.

---

## Documentation Rules

Every new feature must update

README

Roadmap

API

Architecture

Database

---

## Git Rules

Every commit

Use Conventional Commits

Examples

feat:

fix:

docs:

refactor:

---

## Goal

Build a production-ready enterprise AI platform.

Not a demo.