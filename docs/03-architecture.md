# 03 - Data Ingestion

This step shows how documents enter the RAG pipeline.

Ingestion flow:
1. upload PDF or UTF-8 text document
2. extract and normalize text
3. split text into chunks
4. generate embeddings for each chunk
5. store documents/chunks/vectors in PostgreSQL + pgvector

Result:
- searchable knowledge base ready for semantic retrieval
- full traceability from answer to source chunks

![Data Ingestion](images/ingestion.png)

[Back to README](../README.md)
