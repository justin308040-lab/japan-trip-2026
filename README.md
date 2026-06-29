# 🇯🇵 名古屋・高山・立山黑部 9 日行程網頁

一個**單一自包含 HTML 檔**的互動式旅遊行程網站，搭配 Firebase 即時同步、可多人共同編輯、購物清單上傳、即時天氣預報。

- **線上版**：https://justin308040-lab.github.io/japan-trip-2026/
- **原始碼**：`index.html`（單檔，約 814 行，CSS/JS/SVG 全內嵌，無建置流程）
- **託管**：GitHub Pages（repo: `justin308040-lab/japan-trip-2026`，分支 `main`，根目錄 `index.html`）

---

## ⚠️ 給要修改的 AI / 開發者：最重要的一件事

**行程的真實資料來源是 Firebase Realtime Database，不是 HTML 裡的 `defaultDays`。**

- `defaultDays` / `defaultPacking`（在 `<script>` 內）**只是「種子資料」**，僅在資料庫第一次完全空白時，由 `ensureSeed()` 寫入一次。
- 之後網頁一律從 Firebase 讀取並即時渲染。**改 HTML 裡的 `defaultDays` 文字 → 不會改到線上已存在的內容。**
- 要改線上內容有三種方式：
  1. 在網頁右上角開「✏️ 編輯模式」直接改（會寫回 Firebase）。
  2. 用 Firebase REST API `PATCH`/`PUT`（見下方「如何用程式改資料」）。
  3. 砍掉資料庫重新 seed（會清空所有人改過的內容，慎用）。
- 慣例：每次改動會「**同時更新 Firebase（即時生效）＋ 更新 HTML 的 `defaultDays`（保持原始碼一致）**」。改程式時請維持這個雙寫慣例。

---

## 🏗️ 架構總覽

```
index.html  (單檔)
├── <style>          全部 CSS（日式和紙淺色主題、青海波背景、響應式）
├── <body>           靜態結構：Hero(SVG 場景)、飯店列、控制列、行程容器、購物/清單區、Modal、FAB
└── <script type="module">
    ├── Firebase 初始化（Realtime Database）
    ├── defaultDays / defaultPacking   ← 種子資料
    ├── ensureSeed()                    ← 首次空庫才寫入
    ├── watch() / onValue()             ← 即時監聽三個集合
    ├── renderDays / renderPacking / renderShopping  ← 渲染
    ├── 編輯：openDayEditor / openShopEditor / Modal
    ├── 天氣：Open-Meteo API（每日 + 逐時 modal）
    └── 其他：倒數計時、捲動進度、側邊導航、路線/列印、購物排序
```

**無框架、無 npm、無建置**。純瀏覽器原生 ES Module，直接開 `index.html` 或丟上任何靜態主機即可。

---

## 🔌 後端：Firebase Realtime Database

設定在 `<script>` 頂部：

```js
const firebaseConfig = { apiKey:"…", authDomain:"…", projectId:"japan-trip-3f1a0", … };
const DB_URL = "https://japan-trip-3f1a0-default-rtdb.asia-southeast1.firebasedatabase.app";
const db = getDatabase(app, DB_URL);
```

- 用的是 **Realtime Database**（不是 Firestore；Firestore 在此專案需綁帳單故未用）。
- 規則設為公開讀寫（家庭共用、免登入）：
  ```json
  { "rules": { ".read": true, ".write": true } }
  ```
- **照片不另用 Firebase Storage**（Storage 需付費方案）。購物照片以 canvas 壓縮成 `dataURL`（JPEG, max 700px, q=0.6）後**直接存進 Realtime DB 的 `photoUrl` 欄位**。

### 資料結構（Realtime DB 根節點）

```
/meta/init                      { seededAt }            ← seed 哨兵，存在就不重複 seed
/itinerary/{id}                 ← 每日行程（id 種子為 d1…d9，新增為亂數 key）
/packing/{key}                  ← 行前清單（前端已隱藏，資料仍在）
/shopping/{key}                 ← 購物清單
```

#### `/itinerary/{id}` 欄位
| 欄位 | 型別 | 說明 |
|------|------|------|
| `order` | number | 排序用（1,2,3…），渲染依此排 |
| `date` | string | 顯示日期，如 `"6/18 (四)"` |
| `wdate` | string | 天氣用 ISO 日期，如 `"2026-06-18"`（對應天氣徽章 / 逐時 modal） |
| `title` | string | 標題 |
| `color` | string | 主題色 hex，如 `"#c0392b"`（左側色條、D 字圓圈） |
| `tags` | array | `[{l:"標籤名", t:"類型"}]`；t ∈ `transport|food|highlight|nature|culture` |
| `itinerary` | string | 行程內容，**允許 HTML**（`<br>`、`<a href>` Google Maps 連結等），以 `innerHTML` 渲染 |
| `transport` | string | 交通，允許 HTML |
| `food` | string | 美食（可空），允許 HTML |
| `highlight` | string | 重點/備註（可空），允許 HTML |
| `costs` | array | `[{item:"項目", amt:數字, note:"可選備註"}]`；目前**前端不顯示花費表**（已移除），但資料保留 |
| `done` | bool | 是否標記完成（影響進度環 + 刪除線） |

#### `/packing/{key}` 欄位
`{ text, checked(bool), order }`

#### `/shopping/{key}` 欄位
`{ name, note, bought(bool), photoUrl(dataURL 或 ""), order }`
- 渲染時：**未買的排前面、已買的排到最後**（`renderShopping` 內排序）。

---

## 🛠️ 如何用程式改資料（Firebase REST API）

Realtime DB 支援 REST，規則公開所以可直接 `PATCH`/`PUT`（不需金鑰）：

```bash
BASE="https://japan-trip-3f1a0-default-rtdb.asia-southeast1.firebasedatabase.app/itinerary"

# 改某天的部分欄位（PATCH＝合併）
curl -X PATCH "$BASE/d6.json" \
  --data-binary '{"title":"新標題","highlight":"新備註"}' \
  -H "Content-Type: application/json"

# 整天覆寫（PUT＝取代整個節點，要含所有欄位)
curl -X PUT "$BASE/d6.json" --data-binary @d6.json -H "Content-Type: application/json"
```

> 中文/emoji/`<a href="…">` 內含雙引號時，建議寫成 JSON 檔再用 `--data-binary @file.json`，避免跳脫地獄。

---

## 🌤️ 天氣（Open-Meteo，免金鑰）

- `loadWeather()` 對四個地區座標各打一次 `api.open-meteo.com/v1/forecast`，取 `daily` + `hourly`。
- 預報只涵蓋未來約 16 天；超過範圍用各地區「六月氣候概況」後備值（`regions[].norm`）。
- 每張行程卡的**天氣徽章可點開「逐時預報」modal**（`openHourly()`，顯示 6–21 時溫度 + 降雨機率長條）。
- 地區→日期對應寫在 `regions{}` 的 `dates` 陣列；改行程日期時記得同步。

---

## 🎨 前端功能清單

| 功能 | 位置 / 函式 |
|------|-------------|
| 日式淺色主題 + 青海波固定背景 | `<style>` body background |
| Hero 手繪 SVG（名古屋城、鳥居、雪山、櫻花瓣） | `.hero-scene` SVG |
| 出發倒數 / 旅行第幾天 | `updateCountdown()` |
| 每日卡片展開、標記完成、編輯/刪除、新增 | `renderDays()` + 編輯模式 |
| 友善編輯表單（標題/日期/行程/交通/美食/備註） | `openDayEditor()`（不暴露標籤/花費的原始格式） |
| 購物清單 + 照片上傳（壓縮存 DB）+ 已買排末 | `renderShopping()` / `openShopEditor()` / `resizePhoto()` |
| 即時天氣 + 逐時 modal | `loadWeather()` / `openHourly()` |
| 側邊 D1–D9 導航、捲動進度條、回頂 FAB + 進度環 | scroll listener / `updateRing()` |
| 周遊券攻略可展開卡片 | `#pass-head` 區塊 |
| 列印 / PDF | `window.print()` + `@media print` |
| 行前清單（**目前隱藏**） | `#packing` section `display:none` |

---

## 🚀 部署

```bash
git add index.html && git commit -m "…" && git push   # → GitHub Pages 約 1 分鐘後生效
```

---

## 📝 常見修改提示（給接手的 AI）

- **改某天行程**：同時 `PATCH` Firebase 的 `/itinerary/dN` ＋ 改 HTML 的 `defaultDays` 對應物件（雙寫）。
- **行程內想加地圖連結**：用 `<a href="https://www.google.com/maps/search/?api=1&query=地名" target="_blank">名稱</a>`。
- **加一整天**：在編輯模式按「新增一天」，或 `addItem('itinerary', {…, order:max+1, done:false})`。
- **標籤顏色**：在 CSS `.tag-xxx` 五種類型；`t` 值要對應其一。
- **不要把行程內容硬寫死在 HTML 期待生效** —— 一定要寫進 Firebase 才會在已 seed 的環境顯示。
- **照片別改用 Firebase Storage** —— 會需付費方案；維持 `dataURL` 存 DB 的做法。

---

*單檔、無建置、Firebase 即時同步、家庭共用。Have a nice trip 🌸*
