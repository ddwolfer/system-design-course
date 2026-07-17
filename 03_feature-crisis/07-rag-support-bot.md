# L7|AI 客服:讓模型先查自家資料再回答 🔨

🎯 這課結束時:你知道為什麼 AI 客服會亂編規則,並親手做出一個「先查自家 FAQ、再回答」的最小可行客服。
🧩 需要先會:L3 的向量/索引直覺(有沒有做過那課都不影響,這課會重新解釋)。
📚 想深挖:pgvector 官方 README(github.com/pgvector/pgvector);任一家模型供應商的 API 文件(embedding、chat completion)。

## 阿哲的便利貼:能不能先讓 AI 擋一波客服

阿哲接了一個現成的語言模型接到客服對話窗,結果客人問「我的商品可以退貨嗎」,
模型煞有其事地回答一套退貨規則 —— 內容聽起來很合理,但**跟「好物市集」真正的退貨政策完全對不上**。

## 為什麼模型會亂編

語言模型回答問題,靠的是它訓練時「看過的天量文字」歸納出的通用知識與語感,
它**沒有讀過阿哲的退貨政策文件**,也不知道「好物市集」這家店真正的規則是什麼。
問它一個它沒有依據的具體問題,它不會誠實地說「我不知道」,而是用最像答案的方式生成一段話 ——
這種看起來煞有其事但其實編造的內容,通常被稱作模型的**幻覺**。

解法不是換一個「更聰明」的模型,是換一個做法:**讓模型回答前,先去讀真正的文件**。

## RAG 白話:先查、再答

**RAG**(Retrieval-Augmented Generation)拆開來看就兩步:

1. **Retrieval(檢索)**:使用者問「可以退貨嗎」,系統先去自家的 FAQ / 政策文件裡,
   找出「跟這個問題最相關」的幾段內容。
2. **Augmented Generation(增強生成)**:把找到的那幾段內容,連同使用者的問題,
   一起放進餵給模型的提示詞(prompt)裡,請模型「根據這些資料回答」,
   而不是憑空回答。

模型此時做的事情,比較接近「幫你把一份文件的重點,組成一句人話回答」,
而不是「憑訓練記憶編一個答案」—— 這就是為什麼 RAG 能大幅降低幻覺。

## 「相關內容」是怎麼找到的:向量相似度

FAQ 文件那麼多段,怎麼知道哪幾段跟問題「相關」?做法是把每一段文字,
透過一個 **embedding** 模型轉成一串數字(向量),
語意相近的句子,轉出來的向量在空間裡的距離也相近。
查詢時,把使用者的問題也轉成向量,去找**距離最近的幾段 FAQ 向量**,
就是「檢索」實際在做的事。

## 動手:pgvector 存 FAQ 向量 + 組 prompt

**pgvector** 是 PostgreSQL 的擴充套件,讓資料庫多一種欄位型別能存向量、
並且能做相似度查詢 —— 不用額外架一套獨立的向量資料庫。

```bash
docker run -d --name lab-pgvector -e POSTGRES_PASSWORD=lab -p 5432:5432 pgvector/pgvector:pg17
```

```sql
CREATE EXTENSION IF NOT EXISTS vector;

CREATE TABLE faq (
  id bigint GENERATED ALWAYS AS IDENTITY PRIMARY KEY,
  question text NOT NULL,
  answer text NOT NULL,
  embedding vector(1536)  -- 維度依你使用的 embedding 模型而定,這裡用常見的 1536 示範
);
```

以下是一段最小可行的 Python 概念示範(呼叫模型的部分寫成可替換的函式,
可接任一家提供 embedding 與 chat completion 的模型 API):

```python
import psycopg2

def embed_text(text: str) -> list[float]:
    """呼叫任一模型供應商的 embedding API,回傳一串浮點數向量。"""
    ...  # 依你選用的模型 API 實作

def chat_complete(prompt: str) -> str:
    """呼叫任一模型供應商的 chat completion API,回傳生成的文字。"""
    ...  # 依你選用的模型 API 實作

def answer_question(user_question: str) -> str:
    conn = psycopg2.connect("dbname=postgres user=postgres password=lab host=localhost")
    cur = conn.cursor()

    # ① 檢索:把問題轉向量,查最相近的 3 筆 FAQ
    q_vec = embed_text(user_question)
    cur.execute(
        "SELECT question, answer FROM faq ORDER BY embedding <-> %s::vector LIMIT 3",
        (q_vec,),
    )
    matches = cur.fetchall()

    # ② 組 prompt:把檢索到的內容一起餵給模型
    context = "\n".join(f"Q: {q}\nA: {a}" for q, a in matches)
    prompt = f"""你是好物市集的客服。只根據下面的資料回答問題,
資料裡沒有的內容,要老實說「這個問題我需要請人工客服協助」。

資料:
{context}

使用者問題:{user_question}
"""
    return chat_complete(prompt)
```

`embedding <-> %s::vector` 裡的 `<->` 是 pgvector 提供的距離運算子,
`ORDER BY ... LIMIT 3` 就是在做「找最相近的 3 筆」這件事,
語法跟一般 SQL 排序、限制筆數完全一樣,只是排序依據換成了「向量距離」。

## 誠實但書:垃圾進,垃圾出

RAG 不是魔法,它只是讓模型「照著你給的資料回答」——**檢索品質決定答案品質**。
FAQ 資料本身寫得模糊、過時、或這次真的沒收錄使用者要問的內容,
模型能做的最好的事就是老實承認「不知道」(這也是為什麼上面的 prompt 特別交代這件事),
而不是硬湊一個聽起來合理但其實還是編的答案。RAG 解決的是「模型沒讀過你家資料」,
不解決「你家資料本身就沒寫清楚」。

想更深入了解「模型怎麼規劃、呼叫工具、串成一整個 agent」,可以看本頻道的
《Zero to Agent》系列,那邊會接著講 RAG 之外的 agent 設計。

## 常見坑

| 症狀 | 原因 |
|---|---|
| 檢索到的內容跟問題不相關 | 每段 FAQ 切太大塊(chunk 太大),一段裡混了太多主題,向量變得模糊 |
| 明明有答案卻查不到 | chunk 切太細,關鍵資訊被拆散在不同段落裡 |
| 資料庫沒有答案,模型還是編答案 | prompt 沒有明確交代「沒資料就說不知道」,模型預設仍會盡力生成 |
| FAQ 更新了,答案還是舊的 | 忘記在文件內容變更時重新產生 embedding、更新資料庫,向量沒同步 |

## 小挑戰

把 `LIMIT 3` 改成 `LIMIT 1`,問幾個你設計的測試問題,比較答案品質有沒有變差。
為什麼「檢索太少段落」跟「檢索太多段落」都可能讓答案變差?

## 收尾一問

如果有人問「乾脆把整份退貨政策文件全部塞進 prompt,不用檢索了」,
這樣做在 FAQ 只有 5 條的時候可不可行?FAQ 長到 500 條之後呢?

→ 下一課:五個功能都落地了,放手課 —— 挑一張便利貼,自己做完整設計。

## 📇 名詞卡

- **幻覺 (Hallucination)** — 語言模型在沒有可靠依據的情況下,生成聽起來合理但實際上是編造或錯誤的內容。降低幻覺的常見做法之一,就是本課教的 RAG:讓模型有真實資料可以依據。
  - 想更深可以想想:任一家模型供應商的文件通常會有專門討論 hallucination 與如何緩解的章節。
- **RAG (Retrieval-Augmented Generation)** — 先檢索跟問題相關的自家資料,再把檢索結果連同問題一起交給模型生成答案,而不是讓模型單憑訓練記憶回答,能大幅降低幻覺。
  - 想更深可以想想:模組三 L0 有提過這個詞的最初定義,這課是完整動手示範。
- **Embedding** — 把一段文字轉換成一串數字(向量)的過程,語意相近的文字轉出來的向量在空間裡距離也相近,是「用相似度找相關內容」的基礎。
  - 想更深可以想想:任一家模型供應商的 embedding API 文件都會解釋這個概念與呼叫方式。
- **pgvector** — PostgreSQL 的開源擴充套件,讓資料庫多一種向量型別欄位,並支援相似度距離查詢,不需要額外架設獨立的向量資料庫就能做基本的 RAG 檢索。
  - 想更深可以想想:github.com/pgvector/pgvector 的 README。
