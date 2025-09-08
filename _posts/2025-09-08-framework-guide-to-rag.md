---
layout: post
title: "將框架文件轉成 RAG 知識庫 - 讓 Local LLM 隨時檢索"
date: 2025-09-08
categories: [技術筆記, AI]
tags: [RAG, ChromaDB, Local LLM, .NET Core]
---

## 上週的問題:文件太長了

上週寫完 6000 多行的框架文件,興沖沖地丟給 GitHub Copilot (Claude Sonnet 4.5) 測試,發現一個問題:**每次都要把整份文件貼上去**。

而且 Local LLM (DeepSeek-V2.5 16B) 的 Context Window 雖然有 128K,但把整份文件塞進去:
1. **速度慢** - 推理時間從 2 秒變 8 秒
2. **容易跑題** - 資訊太多,AI 有時會混淆
3. **成本高** - 雲端 API 按 Token 數計費,文件越長越貴

理想的狀況是:**AI 只看需要的部分**。

例如我問「怎麼寫 Controller」,它就去查 Controller 章節;問「怎麼處理例外」,就查例外處理章節。

**這就是 RAG (Retrieval-Augmented Generation) 的用途**。

## RAG 快速複習

之前做過 RAG 實驗 (8 月第三週),但那次是處理「公司舊文件」和「Slack 對話紀錄」。這次要處理的是**結構化的技術文件**,有些不一樣。

RAG 核心流程:
1. **Chunking**: 把長文件切成小塊
2. **Embedding**: 每塊轉成向量 (Vector)
3. **Storage**: 存入 Vector Database (我用 ChromaDB)
4. **Retrieval**: 使用者提問時,找出最相關的幾塊
5. **Generation**: 把相關內容 + 問題一起丟給 LLM

## 第一步:切分文件 (Chunking)

### 原本的想法:直接按行數切

最簡單的方式是「每 500 行切一塊」,但問題很大:

```markdown
... (第 499 行)
public class UserService : BaseSystemService
{
    private readonly IUserRepository _userRepository;
--- 切在這裡 ---
    
    public UserService(IUserRepository userRepository)
    {
        _userRepository = userRepository;
    }
```

代碼被切斷了,AI 看到片段會看不懂。

### 改進:按章節切分

我的文件本來就有清楚的結構:

```
# .NET Core 8 框架指南
## 1. 專案架構
### 1.1 整體架構圖
### 1.2 分層架構
## 2. 目錄結構
## 3. 核心元件
### 3.1 Web API 層
#### 3.1.1 Program.cs
#### 3.1.2 BaseController
...
```

**解法**:按 `##` (二級標題) 切分,每個章節獨立成一塊。

```python
def chunk_by_sections(markdown_content):
    chunks = []
    current_chunk = ""
    current_title = ""
    
    for line in markdown_content.split('\n'):
        # 遇到二級標題,切分
        if line.startswith('## '):
            if current_chunk:
                chunks.append({
                    'title': current_title,
                    'content': current_chunk.strip()
                })
            current_title = line.replace('## ', '')
            current_chunk = line + '\n'
        else:
            current_chunk += line + '\n'
    
    # 最後一塊
    if current_chunk:
        chunks.append({
            'title': current_title,
            'content': current_chunk.strip()
        })
    
    return chunks
```

結果切成 **35 個 chunks**:
- 專案架構 (1 chunk)
- 目錄結構 (1 chunk)
- 核心元件 - Web API 層 (4 chunks)
- 核心元件 - Application 層 (3 chunks)
- 核心元件 - Domain 層 (4 chunks)
- 核心元件 - Infrastructure 層 (3 chunks)
- 常用模式 (7 chunks)
- 快速開始範本 (8 chunks)
- 最佳實踐 (4 chunks)

每個 chunk 大小在 **200-800 行**,剛好適合 LLM 處理。

## 第二步:Embedding

用 `sentence-transformers` 的 `all-MiniLM-L6-v2` 模型:

```python
from sentence_transformers import SentenceTransformer

model = SentenceTransformer('all-MiniLM-L6-v2')

embeddings = []
for chunk in chunks:
    # 把標題和內容結合,增加檢索準確度
    text = f"{chunk['title']}\n\n{chunk['content']}"
    embedding = model.encode(text)
    embeddings.append(embedding)
```

這個模型只有 **80MB**,在我的 Mac Mini Pro 上跑超快 (每個 embedding 只要 20ms)。

## 第三步:存入 ChromaDB

```python
import chromadb

# 初始化 ChromaDB
client = chromadb.PersistentClient(path="./chromadb_data")
collection = client.get_or_create_collection(
    name="dotnet_framework_guide",
    metadata={"description": ".NET Core 8 Framework Documentation"}
)

# 存入資料
for i, chunk in enumerate(chunks):
    collection.add(
        ids=[f"chunk_{i}"],
        embeddings=[embeddings[i].tolist()],
        documents=[chunk['content']],
        metadatas=[{
            'title': chunk['title'],
            'section': chunk['title'].split('.')[0]  # 例如 "3" 代表核心元件
        }]
    )

print(f"✅ 成功存入 {len(chunks)} 個文件片段")
```

ChromaDB 會自動建立索引,查詢速度很快。

## 第四步:測試檢索效果

寫個簡單的查詢函數:

```python
def search_framework_guide(query, top_k=3):
    # 將問題轉成 embedding
    query_embedding = model.encode(query)
    
    # 從 ChromaDB 檢索最相關的 chunks
    results = collection.query(
        query_embeddings=[query_embedding.tolist()],
        n_results=top_k
    )
    
    return results['documents'][0], results['metadatas'][0]
```

### 測試 Case 1: 如何寫 Controller?

```python
docs, metas = search_framework_guide("如何建立一個新的 Controller", top_k=3)

# 檢索結果:
# 1. "3.1 Web API 層" (相似度: 0.82)
# 2. "5.1 新增 CRUD API 完整流程" (相似度: 0.79)
# 3. "3.1.2 BaseController" (相似度: 0.76)
```

✅ 完美!前三名都是相關內容。

### 測試 Case 2: 錯誤處理怎麼做?

```python
docs, metas = search_framework_guide("如何處理例外和錯誤", top_k=3)

# 檢索結果:
# 1. "4.1 錯誤處理" (相似度: 0.88)
# 2. "4.1.1 全域例外過濾器" (相似度: 0.83)
# 3. "4.1.2 例外處理鏈" (相似度: 0.81)
```

✅ 也很準!

### 測試 Case 3: JWT 認證怎麼用?

```python
docs, metas = search_framework_guide("如何實作 JWT 認證", top_k=3)

# 檢索結果:
# 1. "4.4 JWT 認證" (相似度: 0.91)
# 2. "4.4.2 JWT Manager" (相似度: 0.85)
# 3. "3.2.1 BaseService" (相似度: 0.72)
```

✅ 第三名抓到 BaseService,因為它有從 JWT Token 解析使用者資訊的邏輯。合理!

## 整合到 Continue.dev

Continue.dev 是 VS Code 的 Local LLM 插件 (之前 8 月第二週用過)。它支援自訂 Context Provider。

### 設定 config.json

```json
{
  "models": [
    {
      "title": "DeepSeek Coder",
      "provider": "ollama",
      "model": "deepseek-coder-v2:16b-lite-instruct-q8_0"
    }
  ],
  "contextProviders": [
    {
      "name": "framework-guide",
      "params": {
        "endpoint": "http://localhost:5000/search",
        "description": ".NET Core 8 Framework Guide"
      }
    }
  ]
}
```

### 建立 RAG API Server

用 FastAPI 快速建一個檢索 API:

```python
from fastapi import FastAPI
from pydantic import BaseModel

app = FastAPI()

class SearchRequest(BaseModel):
    query: str
    top_k: int = 3

@app.post("/search")
async def search(request: SearchRequest):
    docs, metas = search_framework_guide(request.query, request.top_k)
    
    # 格式化成 Continue.dev 需要的格式
    context = "\n\n---\n\n".join([
        f"# {meta['title']}\n\n{doc}" 
        for doc, meta in zip(docs, metas)
    ])
    
    return {"context": context}

# 啟動: uvicorn rag_api:app --port 5000
```

### 在 VS Code 測試

在 Continue.dev 的聊天框輸入:

```
@framework-guide 幫我建立一個 Product CRUD API
```

Continue.dev 會:
1. 自動呼叫 `http://localhost:5000/search?query=建立 CRUD API`
2. 拿到相關的框架文件片段
3. 把片段 + 使用者問題一起丟給 DeepSeek
4. DeepSeek 依照框架規範生成代碼

**實測結果**:
- 生成的 Controller 繼承 `SystemBaseController` ✅
- 有用 `HandleResult()` 統一回應格式 ✅
- Service 繼承 `BaseSystemService` ✅
- Repository 有實作 `IBaseSystemRepository` ✅
- Entity 有審計欄位 (CreateUser, UpdateIp) ✅

**推理時間**: 從 8 秒降到 **3 秒** (因為 Context 變小了)

## 遇到的問題

### 問題 1: 檢索結果不夠精準

有時候問「如何實作分頁」,它會返回「BaseRepository」章節,但我其實想看的是「BasePageReq」的用法。

**原因**: Embedding Model 是英文訓練的,對中文技術用語理解不夠好。

**解法**: 
1. 在 chunk 時,把關鍵類別名稱 (例如 `BasePageReq`) 加到 metadata
2. 檢索時優先匹配 metadata,再匹配內容

```python
collection.add(
    ids=[f"chunk_{i}"],
    embeddings=[embeddings[i].tolist()],
    documents=[chunk['content']],
    metadatas=[{
        'title': chunk['title'],
        'keywords': extract_class_names(chunk['content'])  # 提取類別名
    }]
)
```

改進後準確度提升到 **90%+**。

### 問題 2: 有時需要多個 chunks 才能理解

例如問「如何實作完整的 User CRUD」,需要:
- Controller 範例
- Service 範例
- Repository 範例
- AutoMapper 設定

但我只設定 `top_k=3`,可能漏掉某些部分。

**解法**: 動態調整 `top_k`
- 簡單問題 (例如「BaseController 怎麼用」) → `top_k=2`
- 複雜問題 (例如「完整 CRUD」) → `top_k=5`

可以用簡單的規則判斷:

```python
def get_top_k(query):
    complex_keywords = ['完整', '全部', 'CRUD', '流程', '步驟']
    if any(k in query for k in complex_keywords):
        return 5
    return 2
```

## 效能數據

做了一些測試,對比「直接餵完整文件」vs「用 RAG 檢索」:

| 項目 | 完整文件 | RAG 檢索 |
|------|---------|---------|
| Context Size | 6000 行 (~50K tokens) | 800-1500 行 (~8K tokens) |
| 推理時間 | 8-10 秒 | 2-3 秒 |
| 準確度 | 85% (有時會忽略細節) | 90% (更聚焦) |
| 記憶體用量 | ~25GB | ~18GB |

**結論**: RAG 在速度和準確度上都有明顯提升。

## 小結

這週把框架文件轉成 RAG 知識庫,核心改進:
1. **按章節切分**,保持代碼完整性
2. **Embedding + ChromaDB**,快速檢索相關內容
3. **整合 Continue.dev**,讓 Local LLM 能自動查詢框架規範

現在開發新功能時,只要在 VS Code 輸入:
```
@framework-guide 建立 XXX 模組
```

AI 就會自動檢索相關文件,並生成符合規範的代碼。

**下一步**:
- 訓練 Local LLM 更深入理解框架 (可能需要 Fine-tuning)
- 新增「反向查詢」功能:給 AI 一段代碼,讓它檢查是否符合規範
- 建立「常見錯誤」知識庫,讓 AI 自動修正不符規範的代碼

目標是讓 AI 成為真正的「框架專家」,不只會生成代碼,還能做 Code Review。
