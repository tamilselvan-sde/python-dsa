# Part 16: Filtering

> Author: **Tamilselvan** · ✉️ tamilselvan.sde@gmail.com · 🔗 [LinkedIn](https://www.linkedin.com/in/tamilselvan-ai/)
>

## Metadata Filtering

Vector databases support structured filtering alongside vector similarity search.

### Filter Types by Database

| Filter Type | Qdrant | Milvus | Pinecone | Weaviate | Elastic |
|-------------|--------|--------|----------|----------|---------|
| **Equality** | `MatchValue` | `==` | `$eq` | `Equal` | `term` |
| **Range** | `Range` | `>` `<` | `$gt` `$lt` | `GreaterThan` | `range` |
| **List/IN** | `MatchAny` | `in` | `$in` | `ContainsAny` | `terms` |
| **Boolean** | `MatchValue` | `==` | `$eq` | `Equal` | `term` |
| **Geo** | `GeoRadius` | `geo_distance` | Not built-in | `GeoRange` | `geo_shape` |
| **Text** | `MatchText` | `like` | Not built-in | `Like` | `match` |
| **Exists** | `IsEmpty` | `exists` | `$exists` | `Not` | `exists` |
| **Nested** | `NestedCondition` | `json_contains` | Limited | Limited | `nested` |

---

## Boolean, Range, Geo Filters

### Boolean Filter
```python
# Find published documents
{
    "is_published": True
}
```

### Range Filter
```python
# Find recent documents between page 10-50
{
    "date": {"$gte": "2024-01-01", "$lte": "2024-12-31"},
    "page_number": {"$gte": 10, "$lte": 50}
}
```

### Geo Filter
```python
# Find nearby points of interest
{
    "location": {
        "$geo_radius": {
            "center": [40.7128, -74.0060],
            "radius": 10000  # meters
        }
    }
}
```

### Compound Filter (AND/OR/NOT)
```python
{
    "$and": [
        {"tenant_id": "company_a"},
        {"$or": [
            {"category": "engineering"},
            {"category": "research"}
        ]},
        {"$not": {"status": "archived"}}
    ]
}
```

---

## Tenant Isolation

### Strategy 1: Filter by Tenant ID
```python
# Add tenant_id to every vector
vectors.append({
    "id": doc_id,
    "vector": embedding,
    "payload": {
        "tenant_id": tenant_id,
        "text": chunk_text,
        ...
    }
})

# Always filter by tenant_id in queries
results = db.search(
    query_vector=query_emb,
    filter={
        "tenant_id": {"$eq": current_tenant}
    }
)
```

### Strategy 2: Separate Collections
```python
# One collection per tenant
for tenant in tenants:
    db.create_collection(
        collection_name=f"docs_{tenant}",
        vectors_config=VectorParams(size=768, distance=Distance.COSINE)
    )

# Query tenant-specific collection
results = db.search(
    collection_name=f"docs_{current_tenant}",
    query_vector=query_emb
)
```

### Strategy 3: Namespaces (Pinecone)
```python
# One index, multiple namespaces
index.query(
    vector=query_emb,
    top_k=10,
    namespace=f"tenant_{tenant_id}"  # Isolated namespace
)
```

---

### Production Tip

> **Multi-tenant performance considerations:**
> - **Metadata filter:** Best for many small tenants. Each query still scans the index but narrows results via filter.
> - **Separate collections:** Best for few large tenants. Each collection gets its own index, optimized for that tenant's data size.
> - **Separate DB instances:** Best for compliance/isolation requirements. Most expensive.

---

