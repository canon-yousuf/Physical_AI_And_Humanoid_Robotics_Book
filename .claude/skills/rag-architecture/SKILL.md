---
name: rag-architecture
description: Architecture patterns and best practices for building RAG (Retrieval-Augmented Generation) chatbots. Use when designing the chatbot backend, implementing vector search, chunking documents, or integrating LLMs with document retrieval for the Physical AI textbook.
---

# RAG Architecture Patterns

This skill provides patterns for building the RAG chatbot that answers questions about the Physical AI textbook. It covers the complete pipeline from content ingestion to response generation.

## System Architecture Overview

```
┌─────────────────────────────────────────────────────────────────┐
│                        USER INTERFACE                           │
│                   (React Component in Docusaurus)               │
└─────────────────────────┬───────────────────────────────────────┘
                          │ HTTP POST /chat
                          ▼
┌─────────────────────────────────────────────────────────────────┐
│                        FASTAPI BACKEND                          │
│  ┌─────────────┐  ┌─────────────┐  ┌─────────────────────────┐ │
│  │   /chat     │  │  /chat/     │  │  /ingest                │ │
│  │  endpoint   │  │  selection  │  │  (admin only)           │ │
│  └──────┬──────┘  └──────┬──────┘  └─────────────────────────┘ │
└─────────┼────────────────┼──────────────────────────────────────┘
          │                │
          ▼                ▼
┌─────────────────────────────────────────────────────────────────┐
│                     RAG PIPELINE                                │
│  1. Embed Query  →  2. Search Qdrant  →  3. Build Prompt       │
│                                          →  4. Call LLM         │
└─────────┬────────────────┬──────────────────────┬───────────────┘
          │                │                      │
          ▼                ▼                      ▼
   ┌─────────────┐  ┌─────────────┐      ┌─────────────┐
   │   OpenAI    │  │   Qdrant    │      │   OpenAI    │
   │  Embeddings │  │   Cloud     │      │  GPT-4 /    │
   │    API      │  │  (vectors)  │      │  Claude API │
   └─────────────┘  └─────────────┘      └─────────────┘
```

## Technology Stack

### Required Services (Free Tiers Available)

**Qdrant Cloud** - Vector database for storing embeddings
- Free tier: 1GB storage, 1 cluster
- Sign up: https://cloud.qdrant.io/

**Neon Postgres** - Serverless PostgreSQL for user data
- Free tier: 0.5GB storage, 1 project
- Sign up: https://neon.tech/

**OpenAI API** - For embeddings and chat completion
- Pay-as-you-go pricing
- Embeddings: ~$0.0001 per 1K tokens
- GPT-4: ~$0.03 per 1K tokens

### Python Dependencies

```python
# requirements.txt
fastapi>=0.109.0
uvicorn>=0.27.0
qdrant-client>=1.7.0
openai>=1.10.0
python-dotenv>=1.0.0
pydantic>=2.5.0
python-frontmatter>=1.1.0
httpx>=0.26.0
```

## Content Ingestion Pipeline

### Step 1: Load Documents from Docusaurus

```python
from pathlib import Path
import frontmatter
from typing import List, Dict, Any

def load_markdown_files(docs_dir: str) -> List[Dict[str, Any]]:
    """
    Load all markdown files from the Docusaurus docs directory.
    
    Args:
        docs_dir: Path to the docs/ folder in your Docusaurus project
        
    Returns:
        List of documents with content and metadata
    """
    documents = []
    docs_path = Path(docs_dir)
    
    for md_file in docs_path.rglob("*.md"):
        # Skip files that start with underscore (like _category_.json companions)
        if md_file.name.startswith("_"):
            continue
            
        # Parse frontmatter and content using python-frontmatter
        post = frontmatter.load(md_file)
        
        # Extract module name from path (e.g., "module-1-ros2" from "docs/module-1-ros2/intro.md")
        relative_path = md_file.relative_to(docs_path)
        parts = relative_path.parts
        module = parts[0] if len(parts) > 1 else "introduction"
        
        # Build document object with metadata for filtering later
        documents.append({
            "content": post.content,
            "metadata": {
                "title": post.get("title", md_file.stem.replace("-", " ").title()),
                "source": str(relative_path),
                "module": module,
                "sidebar_position": post.get("sidebar_position", 999),
                "description": post.get("description", ""),
                "keywords": post.get("keywords", [])
            }
        })
    
    print(f"Loaded {len(documents)} documents from {docs_dir}")
    return documents
```

### Step 2: Chunk Documents for Embedding

```python
from typing import List, Dict, Any

def chunk_documents(
    documents: List[Dict[str, Any]], 
    chunk_size: int = 500, 
    overlap: int = 100
) -> List[Dict[str, Any]]:
    """
    Split documents into overlapping chunks suitable for embedding.
    
    Uses semantic splitting (by headers) when possible, falls back to 
    character-based splitting. Overlap ensures context isn't lost at boundaries.
    
    Args:
        documents: List of documents from load_markdown_files()
        chunk_size: Target size of each chunk in characters
        overlap: Number of characters to overlap between chunks
        
    Returns:
        List of chunks with inherited metadata plus chunk-specific info
    """
    # Define split points in order of preference (most semantic to least)
    separators = [
        "\n## ",      # H2 headers - major sections
        "\n### ",     # H3 headers - subsections  
        "\n#### ",    # H4 headers - minor sections
        "\n\n",       # Paragraph breaks
        "\n",         # Line breaks
        ". ",         # Sentences
        " ",          # Words (last resort)
    ]
    
    chunks = []
    
    for doc in documents:
        content = doc["content"]
        doc_chunks = []
        
        # Try to split by the most semantic separator available
        current_text = content
        
        for separator in separators:
            if len(current_text) <= chunk_size:
                break
                
            if separator in current_text:
                parts = current_text.split(separator)
                current_chunk = ""
                
                for part in parts:
                    # If adding this part exceeds chunk size, save current chunk
                    if len(current_chunk) + len(part) > chunk_size and current_chunk:
                        doc_chunks.append(current_chunk.strip())
                        # Start new chunk with overlap from previous
                        overlap_text = current_chunk[-overlap:] if len(current_chunk) > overlap else current_chunk
                        current_chunk = overlap_text + separator + part
                    else:
                        current_chunk += separator + part if current_chunk else part
                        
                if current_chunk.strip():
                    doc_chunks.append(current_chunk.strip())
                break
        else:
            # If no separators found, just use the whole content
            doc_chunks.append(current_text.strip())
        
        # Add metadata to each chunk
        for i, chunk_text in enumerate(doc_chunks):
            if chunk_text:  # Skip empty chunks
                chunks.append({
                    "content": chunk_text,
                    "metadata": {
                        **doc["metadata"],
                        "chunk_index": i,
                        "total_chunks": len(doc_chunks)
                    }
                })
    
    print(f"Created {len(chunks)} chunks from {len(documents)} documents")
    return chunks
```

### Step 3: Generate Embeddings

```python
from openai import OpenAI
from typing import List
import os

# Initialize client (reads OPENAI_API_KEY from environment)
client = OpenAI()

def generate_embeddings(
    texts: List[str], 
    model: str = "text-embedding-3-small"
) -> List[List[float]]:
    """
    Generate embeddings for a list of texts using OpenAI's API.
    
    Args:
        texts: List of text strings to embed
        model: OpenAI embedding model to use
               - "text-embedding-3-small": Cheaper, 1536 dimensions
               - "text-embedding-3-large": Better quality, 3072 dimensions
               
    Returns:
        List of embedding vectors (same order as input texts)
    """
    # OpenAI API can handle batches up to 2048 texts
    # For larger batches, we'd need to split them
    batch_size = 100
    all_embeddings = []
    
    for i in range(0, len(texts), batch_size):
        batch = texts[i:i + batch_size]
        
        response = client.embeddings.create(
            model=model,
            input=batch
        )
        
        # Extract embeddings in order
        batch_embeddings = [item.embedding for item in response.data]
        all_embeddings.extend(batch_embeddings)
        
        print(f"Embedded batch {i//batch_size + 1}/{(len(texts)-1)//batch_size + 1}")
    
    return all_embeddings
```

### Step 4: Store in Qdrant

```python
from qdrant_client import QdrantClient
from qdrant_client.models import VectorParams, Distance, PointStruct
from typing import List, Dict, Any
import uuid

def create_qdrant_client(url: str, api_key: str) -> QdrantClient:
    """
    Create a Qdrant client for cloud deployment.
    
    Get these values from your Qdrant Cloud dashboard.
    """
    return QdrantClient(url=url, api_key=api_key)

def create_collection(
    client: QdrantClient, 
    collection_name: str, 
    vector_size: int = 1536
) -> None:
    """
    Create a Qdrant collection for storing document embeddings.
    
    Args:
        client: Qdrant client instance
        collection_name: Name for the collection
        vector_size: Dimension of embeddings (1536 for text-embedding-3-small)
    """
    # Check if collection already exists
    collections = client.get_collections().collections
    exists = any(c.name == collection_name for c in collections)
    
    if exists:
        print(f"Collection '{collection_name}' already exists, skipping creation")
        return
    
    client.create_collection(
        collection_name=collection_name,
        vectors_config=VectorParams(
            size=vector_size,
            distance=Distance.COSINE  # Cosine similarity works well for text
        )
    )
    print(f"Created collection '{collection_name}'")

def upload_chunks(
    client: QdrantClient, 
    collection_name: str, 
    chunks: List[Dict[str, Any]], 
    embeddings: List[List[float]]
) -> None:
    """
    Upload document chunks with their embeddings to Qdrant.
    
    Args:
        client: Qdrant client instance
        collection_name: Target collection
        chunks: List of chunk dictionaries with content and metadata
        embeddings: Corresponding embedding vectors
    """
    # Build point objects for Qdrant
    points = []
    for chunk, embedding in zip(chunks, embeddings):
        points.append(
            PointStruct(
                id=str(uuid.uuid4()),  # Unique ID for each chunk
                vector=embedding,
                payload={
                    "content": chunk["content"],
                    **chunk["metadata"]
                }
            )
        )
    
    # Upload in batches
    batch_size = 100
    for i in range(0, len(points), batch_size):
        batch = points[i:i + batch_size]
        client.upsert(
            collection_name=collection_name,
            points=batch
        )
        print(f"Uploaded batch {i//batch_size + 1}/{(len(points)-1)//batch_size + 1}")
    
    print(f"Uploaded {len(points)} chunks to '{collection_name}'")
```

### Complete Ingestion Script

```python
# ingest.py - Run this once to populate your vector database

import os
from dotenv import load_dotenv

load_dotenv()

def main():
    """Ingest all documentation into Qdrant."""
    
    # Configuration
    DOCS_DIR = "./docs"  # Path to your Docusaurus docs folder
    COLLECTION_NAME = "physical_ai_textbook"
    
    # Load environment variables
    qdrant_url = os.getenv("QDRANT_URL")
    qdrant_api_key = os.getenv("QDRANT_API_KEY")
    
    # Step 1: Load documents
    print("Loading documents...")
    documents = load_markdown_files(DOCS_DIR)
    
    # Step 2: Chunk documents
    print("Chunking documents...")
    chunks = chunk_documents(documents, chunk_size=500, overlap=100)
    
    # Step 3: Generate embeddings
    print("Generating embeddings...")
    texts = [chunk["content"] for chunk in chunks]
    embeddings = generate_embeddings(texts)
    
    # Step 4: Upload to Qdrant
    print("Uploading to Qdrant...")
    client = create_qdrant_client(qdrant_url, qdrant_api_key)
    create_collection(client, COLLECTION_NAME)
    upload_chunks(client, COLLECTION_NAME, chunks, embeddings)
    
    print("Ingestion complete!")

if __name__ == "__main__":
    main()
```

## Query Pipeline (FastAPI Backend)

### API Models

```python
# models.py
from pydantic import BaseModel, Field
from typing import Optional, List

class ChatRequest(BaseModel):
    """Request body for the /chat endpoint."""
    question: str = Field(..., description="User's question about the textbook")
    selected_text: Optional[str] = Field(
        None, 
        description="Text selected by user for context-specific questions"
    )
    module_filter: Optional[str] = Field(
        None,
        description="Limit search to specific module (e.g., 'module-1-ros2')"
    )

class Source(BaseModel):
    """Citation source for an answer."""
    title: str
    source: str
    module: str

class ChatResponse(BaseModel):
    """Response body from the /chat endpoint."""
    answer: str
    sources: List[Source]
    
class HealthResponse(BaseModel):
    """Response for health check endpoint."""
    status: str
    version: str
```

### FastAPI Application

```python
# main.py
from fastapi import FastAPI, HTTPException
from fastapi.middleware.cors import CORSMiddleware
from contextlib import asynccontextmanager
import os

from models import ChatRequest, ChatResponse, HealthResponse
from rag import RAGPipeline

# Initialize RAG pipeline as global (loaded once at startup)
rag_pipeline: RAGPipeline = None

@asynccontextmanager
async def lifespan(app: FastAPI):
    """Load RAG pipeline on startup, cleanup on shutdown."""
    global rag_pipeline
    rag_pipeline = RAGPipeline(
        qdrant_url=os.getenv("QDRANT_URL"),
        qdrant_api_key=os.getenv("QDRANT_API_KEY"),
        collection_name="physical_ai_textbook"
    )
    yield
    # Cleanup if needed

app = FastAPI(
    title="Physical AI Textbook RAG API",
    description="Answer questions about Physical AI & Humanoid Robotics",
    version="1.0.0",
    lifespan=lifespan
)

# Enable CORS for Docusaurus frontend
app.add_middleware(
    CORSMiddleware,
    allow_origins=["*"],  # Configure for your domain in production
    allow_credentials=True,
    allow_methods=["*"],
    allow_headers=["*"],
)

@app.get("/health", response_model=HealthResponse)
async def health_check():
    """Health check endpoint."""
    return HealthResponse(status="healthy", version="1.0.0")

@app.post("/chat", response_model=ChatResponse)
async def chat(request: ChatRequest):
    """
    Answer questions about the Physical AI textbook.
    
    The endpoint:
    1. Embeds the user's question
    2. Searches Qdrant for relevant content chunks
    3. Builds a prompt with the retrieved context
    4. Calls the LLM to generate an answer
    5. Returns the answer with source citations
    """
    try:
        result = await rag_pipeline.query(
            question=request.question,
            selected_text=request.selected_text,
            module_filter=request.module_filter
        )
        return result
    except Exception as e:
        raise HTTPException(status_code=500, detail=str(e))

@app.post("/chat/selection", response_model=ChatResponse)
async def chat_with_selection(request: ChatRequest):
    """
    Answer questions about specific selected text.
    
    This endpoint is used when users highlight text in the book
    and want to ask questions specifically about that selection.
    """
    if not request.selected_text:
        raise HTTPException(
            status_code=400, 
            detail="selected_text is required for this endpoint"
        )
    return await chat(request)
```

### RAG Pipeline Class

```python
# rag.py
from openai import OpenAI
from qdrant_client import QdrantClient
from qdrant_client.models import Filter, FieldCondition, MatchValue
from typing import Optional, List
from models import ChatResponse, Source

class RAGPipeline:
    """Handles the complete RAG query pipeline."""
    
    def __init__(self, qdrant_url: str, qdrant_api_key: str, collection_name: str):
        self.openai_client = OpenAI()
        self.qdrant_client = QdrantClient(url=qdrant_url, api_key=qdrant_api_key)
        self.collection_name = collection_name
        
    def _embed_query(self, text: str) -> List[float]:
        """Generate embedding for a query."""
        response = self.openai_client.embeddings.create(
            model="text-embedding-3-small",
            input=text
        )
        return response.data[0].embedding
    
    def _search_similar(
        self, 
        query_embedding: List[float], 
        limit: int = 5,
        module_filter: Optional[str] = None
    ) -> List:
        """Search Qdrant for similar chunks."""
        
        # Build filter if module specified
        search_filter = None
        if module_filter:
            search_filter = Filter(
                must=[
                    FieldCondition(
                        key="module",
                        match=MatchValue(value=module_filter)
                    )
                ]
            )
        
        results = self.qdrant_client.search(
            collection_name=self.collection_name,
            query_vector=query_embedding,
            limit=limit,
            query_filter=search_filter
        )
        
        return results
    
    def _build_prompt(
        self, 
        question: str, 
        results: List, 
        selected_text: Optional[str] = None
    ) -> str:
        """Build the prompt for the LLM with retrieved context."""
        
        # Format retrieved chunks as context
        context_parts = []
        for i, result in enumerate(results, 1):
            payload = result.payload
            context_parts.append(
                f"[Source {i}: {payload['title']} - {payload['source']}]\n"
                f"{payload['content']}"
            )
        context = "\n\n---\n\n".join(context_parts)
        
        # Add selected text section if provided
        selected_section = ""
        if selected_text:
            selected_section = f"""

SELECTED TEXT BY USER:
The student has highlighted this specific text from the book:
\"\"\"
{selected_text}
\"\"\"

Focus your answer on explaining this highlighted content. Reference the selected text directly in your explanation.
"""
        
        system_prompt = f"""You are a helpful teaching assistant for the Physical AI & Humanoid Robotics course. Your job is to answer student questions using the provided context from the textbook.

IMPORTANT RULES:
1. ONLY use information from the provided context to answer
2. If the context doesn't contain the answer, say "I don't have information about that in the textbook. You might want to check the module on [suggest related topic]."
3. Always cite your sources using [Source N] format
4. Explain concepts clearly - assume the student knows Python but NOT robotics
5. Use analogies when they help clarify complex concepts
6. Keep answers focused and educational

CONTEXT FROM TEXTBOOK:
{context}
{selected_section}"""

        return system_prompt
    
    async def query(
        self, 
        question: str, 
        selected_text: Optional[str] = None,
        module_filter: Optional[str] = None
    ) -> ChatResponse:
        """
        Execute the complete RAG pipeline.
        
        Args:
            question: User's question
            selected_text: Optional highlighted text for context
            module_filter: Optional module to limit search
            
        Returns:
            ChatResponse with answer and sources
        """
        # Step 1: Embed the question
        query_embedding = self._embed_query(question)
        
        # Step 2: Search for relevant chunks
        results = self._search_similar(
            query_embedding, 
            limit=5, 
            module_filter=module_filter
        )
        
        # Handle no results
        if not results:
            return ChatResponse(
                answer="I couldn't find relevant information in the textbook for your question. Try rephrasing or asking about a specific topic like ROS 2 nodes, Gazebo simulation, or NVIDIA Isaac.",
                sources=[]
            )
        
        # Step 3: Build prompt with context
        system_prompt = self._build_prompt(question, results, selected_text)
        
        # Step 4: Call LLM
        response = self.openai_client.chat.completions.create(
            model="gpt-4-turbo-preview",  # Or "gpt-3.5-turbo" for cheaper option
            messages=[
                {"role": "system", "content": system_prompt},
                {"role": "user", "content": question}
            ],
            temperature=0.3,  # Lower temperature for more factual answers
            max_tokens=1000
        )
        
        answer = response.choices[0].message.content
        
        # Step 5: Format sources
        sources = [
            Source(
                title=r.payload["title"],
                source=r.payload["source"],
                module=r.payload["module"]
            )
            for r in results
        ]
        
        return ChatResponse(answer=answer, sources=sources)
```

## Environment Configuration

```bash
# .env file (DO NOT commit this to git!)
OPENAI_API_KEY=sk-your-openai-api-key
QDRANT_URL=https://your-cluster.qdrant.io
QDRANT_API_KEY=your-qdrant-api-key
NEON_DATABASE_URL=postgresql://user:pass@host/dbname
```

## Running the API

```bash
# Install dependencies
pip install -r requirements.txt

# Run ingestion (once, to populate Qdrant)
python ingest.py

# Start the API server
uvicorn main:app --reload --port 8000
```

## Error Handling Best Practices

```python
class RAGError(Exception):
    """Base exception for RAG errors."""
    pass

class NoRelevantContentError(RAGError):
    """Raised when no relevant content is found."""
    pass

class EmbeddingError(RAGError):
    """Raised when embedding generation fails."""
    pass

# In your endpoint:
@app.post("/chat")
async def chat(request: ChatRequest):
    try:
        result = await rag_pipeline.query(request.question)
        return result
    except NoRelevantContentError:
        return ChatResponse(
            answer="I couldn't find relevant information for your question.",
            sources=[]
        )
    except EmbeddingError:
        raise HTTPException(status_code=503, detail="Embedding service unavailable")
    except Exception as e:
        logger.error(f"RAG error: {e}")
        raise HTTPException(status_code=500, detail="Internal server error")
```

## Response Quality Checklist

Every good RAG response should include:

1. **Direct Answer**: Address the question in the first sentence
2. **Explanation**: Break down the concept for beginners
3. **Code Example**: If relevant, show a small code snippet
4. **Source Citations**: Always cite using [Source N] format
5. **Follow-up Suggestion**: Suggest related topics to explore

Example high-quality response:

```
ROS 2 topics are named channels where nodes can publish and subscribe to messages 
[Source 1].

Think of a topic like a group chat channel: any node can post messages (publish), 
and any node that's interested can receive those messages (subscribe). This 
"publish-subscribe" pattern allows nodes to communicate without knowing about 
each other directly [Source 2].

Here's a simple example of creating a publisher:

```python
# Create a publisher that sends String messages to 'robot_status' topic
self.publisher = self.create_publisher(String, 'robot_status', 10)
```

The `10` is the queue size - how many messages to buffer if subscribers are slow.

You might also want to learn about Services (for request-reply patterns) or 
Actions (for long-running tasks with feedback) in the next sections.

Sources:
[Source 1]: Module 1 - Introduction to ROS 2
[Source 2]: Module 1 - Nodes, Topics, and Services
```
