# youtube-vector-rag-llamaindex
Vector RAG chatbot for YouTube videos using LlamaIndex. Fetches the transcript with a self-written loader, embeds it with Gemini into ChromaDB Cloud, reranks with a cross-encoder, and answers strictly from the video in a CLI chat with PostgreSQL memory. Includes Phoenix tracing, token counting, and cost tracking. Vector baseline of a RAG series.
