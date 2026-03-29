---
name: use-milvus
description: how to use Milvus vector database for embeddings storage and search
license: Apache 2.0
---

To use Milvus in an endpoint, first use the tool `add-milvus` to add the connection to your endpoint. This makes `ctx.MILVUS` available as a `MilvusClient` instance.

For generating embeddings, use the tool `add-secret` to inject these OpenAI-compatible secrets into your endpoint:

- `OPENAI_API_KEY` — the API key
- `OPENAI_BASE_URL` — the base URL of the OpenAI-compatible server (e.g. `https://api.openai.com/v1` or `http://localhost:11434/v1` for Ollama)
- `OPENAI_MODEL` — the embedding model name (e.g. `text-embedding-3-small`)

## Generating embeddings with OpenAI

Use the OpenAI SDK to generate embeddings. The dimension must match your Milvus collection schema.

```python
from openai import OpenAI

DIMENSION = 1536  # text-embedding-3-small default; use 1024 for text-embedding-3-large with dimensions param

def generate_embedding(ctx, text):
    ai = OpenAI(base_url=ctx.OPENAI_BASE_URL, api_key=ctx.OPENAI_API_KEY)
    response = ai.embeddings.create(
        model=ctx.OPENAI_MODEL,
        input=text,
    )
    return response.data[0].embedding
```

You can also generate embeddings for multiple texts in a single call:

```python
def generate_embeddings(ctx, texts):
    ai = OpenAI(base_url=ctx.OPENAI_BASE_URL, api_key=ctx.OPENAI_API_KEY)
    response = ai.embeddings.create(
        model=ctx.OPENAI_MODEL,
        input=texts,
    )
    return [item.embedding for item in response.data]
```

## Creating a collection with vector schema

Define a collection with an auto-increment primary key, a text field, and a vector field. The `dim` parameter must match the dimension of your embedding model.

```python
from pymilvus import DataType

COLLECTION = "my_collection"
DIMENSION = 1536

def create_collection(db):
    if COLLECTION not in db.list_collections():
        schema = db.create_schema()
        schema.add_field(field_name="id", datatype=DataType.INT64, is_primary=True, auto_id=True)
        schema.add_field(field_name="text", datatype=DataType.VARCHAR, max_length=65535)
        schema.add_field(field_name="embeddings", datatype=DataType.FLOAT_VECTOR, dim=DIMENSION)

        index_params = db.prepare_index_params()
        index_params.add_index("embeddings", index_type="AUTOINDEX", metric_type="IP")
        db.create_collection(collection_name=COLLECTION, schema=schema, index_params=index_params)
```

You can add extra fields to store metadata alongside each vector:

```python
schema.add_field(field_name="source", datatype=DataType.VARCHAR, max_length=512)
schema.add_field(field_name="timestamp", datatype=DataType.INT64)
```

Common metric types for the index:
- `IP` — Inner Product (best when embeddings are normalized, e.g. OpenAI models)
- `L2` — Euclidean distance
- `COSINE` — Cosine similarity

## Inserting vectors

Generate the embedding and insert it together with the text into the collection:

```python
def main(args, ctx=None):
    db = ctx.MILVUS
    create_collection(db)

    text = args.get("input", "")
    vec = generate_embedding(ctx, text)
    result = db.insert(COLLECTION, [{"text": text, "embeddings": vec}])
    return f"Inserted: {result.get('insert_count', 0)}"
```

For bulk insertion, batch the texts and embeddings together:

```python
def insert_batch(db, ctx, texts):
    vecs = generate_embeddings(ctx, texts)
    data = [{"text": t, "embeddings": v} for t, v in zip(texts, vecs)]
    result = db.insert(COLLECTION, data)
    return f"Inserted: {result.get('insert_count', 0)}"
```

## Searching by vector similarity

Embed the query text, then search the collection for the closest vectors:

```python
def search(db, ctx, query_text, limit=5):
    query_vec = generate_embedding(ctx, query_text)
    results = db.search(
        collection_name=COLLECTION,
        search_params={"metric_type": "IP"},
        limit=limit,
        anns_field="embeddings",
        data=[query_vec],
        output_fields=["text"],
    )
    out = ""
    for item in results[0]:
        text = item.get("entity", {}).get("text", "")
        dist = item.get("distance", 0)
        out += f"- ({dist:.2f}) {text}\n"
    return out or "Not found"
```

## Deleting entries

Delete entries by matching a substring in their text field:

```python
def delete_matching(db, substring):
    ids = []
    qit = db.query_iterator(collection_name=COLLECTION, batchSize=100, output_fields=["text"])
    res = qit.next()
    while len(res) > 0:
        for ent in res:
            if substring in ent.get("text", ""):
                ids.append(ent.get("id"))
        res = qit.next()
    result = db.delete(collection_name=COLLECTION, ids=ids)
    return f"Deleted: {result['delete_count']}"
```

## Dropping a collection

```python
db.drop_collection("my_collection")
```
