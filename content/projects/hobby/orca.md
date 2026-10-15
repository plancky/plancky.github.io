---
title: "Orca"
build:
    render: never
params:
    url: "https://github.com/plancky/orca"
    thumbnail: "/projects/imgs/orca_thumbnail.png"
    tags:
        - Python
        - FastAPI
        - Postgres
        - pgvector
        - Redis
        - Celery
        - LLM
        - Modal
weight: 1
---

An async orchestrator that answers natural-language questions across Gmail, Google Calendar and Google Drive. It classifies intent, plans an execution DAG, fans out to per-service agents in parallel, runs hybrid pgvector semantic search over the user's data and synthesizes an answer, pausing for confirmation before any write like sending an email. Built from scratch on FastAPI, Postgres, Redis and Celery with no LangChain or agent framework, and with pluggable inference (Modal-hosted Qwen + BGE embeddings, or Gemini).
