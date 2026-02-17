# 04 - RAG Query Execution

This page shows the runtime query experience after ingestion.

Execution path:
1. user asks a financial question
2. backend retrieves most relevant embedded chunks
3. model composes grounded answer using retrieved context
4. UI shows answer, score, token usage, and evidence chunks

This is the core interactive layer of the platform.

![RAG Query](images/rag_query.png)

[Back to README](../README.md)
