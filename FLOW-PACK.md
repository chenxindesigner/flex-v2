# 電子名片推薦系統流程包

最後更新：2026.09.30 21:28（台北時間 UTC+8）

目前版本：v3.2  
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
2. 好友只要點擊名片內任一功能按鈕（作品集／預約諮詢／LINE 官網／意念空間），就會進入互動名單。
3. LIFF Router 讀取 `r` referral token 與 `a` action。
4. 若參數被 LINE 放進 `liff.state`，前端也會正確解析。
5. 系統驗證好友 LINE 身分。
6. 若該好友尚未有正式推薦人，第一次有效功能按鈕建立推薦關係。
7. 後來從其他人的卡按一般功能按鈕，只保留來源證據，不覆蓋正式推薦人。
8. 只有從其他來源的卡真正完成「好友分享」後，正式推薦人才可改變；原本來源與歸屬歷程永久保留。

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

> 推薦歸屬先以受邀者第一次可驗證的功能按鈕互動來源建立；之後一般按鈕不覆蓋，只有從另一來源的卡真正完成「好友分享」才可變更正式推薦人。

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
- 第一筆有效功能按鈕先建立正式推薦人
- 一般後續按鈕不覆蓋正式推薦人
- 只有好友分享完成可變更正式推薦人，且每次變更都寫入不可覆蓋歷程
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
好友點開名片按鈕，即會建立推薦來源
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

## 8.2 2026.09.24 推薦成立改為「預約諮詢」單一入口（歷史版本，已由 8.3 取代）

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

## 8.3 2026.09.27 永久 Router／互動名單／推薦歷程

本版將「卡片永久可用」列為固定架構規則。

### 永久按鈕入口

Flex 卡送出後，按鈕 URI 不再因後端、追蹤、Loading、真正目標網址或管理後台修改而更換。

新產生的名片固定使用同一個電子名片 LIFF Router：

```text
作品集 / 預約諮詢 / LINE 官網 / 意念空間
→ 固定 LIFF Router
→ LOADING 極輕量轉場
→ LINE 身分驗證
→ 記錄互動與來源
→ 真正目標網址
```

只要舊卡本身已經使用這個 LIFF Router，即使數日後才點擊，也會執行當下最新的後端規則，不需要重新把名片傳給好友。

已送出的 Flex 訊息「畫面／文字／原始 URI」本身仍無法被 LINE 遠端改寫；因此未來不得為一般程式或後台修改任意更換 Router URI。只有名片前台視覺、文字、按鈕結構本身改版時，才需要重傳新版。

### LOADING 轉場

功能按鈕不再載入完整電子名片作為中繼畫面。

轉場只顯示：

- `LOADING...`
- 純 CSS 沙漏旋轉動畫
- 品牌淡米粉背景

轉場頁不載入大頭貼與完整名片 UI；只處理必要的 LIFF 身分驗證、互動寫入與轉址。

### 互動名單

只要使用者點擊下列任一功能按鈕，就建立／更新會員與互動：

- portfolio
- booking
- line_oa
- idea_space

同一會員＋同一 action 只保留一筆目前狀態：

- first_interacted_at：第一次按
- last_interacted_at：最後一次按
- interaction_count：內部累計
- last source：最後一次該 action 是從哪一個來源進入

管理後台不需要顯示每一次重複 click；重複點擊只更新最後互動時間。

### 推薦來源與正式歸屬

推薦來源證據不可覆蓋。

新增 append-only `referral_history`：

- `source_seen`：第一次確認曾從某位推薦人的卡進入
- `assigned`：第一次有效功能按鈕建立正式推薦人
- `reassigned`：從另一位來源的卡真正完成「好友分享」，正式推薦人改變

固定規則：

```text
D 第一次從 A 的卡按任一功能按鈕
→ 正式推薦人先為 A

D 後來從 C 的卡只按作品集／預約／LINE 官網／意念空間
→ 保留 C 曾是來源的證據
→ 正式推薦人仍是 A

D 後來從 C 的卡按「好友分享」並完成分享
→ 正式推薦人改為 C
→ A 的來源與原本歸屬歷程永久保留
```

LINE 原生「轉傳」不會觸發歸屬變更；只有系統內的「好友分享」成功完成後，才可依上述規則改變既有正式推薦人。

### Edge Function 速度優化

`flex-card-track` 已升級為 version 2（ACTIVE）。

同一 LIFF access token 在同一個 warm Edge instance 內使用 10 分鐘短快取，避免使用者短時間連續按不同按鈕時，每一顆都重新呼叫 LINE Profile API。第一次仍會向 LINE 驗證；快取只用於降低重複驗證延遲，不改推薦判定。

### 資料表

新增：

- `member_interactions`：去重後的每位會員／每顆按鈕最後互動狀態
- `referral_history`：不可覆蓋的來源／歸屬歷程

`referral_relationships` 保留為「目前正式推薦人」。

### 2026.09.27 LOADING 視覺簡化

依實機回饋，中繼轉場移除沙漏圖形，只保留極簡 `LOADING...` 動態點。

同時加入 Supabase tracking endpoint 的 `preconnect`／`dns-prefetch`，減少第一次建立連線的等待時間。身分驗證與資料寫入仍需完成後才轉址，不以犧牲推薦／互動紀錄可靠性換取假性速度。

注意：LINE 聊天室中已經送出的 Flex 訊息，其 `action.uri` 無法被遠端改寫。若某張歷史卡的個別按鈕在送出當下就是直接目標網址，該按鈕無法套用後來的 Router；永久 Router 規則適用於已使用 Router 的舊卡與之後新產生的卡。

### Face 分享器 JSON 自動 Router 化

使用者不需要修改既有 Flex JSON。

Face 分享器 `index.html` 在送出前會辨識 CHEN.XIN 名片的固定目標網址：

- 作品集
- 預約諮詢
- LINE 官網
- 意念空間
- 名片分享

若命中上述網址，分享器會先取得目前分享者的 referral token，再把該 URI 自動改成 dedicated card LIFF Router：

```text
原本 JSON 直接網址
→ Face 分享器送出前自動轉換
→ 固定 card LIFF + r=<分享者 token> + a=<action>
→ LINE 好友收到可追蹤的永久 Router 卡
```

一般不屬於 CHEN.XIN 名片的 Flex URI 不做修改。

本次只修改「送出前 URI 處理」與分享完成紀錄；不修改 root LIFF 的 login／redirect／400 相關初始化邏輯。

### Face 分享器辨識修正：按鈕名稱組合

Face 分享器送出前不再要求目標 URI 與內建字串完全相同。

只要同一卡片至少包含 3 個核心按鈕「作品集／預約諮詢／LINE 官網／意念空間」，即視為 CHEN.XIN 名片，並依按鈕 label 自動轉成永久 Router。label 比對會忽略空白；「名片分享／好友分享」都對應 share。

使用者仍照原流程貼原本 JSON，不需要手動修改 URI。

### 分享器舊版快取防護

2026.09.28 實機測試發現，Chrome／GitHub Pages 在剛部署後仍可能沿用先前已開啟的 root 分享器 HTML。判斷依據：舊版成功提示仍顯示「如卡片原本含文字觸發……」，表示瀏覽器執行的不是最新程式。

已補：

- root `index.html` 加入 no-cache／no-store meta。
- 頁面顯示版本更新為 v3.2。
- 可見更新日期改為 2026.09.28。
- 部署剛完成時若既有分頁仍載入舊 HTML，可用一次性 query cache-bust URL 開啟；之後正常新開頁面即可。

### 舊卡「好友分享」可自動承接新版規則

已送出的歷史卡片，只要「好友分享／名片分享」按鈕本身仍指向目前的分享器 LIFF，就可以繼續沿用，不需要重新傳給原本的好友。

固定流程：

```text
A 以前把舊卡傳給 B
→ B 過幾天從舊卡按「好友分享」
→ 開啟目前最新版分享器
→ 系統辨識 B 並取得 B 的 referral token
→ 分享器依目前最新版規則重新產生可追蹤卡
→ B 分享給 C
→ C 收到的是新版 Router 卡
→ C 後續按功能按鈕時，來源記為 B
```

因此：

- B 手上的舊卡可以作為「進入最新版分享器」的入口。
- B 不需要先取得一張新版卡，才能再分享給 C。
- C 收到的卡會套用當下最新的 Router／LOADING／互動紀錄／推薦規則。
- 新一層推薦來源是 B，不會沿用 A。
- 這個承接能力只依賴「舊卡的好友分享按鈕仍可進入現行分享器」。
- 歷史卡中如果某些一般功能按鈕當時是直接目標網址，這些既有按鈕本身仍無法被遠端改寫；但不影響其「好友分享」重新產生新版卡的能力。

### 2026.09.30 LIFF 登入流程修正

實機出現：新使用者由 Flex 功能按鈕進入 card LIFF 後，被再次導向 LINE Login，並顯示「錯誤／無法正常執行」。

原因鎖定為 card LIFF 仍使用舊登入流程：

```text
liff.init()
→ 若 isLoggedIn=false
→ 在 LIFF browser 內再次 liff.login({redirectUri: window.location.href})
```

已改為：

- `liff.init({ liffId, withLoginOnExternalBrowser: true })`
- LIFF browser 內由 `liff.init()` 自動處理登入，不再手動呼叫 `liff.login()`
- 只有非 LIFF 環境仍未登入時才顯式呼叫 `liff.login()`
- 外部登入 redirect 改用固定 `https://chenxindesigner.github.io/flex-v2/card/`，並只帶必要的 `r` 與 `a`
- `r/a` 參數於 `liff.init()` 完成後再解析

此修正不改推薦歸屬、互動紀錄、Router action 與目標網址。

### 2026.09.30 恢復已驗證可用的大頭貼認證流程

實機再次測試後，比對歷史 commit `0670ca4a`／`baa00fc1`，確認過去「大頭貼中繼頁」可正常完成新使用者 LINE 認證的登入流程為：

```text
liff.init({liffId})
→ isLoggedIn()
→ 未登入：liff.login({redirectUri: window.location.href})
→ 登入後回原 URL
→ 解析 r / a
→ track
→ 轉址
```

2026.09.30 前一版改用 `withLoginOnExternalBrowser`／固定 canonical redirect 後，實機仍出現 LINE Login「無法正常執行」，因此撤回該實驗性修改，恢復已在本專案實際驗證過的原登入流程。

同時調整畫面時序：

- 第一次尚未認證：顯示原本有陳芯大頭貼的中繼頁並進行 LINE 認證。
- 已經認證：功能按鈕才切換到極輕量 `LOADING...`，完成追蹤後快速轉址。
- 不改永久 Router、四顆功能按鈕追蹤、推薦來源歷程與後台規則。

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
