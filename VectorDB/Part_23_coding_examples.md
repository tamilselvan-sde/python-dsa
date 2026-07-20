# Part 23: Coding Examples

> Author: **Tamilselvan** · ✉️ tamilselvan.sde@gmail.com · 🔗 [LinkedIn](https://www.linkedin.com/in/tamilselvan-ai/)
>

## FAISS

```python
import faiss
import numpy as np
from typing import List, Tuple

class FaissVectorStore:
    def __init__(self, dim: int = 768, index_type: str = "HNSW"):
        self.dim = dim
        self.index_type = index_type
        self.index = None
        self.ids = []
        
    def create_index(self):
        if self.index_type == "HNSW":
            self.index = faiss.IndexHNSWFlat(self.dim, 32)
            self.index.hnsw.efConstruction = 200
        elif self.index_type == "IVF":
            quantizer = faiss.IndexFlatL2(self.dim)
            self.index = faiss.IndexIVFFlat(quantizer, self.dim, 100)
            self.index.nprobe = 10
        elif self.index_type == "IVFPQ":
            quantizer = faiss.IndexFlatL2(self.dim)
            self.index = faiss.IndexIVFPQ(quantizer, self.dim, 100, 96, 8)
            self.index.nprobe = 10
        elif self.index_type == "Flat":
            self.index = faiss.IndexFlatIP(self.dim)
            
    def add(self, vectors: np.ndarray, ids: List[str]):
        if not self.index.is_trained:
            self.index.train(vectors)
        self.index.add_with_ids(vectors, np.array([int(i) for i in ids]))
        self.ids.extend(ids)
        
    def search(self, query: np.ndarray, k: int = 10) -> Tuple[np.ndarray, np.ndarray]:
        return self.index.search(query.reshape(1, -1), k)

# Usage
store = FaissVectorStore(dim=768, index_type="HNSW")
store.create_index()

vectors = np.random.random((10000, 768)).astype('float32')
store.add(vectors, [str(i) for i in range(10000)])

query = np.random.random(768).astype('float32')
distances, indices = store.search(query, k=5)
```

## Qdrant

```python
from qdrant_client import QdrantClient, models
from qdrant_client.models import PointStruct, VectorParams, Distance

client = QdrantClient(host="localhost", port=6333)

# Create collection
client.create_collection(
    collection_name="documents",
    vectors_config=VectorParams(size=768, distance=Distance.COSINE),
    optimizers_config=models.OptimizersConfigDiff(
        indexing_threshold=10000,
        memmap_threshold=20000,
    ),
)

# Batch insert
points = [
    PointStruct(
        id=i,
        vector=np.random.random(768).tolist(),
        payload={"text": f"Document {i}", "category": "tech"}
    )
    for i in range(1000)
]
client.upsert(
    collection_name="documents",
    points=points,
    wait=True,
)

# Search with filter
results = client.search(
    collection_name="documents",
    query_vector=np.random.random(768).tolist(),
    query_filter=models.Filter(
        must=[
            models.FieldCondition(
                key="category",
                match=models.MatchValue(value="tech")
            )
        ]
    ),
    limit=10,
    with_payload=True,
)
```

## Milvus

```python
from pymilvus import (
    connections, Collection, FieldSchema, 
    CollectionSchema, DataType, utility
)

connections.connect(host="localhost", port="19530")

# Create schema
fields = [
    FieldSchema("id", DataType.INT64, is_primary=True, auto_id=False),
    FieldSchema("vector", DataType.FLOAT_VECTOR, dim=768),
    FieldSchema("text", DataType.VARCHAR, max_length=65535),
    FieldSchema("category", DataType.VARCHAR, max_length=100),
]
schema = CollectionSchema(fields, "Document collection")
collection = Collection("documents", schema)

# Create index
index_params = {
    "metric_type": "COSINE",
    "index_type": "IVF_FLAT",
    "params": {"nlist": 100}
}
collection.create_index("vector", index_params)
collection.load()

# Insert
import random
vectors = [[random.random() for _ in range(768)] for _ in range(1000)]
entities = [
    [i for i in range(1000)],  # id
    vectors,
    [f"doc_{i}" for i in range(1000)],
    ["tech" for _ in range(1000)]
]
collection.insert(entities)

# Search
search_params = {"metric_type": "COSINE", "params": {"nprobe": 10}}
results = collection.search(
    data=[vectors[0]],
    anns_field="vector",
    param=search_params,
    limit=10,
    expr="category == 'tech'",
    output_fields=["text"]
)
```

## Chroma

```python
import chromadb

client = chromadb.PersistentClient(path="./chroma_db")

# Create collection
collection = client.create_collection(
    name="documents",
    metadata={"hnsw:space": "cosine"}
)

# Add documents (auto-embeds)
collection.add(
    documents=[
        "Vector databases store embeddings",
        "Machine learning is transforming AI",
        "HNSW is a popular ANN algorithm",
    ],
    metadatas=[
        {"category": "vector-db"},
        {"category": "ai"},
        {"category": "algorithms"},
    ],
    ids=["doc1", "doc2", "doc3"]
)

# Query (auto-embeds query)
results = collection.query(
    query_texts=["What are vector databases?"],
    n_results=2,
    where={"category": "vector-db"}
)

print(results['documents'])
```

## pgvector

```python
import psycopg2
import numpy as np

conn = psycopg2.connect("dbname=vectordb")
cur = conn.cursor()

# Enable extension
cur.execute("CREATE EXTENSION IF NOT EXISTS vector")

# Create table and index (run once)
cur.execute("""
    CREATE TABLE IF NOT EXISTS documents (
        id SERIAL PRIMARY KEY,
        text TEXT,
        embedding vector(768)
    )
""")
cur.execute("""
    CREATE INDEX IF NOT EXISTS docs_ivf_idx 
    ON documents 
    USING ivfflat (embedding vector_cosine_ops)
    WITH (lists = 100)
""")
conn.commit()

# Insert
embedding = np.random.random(768).tolist()
cur.execute(
    "INSERT INTO documents (text, embedding) VALUES (%s, %s::vector)",
    ("Machine learning is fun", embedding)
)
conn.commit()

# Search
query_embedding = np.random.random(768).tolist()
cur.execute("""
    SELECT text, 1 - (embedding <=> %s::vector) AS similarity
    FROM documents
    ORDER BY embedding <=> %s::vector
    LIMIT 10
""", (query_embedding, query_embedding))
results = cur.fetchall()
```

## LanceDB

```python
import lancedb
import numpy as np
import pyarrow as pa

db = lancedb.connect("./lancedb_data")

# Create table
data = [
    {"vector": np.random.random(768).tolist(), "text": f"doc {i}", "id": i}
    for i in range(1000)
]
table = db.create_table("documents", data)

# Create index
table.create_index(
    metric="cosine",
    num_partitions=256,
    num_sub_vectors=96,
)

# Search
results = (
    table.search(np.random.random(768).tolist())
    .metric("cosine")
    .limit(10)
    .to_pandas()
)
```

## OpenSearch

```python
from opensearchpy import OpenSearch

client = OpenSearch(hosts=[{"host": "localhost", "port": 9200}])

# Create index with k-NN
index_body = {
    "settings": {
        "index.knn": True
    },
    "mappings": {
        "properties": {
            "vector": {
                "type": "knn_vector",
                "dimension": 768,
                "method": {
                    "name": "hnsw",
                    "space_type": "cosinesimil",
                    "engine": "nmslib",
                    "parameters": {
                        "ef_construction": 200,
                        "m": 16
                    }
                }
            },
            "text": {"type": "text"},
            "category": {"type": "keyword"}
        }
    }
}
client.indices.create("documents", body=index_body)

# Insert
import random
doc = {
    "vector": [random.random() for _ in range(768)],
    "text": "Vector search is powerful",
    "category": "tech"
}
client.index(index="documents", body=doc, id=1, refresh=True)

# Search
query = {
    "size": 10,
    "query": {
        "knn": {
            "vector": {
                "vector": [random.random() for _ in range(768)],
                "k": 10
            }
        }
    }
}
results = client.search(index="documents", body=query)
```

---

