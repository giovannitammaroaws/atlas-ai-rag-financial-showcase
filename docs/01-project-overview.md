# 01 - Query Flow

This is the main entry point of the system.  
From this screen, the user submits a question, the backend retrieves relevant chunks from the vector database, and the model returns a grounded answer with evidence.

What happens in sequence:
1. user writes a question
2. backend identifies intent/ticker and retrieves top chunks
3. LLM generates the answer from retrieved context
4. UI shows the answer and traceable source snippets

![Query Flow](images/query.png)

[Back to README](../README.md)
