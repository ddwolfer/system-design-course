# L3|搜尋:LIKE 為什麼不行 🔨

🎯 這課結束時:你知道為什麼 `LIKE '%關鍵字%'` 用不到一般索引,並親手做出一個真的能用的搜尋端點。
🧩 需要先會:模組一「第一刀:索引」的 B-tree 概念(排好序、每次砍半)。
📚 想深挖:PostgreSQL 官方文件 Chapter 12「Full Text Search」;關鍵字:tsvector、tsquery、GIN index。

## 客常的便利貼:搜尋像壞掉一樣

「帆布包」明明存在,搜尋卻找不到。工程師第一反應通常是:

```sql
SELECT * FROM products WHERE description LIKE '%帆布包%';
```

能動,但慢,而且慢到規模一大幾乎不能用。問題不在資料庫壞了,在**用錯工具**。

## 為什麼 B-tree 索引救不了 LIKE '%關鍵字%'

模組一講過:**B-tree** 索引的威力來自「排好序、每次砍半」——
前提是你要找的東西**從頭開始比對**才有意義,像電話簿能幫你找「陳」開頭的名字。

但 `'%帆布包%'` 的萬用字元在**前面**,等於在問「有哪些描述『中間』含這幾個字」——
電話簿沒辦法回答「名字裡任何位置含某個字的人有誰」,唯一辦法還是整本翻一遍。
所以就算 `description` 欄位加了 B-tree 索引,這句查詢照樣是全表掃描,一個字都沒省到。

## 倒排索引白話:反過來建目錄

搜尋引擎的解法是換一種目錄,叫**倒排索引**:

- 一般想法是「文件 → 這篇文件有哪些詞」(正向,人類寫作的順序)。
- 倒排索引反過來記「詞 → 有哪些文件包含這個詞」。

想成一本書的**索引頁**:你不會為了找「帆布」這個詞,把整本書從頭讀到尾;
你翻到書後面的索引頁,直接看「帆布」出現在第 12、45、108 頁。
建好這份「詞 → 頁碼清單」,查任何詞都是查表,不用整本翻。

## 動手:PostgreSQL 內建的全文檢索

不用馬上上重型搜尋引擎,PostgreSQL 內建的**全文檢索**(full text search)
已經是一個小型倒排索引,拿來練手感剛好。示範用英文商品描述
(中文需要額外的分詞器如 zhparser,超出這課範圍,先記住這個限制)。

```sql
CREATE TABLE products (
  id bigint GENERATED ALWAYS AS IDENTITY PRIMARY KEY,
  name text NOT NULL,
  description text NOT NULL
);

INSERT INTO products (name, description) VALUES
  ('Canvas Tote Bag', 'A sturdy canvas tote bag for daily errands and market trips'),
  ('Leather Wallet', 'Handmade leather wallet with minimalist stitching'),
  ('Ceramic Mug', 'Hand-thrown ceramic mug, dishwasher safe');
```

**① 先感受沒有全文檢索的窘境**:

```sql
EXPLAIN ANALYZE SELECT * FROM products WHERE description LIKE '%canvas%';
```

資料量小時看不出差,但 `Seq Scan` 已經在那裡了 —— 資料一多就是災難。

**② 建一個 tsvector 欄位 + GIN 索引**:

```sql
ALTER TABLE products ADD COLUMN search_vec tsvector
  GENERATED ALWAYS AS (to_tsvector('english', name || ' ' || description)) STORED;

CREATE INDEX idx_products_search ON products USING GIN (search_vec);
```

`to_tsvector` 把文字拆成詞、去掉贅字(如 a、the)、做詞形還原,存成一份「這篇文件有哪些詞」的清單;
GIN 索引則是幫這份清單建出「詞 → 文件」的反向查詢結構。

**③ 用 tsquery 查詢**:

```sql
EXPLAIN ANALYZE
SELECT name FROM products
WHERE search_vec @@ to_tsquery('english', 'canvas & bag');
```

看 `EXPLAIN` 輸出,應該會看到用上 `idx_products_search` 而不是 Seq Scan;
`to_tsquery` 讓你能寫 `&`(且)、`|`(或)這類邏輯組合,這是 `LIKE` 完全做不到的。

## 什麼規模該搬去專職搜尋引擎

PostgreSQL 全文檢索能撐一段路,但下面這些需求出現時,值得認真評估
**Elasticsearch** 或 OpenSearch 這類專職搜尋引擎:

- 需要「相關性排序」(哪個結果最貼近使用者的意圖排前面,不只是有沒有匹配)
- 需要「錯字容忍」(使用者打錯字也要找得到,例如 fuzzy matching)
- 需要「facet」篩選(側邊欄「依分類、價格區間」即時算出各選項的筆數)

## 搜尋資料要跟資料庫同步

搜尋引擎通常是一份**獨立的資料副本**,商品資料改了,搜尋那邊也要跟著更新,
不然使用者會搜到「已經下架」的商品。這個「資料怎麼從正式資料庫流向搜尋引擎」的問題,
本質上跟模組三後面會教的資料管線是同一件事,這裡先點到為止。

## 常見坑

| 症狀 | 原因 |
|---|---|
| 建了 tsvector 欄位還是慢 | 忘記在 tsvector 欄位上建 GIN 索引,或索引還沒建完就下查詢 |
| `to_tsquery` 報錯 | 語法問題,例如詞中間有空白但沒用 `&` 連接;先用 `plainto_tsquery` 簡化 |
| 查得到英文查不到中文 | 語言設定用了 `'english'` 分析中文,中文需要專門分詞器,不能直接沿用 |
| 搜尋結果跟商品頁對不上 | 商品資料更新了但搜尋索引沒同步更新,是資料管線沒接上 |

## 小挑戰

試著把 `to_tsquery('english', 'canvas & bag')` 換成 `plainto_tsquery('english', 'canvas bag daily')`,
比較兩者查詢結果與寫法的差異。哪一種比較適合直接接使用者在搜尋框打的原始文字?

## 收尾一問

朋友問你「我們的搜尋到底是不是資料庫做的」,你會怎麼跟他解釋
「一般資料庫查詢」跟「全文檢索」的本質差異?

→ 下一課:圖片跟搜尋都解決了,接下來是「到貨了要馬上通知我」—— 即時更新怎麼做。

## 📇 名詞卡

- **B-tree 索引** — 資料庫預設的索引結構:把欄位值排好序、組成一棵每層都能大幅縮小範圍的樹,靠「從頭比對、每次砍半」加速查詢,但幫不上「萬用字元在開頭」的模糊比對。
  - 想更深可以想想:模組一「第一刀:索引」那一課有完整動手示範。
- **倒排索引 (Inverted Index)** — 把「文件 → 詞」反過來記成「詞 → 有哪些文件包含它」的目錄結構,查任何詞都是查表而非整篇翻找,是幾乎所有搜尋引擎的核心資料結構。
  - 想更深可以想想:PostgreSQL 文件 Chapter 12 開頭的概念說明。
- **全文檢索 (Full Text Search)** — 針對「文字內容是否包含某些詞、詞的相關性」設計的查詢方式,PostgreSQL 內建 tsvector/tsquery 實作了一個輕量版本,足以應付中小規模需求。
  - 想更深可以想想:PostgreSQL 官方文件 Chapter 12「Full Text Search」。
- **Elasticsearch** — 業界常用的專職搜尋引擎,擅長相關性排序、錯字容忍、facet 篩選等資料庫全文檢索做不到或做不好的事,代價是要多維護一套獨立系統與資料同步機制。
  - 想更深可以想想:Elastic 官方文件「What is Elasticsearch」。
