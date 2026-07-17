# L2|起跑線:把會掛的網站跑起來 🔨

🎯 這課結束時:你的筆電上會跑著一個迷你版「好物市集」(FastAPI + PostgreSQL + Redis,全部用 Docker 跑),而且你會親手用壓測工具把它打掛,記下「打掛時的數字」當這個模組的基準線。
🧩 需要先會:安裝好 Docker Desktop(或任何 Docker 環境)。L1 的請求全景(知道等一下卡住的會是哪一段)。
📚 想深挖:Docker 官方文件「Get Started」;hey 壓測工具:github.com/rakyll/hey;PostgreSQL 官方文件 Chapter 4「SQL Syntax」。

## 為什麼要先打掛它

L0 講了故事,但「網站被打掛」如果只是聽,很難有體感。這課要讓你**親手做一次**——
先蓋一個和阿哲網站一樣脆弱的迷你版「好物市集」,再用壓測工具往它身上招呼,
親眼看數字從正常變成災難。之後每一課加一把武器,都拿這個基準線比較。

## 三個容器,一個 docker compose

我們要跑三個服務:一顆 PostgreSQL(資料庫)、一顆 Redis(快取,這一課先擺著,
下一刀才會用到)、一個用 Python FastAPI 寫的商品列表 app。全部寫進
`docker-compose.yml`,一行指令全部啟動。

先建立資料夾結構:

```
lab/
├── docker-compose.yml
└── app/
    └── main.py
```

**`docker-compose.yml`**:

```yaml
services:
  db:
    image: postgres:17
    environment:
      POSTGRES_PASSWORD: lab
      POSTGRES_DB: shop
    ports:
      - "5432:5432"
    volumes:
      - dbdata:/var/lib/postgresql/data

  cache:
    image: redis:7
    ports:
      - "6379:6379"

  app:
    image: python:3.12-slim
    working_dir: /app
    volumes:
      - ./app:/app
    command: >
      sh -c "pip install --quiet fastapi 'uvicorn[standard]' psycopg2-binary redis &&
             uvicorn main:app --host 0.0.0.0 --port 8000"
    ports:
      - "8000:8000"
    depends_on:
      - db
      - cache

volumes:
  dbdata:
```

**`app/main.py`**(商品列表 app,約 40 行):

```python
import psycopg2
from fastapi import FastAPI

app = FastAPI()


def get_conn():
    return psycopg2.connect(
        host="db", dbname="shop", user="postgres", password="lab"
    )


@app.get("/health")
def health():
    return {"status": "ok"}


@app.get("/products")
def list_products(category: str, limit: int = 20):
    """依分類撈商品,依價格排序——這句查詢還沒加任何索引。"""
    conn = get_conn()
    cur = conn.cursor()
    cur.execute(
        "SELECT id, name, category, price FROM products "
        "WHERE category = %s ORDER BY price LIMIT %s",
        (category, limit),
    )
    rows = cur.fetchall()
    cur.close()
    conn.close()
    return [
        {"id": r[0], "name": r[1], "category": r[2], "price": r[3]}
        for r in rows
    ]
```

先講為什麼,再動手:

- `app` 用 `python:3.12-slim` 現場 `pip install`,不另外寫 Dockerfile——
  這是實驗室求快;真的要上線會另外做正式映像檔,那是題外話。
- `/products` 這句查詢先天就是慢種子,因為 `category` 和 `price`
  兩個欄位目前**完全沒有索引**——這是刻意留著的,下一課(L3)才補這一刀。

## 啟動

```bash
cd lab
docker compose up -d
docker compose ps
```

看到三個服務都是 running / healthy 再往下走。

## 灌 300 萬筆商品資料

商品表的欄位是 id / name / category / price(下一課會沿用同一張表加索引,
所以先用同一個 schema)。連進資料庫貼上這段 SQL:

```bash
docker exec -it $(docker compose ps -q db) psql -U postgres -d shop
```

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
```

跑完(約十幾秒)確認一下 `SELECT count(*) FROM products;` 應該是 300 萬。

## 先手動打一次

```bash
curl "http://localhost:8000/products?category=leather"
```

應該會拿到 20 筆商品的 JSON——但你可能已經感覺到它「回得有點久」。
單一個人用還好,問題是等一下要模擬**很多人同時**用。

## 動手:把它打掛

用 **hey** 這個壓測工具,同時發出大量請求。Docker Desktop
使用者可以直接用容器打宿主機:

```bash
docker run --rm williamyeh/hey -z 20s -c 100 \
  "http://host.docker.internal:8000/products?category=leather"
```

`-z 20s` 是「跑 20 秒」,`-c 100` 是「同時維持 100 個連線」。跑完看報告,
重點看兩個數字:

- ****p95 延遲****:100 個請求裡,95% 的人等待時間落在哪個數字以下。
- **狀態碼分布**:是不是開始出現非 200(逾時、連線被拒)。

在一般筆電上,你應該會看到 p95 從「還可以接受」的範圍,滑向**幾百毫秒到
數秒**,甚至開始出現逾時或錯誤——這就是阿哲網站爆紅那晚發生的事,
只是縮小版重演。(實際數字因機器而異,別背數字,記的是「質變」:
從穩定回應,變成排隊、逾時。)

## 記下你的基準線

把這次的結果存下來(截圖或抄數字都行):總請求數、p95、有沒有非 200
狀態碼。之後每加一把武器,都拿它跟這個基準線比。

## 常見坑

| 症狀 | 原因 |
|---|---|
| app 一啟動就連線失敗 | db 容器還沒「準備好接受連線」就被 app 連了,depends_on 只保證容器啟動順序,不保證資料庫真的就緒;重跑一次 app 或加重試邏輯即可 |
| hey 連不到 host.docker.internal | 這是 Docker Desktop(Windows/Mac)才有的功能;Linux 版 Docker 需另外加 `--add-host=host.docker.internal:host-gateway`,或改用容器間的網路名稱連線 |
| SQL 灌很慢 | 檢查是不是忘了先建表,或資料庫容器資源被其他程式占用 |
| curl 拿到空陣列 | 檢查 category 拼字要跟 SQL 陣列裡的值一致,例如 leather 不是 leathers |

## 小挑戰

自己動手把 `-c 100` 改成 `-c 300` 再跑一次,比較 p95 和錯誤率的變化。
自我驗收:你能不能講出「同時連線數變多」和「p95 惡化」之間的因果關係?

## 收尾一問

如果現在把 `-c` 調到 1(等於一次只有一個人在用),你猜這句查詢的反應時間
還會不會像多人同時打的時候那麼糟?為什麼?

→ 下一課:別急著加機器——先檢查資料庫是不是在用最笨的方式找資料。
第一刀:**索引**。

## 📇 名詞卡

- **Load Testing 壓力測試** — 模擬大量使用者同時打同一個服務,觀察它在壓力下的表現(反應時間、錯誤率)。本課用 hey 這個小工具,一行指令就能發出大量並行請求。
  - 想更深可以想想:hey 專案文件:github.com/rakyll/hey。
- **p95 延遲(95th percentile latency)** — 把所有請求的反應時間由快到慢排序,取排在第 95% 那個位置的數字——代表「95% 的人等待時間都在這個數字以下」。比看平均值更能反映「最慘的那一群人」的體驗。
  - 想更深可以想想:關鍵字:percentile latency、tail latency。
