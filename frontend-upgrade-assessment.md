# CDM Server 前端框架升級評估

**評估日期：** 2026-08-13
**評估對象：** `src/Admin` 後臺網頁的 Bootstrap 與 AdminLTE
**用途：** 週會報告，供決定是否投入與投入規模

---

## 一、現況

### 1.1 目前使用的版本

| 項目 | 目前版本 | 釋出時間 | 官方支援狀態 |
| --- | --- | --- | --- |
| Bootstrap | 3.3.7 | 2016-07 | **已終止支援**(2019-07 EOL) |
| AdminLTE | 2.x(`skin-blue` / `sidebar-mini` / `wrapper` 語法) | 2018 | **已終止支援**(AdminLTE 2 不再維護) |
| jQuery | 3.7.1(另有一份舊的 3.0 並存) | 2023-08 | 支援中 |
| Vue | 2.5.17 | 2018-08 | **已終止支援**(Vue 2 於 2023-12-31 EOL) |

### 1.2 站上同時存在兩套設計系統

這是評估過程中最值得先講的一件事：

- **首頁**(`Views/Home/Index.cshtml`)在先前的改版中已改用 **SB Admin 2** 的樣式(Bootstrap 4 風格的 `card` 元件)。
- **其餘 98 個頁面**仍是 **AdminLTE 2 + Bootstrap 3**(`panel` 元件)。

也就是說「要不要統一」這件事其實已經發生一半了，目前是混雜狀態。這會持續造成兩個成本：改樣式時要判斷改哪一套、新頁面要決定跟哪一邊走。

### 1.3 規模

| 項目 | 數量 |
| --- | --- |
| Razor View(`.cshtml`) | 99 個 |
| View 總行數 | 20,486 行 |
| 含 inline `<style>` 的 View | 9 個 |
| 專案自訂 JS(`yui.js`，包裝表格與彈窗的共用層) | 32 KB |

---

## 二、為什麼要升

1. **兩個核心框架都已終止支援。** Bootstrap 3 自 2019 年、AdminLTE 2 自更早就不再收到安全性修補。往後若被通報漏洞，官方不會出修正版，只能自行 patch 或緊急升級。
2. **資安掃描的持續告警。** 目前 CSP 的 `unsafe-inline` / `unsafe-eval` 告警已決議先開 Bugzilla 緩處理，其中 `unsafe-eval` 是卡在 Vue 2 完整版(runtime compiler)。前端整包現代化才有機會一次解掉。
3. **維護成本。** 兩套設計系統並存，加上多個已停止維護的 jQuery 外掛，改動的風險與時間都在上升。

---

## 三、相依套件盤點(成本的主要來源)

升級的工時**不是花在 Bootstrap 本身，而是花在綁在 Bootstrap 3 上的外掛**。以下是實際掃描結果：

| 套件 | 用到的 View 數 | 是否支援 Bootstrap 5 | 處置 |
| --- | --- | --- | --- |
| `layer`(彈窗) | **72** | 與 Bootstrap 無關 | **可保留**，只需確認樣式不衝突 |
| `bootstrap-table` | **32** | 官方有 BS5 主題 | 換主題即可，風險低 |
| `bootstrap-datetimepicker`(eonasdan v4) | **25** | 否，僅支援 BS3 | **必須更換**(改用 Tempus Dominus 或原生 `input[type=date]`) |
| `bootstrap-treeview` | **10** | 否，且該專案自 2016 年起停止維護 | **必須更換**(需另尋樹狀元件) |
| `chosen`(下拉選單) | 5 | 與 Bootstrap 無關 | 可保留 |
| `bootstrap-dialog` | 2 | 否，僅支援 BS3 | 改用 Bootstrap 5 原生 Modal |

> **重點：** 用量最大的 `layer`(72 個 View)反而不受影響，因為它不依賴 Bootstrap。真正的痛點是 `bootstrap-treeview` 與 `bootstrap-datetimepicker` 這兩個**已無人維護、且沒有直接對應替代品**的套件，合計影響 35 個 View。

---

## 四、需要修改的樣式類別數量

Bootstrap 3 → 5 有大量 class 更名或移除。以下是實際統計：

| Bootstrap 3 寫法 | 出現次數 | 影響檔案數 | Bootstrap 5 的對應 |
| --- | --- | --- | --- |
| `form-group` | 225 | 43 | 語意變更，需重寫表單結構 |
| `control-label` | 200 | 41 | 改為 `form-label` |
| `glyphicon` | **197** | 31 | **已完全移除**，需改用 Font Awesome 或 Bootstrap Icons |
| `btn-default` | 102 | 33 | 改為 `btn-secondary` |
| `pull-left` / `pull-right` | 90 | 72 | 改為 `float-start` / `float-end` |
| `col-xs-*` | 65 | 40 | 改為 `col-*` |
| `form-horizontal` | 46 | 42 | 已移除，改用 grid 排版 |
| `input-group-addon` | 35 | 28 | 改為 `input-group-text` |
| `panel` | 34 | 5 | 改為 `card` |
| `data-toggle` / `data-target` | 29 | 9 | 改為 `data-bs-toggle` / `data-bs-target` |
| **合計** | **約 1,023 處** | — | — |

其中約六成可用指令碼機械式替換，但**每一頁仍須人工目視確認排版沒有跑掉**——這是工時的主要組成。

---

## 五、方案比較

### 方案 A：維持現狀，只處理資安告警

只針對掃描報出的項目個別處理，不動框架。

- **工時：** 3～5 人天
- **優點：** 立即、風險最低
- **缺點：** 框架仍在 EOL 狀態，掃描報告每次都會重複出現同樣的項目；技術債持續累積

### 方案 B：Bootstrap 3→4 + AdminLTE 2→3

AdminLTE 3 是基於 Bootstrap 4 的版本，遷移幅度比直上 5 小。

- **工時：** 25～35 人天
- **缺點：** **Bootstrap 4 已於 2023-01 終止支援。**等於花了相當的成本，升到另一個也過期的版本，一兩年內還要再做一次。**不建議。**

### 方案 C：Bootstrap 3→5 + AdminLTE 2→4(建議)

AdminLTE 4 基於 Bootstrap 5，且不再依賴 jQuery。

- **工時：** 50～65 人天
- **優點：**
  - 一次到位，落在目前有支援的版本上
  - **UI 原型已完成 7 頁**(`docs/ui-prototype/` 底下的登入、設備、日誌、地圖、角色、使用者、總覽)，設計方向已經看過，不確定性比從零開始低很多
  - 可順帶統一首頁與其他頁面的設計系統
- **缺點：** 工時最高；`bootstrap-treeview` 與 `datetimepicker` 的替代品需要另外評估

### 方案 D：升 Bootstrap 5，但不採用 AdminLTE

拿掉 AdminLTE 依賴，直接用 Bootstrap 5 加少量自訂樣式做骨架。

- **工時：** 45～60 人天
- **優點：** 未來不再被 AdminLTE 的版本節奏綁住
- **缺點：** 側邊欄、選單、版型等 AdminLTE 現成的東西要自己做；已完成的 7 頁 AdminLTE 4 原型會用不上

---

## 六、方案 C 的工時拆解

| 階段 | 工作內容 | 工時(人天) |
| --- | --- | --- |
| 1 | 前置：相依套件替代方案定案(樹狀元件、日期選擇器選型與試作) | 3 |
| 2 | 骨架與共用層：`_Layout` / `_LayoutH` / `_Header` / `_Sidebar` / `_Aside` / `_Css` / `_Js` 與 `yui.js` 共用函式改寫 | 8 ～ 10 |
| 3 | 替換 `bootstrap-treeview`(10 個 View) | 5 ～ 7 |
| 4 | 替換 `bootstrap-datetimepicker`(25 個 View) | 5 ～ 7 |
| 5 | `bootstrap-table` 換 BS5 主題(32 個 View) | 3 ～ 4 |
| 6 | `bootstrap-dialog` 改原生 Modal(2 個 View)、確認 `layer` 相容性 | 2 ～ 3 |
| 7 | View 樣式類別遷移(99 個 View，約 1,023 處) | 20 ～ 25 |
| 8 | 首頁由 SB Admin 2 統一到 AdminLTE 4 | 2 ～ 3 |
| 9 | 測試與修正(99 頁目視 + E2E 迴歸) | 8 ～ 12 |
| | **合計** | **56 ～ 74** |

> 取整後對外報告用 **50～65 人天**(約 2.5 ～ 3.5 個月單人工時)。上表刻意列出上限 74，是因為第 3、4 項的替代品選型還沒做，那兩項的變異最大。

---

## 七、能不能分階段做

**可以，但有一個技術限制：Bootstrap 3 與 5 無法在同一個頁面共存。**

不過因為本專案的 CSS 是由 `_Css.cshtml` / `_CssAdd.cshtml` 這兩個共用 partial 統一載入，可以採用**新舊並行**的過渡策略：

1. 新增一組 `_Css5.cshtml` / `_Layout5.cshtml`，只載入 Bootstrap 5 + AdminLTE 4
2. 每次遷移一個功能模組，把該模組的 View 指向新的 layout
3. 未遷移的模組繼續走舊 layout，功能不受影響
4. 全部遷移完成後移除舊的一組

**代價：** 過渡期間兩套 CSS 都要保留，站台體積暫時變大，且使用者在不同頁面之間切換時會看到外觀不一致。過渡期建議控制在兩到三個月內。

**建議的遷移順序**(由風險低到高)：

1. 登入頁(獨立、不含表格與樹狀元件)
2. 使用者、角色、公司等單純的 CRUD 頁
3. 設備清單等使用 `bootstrap-table` 的頁面
4. 含樹狀元件與日期選擇器的頁面(群組、任務、日誌查詢)
5. 首頁儀表板

---

## 八、建議

1. **採方案 C**，但**先做第 1 階段(3 人天)** 把 `bootstrap-treeview` 與 `bootstrap-datetimepicker` 的替代品試作出來，再回頭確認總工時。這兩項是目前估算中變異最大的部分。
2. 若本季沒有 50～65 人天的空間，**先做方案 A(3～5 人天)** 止血，並把方案 C 排入下一季規劃。不建議採方案 B。
3. **Vue 2 的處理需另外評估。** 它與 Bootstrap 升級技術上可以分開做，但兩者都牽動所有 View，同時做可以省下一次全站迴歸測試的成本；分開做則風險較低。這部分尚未估算。

---

## 九、本評估的前提與不確定性

- 工時為**單人專職**的估算，未計入需求討論、設計確認、跨團隊溝通的時間。
- 未計入 Client 端的任何改動(本次評估僅涵蓋 Server 後臺網頁)。
- `bootstrap-treeview` 與 `bootstrap-datetimepicker` 的替代品**尚未選型**，第 3、4 階段的工時可能上下浮動。
- 已完成的 7 頁 AdminLTE 4 原型**尚未 commit**，目前在工作區。若原型的設計方向後續有變更，第 2、8 階段需重估。
- Vue 2 → Vue 3 的遷移**不在本次工時內**。
