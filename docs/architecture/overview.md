```mermaid
graph TD

User --> Vue

Vue --> FastAPI

FastAPI --> MySQL

FastAPI --> Redis

FastAPI --> LangGraph

LangGraph --> LLM

LangGraph --> RAG

RAG --> KnowledgeBase
```