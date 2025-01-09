---
layout: post
title: 构建基于 FastAPI 的简单语义搜索服务
categories: [ fastapi, rag ]
description: some word here
keywords: rag, fastapi
mermaid: false
sequence: false
flow: false
mathjax: false
mindmap: false
mindmap2: false
---

# 构建基于 FastAPI 的简单语义搜索服务

## 背景与目标

随着自然语言处理（NLP）和向量化技术的进步，构建能够理解和处理语义查询的服务变得更加容易。本项目使用 FastAPI
框架、SentenceTransformer 模型和 MongoDB 数据库，实现了一个简洁高效的语义搜索服务。

本文将总结代码的实现思路和亮点，帮助读者理解如何快速构建类似的服务。

---

## 核心功能

本项目的主要功能包括：

1. **文档向量化存储**：将文本内容向量化后存入 MongoDB。
2. **语义检索**：基于余弦相似度，检索与查询最相关的文档，并返回相似度分数。
3. **动态文档添加**：通过 API 动态添加新的文档至知识库。

---

## 技术栈

- **FastAPI**：用于构建 RESTful API。
- **SentenceTransformer**：生成文本向量的预训练模型。
- **MongoDB**：存储文档及其对应的向量。
- **NumPy**：用于计算向量之间的余弦相似度。

---

## 代码结构与实现

### 1. 初始化向量化模型

使用 `SentenceTransformer` 的预训练模型 `all-MiniLM-L6-v2`，将文本内容转换为高维向量。此模型兼具精度和性能，非常适合轻量级应用。

```python
model = SentenceTransformer('all-MiniLM-L6-v2')
```

### 2. 数据存储与初始化

知识库存储在 MongoDB 中。文档内容被向量化后，与原始内容一起存储。

#### 数据初始化

以下函数用于将初始文档插入数据库：

```python
def populate_knowledge_base():
    documents = [
        {"content": "MongoDB is a NoSQL database that stores data in JSON-like documents."},
        {"content": "RAG combines retrieval and generation for enhanced QA performance."},
        {"content": "Atlas Search in MongoDB supports efficient text and vector queries."}
    ]

    for doc in documents:
        vector = model.encode(doc["content"]).tolist()
        doc["vector"] = vector
        collection.insert_one(doc)

    print("Knowledge base populated with vectorized documents.")
```

### 3. 文档检索

通过以下步骤实现语义检索：

1. 向量化查询内容。
2. 计算查询向量与每个文档向量的余弦相似度。
3. 按相似度排序并返回结果。

核心代码如下：

```python
def retrieve_documents(query: str, top_k: int = 3) -> List[dict]:
    query_vector = model.encode(query).tolist()

    all_documents = collection.find()
    scored_docs = []
    for doc in all_documents:
        similarity = cosine_similarity(query_vector, doc["vector"])
        scored_docs.append({"content": doc["content"], "similarity": similarity})

    scored_docs = sorted(scored_docs, key=lambda x: x["similarity"], reverse=True)
    return scored_docs[:top_k]
```

### 4. 插入文档

新增文档时，服务会自动向量化内容并存储。

```python
def insert_document(content: str):
    vector = model.encode(content).tolist()
    document = {"content": content, "vector": vector}
    collection.insert_one(document)
```

### 5. API 路由

FastAPI 提供了两个主要路由：

- **查询知识库**：

  ```python
  @app.post("/query")
  async def query_knowledge_base(query: Query):
      results = retrieve_documents(query.query, query.top_k)
      return {
          "query": query.query,
          "results": [
              {"content": result["content"], "similarity": result["similarity"]}
              for result in results
          ]
      }
  ```

- **插入新文档**：

  ```python
  @app.post("/insert")
  async def add_document_to_knowledge_base(document: Document):
      insert_document(document.content)
      return {"message": "Document inserted successfully."}
  ```

---

## 思路亮点

1. **轻量高效**：
    - 使用 SentenceTransformer 提供高效的文本向量化。
    - MongoDB 存储 JSON 数据，自然支持扩展。

2. **动态性强**：
    - 支持动态插入新文档，知识库内容可随时更新。

3. **相似度返回**：
    - 检索结果中包含相似度分数，便于评估文档相关性。

4. **易扩展性**：
    - 可进一步集成更多模型或支持多种查询方式。

---

## 测试数据

### 插入文档

```json
POST /insert
{
  "content": "FastAPI is a modern web framework for building APIs with Python."
}
```

### 查询示例

```json
POST /query
{
  "query": "Explain MongoDB and its features.",
  "top_k": 2
}
```

### 响应示例

```json
{
  "query": "Explain MongoDB and its features.",
  "results": [
    {
      "content": "MongoDB is a NoSQL database that stores data in JSON-like documents.",
      "similarity": 0.94
    },
    {
      "content": "Atlas Search in MongoDB supports efficient text and vector queries.",
      "similarity": 0.87
    }
  ]
}
```

---

## 总结

本项目展示了如何利用 FastAPI 和 NLP 技术快速搭建语义搜索服务。其代码清晰、逻辑简洁，适合初学者学习和实际应用场景扩展。

