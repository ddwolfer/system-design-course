# L2|過載保護:給系統裝上保險絲 🔨

🎯 這課結束時:你會三種「保險絲」—— rate limit、circuit breaker、優雅降級 —— 知道各自擋什麼、怎麼用最少的程式碼做出一個能動的版本。
🧩 需要先會:上一課的「四個黃金訊號」;讀得懂約 25 行的 Python。
📚 想深挖:關鍵字 token bucket algorithm、circuit breaker pattern(Martin Fowler 的公開文章「CircuitBreaker」是這個模式最常被引用的說明)、exponential backoff and jitter(AWS Architecture Blog 的公開文章談得很清楚)。

## 上一課看見了,這一課要擋住

L1 幫你把「看不見」補起來了。但看見推薦服務變慢,不代表它就不會拖垮全站 —— 真正救命的,是在它慢下去的當下,有東西**主動擋住它擴散**。這一課給系統裝三種保險絲。

## 保險絲一:Rate Limit(限流)—— 令牌桶

概念很簡單:一個桶子,裝著「令牌」,每個請求進來要拿一個令牌才准通行,桶子固定用**穩定的速度**在補新令牌。桶子空了,新請求就被擋在外面(回一個「等一下再試」,不是讓它排隊等到系統累死)。

```python
import time

class TokenBucket:
    def __init__(self, capacity, refill_per_sec):
        self.capacity = capacity              # 桶子最多裝幾個令牌
        self.tokens = capacity
        self.refill_per_sec = refill_per_sec  # 每秒補幾個令牌
        self.last_check = time.monotonic()

    def allow(self):
        now = time.monotonic()
        elapsed = now - self.last_check
        self.last_check = now
        self.tokens = min(self.capacity, self.tokens + elapsed * self.refill_per_sec)
        if self.tokens >= 1:
            self.tokens -= 1
            return True
        return False

limiter = TokenBucket(capacity=20, refill_per_sec=10)

def handle_request(req):
    if not limiter.allow():
        return "429 Too Many Requests, 請稍後再試"
    return real_handler(req)
```

桶子大小決定「能接受多大的瞬間爆量」,補充速度決定「長期能扛的流量」—— 兩個數字都要根據下游(通常是資料庫或某個較慢的服務)真正撐得住多少去抓,不是憑感覺。

## 保險絲二:Circuit Breaker(斷路器)—— 別再對一個已經倒下的下游窮追猛打

上一課的推薦服務慢下去之後,首頁服務還在**傻傻地一直等它回話**—— 每個等待都佔用一份連線。斷路器的想法是:如果一個下游最近失敗率太高,乾脆別再呼叫它了,直接快速失敗(**fail fast**),把資源留給還健康的請求。

斷路器有三態:

- **關(closed)**:正常運作,請求照樣送出去,同時計算失敗率。
- **開(open)**:失敗率超過門檻,直接拒絕呼叫、不再浪費時間等下游回話,等一段冷卻時間。
- **半開(half-open)**:冷卻時間到,放一個請求探路 —— 成功就轉回「關」,失敗就繼續「開」。

```python
class CircuitBreaker:
    def __init__(self, fail_threshold=5, cooldown_sec=30):
        self.fail_threshold = fail_threshold
        self.cooldown_sec = cooldown_sec
        self.fail_count = 0
        self.state = "closed"
        self.opened_at = None

    def call(self, func, *args):
        if self.state == "open":
            if time.monotonic() - self.opened_at > self.cooldown_sec:
                self.state = "half-open"      # 放一個請求探路
            else:
                raise Exception("斷路器開啟中,直接快速失敗")
        try:
            result = func(*args)
        except Exception:
            self.fail_count += 1
            if self.fail_count >= self.fail_threshold:
                self.state = "open"
                self.opened_at = time.monotonic()
            raise
        else:
            self.fail_count = 0
            self.state = "closed"
            return result
```

如果上一課的首頁服務對推薦服務包了這一層,推薦服務一慢,斷路器很快跳開,首頁服務就不會被拖著一起沉。

## 保險絲三:優雅降級(graceful degradation)

斷路器跳開之後,呼叫方不能只是回一個錯誤畫面 —— 「猜你喜歡」是錦上添花,不是必需品。優雅降級的做法:推薦服務叫不動,首頁就顯示一份**預先算好、放在快取裡的熱門商品清單**當替代品。使用者根本不會發現差別,但系統不再被一個非核心功能拖垮。

原則:**先分清楚什麼是核心(結帳、看商品),什麼是錦上添花(推薦、個人化排序)**,非核心功能故障時,降級也要撐住核心不受影響。

## 重試要有上限、有退避、還要加一點隨機

上一課的重試風暴,是「使用者手動重刷」加「系統自動重試」疊加出來的。重試本身沒有錯,錯的是沒有紀律的重試。基本規則:

1. **有上限**:重試 3 次還失敗就放棄,別無限重試。
2. **退避(**exponential backoff**)**:每次重試等更久 —— 第 1 次等 1 秒、第 2 次等 2 秒、第 3 次等 4 秒,而不是每次都馬上重試。
3. **加入抖動(jitter)**:在等待時間裡加一點隨機值,避免一大群客戶端在完全同一秒同時重試,製造出新的一波尖峰。

## 常見坑

| 症狀 | 原因 |
|---|---|
| 加了斷路器,系統還是被拖垮 | 忘了同時設定 timeout —— 沒有逾時,請求根本不會被斷路器記成「失敗」,只會一直卡著 |
| 限流門檻設太鬆 | 桶子容量抓得比下游真正撐得住的量還大,爆量時下游還是被打死 |
| 優雅降級用在核心流程 | 把降級邏輯加在結帳上,故障時「靜默失敗」讓使用者以為下單成功,其實沒有 —— 降級只該用在非核心功能 |
| 重試沒有上限或退避 | 每次失敗立刻重試、沒有間隔,失敗的下游被重試流量二次打死,這正是上一課「重試風暴」的成因 |

## 小挑戰

回頭看上一課的時間軸:如果首頁服務呼叫推薦服務時,同時具備「逾時(timeout)+ 斷路器 + 優雅降級」,故障會在哪一步被擋下(03:04?03:06?03:09?)?再想一步:光有斷路器、沒有給推薦服務的呼叫設定**獨立**的連線池,會不會還是不夠?為什麼?

## 收尾一問

「好物市集」的搜尋功能最近常被少數幾個帳號用程式瘋狂呼叫。你會選這三種保險絲裡的哪一個當第一道防線?為什麼?

→ 下一課:保險絲擋住了故障擴散,但故障當下如果有一筆訂單的通知或扣庫存訊息剛好卡在半路 —— 它會丟掉,還是重複?第三課:可靠傳遞。

## 📇 名詞卡

- **Fail Fast(快速失敗)** — 與其讓請求傻等一個已經出問題的下游,不如立刻回一個失敗,把資源(連線、執行緒)留給還健康的請求。斷路器「開」的狀態就是在做這件事。
  - 想更深可以想想:關鍵字:fail-fast design。
- **Exponential Backoff(指數退避)** — 每次重試都拉長等待時間(1 秒、2 秒、4 秒⋯),避免對一個正在掙扎的下游持續施壓;通常會再加入隨機的 jitter,避免大量客戶端在同一秒同時重試。
  - 想更深可以想想:關鍵字:exponential backoff and jitter(AWS Architecture Blog 有公開文章專門講這個)。
