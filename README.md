# 🇯🇵 名古屋・高山・立山黑部 9 日行程網頁

一個**單一自包含 HTML 檔**的互動式旅遊行程網站(唯讀靜態版)。行程、打包、購物清單資料全部內嵌於檔案中,搭配即時天氣預報、倒數計時、側邊導航等功能。

- **線上版**:https://justin308040-lab.github.io/japan-trip-2026/
- **原始碼**:`index.html`(單檔,CSS/JS/SVG 全內嵌,無建置流程)
- **託管**:GitHub Pages(repo: `justin308040-lab/japan-trip-2026`,分支 `main`,根目錄 `index.html`)

---

## ⚠️ 給要修改的 AI / 開發者:最重要的一件事

**本網頁為唯讀靜態版,已不連任何後端資料庫。**

- 2026 旅程結束後,原本的 Firebase Realtime Database 連線已完全移除,資料庫最終快照被內嵌成 `<script>` 內的 `TRIP_DATA` 物件。
- 網頁一律從 `TRIP_DATA` 讀取並渲染;**沒有任何寫入 / 同步 / 登入**。
- 編輯模式、新增、編輯、刪除等控制項已用 CSS 隱藏(對應的寫入函式 `addItem/updateItem/removeItem` 已改為空函式)。

### 要改內容怎麼做

直接修改 `<script>` 內的 `TRIP_DATA` 物件即可,存檔後重新整理就生效:

```js
const TRIP_DATA = { itinerary:{…}, packing:{…}, shopping:{…}, meta:{…} };
```

- `itinerary`:每日行程,key 為 `d1…d9`(或亂數 key)。
- `packing`:行前清單(前端已隱藏,資料仍在)。
- `shopping`:購物清單。

---

## 🏗️ 架構總覽

```
index.html  (單檔)
├── <style>          全部 CSS(日式和紙淺色主題、青海波背景、響應式)
│                    + 唯讀版隱藏編輯控制項的規則
├── <body>           靜態結構:Hero(SVG 場景)、飯店列、控制列、行程容器、購物/清單區、Modal、FAB
└── <script type="module">
    ├── TRIP_DATA                      ← 內嵌的行程資料(唯一資料來源)
    ├── watch()                        ← 從 TRIP_DATA 取出並排序後渲染(一次性,無監聽)
    ├── renderDays / renderPacking / renderShopping  ← 渲染
    ├── 天氣:Open-Meteo API(每日 + 逐時 modal)
    └── 其他:倒數計時、捲動進度、側邊導航、列印、購物排序
```

**無框架、無 npm、無建置、無後端**。純瀏覽器原生 ES Module,直接開 `index.html` 或丟上任何靜態主機即可。

---

## 📦 資料結構(`TRIP_DATA`)

```
TRIP_DATA.itinerary.{id}        ← 每日行程(id 為 d1…d9)
TRIP_DATA.packing.{key}         ← 行前清單(前端已隱藏,資料仍在)
TRIP_DATA.shopping.{key}        ← 購物清單
TRIP_DATA.meta                  ← 舊資料庫遺留的中繼資料(不影響渲染)
```

#### `itinerary.{id}` 欄位
| 欄位 | 型別 | 說明 |
|------|------|------|
| `order` | number | 排序用(1,2,3…),渲染依此排 |
| `date` | string | 顯示日期,如 `"6/18 (四)"` |
| `wdate` | string | 天氣用 ISO 日期,如 `"2026-06-18"`(對應天氣徽章 / 逐時 modal) |
| `title` | string | 標題 |
| `color` | string | 主題色 hex,如 `"#c0392b"`(左側色條、D 字圓圈) |
| `tags` | array | `[{l:"標籤名", t:"類型"}]`;t ∈ `transport\|food\|highlight\|nature\|culture` |
| `itinerary` | string | 行程內容,**允許 HTML**(`<br>`、`<a href>` Google Maps 連結等),以 `innerHTML` 渲染 |
| `transport` | string | 交通,允許 HTML |
| `food` | string | 美食(可空),允許 HTML |
| `highlight` | string | 重點/備註(可空),允許 HTML |
| `costs` | array | `[{item:"項目", amt:數字, note:"可選備註"}]`;前端不顯示花費表,但資料保留 |
| `done` | bool | 是否標記完成(影響進度環 + 刪除線;唯讀版沿用快照當時的狀態) |

#### `packing.{key}` 欄位
`{ text, checked(bool), order }`

#### `shopping.{key}` 欄位
`{ name, note, bought(bool), photoUrl(dataURL 或 ""), order }`
- 渲染時:**未買的排前面、已買的排到最後**(`renderShopping` 內排序)。
- 照片以 `dataURL`(壓縮後的 JPEG)直接存在 `photoUrl` 欄位。

---

## 🌤️ 天氣(Open-Meteo,免金鑰)

- `loadWeather()` 對四個地區座標各打一次 `api.open-meteo.com/v1/forecast`,取 `daily` + `hourly`。
- 預報只涵蓋未來約 16 天;超過範圍用各地區「六月氣候概況」後備值(`regions[].norm`)。
- 每張行程卡的**天氣徽章可點開「逐時預報」modal**(`openHourly()`,顯示 6–21 時溫度 + 降雨機率長條)。
- 地區→日期對應寫在 `regions{}` 的 `dates` 陣列;改行程日期時記得同步。

> 這是唯一仍會對外連線的功能(公開 API、免金鑰、免登入)。行程已結束、日期超過預報範圍時,會自動改用氣候概況後備值。

---

## 🎨 前端功能清單

| 功能 | 位置 / 函式 |
|------|-------------|
| 日式淺色主題 + 青海波固定背景 | `<style>` body background |
| Hero 手繪 SVG(名古屋城、鳥居、雪山、櫻花瓣) | `.hero-scene` SVG |
| 出發倒數 / 旅行第幾天 / 圓滿結束 | `updateCountdown()` |
| 每日卡片展開、標記完成狀態顯示 | `renderDays()` |
| 購物清單 + 照片 + 已買排末 | `renderShopping()` |
| 即時天氣 + 逐時 modal | `loadWeather()` / `openHourly()` |
| 側邊 D1–D9 導航、捲動進度條、回頂 FAB + 進度環 | scroll listener / `updateRing()` |
| 周遊券攻略可展開卡片 | `#pass-head` 區塊 |
| 列印 / PDF | `window.print()` + `@media print` |
| 行前清單(**目前隱藏**) | `#packing` section `display:none` |

---

## 🚀 部署

```bash
git add index.html && git commit -m "…" && git push   # → GitHub Pages 約 1 分鐘後生效
```

---

## 🕘 歷史沿革

- 原始版本使用 **Firebase Realtime Database** 做多人即時同步、共同編輯、照片上傳。
- 因資料庫規則設為公開讀寫,Google 會寄「安全性較低」的警告信;2026 旅程結束後改為唯讀靜態版:移除所有 Firebase 連線、內嵌最終資料快照、隱藏編輯功能。
- 若日後要重新啟用線上編輯,需重新導入後端與適當的安全規則(建議搭配登入驗證,避免公開讀寫)。

---

*單檔、無建置、無後端、純唯讀靜態。Have a nice trip 🌸*
