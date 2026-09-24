# 電子名片推薦系統流程包

最後更新：2026.09.24 20:48（台北時間 UTC+8）

目前版本：v3.1  
狀態：內部測試 OK，待 2026.09.21 外部好友實測

> 本文件只記錄流程與架構。Supabase Vault 內部密鑰、Service Role 等敏感資料不寫入 GitHub。

---

## 1. 對外入口

### 電子名片 LIFF
- LIFF ID：`2007730669-c8K2cktB`
- Endpoint：`https://chenxindesigner.github.io/flex-v2/card/`
- LIFF URL：`https://liff.line.me/2007730669-c8K2cktB`

### GitHub
- 名片／分享引擎：`chenxindesigner/flex-v2`
- 管理後台：`chenxindesigner/flex-card-admin`
- 管理後台網址：`https://chenxindesigner.github.io/flex-card-admin/`

### Supabase
- Project：`flex-card`
- Project ref：`tzdaqrfsgwijbswfxlwj`
- Region：`ap-northeast-1`

---

## 2. 目前使用者流程

### A. 使用者第一次開啟名片分享頁
1. LIFF 初始化。
2. 若尚未登入 LINE，進行 LINE Login。
3. 前端取得 LINE access token。
4. 呼叫 `flex-card-track` Edge Function。
5. Edge Function 向 LINE Profile API 驗證 access token。
6. 驗證成功後才用 LINE user ID 建立或更新會員。
7. 系統產生會員推薦編號與隨機 referral token。
8. 頁面顯示本人推薦編號。

### B. 分享名片
1. 使用者按「分享給 LINE 好友」。
2. 系統先確認目前分享者的 referral token。
3. 使用 `shareTargetPicker` 產生新的 Flex 名片。
4. 名片內每個可追蹤按鈕都帶目前分享者的 referral token。
5. 分享完成後記錄 `share_completed`。

### C. 好友收到名片
1. 單純收到或看到訊息，不會立即建立推薦關係。
2. 好友必須點擊名片內「預約諮詢」。
3. LIFF 讀取 `r` referral token。
4. 若參數被 LINE 放進 `liff.state`，前端也會正確解析。
5. 系統驗證好友 LINE 身分。
6. 若該好友尚未有直接推薦人，建立推薦關係。
7. 直接推薦人建立後鎖定，不會被後來其他分享者覆蓋。

---

## 3. 直接推薦規則

系統只做一層推薦，不做多層分潤。

標準流程：

```text
A 分享給 B
B 點擊卡片
=> A → B

B 再從卡片內按「名片分享」分享給 C
C 點擊卡片
=> B → C
```

若 C 最後成交：
- B 有推薦獎金
- A 不會因為 C 成交而再拿第二層獎金

### LINE 原生轉傳要特別注意

Flex 卡片在產生時就已經帶著分享者 referral token。

因此：

```text
A 產生的卡片 → B 收到
B 直接用 LINE 原生「轉傳」給 C
=> 卡片仍然是 A 的 token
=> C 點擊後仍歸 A
```

如果 B 要成為 C 的直接推薦人：

```text
B 必須打開卡片
→ 按卡片內「名片分享」
→ 系統用 B 的 LINE 身分重新產生卡片
→ 再分享給 C
```

---

## 4. 推薦成立判定

正式規則：

> 推薦歸屬以受邀者第一次點擊推薦名片內「預約諮詢」，並由系統成功建立的直接推薦關係為準。

不是以：
- 誰先在 LINE 傳訊息
- 誰先截圖
- 誰說自己曾推薦
- 單純的 LINE 轉傳紀錄

來認定。

LINE `shareTargetPicker` 不會提供實際收件人的 LINE user ID，因此真正的推薦關係是在好友點入後才成立。

---

## 5. 防止重複與錯誤推薦

目前已實作：

- `line_user_id` 唯一
- 同一 LINE 使用者不會因重複點擊產生多個會員
- 每位被推薦者只能有一個直接推薦人
- 第一筆成功推薦關係鎖定
- 禁止自我推薦
- 禁止形成推薦循環
- 互動事件可以持續記錄，不影響會員唯一性
- 後台可在必要時人工補登直接推薦人，但不能覆蓋既有關係

---

## 6. 成交與獎金規則

範例：

```text
案件金額：20,000
推薦比例：10%
總獎金：2,000
```

獎金分兩階段：

1. 訂金實際收到後，可發 50%
2. 專案完成／尾款完成後，再發 50%

每一期獎金可分別標記：
- pending
- approved
- paid
- void

「待發獎金」只統計目前已符合發放條件但尚未支付的金額，不包含未來尚未到期的那一半。

---

## 7. 我的推薦紀錄

LIFF 分享頁已新增「我的推薦紀錄」。

驗證方式：
1. 使用者在 LINE LIFF 內開啟。
2. 前端傳 LINE access token 至 `flex-card-records`。
3. Edge Function 驗證 LINE Profile。
4. 依本人 LINE user ID 讀取自己的推薦資料。
5. 不提供其他會員完整後台資料。

目前顯示：
- 已推薦人數
- 推薦成交數
- 待發獎金
- 每一筆直接推薦紀錄
- 推薦成立時間
- 成交狀態／獎金狀態

---

## 8. 名片分享頁 UI 固定規則

目前確認版：

- 淡粉霧玫瑰色
- 白色圓角主卡
- 背景只使用很淡的抽象線條，不放花
- 雜誌感、留白多
- `A BETTER YOU` 小字
- `CHEN.XIN` 不可過大
- 中文「陳芯」使用宋體系
- 名字下方細線靠近名字
- 三顆按鈕左右總寬完全對齊
- 主按鈕：`分享給 LINE 好友`
- 次按鈕：
  - `我的推薦紀錄`
  - `直接洽詢`
- 按鈕不要過厚
- 不使用純黑色字，使用柔和灰紫／灰粉深色
- 大頭貼固定使用 `card/avatar-chenxin.jpg`
- 大頭貼只能等比縮放，不得拉伸變形

固定提示文字：

```text
歡迎分享給身邊有需求的朋友
成功合作可享推薦獎金
好友需點「預約諮詢」，推薦才會成立
```

直接洽詢：
`https://line.me/ti/p/_YTgB9L_px`

---

## 8.1 2026.09.24 名片按鈕快速跳轉

問題：

- Flex 名片內「作品集／預約諮詢／LINE 官網／意念空間」原本全部先開完整 LIFF 名片頁。
- LIFF 頁面會先顯示「準備中…」，再等待 `liff.init → LINE Profile 驗證 → Supabase 推薦追蹤` 完成後才轉址。
- 使用者體感為多一個無意義中繼畫面，而且按鈕明顯變慢。

本次修正僅調整前端路由，不改推薦資料結構與成立規則：

- 作品集／預約諮詢／LINE 官網／意念空間：
  - 在頁面第一幀前辨識 action。
  - 不再顯示完整名片頁。
  - 取得既有 LINE access token 後，以 `sendBeacon`；不支援時用 `fetch keepalive` 在背景送出原本相同的推薦追蹤資料。
  - 立即 `window.location.replace()` 到真正目標頁，不再等待追蹤 API 回應。
- 背景追蹤仍包含：
  - LINE access token
  - referral token
  - action
  - sessionId
- `flex-card-track` Edge Function、LINE Profile 驗證、直接推薦第一筆鎖定、禁止自我推薦／循環推薦全部未修改。
- 「名片分享」仍使用原本 `track("share") + shareTargetPicker` 正式流程，只把分享選擇器開啟前的完整名片畫面隱藏；若分享取消或失敗才顯示原頁供重試。
- 一般直接開啟電子名片頁仍維持原 UI、我的推薦紀錄、直接洽詢與分享功能。

## 8.2 2026.09.24 推薦成立改為「預約諮詢」單一入口

使用者最終確認：

> 推薦獎金歸屬只以好友點擊 Flex 名片內「預約諮詢」為準。

因此改為「速度優先＋單一推薦入口」：

- 作品集：
  - 直接開 `https://chenxinstudio.com/#portfolio`
  - 不經 LIFF
  - 不建立推薦關係
  - 不記錄是哪一位 LINE 使用者點擊
- LINE 官網：
  - 直接開 LINE OA
  - 不經 LIFF
  - 不建立推薦關係
- 意念空間：
  - 直接開對應 LINE
  - 不經 LIFF
  - 不建立推薦關係
- 預約諮詢：
  - 保留 referral token
  - 經 LIFF 驗證點擊者 LINE 身分
  - 只有 `booking` action 可建立直接推薦關係
  - 推薦完成後進入正式預約系統
- 名片分享：
  - 保留原本 LIFF `shareTargetPicker`
  - 本身不建立被推薦關係

舊卡片相容：

- 舊 Flex 卡即使仍帶 `a=portfolio / line_oa / idea_space` 的 LIFF URL，
  `card/index.html` 會在載入 LIFF SDK 前立即轉到真正網址。
- 因此舊卡不需要為了作品集／LINE 官網／意念空間重新傳送，也不再等待完整 LIFF 初始化。
- 舊卡只有「預約諮詢」仍會保留推薦追蹤。

推薦成立規則已由「好友點任一按鈕」改成：

```text
A 分享名片給 B
B 點「預約諮詢」
=> A → B 推薦成立
```

作品集等其他按鈕僅作為正常功能入口，不再參與推薦獎金歸屬。

## 9. 本次已修正問題

### 2026.09.20

#### 推薦 Token
已修正 LINE LIFF 會將 deep-link 參數包入 `liff.state`，造成原本只讀 `window.location.search` 時 referral token 遺失的問題。

現在：
- 先讀 URL 頂層 `r`、`a`
- 若沒有，再解析 `liff.state`
- 可正常保留分享來源

#### 大頭貼
原先使用 inline base64 造成 LIFF 顯示異常。

已改為 GitHub 實體圖片：
`card/avatar-chenxin.jpg`

目前 GitHub Pages 已成功部署，圖片已確認可載入。

#### 我的推薦紀錄
新增：
- SQL function：`get_flex_card_referral_records`
- Edge Function：`flex-card-records`

使用 LINE access token 驗證本人後才回傳自己的推薦紀錄。

#### 後台測試資料
2026.09.20 凌晨曾執行一次測試資料重置：
- members
- referral_relationships
- interaction_events
- conversions
- commissions

會員流水號亦重新從 R000001 開始。

重置後已重新建立測試資料。

---

## 10. 2026.09.20 04:40 測試快照

目前資料庫測試快照：

- members：2
- referral_relationships：1
- interaction_events：12
- conversions：0
- commissions：0

已成功驗證：
- LINE 身分建立會員
- referral token 產生
- 分享 Flex
- 好友點擊後建立直接推薦關係
- 推薦關係不重複
- 分享頁按鈕可正常操作
- 大頭貼已正常顯示
- 「我的推薦紀錄」入口可使用
- 「直接洽詢」可使用

目前內部測試結果：OK。

下一步：
- 2026.09.21 請不同 LINE 帳號進行 A → B → C 實際測試
- 再測一筆成交
- 再測訂金獎金 50%
- 再測結案獎金 50%
- 最後確認「我的推薦紀錄」與管理後台金額同步

---

## 11. 明日外部實測流程

請準備 3 個不同 LINE 帳號：A、B、C。

### 第一段
1. A 打開自己的名片分享頁。
2. 確認 A 有自己的推薦編號。
3. A 按「分享給 LINE 好友」傳給 B。
4. B 不要用 LINE 原生轉傳，先點名片內「預約諮詢」。
5. 後台確認 A → B。

### 第二段
1. B 打開收到的卡片。
2. B 按卡片內「名片分享」。
3. B 重新產生自己的卡片後分享給 C。
4. C 點名片內「預約諮詢」。
5. 後台確認 B → C。

### 第三段
確認：
- A 的推薦紀錄只有 B
- B 的推薦紀錄只有 C
- C 沒有直接推薦人以外的上層分潤
- 重複點擊不新增第二個會員
- 重複分享不覆蓋第一位直接推薦人

---

## 12. 不要誤動的項目

目前測試正常，之後修改 UI 時不要任意改動：

- LIFF ID
- `readLiffParams()`
- referral token 產生與帶入方式
- `track()`
- `shareTargetPicker`
- `flex-card-track`
- `flex-card-records`
- Supabase Vault 內部密鑰
- 直接推薦唯一性
- 循環推薦檢查
- 第一筆推薦關係鎖定
- 卡片內各追蹤 action URL

若只調整外觀，應盡量限制在 HTML／CSS 顯示層。
