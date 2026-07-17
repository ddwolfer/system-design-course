# L2|大檔案:商品圖直傳 🔨

🎯 這課結束時:你會親手做出一個「瀏覽器直接把圖片傳上雲端儲存」的端點,不經過你的伺服器中轉。
🧩 需要先會:L1 的 API 設計基本概念(資源、動詞、狀態碼)。
📚 想深挖:MinIO 官方文件「Presigned URLs」;AWS 文件「Uploading objects with presigned URLs」(概念相通)。

## 客服的便利貼:圖可以更清楚一點嗎

賣家一直用手機隨手拍,圖糊得像素塊。要清楚,檔案就會從幾百 KB 跳到幾 MB 甚至更大。
先問一個問題:**這麼大的檔案,要走哪條路上雲端?**

## 為什麼不能塞進資料庫,也不能靠自家伺服器中轉

兩個直覺但錯誤的做法,各有各的痛:

- **存進資料庫欄位**:資料庫是為了「結構化資料、快速查詢」設計的,拿來存幾 MB 的二進位檔,
  備份、複寫(模組一提過的讀取副本)全部被拖慢,而且資料庫通常是全系統最貴、最難擴充的一塊。
- **上傳先經過你的 app 伺服器再轉存**:每個上傳請求都佔一條連線、吃你伺服器的頻寬,
  一千個人同時上傳大圖,你的伺服器光「轉手」就先喘不過氣 —— 這件事你的伺服器根本不用做。

## Object Storage 白話

把 **物件儲存** 想成一個超大的公共倉庫:你把檔案丟進去,
換到一把「地址」(一個網址),之後誰拿著這把地址,就能直接跟倉庫要那個檔案 ——
不用經過你家。倉庫自己處理備份、擴充、跟全世界的連線,你只要負責「發地址」跟「管誰能進倉庫」。

## Presigned URL:核心模式

那「誰能進倉庫」怎麼管?不能讓任何人隨便上傳任何東西到你的倉庫。
答案是 **Presigned URL**:你的 app 伺服器**不搬檔案**,只做一件事 ——
簽發一張「限時、限這個檔名」的通行證(一個帶簽名的網址)。
瀏覽器拿著這張通行證,直接跟物件儲存講話,檔案完全不經過你的 app。

流程:

1. 瀏覽器跟你的 app 說「我要傳一張圖,檔名 xxx.jpg」
2. app 跟物件儲存要一張限時通行證(presigned URL),回給瀏覽器
3. 瀏覽器直接 PUT 檔案到那個網址
4. 上傳完成,圖片網址交給模組一講過的 CDN 去served

## 動手:docker 跑 MinIO + FastAPI 產 presigned URL

MinIO 是開源、**S3 相容**的物件儲存,可以在自己筆電上跑。

```bash
docker run -d --name lab-minio -p 9000:9000 -p 9001:9001 \
  -e MINIO_ROOT_USER=admin -e MINIO_ROOT_PASSWORD=admin12345 \
  minio/minio server /data --console-address ":9001"
```

啟動後在瀏覽器打開 `localhost:9001` 用剛剛的帳密登入,建一個 bucket 叫 `product-images`。

接著一段最小可行的 FastAPI 端點,負責「簽發通行證」:

```python
from fastapi import FastAPI
import boto3
from botocore.client import Config
import uuid

app = FastAPI()

s3 = boto3.client(
    "s3",
    endpoint_url="http://localhost:9000",
    aws_access_key_id="admin",
    aws_secret_access_key="admin12345",
    config=Config(signature_version="s3v4"),
)

@app.post("/products/{product_id}/upload-url")
def get_upload_url(product_id: int, content_type: str = "image/jpeg"):
    key = f"products/{product_id}/{uuid.uuid4()}.jpg"
    url = s3.generate_presigned_url(
        "put_object",
        Params={
            "Bucket": "product-images",
            "Key": key,
            "ContentType": content_type,
        },
        ExpiresIn=300,  # 5 分鐘內要用掉,過期作廢
    )
    return {"upload_url": url, "key": key}
```

前端拿到 `upload_url` 後,直接對它發一個 `PUT` 請求、body 放圖片內容,就完成上傳 ——
全程沒有一個位元組流過你的 app 伺服器。

## 常見坑

| 症狀 | 原因 |
|---|---|
| 通行證還沒用就過期 | `ExpiresIn` 設太短,前端網路慢一點就來不及;通常給幾分鐘,別給幾秒 |
| 任何人都能上傳任何東西 | bucket 權限開成公開可寫,或簽發時沒限制 `ContentType`;通行證應該只夠做「這一件事」 |
| 有人塞了 500MB 的檔案 | 沒在應用層或儲存政策限制檔案大小,一張「圖」變成一支影片 |
| 上傳成功但格式不是圖片 | 只信任副檔名,沒有在上傳完成後驗證真實內容型別 |

## 小挑戰

把上面的端點改一版:限制只能上傳 `image/jpeg` 或 `image/png`,拒絕其他型別;
並把 `ExpiresIn` 從 300 秒改成一個你覺得更合理的數字,說說看你的理由
(提示:想想「使用者從按下上傳鍵到檔案選好、開始傳」中間可能卡多久)。

## 收尾一問

如果之後要換掉 MinIO,改用真正的雲端物件儲存(例如某家雲廠商的 S3 相容服務),
上面這段程式碼大概要改動哪幾行?為什麼改動可以這麼小?

→ 下一課:圖片問題解決了,換搜尋出包 —— 為什麼 `LIKE '%關鍵字%'` 找不到東西。

## 📇 名詞卡

- **Object Storage(物件儲存)** — 專門存放檔案(圖片、影片、備份檔)的儲存服務,每個檔案是一個「物件」,用一把 key 換取存取,不像資料庫那樣支援複雜查詢,但便宜、耐用、擴充容易。
  - 想更深可以想想:MinIO 官方文件「Object Storage 101」。
- **Presigned URL** — 一個帶簽名、限時、限定動作(通常是上傳或下載某個檔案)的網址。伺服器不用經手檔案本身,只需要簽發這張通行證,實際傳輸由瀏覽器與儲存服務直接進行。
  - 想更深可以想想:MinIO 文件「Presigned URLs」章節。
- **S3 相容 (S3-compatible)** — 業界把 Amazon S3 的物件儲存 API 當成事實標準,MinIO 等開源系統實作同一套 API,所以程式碼幾乎不用改就能在不同儲存服務之間切換。
  - 想更深可以想想:MinIO 官方文件首頁會直接講它與 S3 API 的相容範圍。
