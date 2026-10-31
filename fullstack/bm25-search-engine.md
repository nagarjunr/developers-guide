# BM25 Search Engine Implementation

A complete guide to building a BM25-based search engine for document retrieval, understanding the algorithm, and implementing it with Python.

## Table of Contents

1. [Introduction](#introduction)
2. [Understanding BM25](#understanding-bm25)
3. [Basic Implementation](#basic-implementation)
4. [Advanced Features](#advanced-features)
5. [Performance Optimization](#performance-optimization)
6. [Comparison with Alternatives](#comparison-with-alternatives)
7. [Use Cases](#use-cases)

---

## Introduction

**BM25 (Best Matching 25)** is a ranking function used in information retrieval to score and rank documents based on keyword relevance. It's the algorithm behind many search engines.

### Why BM25?

✅ **Fast** - No neural networks or embeddings required
✅ **Accurate** - Excellent for keyword-based search
✅ **Simple** - Easy to implement and understand
✅ **Interpretable** - Scores have clear meaning
✅ **No GPU needed** - Pure CPU algorithm

---

## Understanding BM25

### The Formula

```
BM25(D, Q) = Σ IDF(qi) · (f(qi, D) · (k1 + 1)) / (f(qi, D) + k1 · (1 - b + b · |D| / avgdl))

Where:
- D = document
- Q = query
- qi = query term i
- f(qi, D) = frequency of qi in D
- |D| = document length (words)
- avgdl = average document length in corpus
- k1 = term frequency saturation (default: 1.5)
- b = length normalization (default: 0.75)
```

### Key Components

**1. IDF (Inverse Document Frequency)**
```
IDF(qi) = log((N - n(qi) + 0.5) / (n(qi) + 0.5) + 1)

Where:
- N = total documents
- n(qi) = documents containing qi
```

**Intuition:** Rare terms are more important than common terms.

**2. Term Frequency (TF)**
```
TF component = (f(qi, D) · (k1 + 1)) / (f(qi, D) + k1 · ...)
```

**Intuition:** Documents with more occurrences of query terms rank higher, but with diminishing returns.

**3. Length Normalization**
```
Length factor = (1 - b + b · |D| / avgdl)
```

**Intuition:** Penalize very long documents (they naturally have more term occurrences).

---

## Basic Implementation

### Install Dependencies

```bash
pip install rank-bm25
```

### Simple BM25 Search

```python
# bm25_simple.py
from rank_bm25 import BM25Okapi
import re

def tokenize(text: str) -> list[str]:
    """Simple tokenization: lowercase + split."""
    return re.findall(r'\w+', text.lower())

# Sample corpus
documents = [
    "The quick brown fox jumps over the lazy dog",
    "Never jump over the lazy dog quickly",
    "A quick brown dog outpaces a quick fox",
]

# Tokenize corpus
tokenized_corpus = [tokenize(doc) for doc in documents]

# Create BM25 object
bm25 = BM25Okapi(tokenized_corpus)

# Query
query = "quick fox"
tokenized_query = tokenize(query)

# Get scores for all documents
scores = bm25.get_scores(tokenized_query)
print("Scores:", scores)
# Output: [2.45, 0.0, 1.5] (document 0 is most relevant)

# Get top N documents
top_n = bm25.get_top_n(tokenized_query, documents, n=2)
print("Top 2:", top_n)
# Output: ["The quick brown fox...", "A quick brown dog..."]
```

---

## Full Search Engine Implementation

```python
# search_engine.py
from rank_bm25 import BM25Okapi
from pathlib import Path
from typing import List, Dict, Any
import re

class BM25SearchEngine:
    """BM25-based search engine for document retrieval."""

    def __init__(self):
        self.documents: List[Dict[str, Any]] = []
        self.bm25 = None
        self.tokenized_corpus = []

    def tokenize(self, text: str) -> List[str]:
        """
        Tokenize text into words.

        Strategy: Lowercase + extract alphanumeric sequences
        """
        return re.findall(r'\w+', text.lower())

    def index_documents(self, documents: List[Dict[str, Any]]):
        """
        Index documents for search.

        Args:
            documents: List of dicts with keys:
                - content (str): Document text
                - source (str): Document name/ID
                - metadata (dict): Additional metadata
        """
        self.documents = documents

        # Tokenize all documents
        self.tokenized_corpus = [
            self.tokenize(doc["content"])
            for doc in documents
        ]

        # Build BM25 index
        self.bm25 = BM25Okapi(self.tokenized_corpus)

        print(f"Indexed {len(documents)} documents")

    def search(self, query: str, top_k: int = 5) -> List[Dict[str, Any]]:
        """
        Search for documents matching query.

        Args:
            query: Search query string
            top_k: Number of results to return

        Returns:
            List of documents with scores, sorted by relevance
        """
        if not self.bm25:
            raise ValueError("No documents indexed. Call index_documents() first.")

        # Tokenize query
        tokenized_query = self.tokenize(query)

        # Get scores
        scores = self.bm25.get_scores(tokenized_query)

        # Get top-k indices
        top_indices = sorted(
            range(len(scores)),
            key=lambda i: scores[i],
            reverse=True
        )[:top_k]

        # Return documents with scores
        results = []
        for idx in top_indices:
            result = self.documents[idx].copy()
            result["score"] = scores[idx]
            results.append(result)

        return results

    def load_from_directory(self, directory: Path):
        """
        Load and index all text files from a directory.

        Args:
            directory: Path to directory containing text files
        """
        documents = []

        for file_path in directory.rglob("*.txt"):
            with open(file_path, 'r', encoding='utf-8') as f:
                content = f.read()

            documents.append({
                "content": content,
                "source": file_path.name,
                "path": str(file_path)
            })

        self.index_documents(documents)

# Usage
if __name__ == "__main__":
    # Initialize
    engine = BM25SearchEngine()

    # Load documents
    engine.load_from_directory(Path("./documents"))

    # Search
    results = engine.search("machine learning algorithms", top_k=3)

    # Display
    for i, result in enumerate(results):
        print(f"\n{i+1}. {result['source']} (score: {result['score']:.2f})")
        print(f"   {result['content'][:200]}...")
```

---

## Advanced Features

### 1. Custom Tokenization

```python
import nltk
from nltk.corpus import stopwords
from nltk.stem import PorterStemmer

class AdvancedTokenizer:
    """Advanced tokenizer with stopword removal and stemming."""

    def __init__(self):
        # Download NLTK data (run once)
        # nltk.download('stopwords')
        # nltk.download('punkt')

        self.stopwords = set(stopwords.words('english'))
        self.stemmer = PorterStemmer()

    def tokenize(self, text: str) -> List[str]:
        """
        Advanced tokenization:
        1. Lowercase
        2. Extract words
        3. Remove stopwords
        4. Stem words
        """
        # Lowercase and extract words
        words = re.findall(r'\w+', text.lower())

        # Remove stopwords and stem
        tokens = [
            self.stemmer.stem(word)
            for word in words
            if word not in self.stopwords and len(word) > 2
        ]

        return tokens

# Usage
tokenizer = AdvancedTokenizer()
engine = BM25SearchEngine()
engine.tokenize = tokenizer.tokenize  # Override tokenization
```

### 2. Phrase Search

```python
def search_phrase(self, phrase: str, top_k: int = 5) -> List[Dict]:
    """
    Search for exact phrase.

    Uses BM25 for initial retrieval, then filters for exact matches.
    """
    # Get BM25 candidates
    candidates = self.search(phrase, top_k=top_k * 3)

    # Filter for exact phrase
    results = []
    phrase_lower = phrase.lower()

    for candidate in candidates:
        if phrase_lower in candidate["content"].lower():
            results.append(candidate)

        if len(results) >= top_k:
            break

    return results
```

### 3. Filter by Metadata

```python
def search_with_filter(
    self,
    query: str,
    filters: Dict[str, Any],
    top_k: int = 5
) -> List[Dict]:
    """
    Search with metadata filters.

    Example:
        filters = {"category": "research", "year": 2024}
    """
    # Get all results
    all_results = self.search(query, top_k=len(self.documents))

    # Apply filters
    filtered = []
    for result in all_results:
        metadata = result.get("metadata", {})

        # Check if all filters match
        if all(metadata.get(k) == v for k, v in filters.items()):
            filtered.append(result)

        if len(filtered) >= top_k:
            break

    return filtered
```

### 4. Highlighting Matches

```python
import re

def highlight_matches(text: str, query_terms: List[str]) -> str:
    """
    Highlight query terms in text.

    Returns text with <mark>...</mark> around matches.
    """
    highlighted = text

    for term in query_terms:
        # Case-insensitive replacement
        pattern = re.compile(re.escape(term), re.IGNORECASE)
        highlighted = pattern.sub(lambda m: f"<mark>{m.group()}</mark>", highlighted)

    return highlighted

# Usage
def search_with_highlights(self, query: str, top_k: int = 5):
    """Search and highlight matches."""
    results = self.search(query, top_k)
    query_terms = self.tokenize(query)

    for result in results:
        result["highlighted"] = highlight_matches(
            result["content"],
            query_terms
        )

    return results
```

---

## Performance Optimization

### 1. Index Serialization

```python
import pickle

class BM25SearchEngine:
    # ... (previous code)

    def save_index(self, filepath: str):
        """Save index to disk."""
        data = {
            "documents": self.documents,
            "tokenized_corpus": self.tokenized_corpus,
            "bm25": self.bm25
        }

        with open(filepath, 'wb') as f:
            pickle.dump(data, f)

        print(f"Index saved to {filepath}")

    def load_index(self, filepath: str):
        """Load index from disk."""
        with open(filepath, 'rb') as f:
            data = pickle.load(f)

        self.documents = data["documents"]
        self.tokenized_corpus = data["tokenized_corpus"]
        self.bm25 = data["bm25"]

        print(f"Index loaded: {len(self.documents)} documents")

# Usage
engine = BM25SearchEngine()
engine.load_from_directory(Path("./documents"))
engine.save_index("index.pkl")

# Later...
new_engine = BM25SearchEngine()
new_engine.load_index("index.pkl")
```

### 2. Batch Indexing

```python
def index_documents_batch(
    self,
    documents: List[Dict],
    batch_size: int = 1000
):
    """
    Index documents in batches (for large corpora).
    """
    all_tokenized = []

    for i in range(0, len(documents), batch_size):
        batch = documents[i:i + batch_size]

        # Tokenize batch
        batch_tokenized = [
            self.tokenize(doc["content"])
            for doc in batch
        ]

        all_tokenized.extend(batch_tokenized)

        print(f"Processed {i + len(batch)}/{len(documents)} documents")

    self.documents = documents
    self.tokenized_corpus = all_tokenized
    self.bm25 = BM25Okapi(all_tokenized)
```

### 3. Query Caching

```python
from functools import lru_cache

class BM25SearchEngine:
    # ... (previous code)

    @lru_cache(maxsize=128)
    def search_cached(self, query: str, top_k: int = 5) -> tuple:
        """Cached search (for repeated queries)."""
        results = self.search(query, top_k)

        # Convert to tuple for caching (list not hashable)
        return tuple(
            (r["source"], r["score"])
            for r in results
        )
```

---

## Comparison with Alternatives

### BM25 vs TF-IDF

| Feature | BM25 | TF-IDF |
|---------|------|--------|
| **Accuracy** | Better (length normalization) | Good |
| **Speed** | Fast | Fast |
| **Tuning** | k1, b parameters | None |
| **Saturation** | Yes (term frequency saturates) | No (linear) |

**Recommendation:** Use BM25 (it's an improvement over TF-IDF).

### BM25 vs Embeddings (Semantic Search)

| Feature | BM25 | Embeddings |
|---------|------|------------|
| **Speed** | ⚡ Very fast | 🐌 Slower (GPU helps) |
| **Setup** | Simple | Complex (model, GPU) |
| **Keyword search** | ✅ Excellent | ❌ Poor |
| **Semantic search** | ❌ Poor | ✅ Excellent |
| **Corpus size** | Best for <50K docs | Scales to millions |
| **Typo handling** | ❌ Poor | ✅ Good |

**When to use each:**
- **BM25:** Technical docs, legal text, exact terminology matters
- **Embeddings:** Conceptual search, multi-lingual, large corpora
- **Hybrid:** Use both! BM25 for initial retrieval + embeddings for re-ranking

---

## Use Cases

### 1. Document Search

```python
# Search internal documentation
engine = BM25SearchEngine()
engine.load_from_directory(Path("./company_docs"))
results = engine.search("expense reimbursement policy")
```

### 2. Code Search

```python
# Search code repositories
def load_code_files(directory: Path):
    """Load code files (py, js, etc.)."""
    documents = []

    for ext in ['.py', '.js', '.java', '.cpp']:
        for file_path in directory.rglob(f"*{ext}"):
            with open(file_path, 'r') as f:
                content = f.read()

            documents.append({
                "content": content,
                "source": file_path.name,
                "path": str(file_path),
                "language": ext[1:]
            })

    return documents

engine = BM25SearchEngine()
documents = load_code_files(Path("./my-project"))
engine.index_documents(documents)

results = engine.search("database connection pool")
```

### 3. Q&A System

```python
# FAQ search
faqs = [
    {
        "content": "Q: How do I reset my password? A: Click Forgot Password...",
        "source": "password_reset.txt"
    },
    {
        "content": "Q: What is the refund policy? A: Full refunds within 30 days...",
        "source": "refund_policy.txt"
    }
]

engine = BM25SearchEngine()
engine.index_documents(faqs)

query = "I forgot my password"
results = engine.search(query, top_k=1)
print(results[0]["content"])
```

---

## Summary

**Key Takeaways:**

1. **BM25 is fast and accurate** for keyword search
2. **Simple to implement** with rank-bm25 library
3. **No GPU required** - pure CPU algorithm
4. **Best for small-medium corpora** (<50K documents)
5. **Combine with fuzzy matching** for typo tolerance
6. **Use embeddings for semantic search** (conceptual queries)

**Implementation Checklist:**

- [ ] Install rank-bm25
- [ ] Implement tokenization function
- [ ] Index documents (tokenize + build BM25 object)
- [ ] Implement search function
- [ ] Add advanced features (filters, highlights, caching)
- [ ] Save/load index for reuse
- [ ] Test with your corpus

## Further Reading

- [BM25 Wikipedia](https://en.wikipedia.org/wiki/Okapi_BM25)
- [rank-bm25 Library](https://github.com/dorianbrown/rank_bm25)
- [Information Retrieval Book](https://nlp.stanford.edu/IR-book/)

---

**Created:** 2026-02-06
**Tags:** #bm25 #search #information-retrieval #ranking #nlp #python
