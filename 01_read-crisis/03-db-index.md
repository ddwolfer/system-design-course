# L3|第一刀:索引 —— 同一句查詢,快一百倍 🔨

🎯 這課結束時:你會親手讓一句 300 萬筆資料的查詢從「幾百毫秒」掉到「不到 1 毫秒」,並且知道為什麼。
🧩 需要先會:L2 的 lab 環境(或用本課附的獨立起跑指令,一行就有資料庫)。
📚 想深挖:PostgreSQL 官方文件 Chapter 11「Indexes」;關鍵字:B-tree、EXPLAIN ANALYZE、composite index、partial index。

## 為什麼從索引開刀

L2 我們把「好物市集」打掛了。在買機器、加快取之前,先問一個更便宜的問題:
**資料庫是不是在用最笨的方式找資料?**

十次「網站好慢」有九次,兇手不是機器不夠力,是一句沒有索引的查詢。
這一刀不用加任何新機器、不改架構,幾秒鐘就砍完 —— 所以它永遠是第一刀。

## 概念三十秒:電話簿 vs 名片箱

想像 300 萬張名片。

- **沒有索引**:名片全部亂序丟在一個大紙箱裡。要找「陳小姐」?
  只能一張一張翻,平均翻 150 萬張。這就是資料庫的 **full table scan**。
- **有索引**:同樣 300 萬個名字,印成一本**按姓名排好序**的電話簿。
  翻開中間,看目標在前半還後半,再翻那半的中間…每次砍掉一半,
  300 萬筆只要約 22 次就找到。這就是 **B-tree** 索引在做的事。

排好序,就能每次砍半 —— 300 萬 vs 22,這是索引威力的全部秘密。

## 動手

沒跑 L2 的話,一行先起一顆資料庫(有 L2 環境就直接 `psql` 進去):

```bash
docker run -d --name lab-pg -e POSTGRES_PASSWORD=lab -p 5432:5432 postgres:17
docker exec -it lab-pg psql -U postgres
```

**① 造 300 萬筆商品**(貼進 psql,約十幾秒):

```sql
CREATE TABLE products (
  id bigint GENERATED ALWAYS AS IDENTITY PRIMARY KEY,
  name text NOT NULL,
  category text NOT NULL,
  price int NOT NULL
);

INSERT INTO products (name, category, price)
SELECT 'item-' || i,
       (ARRAY['leather','wood','ceramic','fabric','metal'])[1 + i % 5],
       (i * 37) % 5000 + 100
FROM generate_series(1, 3000000) AS i;

\timing on
```

**② 先感受「沒索引」**:

```sql
SELECT * FROM products WHERE name = 'item-2718281';
```

在一般筆電上這句大約要**上百毫秒** —— 對一句只找一筆資料的查詢來說,慢得離譜。
讓資料庫自己招供它做了什麼:

```sql
EXPLAIN ANALYZE SELECT * FROM products WHERE name = 'item-2718281';
```

看第一行:`Seq Scan on products` —— sequential scan,「我把 300 萬筆全翻了一遍」。
就是那個大紙箱。

**③ 下刀**:

```sql
CREATE INDEX idx_products_name ON products (name);
```

**④ 再跑同一句**:

```sql
EXPLAIN ANALYZE SELECT * FROM products WHERE name = 'item-2718281';
```

這次第一行變成 `Index Scan using idx_products_name`,執行時間掉到**遠低於 1 毫秒**。
同一句查詢、同一台機器、同樣的資料 —— 差了兩三個數量級。
(實際毫秒數每台機器不同,別背數字,看的是 Seq Scan → Index Scan 的質變。)

## 有一個索引你其實一直在用

回頭做個實驗:`SELECT * FROM products WHERE id = 42;` 從頭到尾都飛快。
因為建表時的 `PRIMARY KEY` **自動帶了一個索引**。
很多網站就是這樣被騙的:開發時都用 id 查(快),上線後使用者用名字、
用分類、用價格區間查(全都沒索引)—— 開發環境永遠測不出來的慢,上線就爆。

## 那…全部欄位都加索引不就好了?

不行,索引不是免費的:

1. **寫入變慢**:每次新增/修改資料,所有相關索引都要跟著更新。
   一張表掛十個索引,等於每次寫入要多做十份工。
2. **佔空間**:索引本身就是一份排好序的資料副本。

原則很簡單:**看你的查詢長什麼樣,為真正會出現在 WHERE、JOIN、ORDER BY 的欄位加**。
「好物市集」使用者會用名字搜、按分類逛、照價格排 —— 這三個欄位值得;
一個從來不會拿來查的備註欄位,加了純虧。

## 常見坑

| 症狀 | 原因 |
|---|---|
| 加了索引還是 Seq Scan | 條件對欄位動了手腳:`WHERE lower(name) = …` 用不到 name 的索引(對「運算結果」建 expression index 才行) |
| `LIKE '%皮件%'` 沒變快 | 開頭是萬用字元,排序幫不上忙 —— 電話簿沒辦法幫你找「名字裡含某個字」的人;這是搜尋引擎的活(模組三) |
| 資料很少時索引沒作用 | 正常:幾百筆資料全翻反而比查目錄快,資料庫自己會選 |
| 寫入變慢了 | 檢查是不是索引掛太多,刪掉沒人用的 |

## 小挑戰

「好物市集」的分類頁要撈「某分類下最便宜的 20 件」:

```sql
SELECT * FROM products WHERE category = 'ceramic' ORDER BY price LIMIT 20;
```

先 `EXPLAIN ANALYZE` 看現況,然後試著建一個**同時包含兩個欄位**的索引
`CREATE INDEX ... ON products (category, price);` 再看一次。
快了多少?為什麼欄位順序是 (category, price) 而不是 (price, category)?
(提示:先想電話簿是「先按姓、再按名」排的。)

## 收尾一問

朋友的網站慢,你懷疑是資料庫。你會請他跑哪一個指令、看哪一個關鍵字,
來判斷「是不是缺索引」?

→ 下一課:索引救不了的場景 —— 同一秒十萬人看同一個爆款商品頁,
每次都翻資料庫太傻了。第二刀:**快取**。

## 📇 名詞卡

- **Full Table Scan (Seq Scan)** — 資料庫把整張表逐筆翻過一遍來找資料的方式。資料量小沒差,資料量大就是災難;EXPLAIN 輸出裡看到 Seq Scan 配上大表,通常就是缺索引的訊號。
  - 想更深可以想想:PostgreSQL 文件:Using EXPLAIN。
- **B-tree 索引** — 資料庫預設的索引結構:把欄位值排好序、組成一棵每層都能大幅縮小範圍的樹,等值查詢與範圍查詢都是對數時間 —— 300 萬筆約 22 步。CREATE INDEX 不指定型別時建的就是它。
  - 想更深可以想想:PostgreSQL 文件 Chapter 11.2:Index Types。
