This project addresses a critical limitation of standard language models: **hallucinations and lack of access to specialized knowledge**. Rather than relying on a model's internal parameters, RAG systems retrieve relevant context from external documents and feed that context directly into generation, producing grounded, factually accurate responses.


## Features

- **100% Local Execution** - No cloud APIs, no vendor lock-in, complete data privacy
- **Multi-Document Reasoning** - Query across multiple documents with cross-document context fusion
- **Query Expansion** - Automatically generate alternative phrasings to improve retrieval coverage
- **Maximal Marginal Relevance (MMR)** - Retrieve diverse results to avoid redundancy
- **LLM-as-Judge Evaluation** - Built-in benchmarking framework to measure response quality
- **Easy Integration** - Modular architecture with LangChain for seamless component swapping
- **Fast Inference** - 3B parameter Llama model ensures sub-second response times

##  Architecture

```
┌─────────────────┐
│   PDF Documents │
└────────┬────────┘
         │
         ▼
┌─────────────────────────────────────────┐
│  PyPDFLoader + RecursiveTextSplitter    │
│  (1000-char chunks, 200-char overlap)   │
└────────┬────────────────────────────────┘
         │
         ▼
┌──────────────────────────────────────────┐
│  sentence-transformers/all-MiniLM-L6-v2  │
│  (384-dimensional dense embeddings)      │
└────────┬─────────────────────────────────┘
         │
         ▼
┌──────────────────────────────────────────┐
│  FAISS Flat L2 Index                     │
│  (Vector similarity search)              │
└────────┬─────────────────────────────────┘
         │
         ▼
┌──────────────────────────────────────────┐
│  Query Expansion Layer (LLM)             │
│  (Generate 3 alternative phrasings)      │
└────────┬─────────────────────────────────┘
         │
         ▼
┌──────────────────────────────────────────┐
│  Ollama + Llama 3.2 (3B)                 │
│  (Local inference engine)                │
└──────────────────────────────────────────┘
         │
         ▼
┌──────────────────────────────────────────┐
│  Grounded Answer with Source Context     │
└──────────────────────────────────────────┘
```

## Performance Results

Evaluated on a 20-question test suite across three difficulty levels:

| Configuration | Factual Retrieval | Multi-Hop Reasoning | Contextual Synthesis | **Overall** |
|---------------|-------------------|-------------------|----------------------|------------|
| **Config A** (Baseline) | 4.6/5.0 | 2.6/5.0 | 2.4/5.0 | **3.19/5.0** |
| **Config B** (MMR + Larger Context) | 4.4/5.0 | 3.9/5.0 | 3.7/5.0 | **4.12/5.0** |
| **+ Query Expansion** | 4.5/5.0 | **4.65/5.0** ⭐ | 4.0/5.0 | **4.35/5.0** |

**Improvement**: Query expansion boosted multi-hop reasoning by **+1.45 points** (55% improvement).

## Quick Start

### Prerequisites

- Python 3.8+
- Ollama (download from [ollama.ai](https://ollama.ai))
- 4GB+ RAM (8GB+ recommended for smooth inference)
- Virtual environment (recommended)

### Installation

1. **Clone the repository**
   ```bash
   git clone https://github.com/yourusername/local-rag-pipeline.git
   cd local-rag-pipeline
   ```

2. **Create and activate virtual environment**
   ```bash
   python -m venv venv
   source venv/bin/activate  # On Windows: venv\Scripts\activate
   ```

3. **Install dependencies**
   ```bash
   pip install -r requirements.txt
   ```

4. **Start Ollama and load Llama 3.2**
   ```bash
   # In a separate terminal, start Ollama service
   ollama serve
   
   # In another terminal, pull the model
   ollama pull llama2:3b
   ```

5. **Place your PDFs**
   ```bash
   mkdir -p data/pdfs
   cp your-documents/*.pdf data/pdfs/
   ```

## 💻 Usage

### Basic Query

```python
from rag_pipeline import RAGSystem

# Initialize the system
rag = RAGSystem(
    pdf_directory="data/pdfs",
    chunk_size=1000,
    chunk_overlap=200,
    use_mmr=True,
    k_results=5
)

# Ask a question
query = "What are the main themes discussed in the documents?"
answer = rag.query(query)
print(answer)
```

### Advanced: With Query Expansion

```python
# Enable query expansion for better multi-hop reasoning
rag = RAGSystem(
    pdf_directory="data/pdfs",
    use_query_expansion=True,
    expansion_count=3  # Generate 3 alternative phrasings
)

answer = rag.query(
    "Compare the structural differences between document 1 and document 3",
    return_source_chunks=True
)

print(f"Answer: {answer['response']}")
print(f"Sources: {answer['sources']}")
```

### Conversational Chat

```python
# Start an interactive chat session
rag = RAGSystem(pdf_directory="data/pdfs")
rag.start_chat()

# Interact in the CLI
# >>> What is the publication date?
# >>> Compare themes across documents
# >>> exit
```

### Evaluation/Benchmarking

```python
from evaluation import evaluate_system

# Evaluate against a test suite
results = evaluate_system(
    rag_system=rag,
    questions=[
        "What year was this published?",
        "Who is the author?",
        # ... more questions
    ],
    use_llm_judge=True
)

print(f"Composite Score: {results['composite_mean']}/5.0")
print(f"Multi-hop Performance: {results['multihop_mean']}/5.0")
```

## Project Structure

```
local-rag-pipeline/
├── README.md
├── requirements.txt
├── config.yaml                 # Configuration parameters
├── src/
│   ├── rag_pipeline.py         # Main RAG system class
│   ├── document_loader.py      # PDF loading and chunking
│   ├── embeddings.py           # Embedding generation
│   ├── vector_store.py         # FAISS integration
│   ├── retriever.py            # Retrieval strategies (standard + MMR)
│   ├── query_expansion.py      # Query expansion logic
│   └── llm_interface.py        # Ollama/Llama interface
├── evaluation/
│   ├── benchmark.py            # 20-question test suite
│   ├── llm_judge.py            # LLM-as-Judge scoring
│   └── results.json            # Evaluation results
├── data/
│   └── pdfs/                   # Place your PDFs here
├── notebooks/
│   └── rag_demo.ipynb          # Interactive demo
└── tests/
    ├── test_retrieval.py
    ├── test_embeddings.py
    └── test_rag_system.py
```

## Configuration

Edit `config.yaml` to customize behavior:

```yaml
# Document Processing
chunk_size: 1000
chunk_overlap: 200
recursive_split: true

# Embeddings
embedding_model: "sentence-transformers/all-MiniLM-L6-v2"
embedding_dimension: 384

# Retrieval
use_mmr: true
k_results: 5
similarity_threshold: 0.5

# Query Expansion
enable_query_expansion: true
expansion_count: 3
expansion_k_per_query: 2

# Generation
model_name: "llama2:3b"
ollama_base_url: "http://localhost:11434"
temperature: 0.7
max_tokens: 512

# Evaluation
llm_judge_model: "llama2:3b"
evaluation_questions_file: "evaluation/questions.json"
```

## Key Implementation Details

### Chunking Strategy

Documents are split using **RecursiveCharacterTextSplitter** with hierarchical delimiters:
1. Newline pairs (`\n\n`)
2. Single newlines (`\n`)
3. Spaces (` `)
4. Characters (fallback)

This prevents mid-word/mid-sentence truncation that would destroy semantic meaning.

### Embedding Model

**sentence-transformers/all-MiniLM-L6-v2** is chosen for:
- Lightweight (22M parameters)
- Fast inference on CPU
- Excellent semantic alignment (384 dimensions)
- Supports cosine similarity search

### Retrieval Strategies

**Configuration A (Baseline)**
- Small chunks (1,000 chars)
- Standard cosine distance
- k=3 results
- Best for: Factual lookups, low latency

**Configuration B (Enhanced)**
- Larger chunks (2,000 chars)
- Maximal Marginal Relevance (MMR)
- k=5 results
- Best for: Cross-document reasoning, synthesis

### Query Expansion

For complex queries, automatically generate three alternative phrasings:
```
Original: "What's the legal disclaimer?"
Variant 1: "List the legal terms and conditions"
Variant 2: "What are the licensing requirements?"
Variant 3: "What restrictions apply to usage?"
```

All four are embedded and searched. Results are deduplicated to keep context concise.

## Scaling to Production (10,000+ Documents)

The current architecture works well for <1,000 documents. For production scale:

1. **Replace FAISS with distributed vector database**
   - Qdrant, Milvus, or Pinecone
   - HNSW indexing for sub-millisecond search
   - Scalar quantization for 75% memory savings

2. **Async document processing**
   - Celery/Kafka task queues
   - Parallel embedding generation
   - Non-blocking vector index updates

3. **Inference serving**
   - vLLM, Triton, or Text Generation Inference
   - PagedAttention for memory optimization
   - Load balancing across multiple inference servers

4. **Monitoring & observability**
   - Query latency tracking
   - Retrieval quality metrics
   - Model performance profiling

See `SCALING.md` for detailed architecture.


## Testing

Run the test suite:

```bash
# Unit tests
pytest tests/ -v

# Integration tests
pytest tests/ -v --integration

# Benchmark evaluation
python -m evaluation.benchmark --config config.yaml

# Full evaluation with LLM-as-Judge
python -m evaluation.benchmark --judge --config config.yaml
```




---


---


This project demonstrates:
- Document parsing and text preprocessing at scale
- Dense vector embeddings and semantic search
- RAG architecture design and optimization
- Evaluation frameworks for LLM outputs
- Local LLM deployment and inference
- Production system considerations

Perfect for learning RAG concepts hands-on with real evaluation data!

---

<div align="center">



[⬆ back to top](#local-retrieval-augmented-generation-rag-pipeline)

</div>
