---
layout: post
title: "Week 3: RAG 實驗 - 讓 AI 讀懂專案文件"
date: 2025-08-18 11:30:00 +0800
categories: [AI工程, RAG]
tags: [RAG, Embedding, ChromaDB, Vector Database]
---

## RAG 是什麼?為什麼需要它?

上週用 Local LLM 時發現一個問題:它不懂我們專案的 domain knowledge。

比如我問它「OCPI 的 Session.status 有哪些可能的值?」,它會瞎猜,或是給一個很通用的答案。但這個問題的答案就寫在我們的技術文件裡。

RAG (Retrieval-Augmented Generation) 的概念很簡單:**讓 AI 在回答問題前,先去查詢相關資料**。

流程大概是:
1. 使用者問問題
2. 系統把問題轉成向量 (Embedding)
3. 在向量資料庫搜尋相似的文件
4. 把查到的文件餵給 LLM
5. LLM 根據文件內容回答

聽起來很合理,但實際做起來有很多眉角。

## 第一步:文件準備與切分

我整理了這些文件準備餵給系統:
- `docs/ocpi-2.1-spec.md` - OCPI 規範文件 (50 頁)
- `docs/architecture.md` - 系統架構說明
- `README.md` - 專案說明
- 過去 3 個月的重要 PR 討論

**問題來了:這些文件太長怎麼辦?**

一個 50 頁的 PDF 轉成 Markdown 大概 3 萬字,直接丟給 LLM 會爆 Context Window。所以要「切分」(Chunking)。

我用了最簡單的方法:**按標題切**。每個 `## 標題` 就是一個 chunk。

```python
def split_by_headers(markdown_text):
    chunks = []
    current_chunk = ""
    
    for line in markdown_text.split('\n'):
        if line.startswith('## '):
            if current_chunk:
                chunks.append(current_chunk)
            current_chunk = line + '\n'
        else:
            current_chunk += line + '\n'
    
    if current_chunk:
        chunks.append(current_chunk)
    
    return chunks
```

切完之後大概有 120 個 chunks,每個 200-500 字。

## 第二步:Embedding - 把文字變成數字

向量資料庫需要「向量」,不能直接存文字。所以要把每個 chunk 轉成一串數字 (Embedding)。

我用了 `sentence-transformers` 這個 library:

```python
from sentence_transformers import SentenceTransformer

model = SentenceTransformer('all-MiniLM-L6-v2')

embeddings = []
for chunk in chunks:
    vector = model.encode(chunk)
    embeddings.append(vector)
```

每個 chunk 會變成一個 384 維的向量。什麼意思?就是 384 個浮點數組成的陣列。這些數字代表了文字的「語義」。

**神奇的地方**:如果兩段文字意思相近,它們的向量距離就會很近。比如「充電站故障」和「充電樁異常」,雖然用詞不同,但向量很相似。

## 第三步:ChromaDB - 向量資料庫

本來想用 PostgreSQL + pgvector,但要改 DB schema 有點麻煩。後來發現 ChromaDB 超簡單,直接用檔案儲存,不用架資料庫。

```python
import chromadb

client = chromadb.Client()
collection = client.create_collection("project_docs")

# 存入資料
for i, (chunk, embedding) in enumerate(zip(chunks, embeddings)):
    collection.add(
        ids=[f"doc_{i}"],
        embeddings=[embedding.tolist()],
        documents=[chunk]
    )
```

就這樣,120 個文件 chunks 都進向量資料庫了。

## 第四步:查詢測試

現在可以測試「語義搜尋」了:

```python
query = "充電站的 Heartbeat 機制是什麼?"
query_embedding = model.encode(query)

results = collection.query(
    query_embeddings=[query_embedding.tolist()],
    n_results=3
)

for doc in results['documents'][0]:
    print(doc)
```

結果它找到 3 段相關文件:
1. 架構文件中「Heartbeat Service 設計」的段落
2. OCPI 規範中「Status Notification」的說明
3. README 中「系統監控機制」的描述

**完全沒有用關鍵字比對**,而是理解「Heartbeat」、「充電站」、「機制」這些詞的語義,找到相關內容。這比傳統的全文搜尋強太多。

## 整合到 LLM:第一次成功體驗

把 RAG 接到 DeepSeek:

```python
def ask_with_rag(question):
    # 1. 搜尋相關文件
    query_embedding = model.encode(question)
    results = collection.query(
        query_embeddings=[query_embedding.tolist()],
        n_results=3
    )
    context = "\n\n".join(results['documents'][0])
    
    # 2. 組合 prompt
    prompt = f"""
根據以下文件回答問題:

{context}

問題: {question}
"""
    
    # 3. 呼叫 LLM
    response = ollama.generate(
        model='deepseek-coder-v2:16b',
        prompt=prompt
    )
    
    return response['response']
```

測試:

**問題**: 「OCPI 的 Session 有哪些狀態?」

**沒有 RAG 的回答** (瞎猜):
> Session 通常有 ACTIVE, COMPLETED, FAILED 等狀態...

**有 RAG 的回答** (根據文件):
> 根據 OCPI 2.1 規範,Session.status 包含:ACTIVE (充電中)、COMPLETED (正常結束)、INVALID (無效)、PENDING (待確認)。我們系統還自訂了 INTERRUPTED 狀態處理異常斷線。

差很多!有了 RAG,它就像有「記憶」一樣,能引用專案文件回答問題。

## 實戰應用:自動回答新人問題

我建了一個簡單的 Slack Bot,連接 RAG 系統。新人在頻道問問題,Bot 會:
1. 搜尋專案文件
2. 找相關段落
3. 用 LLM 生成回答
4. 標註資料來源 (哪份文件的哪個章節)

**實際案例**:

新人問:「為什麼 ProviderA 的 Location 更新會失敗?」

Bot 找到兩段相關內容:
- 技術文件:「ProviderA 的 Location ID 必須是 UUID 格式」
- PR 討論:「踩坑記錄:ProviderA 不接受帶破折號的 UUID」

然後回答:
> ProviderA 的 Location ID 格式比較嚴格,必須是不含破折號的 UUID (例如:550e8400e29b41d4a716446655440000)。你可以用 Guid.ToString("N") 來生成。詳見 docs/providers/providerA.md 第 3.2 節。

這個 Bot 上線後,**Junior 開發者的重複問題減少了 30%**。Senior 不用一直回答同樣的問題,可以專心寫代碼。

## 遇到的挑戰

**挑戰 1: Chunking 策略很重要**

一開始我是「每 500 字切一段」,結果查詢效果很差。因為切的位置很隨機,可能把一個完整的說明切成兩半。

改成「按標題切」後好很多。但這需要文件本身有良好的結構。如果是沒標題的純文字,就得用更複雜的策略 (例如用 LLM 幫忙切)。

**挑戰 2: 找到的文件不一定相關**

有時候語義搜尋會搞笑。我問「如何處理充電失敗」,它找到「如何處理登入失敗」的文件 (因為都有「處理」和「失敗」這兩個詞)。

**解法**: 增加搜尋結果數量,從 3 個改成 5 個,然後讓 LLM 自己判斷哪些真的相關。或是加上 metadata filter (例如只搜尋「充電」相關的文件)。

**挑戰 3: 更新文件後要重新 Embedding**

改了技術文件後,要記得重新跑 embedding,不然 RAG 會查到舊資料。

我寫了個簡單的 script,每次 commit 到 `docs/` 資料夾就自動觸發更新:

```bash
# .git/hooks/post-commit
#!/bin/bash
if git diff-tree --name-only HEAD | grep -q '^docs/'; then
    python scripts/update_embeddings.py
fi
```

## 效能觀察

**Embedding 速度**: 120 個文件,大概 5 秒跑完。還行。

**查詢速度**: 向量搜尋很快,10ms 內。瓶頸在 LLM 生成回答 (2-3 秒)。

**記憶體**: ChromaDB 的資料檔案約 50MB,載入記憶體後佔 200MB。可以接受。

如果文件量增加到 1 萬筆以上,可能需要換成真正的向量資料庫 (Pinecone / Weaviate),或是自己架 pgvector。

## 意外收穫:Code Search

RAG 不只能搜尋文件,也能搜尋代碼。

我把專案的所有 `.cs` 檔案也丟進 ChromaDB (每個 class 當作一個 chunk)。現在可以用自然語言搜尋代碼:

**問**: 「哪個 Service 處理 Heartbeat?」

**答**: 找到 `HeartbeatService.cs`、`OcpiProviderBase.cs` (有 SendHeartbeat 方法)、`StationMonitorService.cs` (處理 Heartbeat 失敗)。

比 Visual Studio 的 Find All References 好用,因為它理解「語義」而不只是「關鍵字」。

## 下週計畫

**目標: MCP - 讓 AI 自己執行任務**

這週做的 RAG 還是「被動」的:我問它問題,它查資料回答。

下週要玩更進階的:**讓 AI 主動執行任務**。比如:
- 「檢查最近 3 天有哪些充電站 Heartbeat 異常」→ AI 自己去查資料庫
- 「生成本週的系統健康度報告」→ AI 自己統計數據,生成圖表
- 「找出所有直接呼叫資料庫的代碼」→ AI 掃描專案,列出違規檔案

這需要用到 **MCP (Model Context Protocol)** 或類似的機制,讓 LLM 能呼叫外部工具 (API / Database / File System)。

如果成功,就等於有了一個「可以寫代碼、查資料、生成報告」的 AI Agent。
