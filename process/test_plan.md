# 「月底不吃土」測試計畫（對應規格書 v1.0）

> 由 `spec.md` v1.0（2026-10-10）逐條推導。每個 F-xx 的每一條驗收標準至少對應一個測試案例；另加規格隱含的邊界案例（金額邊界、月份／年份邊界、台灣本地日期、預算 80%／100%、刪最後一筆、壞 JSON、CSV BOM、localStorage 不可用、360px／≥900px 版面、鍵盤與無障礙）。
> 測試案例 ID 格式：`T-<F編號>-<流水號>`；跨功能案例用 `T-D`（日期）、`T-S`（靜態／技術規格）。規格書第 8 節的 T-1～T-6 情境在文末「對照」區對應到這裡的案例。

## 0. 測試環境、慣例與共用工具

### 0.1 執行環境
- 受測物：`index.html` 單檔，以 `file://` 與 `http://localhost` 各跑一次（§6 要求 `file://` 離線可用）。
- 自動化：headless Chrome + CDP（Puppeteer／Playwright／原生 CDP 皆可）。每個案例在**新的 browser context**（乾淨 localStorage）執行，除非「前置」另有指定。
- 時區與時間：所有案例一律 `Emulation.setTimezoneOverride({timezoneId:'Asia/Taipei'})`，並用 `Page.addScriptToEvaluateOnNewDocument` 覆寫 `Date`（固定「現在」為案例指定的本地時間；預設 **2026-10-09 10:00 +08:00**，與規格驗算例一致）。覆寫方式：保存原 `Date`，以固定 epoch 建構 `new Date()`／`Date.now()`，其他方法委派給原物件。
- Console 監聽：整個 suite 全程訂閱 `Runtime.exceptionThrown` 與 `Runtime.consoleAPICalled(type==='error')`，任何案例結束時紅色錯誤數必須為 0（F-08-4）。
- 下載攔截：在 `addScriptToEvaluateOnNewDocument` 覆寫 `URL.createObjectURL` 與 `HTMLAnchorElement.prototype.click`，把 `{name: a.download, text: await blob.text(), bytes: await blob.arrayBuffer()}` 推進 `window.__downloads`；或改用 `Browser.setDownloadBehavior({behavior:'allow', downloadPath})` 讀實體檔。
- 對話框：訂閱 `Page.javascriptDialogOpening`（取得 `message`）後 `Page.handleJavaScriptDialog({accept})`。若實作改用自訂 modal，改抓 modal 文字並點「確定／取消」。兩種都算符合規格（規格只寫「確認」）。

### 0.2 共用 helper（偽碼，harness 自行實作）
| helper | 意義 |
|---|---|
| `KEY = 'hw5-expense-tracker:v1'`, `BKEY = 'hw5-expense-tracker:broken'` | §5 的 localStorage key |
| `clock('2026-10-09T10:00')` | 設定固定本地時間後 reload |
| `seed(state)` | `localStorage.setItem(KEY, JSON.stringify(state))` 後 reload |
| `state()` | `JSON.parse(localStorage.getItem(KEY))` |
| `raw()` | `localStorage.getItem(KEY)`（字串比對用） |
| `byText(sel, text)` | `[...document.querySelectorAll(sel)].find(e => e.textContent.replace(/\s+/g,' ').trim().includes(text))` |
| `spec(id)` | `document.querySelector('[data-spec="F-05"]')` 之類，F-12-3 規定每區塊都有 `data-spec` |
| `setVal(el, v)` | `el.value = v; el.dispatchEvent(new Event('input',{bubbles:true})); el.dispatchEvent(new Event('change',{bubbles:true}))`；金額欄建議改用 CDP `Input.insertText` 以模擬真實輸入 |
| `add({type,amount,category,date,note})` | 透過 UI：點「支出／收入」切換 → 金額欄輸入 → 點分類按鈕（以名稱找）→ 設日期欄 → 備註 → 點「記一筆」 |
| `toast()` | `document.querySelector('[aria-live="polite"]')?.textContent.trim()` |
| `body()` | `document.body.innerText` |
| `errors()` | 自上次 reset 以來的紅色錯誤清單 |
| `viewport(w,h)` | `Emulation.setDeviceMetricsOverride({width:w,height:h,deviceScaleFactor:1,mobile:w<768})` |
| `hscroll()` | `document.documentElement.scrollWidth > document.documentElement.clientWidth` |
| `rec(sel)` | `el.getBoundingClientRect()` |

> DOM id／class 規格未規定；案例以**可見文字**與 `data-spec` 定位。若建置者提供穩定的 `id` 或 `data-test` 屬性，harness 可換成直接選取器。

### 0.3 共用資料集
- **FX-EMPTY**：`{version:1, settings:{monthlyBudget:8000,lastCategory:'food'}, records:[]}`。
- **FX-T2**（規格 T-2，今天 2026-10-09）：
  `10/01 🏠 居住 5500`、`10/01 💰 生活費 8000`、`10/01 💼 打工 4000`、`10/05 🍱 餐飲 2250`、`10/09 🍱 餐飲 235`；預算 8,000。
  推導值：收入 12,000／支出 7,985／結餘 4,015；計入預算已花 2,485；今天以前已花 2,250；剩餘天數 23；今日額度 250；今天還能花 15；已用 31%；剩 5,515；月底預估 round(2485/9×31)=8,559。
- **FX-DONUT**：2026-10 支出 `🍱 餐飲 3000`、`🧋 飲料 600`、`🚇 交通 400`（備註分別「午餐」「珍奶」「捷運」）。
- **FX-XSS**：一筆 `🍱 餐飲 90`，備註 `<img src=x onerror=alert(1)>`。
- 以 `seed()` 植入時，`id` 用 `test-01…`，`createdAt` 依序遞增（同一天的排序測試會指定）。

---

## 1. F-01 快速記一筆

#### T-01-01 預設狀態：支出、本地今天、🍱 餐飲
- 對應：F-01-1
- 前置：乾淨 context；`clock('2026-10-09T10:00')`
- 步驟：開啟頁面。
- 預期：「支出」為選取狀態；日期欄值 `2026-10-09`；分類「🍱 餐飲」為選取；分類按鈕共 8 個（餐飲、飲料、交通、居住、學業、日用、娛樂社交、其他）。
- 自動化：`document.querySelector('input[type=date]').value === '2026-10-09'`；`byText('button','餐飲').matches('[aria-pressed="true"], .active, .selected')`（擇一，依實作）；支出分類按鈕數 `=== 8`。

#### T-01-02 凌晨 00:00–07:59 的「今天」不是昨天（UTC 陷阱）
- 對應：F-01-1、§6 日期規則
- 前置：`clock('2026-10-09T00:30')`（= 2026-10-08T16:30Z）
- 步驟：開啟頁面。
- 預期：日期欄 `2026-10-09`；清單組標題「今天」落在 10/09；預算卡剩餘天數 23 天（非 24）。
- 自動化：同上比對；`body().includes('23 天')`。另跑 `clock('2026-10-09T07:59')` 與 `clock('2026-10-09T23:59')` 都得 `2026-10-09`。

#### T-01-03 沿用上一次分類（含重新整理）
- 對應：F-01-1、§5 `settings.lastCategory`
- 前置：FX-EMPTY
- 步驟：`add({amount:60, category:'飲料'})` → 觀察分類 → reload → 觀察分類。
- 預期：新增後「🧋 飲料」仍為選取；`state().settings.lastCategory === 'drink'`；reload 後預設分類為飲料。
- 自動化：`state().settings.lastCategory`；reload 後 `byText('button','飲料')` 選取態。

#### T-01-04 切到收入換成 4 個收入分類
- 對應：F-01-1
- 步驟：點「收入」→ 數分類按鈕 → 點「支出」。
- 預期：收入分類恰 4 個：💰 生活費、💼 打工、🎓 獎學金、🧧 其他收入；切回支出恢復 8 個；收入時預設選取第一個（或上次用的收入分類——見模糊點 A13）。
- 自動化：切換後 `spec('F-01').querySelectorAll('button')` 中含 emoji 的分類鈕數量；文字集合比對。

#### T-01-05 金額驗證：空白、0、負數、小數、非數字、超過上限
- 對應：F-01-2
- 前置：FX-EMPTY
- 步驟：逐一輸入 `''`、`0`、`-5`、`12.5`、`abc`、`1000000`、`1e3`、`0x10`、` `（只有空白）後按「記一筆」。
- 預期：每次都不儲存（`state().records.length === 0`）；金額欄呈紅色（border 或 class），顯示文字「請輸入 1～999,999 的整數金額」；錯誤文字 `aria-live` 或與欄位 `aria-describedby` 關聯（加分）。
- 自動化：`getComputedStyle(amountEl).borderColor` 為 `rgb(220, 38, 38)`（#DC2626）或 `amountEl.classList` 含錯誤 class；`body().includes('請輸入 1～999,999 的整數金額')`；每輪後 `state().records.length === 0`。
- 備註：`12.0` 是否算小數、`1e3` 是否算非數字，規格未寫（模糊點 A1）；本案例以「拒絕」為預期。

#### T-01-06 金額邊界：1 與 999,999 可存，1,000,000 不可
- 對應：F-01-2
- 步驟：依序 `add({amount:1})`、`add({amount:999999})`、`add({amount:1000000})`。
- 預期：前兩筆成功（toast「已記下 🍱 餐飲 NT$1」「已記下 🍱 餐飲 NT$999,999」），第三筆拒絕；`state().records.map(r=>r.amount)` 為 `[1, 999999]`（順序依實作）。
- 自動化：`state().records.length === 2`；toast 文字含 `NT$999,999`。

#### T-01-07 全形數字與千分位正規化
- 對應：F-01-2
- 步驟：輸入 `１２０` 送出；輸入 `1,200` 送出；輸入 `１,２００`；輸入 ` 90 `（前後空白）；輸入 `0090`。
- 預期：存成 120、1200；`１,２００` → 1200（全形＋半形逗號混用，預期接受）；` 90 ` → 90（trim，模糊點 A1）；`0090` → 90。清單顯示 `NT$120`、`NT$1,200`。
- 自動化：`state().records.map(r=>r.amount)` 含 `120,1200`；注意若金額欄是 `<input type="number">`，瀏覽器會擋全形字、`value` 讀到空字串 → 此案例直接失敗，請建置者用 `type="text" inputmode="numeric"`（模糊點 A1）。

#### T-01-08 未來日期拒絕、今天允許、過去允許
- 對應：F-01-3
- 前置：`clock('2026-10-09T10:00')`
- 步驟：日期欄設 `2026-10-10` 送出；設 `2026-10-09` 送出；設 `2025-12-31` 送出。
- 預期：10/10 不儲存並顯示「不能記錄未來的日期」；10/09 與 2025-12-31 儲存成功（後者 toast「已記到 2025 年 12 月」）。
- 自動化：`setVal(dateEl,'2026-10-10')`（若 `max` 屬性擋住，仍以 JS 設值測邏輯層）；`body().includes('不能記錄未來的日期')`；`state().records.length === 2`。

#### T-01-09 備註最多 30 字、前後空白去除、空白備註
- 對應：F-01-3、§5 note
- 步驟：備註輸入 31 個「字」送出；備註 `　午餐　`（全形空白）送出；備註留空送出。
- 預期：31 字：不超過 30（`maxlength="30"` 截斷，或拒絕並提示——模糊點 A2，二者擇一但要一致，存進去的 `note.length ≤ 30`）；trim 後 `note === '午餐'`；空白備註 `note === ''`，清單該列只顯示分類。
- 自動化：`noteEl.getAttribute('maxlength')`；`state().records.every(r => r.note.length <= 30 && r.note === r.note.trim())`。

#### T-01-10 成功後的表單狀態、焦點與提示
- 對應：F-01-4
- 步驟：`add({amount:90, category:'餐飲', note:'自助餐'})`，立即檢查；等 1.5s；等 2.6s。
- 預期：金額欄與備註清空；分類仍「🍱 餐飲」；日期不變；`document.activeElement === amountEl`；toast「已記下 🍱 餐飲 NT$90」在 1.5s 仍在、2.6s 後消失（允許 ±0.5s）。總覽支出 `NT$90`、清單出現該列、圖表有一個分類。
- 自動化：`amountEl.value === '' && noteEl.value === ''`；`document.activeElement === amountEl`；`toast()` 時序比對；`spec('F-04').textContent.includes('NT$90')`。

#### T-01-11 Enter 送出、不換頁不跳視窗
- 對應：F-01（描述）
- 步驟：在金額欄輸入 `50` 後按 Enter（CDP `Input.dispatchKeyEvent` Enter）。
- 預期：新增成功；`location.href` 不變（無 `?` query、無 reload）；無 `alert`。
- 自動化：監聽 `Page.frameNavigated` 應無事件；`Page.javascriptDialogOpening` 無事件；`state().records.length` +1。

#### T-01-12 記到非檢視月份的提示
- 對應：F-01-4
- 前置：FX-EMPTY，檢視 2026-10
- 步驟：日期 `2026-09-30` 金額 100 送出。
- 預期：toast「已記到 2026 年 9 月」；10 月清單仍空（「這個月還沒有紀錄」）；按「‹」到 9 月看到該筆。
- 自動化：`toast() === '已記到 2026 年 9 月'`；切月後 `byText('li, tr, div','NT$100')` 存在。

#### T-01-13 「今天／昨天」按鈕與月／年邊界
- 對應：F-01（描述）、§6 日期
- 步驟：(a) `clock('2026-10-01T09:00')` 點「昨天」；(b) `clock('2026-01-01T09:00')` 點「昨天」；(c) 改日期後點「今天」。
- 預期：(a) 日期 `2026-09-30`；(b) `2025-12-31`；(c) 回到今天。
- 自動化：`dateEl.value` 比對。

#### T-01-14 收入紀錄儲存與分類相符
- 對應：F-01-1、§5 category 與 type 相符
- 步驟：切「收入」→ 金額 8000 → 分類「生活費」→ 送出。
- 預期：`{type:'income', amount:8000, category:'allowance'}`；總覽收入 `NT$8,000`；toast「已記下 💰 生活費 NT$8,000」。
- 自動化：`state().records[0]` 欄位比對。

#### T-01-15 儲存的資料結構符合 §5
- 對應：§5 資料結構
- 步驟：`add({amount:90, note:'自助餐'})`。
- 預期：`state()` 具 `version:1`、`settings.monthlyBudget`、`settings.lastCategory`、`records[]`；該筆 `id` 符合 `/^[0-9a-z]+-[0-9a-z]{6}$/`、`createdAt` 為整數且 ≈ 當下 `Date.now()`、`date` 符合 `/^\d{4}-\d{2}-\d{2}$/`、`amount` 為 integer。
- 自動化：`Number.isInteger(r.amount) && /^[0-9a-z]+-[0-9a-z]{6}$/.test(r.id)`。

---

## 2. F-02 明細清單

#### T-02-01 依日期新到舊分組、組標題格式
- 對應：F-02-1
- 前置：`clock('2026-10-09T10:00')`；seed 10/09 餐飲 90 + 餐飲 145、10/05 餐飲 2250、10/01 居住 5500。
- 步驟：開啟頁面。
- 預期：組順序 10/09 → 10/05 → 10/01；10/09 標題為「10/09（五）今天　支出 NT$235」（允許全形／半形空白差異，模糊點 A4）；10/05 標題「10/05（一）　支出 NT$2,250」（無「今天」）。
- 自動化：抓各組標題元素（例如 `spec('F-02').querySelectorAll('h3, .group-title')`）→ `textContent.replace(/\s+/g,'')` 比對 `10/09（五）今天支出NT$235`。

#### T-02-02 同一天依建立時間新到舊
- 對應：F-02-1
- 前置：seed 10/09 三筆 `createdAt` 分別 1000、3000、2000，備註 A、C、B。
- 預期：10/09 組內順序 C → B → A。
- 自動化：該組內列的備註文字序列 `['C','B','A']`。

#### T-02-03 每列內容、金額格式與顏色
- 對應：F-02-2、§4 配色
- 前置：seed 10/09 餐飲 90 備註「自助餐」、10/09 收入 打工 4000。
- 預期：支出列含「🍱」「餐飲」「自助餐」「−NT$90」（規格用 U+2212；若實作用 `-` 半形，記為模糊點 A3）、按鈕「編輯」「刪除」；收入列「+NT$4,000」。支出金額色 `#1F2937`（rgb(31,41,55)），收入 `#15803D`（rgb(21,128,61)）。
- 自動化：`getComputedStyle(amountCell).color`；`/[−-]NT\$90/.test(row.textContent)`。

#### T-02-04 無備註只顯示分類
- 對應：F-02-2
- 前置：seed 一筆 `note:''`。
- 預期：該列沒有空的備註節點或多餘分隔符（例如不出現「餐飲 ·」或「餐飲 -」）。
- 自動化：`row.textContent.replace(/\s+/g,' ')` 不含連續分隔符；manual-visual 補看。

#### T-02-05 XSS：備註原樣顯示、不執行
- 對應：F-02-3、§6 安全
- 前置：FX-XSS（經 seed）；另用 UI 新增備註 `<b>test</b>`。
- 預期：清單文字含 `<img src=x onerror=alert(1)>` 與 `<b>test</b>` 字面；`spec('F-02').querySelector('img, b')` 為 null；無 `Page.javascriptDialogOpening`；Console 無錯誤。
- 自動化：`document.body.innerText.includes('<img src=x onerror=alert(1)>')`；`!document.querySelector('[data-spec="F-02"] img')`。

#### T-02-06 空白狀態文字與「載入範例資料」
- 對應：F-02-4
- 前置：FX-EMPTY
- 預期：清單區出現「這個月還沒有紀錄」與按鈕「載入範例資料」；切到 2026-09（也空）同樣顯示。
- 自動化：`byText('[data-spec="F-02"] *','這個月還沒有紀錄')`；`byText('[data-spec="F-02"] button','載入範例資料')`。

#### T-02-07 只有收入的日期之組標題（邊界）
- 對應：F-02-1（規格只舉支出例）
- 前置：seed 10/01 生活費 8000（無支出）。
- 預期：標題仍可讀，不出現「支出 NT$0」以外的 NaN／undefined；若顯示「收入 NT$8,000」或「支出 NT$0」皆接受（模糊點 A4）。
- 自動化：`!/NaN|undefined/.test(title)`；manual-visual。

---

## 3. F-03 編輯、刪除與復原

#### T-03-01 編輯視窗帶入原值
- 對應：F-03-1
- 前置：seed 一筆 `{type:'expense',amount:85,category:'drink',date:'2026-10-05',note:'珍奶'}`。
- 步驟：點該列「編輯」。
- 預期：視窗開啟（`[role=dialog]` 或 `<dialog open>`），類型＝支出、金額欄 `85`、分類「🧋 飲料」選取、日期 `2026-10-05`、備註 `珍奶`；焦點進入視窗。
- 自動化：`dialog.querySelector('input[type=date]').value === '2026-10-05'`；`dialog.contains(document.activeElement)`。

#### T-03-02 編輯時驗證同 F-01
- 對應：F-03-1
- 步驟：編輯視窗內金額改 `0` 儲存；改 `12.5` 儲存；日期改 `2026-10-10`（未來）儲存。
- 預期：都不儲存，顯示同 F-01 的訊息；視窗不關閉；`raw()` 不變。
- 自動化：`raw()` 前後相等；`dialog` 仍 open。

#### T-03-03 儲存後同步更新、id 與 createdAt 不變
- 對應：F-03-2、規格 T-4-3
- 前置：FX-DONUT（含 drink 600 → 先改為 85 方便對照）。
- 步驟：把金額 85 改成 120 儲存。
- 預期：`records[i].id` 與 `createdAt` 不變、`amount === 120`；總覽支出增加 35；甜甜圈重畫（SVG 內容改變）；清單顯示 `−NT$120`；toast（文字規格未定，模糊點 A14）。
- 自動化：記下 `before = state().records.find(...)`；儲存後比對 `id/createdAt` 相等；`svg.innerHTML` 前後不同。

#### T-03-04 編輯改類型、改分類、改日期到其他月份
- 對應：F-03-1、F-03-2
- 步驟：(a) 類型改「收入」→ 分類鈕變 4 個 → 選「打工」儲存；(b) 另一筆日期改 `2026-09-28` 儲存。
- 預期：(a) `type:'income', category:'parttime'`，總覽收入／支出同步；(b) 該筆從 10 月清單消失，切到 9 月可見。
- 自動化：`state()` 比對；`spec('F-02')` 列數 −1。

#### T-03-05 取消、Esc、點視窗外：資料完全不變
- 對應：F-03-2、規格 T-4-3
- 步驟：開啟編輯 → 改金額為 999 → 分別以「取消」／Esc／點 backdrop 關閉（三輪）。
- 預期：`raw()` 三輪前後逐字相同；視窗關閉；清單不變。
- 自動化：`const b = raw(); …; raw() === b`；Esc 用 `Input.dispatchKeyEvent {key:'Escape'}`；backdrop 用 `Input.dispatchMouseEvent` 點視窗外座標。

#### T-03-06 刪除立即移除、不跳確認、5 秒復原提示
- 對應：F-03-3
- 前置：seed 10/09 餐飲 90。
- 步驟：點「刪除」；立即檢查；等 4s；等 5.6s。
- 預期：無 confirm 對話框；該列立即消失；`state().records.length` 減 1（立即寫入）；toast「已刪除 🍱 餐飲 NT$90」加按鈕「復原」；4s 仍在，5.6s 後消失。
- 自動化：`Page.javascriptDialogOpening` 無事件；`byText('[aria-live] button','復原')` 時序。

#### T-03-07 復原：原 id 回到原位、統計恢復
- 對應：F-03-3、規格 T-4-4
- 前置：seed 10/09 三筆（createdAt 1000/2000/3000，備註 A/B/C）。
- 步驟：刪 B → 點「復原」。
- 預期：B 以原 `id`、`createdAt` 回來；清單順序 C、B、A；總覽與圖表回到刪除前；toast 消失或改為一般提示。
- 自動化：`state().records.find(r=>r.id==='test-02')` 存在且 `createdAt === 2000`；列序比對。

#### T-03-08 連刪 A、B 後復原只救回 B
- 對應：F-03-4
- 步驟：刪 A → 刪 B → 點「復原」→（再找「復原」）。
- 預期：只有 B 回來；A 仍不存在；復原後不再出現第二個「復原」。
- 自動化：`state().records.map(r=>r.note)` 不含 A、含 B。

#### T-03-09 刪除最後一筆 → 空白狀態、再復原
- 對應：F-03-3、F-02-4、F-06-5
- 前置：seed 只有一筆支出。
- 步驟：刪除 → 檢查 → 復原。
- 預期：刪後清單「這個月還沒有紀錄」、圖表「這個月還沒有支出紀錄」、總覽 `NT$0`、預算卡 0%（綠）、無 NaN；復原後全部恢復。
- 自動化：`!/NaN|Infinity/.test(body())`；文字存在性。

#### T-03-10 復原逾時與 reload 後不可復原（暫存只在記憶體）
- 對應：F-03-3、§5「不存檔」
- 步驟：(a) 刪除後等 6s；(b) 刪除後立即 reload。
- 預期：(a) 無「復原」按鈕，資料已永久移除；(b) reload 後紀錄不存在、無復原提示。
- 自動化：`!byText('button','復原')`；`state().records.length`。

---

## 4. F-04 月份切換與本月收支摘要

#### T-04-01 預設本月、「›」停用、無「回到本月」
- 對應：F-04-1
- 前置：`clock('2026-10-09T10:00')`
- 預期：月份列文字「2026 年 10 月」；「›」按鈕 `disabled`（或 `aria-disabled="true"` 且點了沒反應）；「回到本月」不存在或隱藏。
- 自動化：`byText('button','›').disabled === true`；`!byText('button','回到本月') || hidden`；點「›」後標題仍 10 月。

#### T-04-02 上一月、「回到本月」出現、「›」恢復
- 對應：F-04-1
- 步驟：點「‹」→ 檢查 → 點「回到本月」。
- 預期：標題「2026 年 9 月」；「回到本月」可見；「›」可用；點「回到本月」後回 10 月且「回到本月」消失。
- 自動化：文字比對；`offsetParent !== null` 判可見。

#### T-04-03 跨年：2026/1 ‹ → 2025/12；12 › → 1
- 對應：F-04-2、§6「年 × 12 ＋ 月」
- 前置：`clock('2026-01-15T10:00')`
- 步驟：點「‹」；再點「›」。
- 預期：「2025 年 12 月」→「2026 年 1 月」。
- 自動化：文字比對。

#### T-04-04 連續切換 14 個月再切回不漂移（setMonth 陷阱）
- 對應：§6「不用 setMonth」
- 前置：`clock('2026-03-31T10:00')`（31 日，容易觸發 setMonth 溢位）
- 步驟：點「‹」14 次 → 點「›」14 次。
- 預期：途中依序 2026/2、2026/1、2025/12…2025/1；回程停在「2026 年 3 月」且「›」停用。
- 自動化：每步記錄標題，與預期序列比對。

#### T-04-05 摘要三格驗算
- 對應：F-04-3、規格 T-2
- 前置：FX-T2
- 預期：收入 `NT$12,000`、支出 `NT$7,985`、結餘 `NT$4,015`；結餘非紅色。
- 自動化：`spec('F-04').innerText` 含三個值；結餘元素 `color !== 'rgb(220, 38, 38)'`。

#### T-04-06 結餘負數紅字
- 對應：F-04-3
- 前置：seed 10 月：收入 1,000、支出 1,500。
- 預期：結餘顯示 `−NT$500`（或 `-NT$500`）且色 `#DC2626`。
- 自動化：`getComputedStyle(balanceEl).color === 'rgb(220, 38, 38)'`。

#### T-04-07 切換月份時所有區塊換成該月資料
- 對應：F-04-4
- 前置：seed 10 月 FX-T2 + 9 月：9/10 餐飲 300、9/15 打工 4000。
- 步驟：點「‹」。
- 預期：摘要 `NT$4,000／NT$300／NT$3,700`；預算卡顯示過去月份結算（見 T-05-09）；甜甜圈中央 `NT$300`；長條圖 30 根；清單只有 9 月兩筆。
- 自動化：各區 `innerText` 比對；`spec('F-06').querySelectorAll('rect').length`（依實作可能含虛線以外的元素，先過濾 `class`）。

#### T-04-08 重新整理後回到本月（檢視月份不存檔）
- 對應：F-04-4、§5
- 步驟：切到 9 月 → reload。
- 預期：標題回「2026 年 10 月」；localStorage 內無檢視月份欄位。
- 自動化：`!('viewMonth' in state()) && !/month/i.test(Object.keys(state().settings).join())`（寬鬆）。

#### T-04-09 月底／月初的本月判定
- 對應：F-04-1
- 步驟：`clock('2026-10-31T23:59')` 開啟；`clock('2026-11-01T00:01')` 開啟。
- 預期：分別為「2026 年 10 月」「2026 年 11 月」，皆「›」停用。
- 自動化：文字比對。

---

## 5. F-05 月預算、進度條與「今天還能花」

#### T-05-01 預算輸入範圍與預設值
- 對應：F-05 描述
- 前置：FX-EMPTY
- 步驟：讀預算欄預設；輸入 `0`、`999999`、`1000000`、`-1`、`12.5`、`abc`（每次觸發 change／blur／或按「儲存預算」——依實作，模糊點 A5）。
- 預期：預設 `8000`；`0` 與 `999999` 接受並寫入 `settings.monthlyBudget`；其餘拒絕（不寫入、欄位變紅或提示；文字規格未定）。
- 自動化：每輪 `state().settings.monthlyBudget` 比對 `[0, 999999, 999999, 999999, 999999, 999999]`。

#### T-05-02 規格驗算（T-2）：今天還能花 NT$15
- 對應：F-05-2、規格 T-2
- 前置：FX-T2，`clock('2026-10-09T10:00')`
- 預期（`spec('F-05').innerText`）：「今天還能花」大數字 `NT$15`；「今日額度 NT$250」；「已用 31%」且狀態「✅ 預算充足」綠色（#22A06B）；「剩 NT$5,515」；「還有 23 天」；「月底預估 NT$8,559」且附「超出 NT$559」橘字（#B45309）；進度條寬 31%。
- 自動化：正規化後 `includes('NT$15')`、`includes('31%')`、`includes('NT$8,559')`、`includes('NT$559')`；`parseFloat(bar.style.width)` ≈ 31（或 `rec(bar).width/rec(track).width ≈ 0.31 ±0.01`）；`getComputedStyle(statusEl).color/backgroundColor` 含 `rgb(34, 160, 107)`。

#### T-05-03 🏠 居住不計入預算、卡片註明
- 對應：F-05 描述
- 前置：FX-T2
- 預期：已花以 2,485 計（非 7,985）；卡片含文字「🏠 居住不計入預算」；長條圖 10/01 的高度不含 5,500。
- 自動化：`spec('F-05').innerText.includes('居住不計入預算')`；已用百分比 31% 而非 99%。

#### T-05-04 恰好 80%：橘色「⚠️ 快用完了，剩 NT$1,600」
- 對應：F-05-1
- 前置：預算 8,000；10/01 餐飲 6,400；`clock('2026-10-09T10:00')`
- 預期：80%；狀態文字「⚠️ 快用完了，剩 NT$1,600」；顏色 #F59E0B（條）／#B45309（字）；對照組 6,399 → 79% 綠「✅ 預算充足」。
- 自動化：兩組 seed 各跑一次，比對文字與 `rgb(245, 158, 11)`／`rgb(180, 83, 9)`。

#### T-05-05 99% 與 100%：橘 → 紅「🚨 預算已用完」
- 對應：F-05-1
- 前置：預算 8,000；支出 7,999 → 7,992（99%，無條件捨去 7,992/8,000=99.9%→99%）→ 8,000。
- 預期：7,999／7,992 皆「99%」橘「⚠️ 快用完了，剩 NT$1」／「剩 NT$8」；8,000 →「100%」紅「🚨 預算已用完」（不是「已超支 NT$0」），條寬 100%。
- 自動化：文字與色 `rgb(220, 38, 38)`。

#### T-05-06 超支（T-3）：預算 3,000 與 2,400
- 對應：F-05-1、F-05-3、規格 T-3
- 前置：FX-T2
- 步驟：預算改 3,000 → 檢查；改 2,400 → 檢查。
- 預期：3,000：「82%」橘「⚠️ 快用完了，剩 NT$515」、今天還能花 紅字「今天已超出 NT$203」（今日額度 ⌊750/23⌋=32，32−235=−203）。2,400：「103%」紅「🚨 已超支 NT$85」、大數字改「🚨 本月已超支 NT$85」、進度條寬度 = 100%（`rec(bar).width ≤ rec(track).width`、不溢出卡片 `rec(bar).right ≤ rec(card).right`）。
- 自動化：文字比對；幾何比對；`parseFloat(bar.style.width) <= 100`。

#### T-05-07 月底預估「5 日起才顯示」
- 對應：F-05 公式
- 前置：預算 8,000；10/01 餐飲 500；分別 `clock('2026-10-04T10:00')`、`clock('2026-10-05T10:00')`。
- 預期：10/04 卡片無「月底預估」字樣；10/05 顯示「月底預估 NT$3,100」（500/5×31=3,100，預估 ≤ 預算時的附註文字規格未定，模糊點 A16）。
- 自動化：`includes('月底預估')` 布林比對。

#### T-05-08 取整規則：今日額度 floor、已用 floor、預估 round
- 對應：F-05 公式
- 前置：預算 8,000；10/01 餐飲 2,251；10/09 餐飲 0 筆；`clock('2026-10-09T10:00')`
- 預期：今日額度 ⌊5,749/23⌋ = 249（非 250）；已用 ⌊2251/8000×100⌋ = 28%；預估 round(2251/9×31)=7,753（7753.4→7753）。
- 自動化：文字含 `NT$249`、`28%`、`NT$7,753`。

#### T-05-09 過去月份：結算文字、無今日額度與預估
- 對應：F-05-4
- 前置：9 月：餐飲 7,000（預算 8,000）；另一組 9 月餐飲 9,000。
- 步驟：點「‹」。
- 預期：第一組「結算：剩 NT$1,000」；第二組「結算：超支 NT$1,000」；卡片無「今日額度」「月底預估」「今天還能花」「還有 N 天」。
- 自動化：`!includes('今日額度') && !includes('月底預估')`。

#### T-05-10 未設預算（0）：只顯示「尚未設定預算」與「設定預算」
- 對應：F-05-4
- 前置：`settings.monthlyBudget = 0` + FX-T2 紀錄。
- 預期：預算卡只有「尚未設定預算」與按鈕「設定預算」；無百分比、無進度條、無 NaN／Infinity；點「設定預算」將焦點移到預算欄（或捲到設定區）。
- 自動化：`!/NaN|Infinity|undefined/.test(body())`；點擊後 `document.activeElement === budgetEl`（寬鬆：`budgetEl` 進入 viewport）。

#### T-05-11 畫面任何狀態不可出現 NaN／Infinity（掃描）
- 對應：F-05-4
- 步驟：對 FX-EMPTY、FX-T2、預算 0、只有收入、過去月份、未來 reload 各狀態執行。
- 預期：`body()` 不符合 `/NaN|Infinity|undefined|null/`。
- 自動化：每個案例結束統一斷言（放進 harness 的 afterEach）。

#### T-05-12 紀錄或預算改變立即更新 + 0.3 秒過場
- 對應：F-05-5
- 前置：FX-T2
- 步驟：新增 10/09 餐飲 5,000 → 立即讀；改預算 → 讀。
- 預期：文字與顏色在同一事件循環後更新（不需 reload）；進度條元素 `transition-duration` ≈ `0.3s`（寬度、顏色）。
- 自動化：`getComputedStyle(bar).transitionDuration` 含 `0.3s`；`await new Promise(r=>setTimeout(r,400))` 後 `rec(bar).width` 變化。

#### T-05-13 月初與月底的剩餘天數
- 對應：F-05 公式
- 步驟：(a) `clock('2026-10-01T09:00')` 預算 8,000 無紀錄；(b) `clock('2026-10-31T09:00')`，10/01 餐飲 7,000。
- 預期：(a) 還有 31 天、今日額度 ⌊8000/31⌋=258、今天還能花 258；(b) 還有 1 天、今日額度 1,000、今天還能花 1,000。
- 自動化：文字比對。

#### T-05-14 2 月與閏年天數
- 對應：§6 `new Date(y, m, 0)`
- 步驟：`clock('2028-02-10T09:00')` 預算 2,900。
- 預期：還有 20 天（29−10+1）；今日額度 ⌊2900/20⌋=145；長條圖 10 根（1–10 日）。
- 自動化：文字比對；bar 數。

#### T-05-15 預算持久化與即時寫入
- 對應：F-05、F-08-1
- 步驟：預算改 5,000 → 立即 `state()` → reload。
- 預期：`settings.monthlyBudget === 5000`，reload 後預算欄 `5000`、卡片以 5,000 計算。
- 自動化：如上。

---

## 6. F-06 圖表（SVG 自繪）

#### T-06-01 圖表為 inline SVG、無圖表函式庫
- 對應：F-06 描述、§6
- 預期：`spec('F-06').querySelectorAll('svg').length >= 2`；`document.querySelectorAll('script[src], link[rel=stylesheet][href]').length === 0`；全域無 `Chart`／`d3`／`echarts`。
- 自動化：`typeof window.Chart === 'undefined' && typeof window.d3 === 'undefined'`。

#### T-06-02 甜甜圈百分比、排行順序、中央總支出（含居住）
- 對應：F-06-1、F-06 描述
- 前置：FX-DONUT；第二輪加 10/01 居住 5,500。
- 預期：排行文字依序「🍱 餐飲 NT$3,000 75%」「🧋 飲料 NT$600 15%」「🚇 交通 NT$400 10%」；中央「NT$4,000」。加居住後：中央 `NT$9,500`，居住在排行第一（5,500 58%，其餘 32%／6%／4%，加總 100±1）。
- 自動化：抓排行項目文字陣列，正則 `/(\d+)%/` 取值，`sum ∈ [99,101]`；順序遞減。

#### T-06-03 新增／修改／刪除／換月後重畫
- 對應：F-06-2
- 前置：FX-DONUT
- 步驟：記下兩張 SVG 的 `innerHTML` 與弧段數；新增 學業 100 → 比對；編輯 → 比對；刪除 → 比對；點「‹」→ 比對。
- 預期：每次動作後至少一張 SVG 內容改變；分類數對應弧段數（path 數 = 有支出的分類數）。
- 自動化：`svg.innerHTML !== prev`；`svg.querySelectorAll('path').length === 4`。

#### T-06-04 點排行分類 → 套用篩選並捲到清單
- 對應：F-06-3、規格 T-5-1
- 前置：FX-DONUT；`viewport(375, 700)`（讓清單在視窗外）。
- 步驟：點排行「🧋 飲料」。
- 預期：分類下拉值變成飲料（或等價篩選態）；清單只剩 1 筆（珍奶）；篩選列「符合 1 筆，支出 NT$600／收入 NT$0」；`window.scrollY` 增加且清單頂端進入 viewport（允許 smooth scroll，等 800ms）；總覽卡與甜甜圈數字不變。
- 自動化：`rec(spec('F-02')).top < innerHeight`；`spec('F-04').innerText` 前後相同。

#### T-06-05 每日長條：1 日到今天、未來不畫、過去月份畫到月底
- 對應：F-06 描述
- 前置：FX-T2，`clock('2026-10-09T10:00')`；9 月有資料。
- 預期：10 月有 9 根長條（或 31 個 x 位置但 10–31 無 rect）；切到 9 月有 30 根；長條只算計入預算支出（10/01 高度對應 0，而非 5,500）。
- 自動化：`[...svg.querySelectorAll('rect')].filter(r => r.dataset.day || r.getAttribute('height') !== null)` 計數；10/01 的 rect `height` 為 0 或不存在。

#### T-06-06 平均預算虛線與超線變橘
- 對應：F-06 描述、規格 T-3-2
- 前置：FX-T2；預算 2,400 → 日均 2400/31 = 77.4。
- 預期：存在 `stroke-dasharray` 的 line；10/05（2,250）與 10/09（235）長條 fill = `#F59E0B`（rgb(245,158,11)）；預算 0 時無虛線、無橘條。
- 自動化：`svg.querySelector('line[stroke-dasharray]')`；`getComputedStyle(rect).fill`。

#### T-06-07 今天長條加外框
- 對應：F-06 描述
- 前置：FX-T2
- 預期：10/09 的 rect 有 `stroke`（非 none）且 stroke-width > 0；其他日無。
- 自動化：比對 `rect.getAttribute('stroke')` 或 computed `stroke`。

#### T-06-08 點或滑過長條顯示「10/05（一）NT$320」
- 對應：F-06-4
- 前置：10/05 餐飲 320。
- 步驟：對該 rect 派發 `mouseover`／`pointerenter`（或 CDP `Input.dispatchMouseEvent mouseMoved` 到其中心）；再 `click`。
- 預期：出現含「10/05（一）NT$320」的 tooltip（DOM 元素、`<title>` 子元素或 `aria-label` 皆接受）。
- 自動化：`body().includes('10/05（一）NT$320') || rect.querySelector('title')?.textContent === '10/05（一）NT$320'`。

#### T-06-09 當月無支出：不畫空圖、顯示「這個月還沒有支出紀錄」
- 對應：F-06-5
- 前置：(a) FX-EMPTY；(b) 只有收入 8,000。
- 預期：兩種情況圖表卡顯示「這個月還沒有支出紀錄」；無 path／rect 的 SVG（或 SVG 隱藏）；(b) 總覽收入仍 `NT$8,000`。
- 自動化：`byText('[data-spec="F-06"] *','這個月還沒有支出紀錄')`；`svg.querySelectorAll('path, rect').length === 0` 或 svg hidden。

#### T-06-10 四捨五入加總容忍 ±1%
- 對應：F-06-1
- 前置：三個分類各 100（33.3%×3）。
- 預期：顯示 33%／33%／33%（加總 99，容許）；無 NaN。
- 自動化：同 T-06-02 的加總檢查。

---

## 7. F-07 篩選與搜尋

#### T-07-01 篩選列三個控制項、分類選項跟著類型變
- 對應：F-07-1
- 前置：FX-DONUT
- 預期：類型 select 選項「全部／支出／收入」；分類 select 在「支出」時含 8 個支出分類＋「全部分類」、「收入」時含 4 個收入分類＋「全部」、「全部」時含全部 12 個（或只有「全部分類」——模糊點 A8，二者皆接受但不得殘留錯誤類型的選項）；關鍵字輸入欄存在且有 label。
- 自動化：`[...typeSel.options].map(o=>o.textContent)`；切換後 `catSel.options.length`。

#### T-07-02 關鍵字即時篩選、不分大小寫與全半形
- 對應：F-07-1、F-07-3
- 前置：seed 備註「珍奶」「Milk Tea」「ｍｉｌｋ」「午餐 12 號」「午餐 １２ 號」。
- 步驟：輸入 `珍奶`；輸入 `milk`；輸入 `MILK`；輸入 `12`；輸入 `１２`（每次只派發 `input` 事件，不按 Enter）。
- 預期：`珍奶` → 1 筆；`milk`／`MILK` → 2 筆（含全形 ｍｉｌｋ）；`12`／`１２` → 2 筆。
- 自動化：每次輸入後 `spec('F-02')` 內紀錄列數。

#### T-07-03 關鍵字比對分類名稱
- 對應：F-07-1
- 前置：FX-DONUT（交通那筆改成空備註）。
- 步驟：輸入 `交通`；輸入 `🚇`（emoji，加分）。
- 預期：`交通` 篩出 1 筆（備註為空，靠分類名稱比對）。
- 自動化：列數 `=== 1`。

#### T-07-04 條件 AND 與統計列「符合 N 筆，支出 NT$X／收入 NT$Y」
- 對應：F-07-2
- 前置：FX-DONUT + 10/15 打工 4,000 備註「珍奶店打工」。
- 步驟：關鍵字 `珍奶` → 類型「收入」。
- 預期：關鍵字後 2 筆「符合 2 筆，支出 NT$600／收入 NT$4,000」；加類型收入後 1 筆「符合 1 筆，支出 NT$0／收入 NT$4,000」；「清除篩選」按鈕可見。
- 自動化：正規化後文字比對。

#### T-07-05 無結果：「找不到符合的紀錄」＋「清除篩選」
- 對應：F-07-3、規格 T-5-2
- 前置：FX-DONUT
- 步驟：關鍵字 `珍奶` → 類型「收入」。
- 預期：清單顯示「找不到符合的紀錄」與「清除篩選」；不顯示「這個月還沒有紀錄」。
- 自動化：文字存在／不存在。

#### T-07-06 篩選只影響清單
- 對應：F-07-4、規格 T-5-1
- 前置：FX-DONUT
- 步驟：記下 `spec('F-04').innerText`、`spec('F-05').innerText`、兩張 SVG `innerHTML`；套用分類「飲料」。
- 預期：四者完全不變。
- 自動化：字串相等。

#### T-07-07 清除篩選重設三個控制項
- 對應：F-07-2
- 步驟：三個條件都設 → 點「清除篩選」。
- 預期：類型「全部」、分類「全部」、關鍵字空；清單回全部；統計列與「清除篩選」隱藏（或顯示全部筆數——模糊點 A9）。
- 自動化：`typeSel.value`、`catSel.value`、`kwEl.value === ''`。

#### T-07-08 篩選狀態不存檔、reload 後清空
- 對應：§5「不存檔」
- 步驟：設關鍵字 → reload。
- 預期：關鍵字欄空、清單全部。
- 自動化：`kwEl.value === ''`。

#### T-07-09 切換月份時篩選行為一致（記錄實作選擇）
- 對應：F-07、F-04-4（規格未明，模糊點 A9）
- 步驟：套用「飲料」→ 點「‹」→ 回「›」。
- 預期：二擇一且一致：(a) 篩選保留並套用到新月份；(b) 切月清除篩選。不得出現統計列與清單不一致（統計 N 筆 ≠ 實際列數）。
- 自動化：`統計列 N === 列數`。

---

## 8. F-08 自動儲存與錯誤處理

#### T-08-01 每個動作後立即寫入、沒有「儲存」按鈕
- 對應：F-08-1
- 步驟：新增 → `state()`；編輯 → `state()`；刪除 → `state()`；改預算 → `state()`。
- 預期：每一步 `state()` 立刻反映；頁面主區（非編輯視窗、非設定區的「儲存預算」）沒有名為「儲存」的全域按鈕。
- 自動化：動作後同步 `localStorage.getItem(KEY)` 比對；`byText('button','儲存')` 僅允許出現在 dialog 內或預算欄旁（若預算用按鈕送出）。

#### T-08-02 重新整理與重開瀏覽器後資料完全相同
- 對應：F-08-1、規格 T-6-1
- 步驟：UI 新增 3 筆 + 預算 6,000 → 記 `raw()` → reload → 比對；關閉 context 以 `Storage.getStorageKeyForFrame`／或用同一 user-data-dir 重啟瀏覽器再開 → 比對。
- 預期：`raw()` 逐字相同；畫面資料相同。
- 自動化：字串相等；`spec('F-02')` 列數相同。

#### T-08-03 主資料不是合法 JSON：黃色橫幅、空白資料、原字串另存
- 對應：F-08-2、規格 T-6-4
- 步驟：`localStorage.setItem(KEY,'abc')` → reload。
- 預期：頁面正常開啟（無未捕捉例外）；清單空白狀態；頂部橫幅「資料讀取失敗，原始內容已另存」，背景 `#FEF3C7`（rgb(254,243,199)）；`localStorage.getItem(BKEY) === 'abc'`；接著新增一筆後 `state()` 為合法結構且 `BKEY` 仍保留 `abc`。
- 自動化：`getComputedStyle(banner).backgroundColor === 'rgb(254, 243, 199)'`；`errors().length === 0`。

#### T-08-04 合法 JSON 但結構錯誤（`[]`、`null`、`{"records":"x"}`、`"str"`）
- 對應：F-08-2（規格只寫「不是合法 JSON」，模糊點 A7）
- 步驟：逐一 seed 字串後 reload。
- 預期：至少不得崩潰、不得出現 NaN、不得把壞資料當作 records 迭代；建議同 T-08-03 處理（橫幅＋另存）。
- 自動化：`errors().length === 0`；`!/NaN|undefined/.test(body())`。

#### T-08-05 localStorage 不可用：橫幅且其他功能照常
- 對應：F-08-3
- 前置：`addScriptToEvaluateOnNewDocument`：`Object.defineProperty(window,'localStorage',{get(){throw new DOMException('denied','SecurityError')}})`；reload。
- 預期：橫幅「目前無法儲存，關閉頁面後資料會遺失，請先匯出備份」；仍可新增紀錄（清單出現）、圖表更新、匯出 CSV 可運作；Console 無紅色錯誤。
- 自動化：文字存在；`add()` 後列數 +1；`errors().length === 0`。

#### T-08-06 localStorage 已滿：寫入時才出現橫幅
- 對應：F-08-3
- 前置：覆寫 `Storage.prototype.setItem = () => { throw new DOMException('quota','QuotaExceededError') }`（讀取正常）。
- 步驟：開啟（可正常讀）→ 新增一筆。
- 預期：新增後出現同上橫幅；該筆仍在記憶體與畫面；無紅色錯誤。
- 自動化：同上。

#### T-08-07 整個 suite Console 無紅色錯誤
- 對應：F-08-4、規格 T-1-1
- 步驟：harness 的全域 afterEach。
- 預期：`Runtime.exceptionThrown` 0 次、`console.error` 0 次（允許 `console.warn`）。
- 自動化：計數器斷言；輸出首個錯誤的 stack 供除錯。

#### T-08-08 `file://` 開啟功能相同
- 對應：§6
- 步驟：以 `file:///…/index.html` 跑 T-01-10、T-08-02、T-09-01。
- 預期：全部通過；無任何網路請求（`Network.requestWillBeSent` 只有 document 本身）。
- 自動化：請求計數 `=== 1`。

---

## 9. F-09 匯出 CSV、JSON 備份與匯入

#### T-09-01 CSV 檔名與月份
- 對應：F-09-1
- 前置：FX-T2 + 9 月資料。
- 步驟：點「匯出 CSV」；切到 9 月再點。
- 預期：檔名 `expenses-2026-10.csv`、`expenses-2026-09.csv`。
- 自動化：`window.__downloads.at(-1).name`。

#### T-09-02 CSV 內容：欄位、整月不受篩選、金額純數字
- 對應：F-09-1
- 前置：FX-DONUT；先套用分類「飲料」篩選。
- 步驟：匯出。
- 預期：第一行（去 BOM）`日期,類型,分類,金額,備註`；資料列 3 行（不受篩選）；`類型` 為「支出」／「收入」；`金額` 欄符合 `/^\d+$/`（無 `NT$`、無逗號）；日期 `YYYY-MM-DD`；分類欄為名稱（含或不含 emoji 皆接受，模糊點 A11）。
- 自動化：`text.replace(/^﻿/,'').split('\r\n').filter(Boolean)` → 解析；支出金額加總 `=== 4000`（對應規格 T-6-2）。

#### T-09-03 CSV BOM 與 `\r\n`
- 對應：F-09-2
- 預期：`bytes[0..2] === [0xEF,0xBB,0xBF]`；每行以 `\r\n` 結尾且不含孤立 `\n`；最後一行可有可無結尾換行。
- 自動化：`new Uint8Array(bytes).slice(0,3)`；`!/[^\r]\n/.test(text)`。

#### T-09-04 CSV 跳脫：逗號、引號、換行、公式前綴
- 對應：F-09-2、規格 T-6-2
- 前置：seed 備註 `雞排, 珍奶`、`他說"好"`、`第一行\n第二行`（只能透過 seed／匯入進入，UI 單行輸入無法輸入換行）、`=1+1`、`+886`、`-5`、`@me`。
- 預期：`"雞排, 珍奶"`；`"他說""好"""`；換行者整格加引號；公式前綴者以 `'` 開頭（如 `'=1+1`；若同時需要引號則 `"'=1+1"`）。
- 自動化：逐行正則；用簡易 CSV parser 回讀後 `備註` 欄位等於原值（公式者等於 `'`+原值）。

#### T-09-05 JSON 備份檔名與內容
- 對應：F-09-3
- 前置：FX-T2 + 9 月資料；`clock('2026-10-09T10:00')`；切到 9 月檢視。
- 步驟：點「匯出備份（JSON）」。
- 預期：檔名 `expense-backup-2026-10-09.json`（以今天為準，與檢視月份無關）；內容 `JSON.parse` 後 deep-equal `state()`（含 `version`、`settings`、**全部月份** records）。
- 自動化：`JSON.stringify(JSON.parse(text)) === JSON.stringify(state())`（或 key 排序後比對）。

#### T-09-06 匯入合法備份：確認文字、取代、整頁更新
- 對應：F-09-4、規格 T-6-3
- 前置：目前 2 筆；備份檔 5 筆、預算 6,000。
- 步驟：`DOM.setFileInputFiles` 設定檔案 → 觸發 `change`。
- 預期：確認訊息「將以備份取代目前資料（目前 2 筆 → 備份 5 筆）」；取消 → `raw()` 不變；確認 → `state()` 等於備份、預算欄 6,000、清單／圖表／摘要全部更新、無需 reload。
- 自動化：`Page.javascriptDialogOpening.message` 比對或 modal 文字；接受後 `state().records.length === 5`。

#### T-09-07 匯入壞檔：不可解析 JSON
- 對應：F-09-4、規格 T-6-3
- 步驟：匯入內容 `{abc`。
- 預期：顯示「檔案格式不正確…未匯入任何資料」（無法解析時「第 N 筆」如何呈現規格未定，模糊點 A6）；`raw()` 不變；無紅色錯誤。
- 自動化：文字含「檔案格式不正確」與「未匯入任何資料」。

#### T-09-08 匯入壞檔：records 非陣列、單筆欄位不合法（指出第 N 筆）
- 對應：F-09-4
- 步驟：逐一匯入：`{"records":{}}`；`{"records":[ok, {type:'x'}]}`；`[ok, {amount:0}]`；`[ok, {amount:1.5}]`；`[ok, {amount:"90"}]`；`[ok, {amount:1000000}]`；`[ok, {type:'income',category:'food'}]`；`[ok, {date:'2026/10/09'}]`；`[ok, {date:'2026-13-01'}]`；`[ok, {date:'2026-02-30'}]`。
- 預期：每次顯示「檔案格式不正確（第 2 筆），未匯入任何資料」（1-based；records 非陣列時 N 的呈現為模糊點 A6）；`raw()` 全程不變。
- 自動化：文字含「第 2 筆」；`raw()` 相等。

#### T-09-09 匯入往返：匯出 → 清除全部 → 匯入 → 完全恢復
- 對應：F-09、規格 T-6-3
- 步驟：FX-T2 匯出 JSON → 「清除全部資料」確認 → 匯入該檔確認。
- 預期：`raw()` 與匯出前 deep-equal（`records` 順序與 id 相同、預算相同）。
- 自動化：deep-equal。

#### T-09-10 匯入邊界：缺 settings、未來日期、備註超過 30 字、重複 id
- 對應：F-09-4（規格未明，模糊點 A6）
- 步驟：分別匯入 `{"version":1,"records":[ok]}`（無 settings）；含 `date:'2027-01-01'`；含 31 字 note；兩筆相同 id。
- 預期：記錄實作選擇即可，但：缺 settings 不得崩潰（預算回預設 8,000 或保留現值）；畫面不得出現 NaN；重複 id 不得造成刪除／編輯錯筆。
- 自動化：`errors().length === 0`；若接受未來日期，F-01-3 的「不能記錄未來的日期」在清單層不受影響但長條圖「未來日期不畫」必須仍成立。

#### T-09-11 匯入控制項可用鍵盤觸發、有 label
- 對應：F-09-4、F-11-4
- 預期：`input[type=file]` 有 `<label>` 或由可聚焦的按鈕觸發；`accept` 含 `.json`。
- 自動化：`fileInput.labels.length > 0 || byText('button','匯入備份')`。

---

## 10. F-10 範例資料與清除全部

#### T-10-01 載入範例資料：範圍、不含未來、內容規則
- 對應：F-10-1、規格 T-1-2
- 前置：FX-EMPTY；`clock('2026-10-09T10:00')`
- 步驟：點空白狀態的「載入範例資料」。
- 預期（對 `state().records` 驗證）：
  - 所有 `date` ∈ [`2026-09-01`, `2026-10-09`]；無 > 今天。
  - 9/1–10/9 每一天至少 2 筆 `food`（早餐、午餐）；部分天有晚餐／`drink`／`transport`；每個週六（9/5、9/12、9/19、9/26、10/3）有 `fun`。
  - 9/1 與 10/1 各有 `housing 5500` 與 `allowance 8000`；9/15 有 `parttime 4000`（10/15 > 今天，不得出現）。
  - 9 月（支出 − 5,500）/30 ∈ [250, 300]。
  - 每筆 `id` 以 `demo-` 開頭、欄位合法（同 F-01 驗證）。
- 畫面：總覽、兩張圖、清單都有資料；點「‹」9 月也有完整資料；10 月的預估與今天還能花非 NaN。
- 自動化：上述規則以 JS 計算；`records.every(r => r.id.startsWith('demo-'))`。

#### T-10-02 再按一次不重複；自己的紀錄保留
- 對應：F-10-2、規格 T-1-2
- 步驟：載入範例 → 記 N → UI 新增一筆自己的紀錄（id 非 demo-）→ 再按「載入範例資料」（設定區的那顆）。
- 預期：demo 筆數 = N（舊 demo 先刪再生，可能 id 不同但數量相同）；自己的紀錄仍在；總筆數 N+1。
- 自動化：`records.filter(r=>r.id.startsWith('demo-')).length === N`；`records.some(r=>!r.id.startsWith('demo-'))`。

#### T-10-03 「移除範例資料」只刪範例
- 對應：F-10-2
- 步驟：接續 T-10-02 → 點「移除範例資料」。
- 預期：`records` 只剩自己的 1 筆；畫面同步；localStorage 已寫入。按鈕在無 demo 時是否顯示為模糊點 A12。
- 自動化：`records.every(r => !r.id.startsWith('demo-'))`。

#### T-10-04 兩個入口都有「載入範例資料」
- 對應：F-10-1
- 預期：空白狀態（清單區）與「設定與資料」區各一顆；載入後清單區那顆隨空白狀態消失，設定區仍在。
- 自動化：`document.querySelectorAll('button')` 中文字為「載入範例資料」者 `=== 2`（空白時）。

#### T-10-05 範例資料的月份邊界：今天是 1 日、今天在 1 月
- 對應：F-10-1
- 步驟：(a) `clock('2026-10-01T09:00')` 載入；(b) `clock('2026-01-10T09:00')` 載入。
- 預期：(a) 10 月只有 10/01 的紀錄（含房租與生活費）、9 月整月；(b) 上個月為 2025-12（整月 31 天），無 2026-02 紀錄。
- 自動化：date 集合的 min／max 比對。

#### T-10-06 清除全部資料：確認文字、取消不變、確認後空白且預算 8,000
- 對應：F-10-3
- 前置：範例資料已載入（N 筆）、預算改 5,000。
- 步驟：點「清除全部資料」→ 取消 → 再點 → 確認。
- 預期：訊息「確定刪除全部 N 筆紀錄？建議先匯出備份」（N 為實際筆數）；取消後 `raw()` 不變；確認後 `records.length === 0`、`monthlyBudget === 8000`、預算欄顯示 8000、清單空白狀態、圖表「這個月還沒有支出紀錄」、復原提示不存在（`lastCategory` 是否重設為模糊點 A15）。
- 自動化：dialog message 正則 `/確定刪除全部 \d+ 筆紀錄？建議先匯出備份/`；`state()` 比對。

#### T-10-07 範例資料不觸發「未來日期」或驗證錯誤
- 對應：F-10-1 + F-01-2/3
- 步驟：載入範例後對每筆跑與表單相同的驗證函式（若有全域暴露）或以匯出 JSON → 匯入的方式驗證。
- 預期：匯入自己的匯出檔不會報「檔案格式不正確」。
- 自動化：T-09-09 流程套用在範例資料上。

---

## 11. F-11 響應式版面與操作回饋

#### T-11-01 手機寬度單欄、無水平捲軸（360／375／430px）
- 對應：F-11-1、規格 T-1-3
- 前置：範例資料已載入（內容最多的狀態）；另跑 FX-T2 預算 2,400（超支狀態）與長備註 30 字。
- 步驟：`viewport(360,740)`、`viewport(375,667)`、`viewport(430,932)`，各自 reload。
- 預期：`hscroll() === false`；所有卡片 `rec(card).left` 相同（單欄）且 `rec(card).right ≤ innerWidth`；SVG 寬度 ≤ 卡片寬；分類按鈕為 4 欄 × 2 列（8 顆的 `top` 值只有 2 種）。
- 自動化：`document.documentElement.scrollWidth <= document.documentElement.clientWidth`；`[...document.querySelectorAll('*')].filter(e => rec(e).right > innerWidth + 1)` 應為空（排除 `position:fixed` 外的溢出）。

#### T-11-02 ≥900px 兩欄、左欄約 380px、最大寬 1,100px
- 對應：F-11-1、§4 桌機
- 步驟：`viewport(899,900)`；`viewport(900,900)`；`viewport(1200,900)`（規格 T-1-3）；`viewport(1600,900)`。
- 預期：899 → 單欄；900／1200 → 兩欄：總覽卡、記一筆、設定區的 `left` 相同（左欄），圖表卡、篩選＋清單的 `left` 較大（右欄），左欄寬 380±40px；頁首與月份列寬 = 內容寬；1600 → 內容容器寬 ≤ 1100px 且水平置中。
- 自動化：`rec(spec('F-01')).left < rec(spec('F-06')).left`；`rec(container).width <= 1100`。

#### T-11-03 手機版區塊順序符合 §4
- 對應：§4 手機版面
- 前置：`viewport(375,667)`
- 預期：依 `rec().top` 排序為：頁首（含「關於」）→ 月份列 → 總覽卡（F-04/F-05）→ 記一筆（F-01）→ 圖表（F-06）→ 篩選＋清單（F-07/F-02）→ 設定與資料（F-09/F-10）→ 頁尾（F-12）。
- 自動化：各 `data-spec` 區塊 `top` 遞增序列比對。

#### T-11-04 可點元素至少 44×44px
- 對應：F-11-2
- 前置：`viewport(360,740)`；範例資料載入；開啟編輯視窗與關於視窗各掃一次。
- 預期：所有可見 `button, a[href], input, select, [role=button]` 的 `rec().height ≥ 44 && width ≥ 44`（清單列的「編輯」「刪除」、分類鈕、月份 ‹ ›、toast 的「復原」皆含）。
- 自動化：列出不合格者 `[tag, text, w, h]`；允許清單 0。

#### T-11-05 輸入欄字級 ≥ 16px、金額欄 inputmode=numeric
- 對應：F-11-2
- 預期：所有 `input, select, textarea`（含編輯視窗內）computed `font-size ≥ 16px`；金額欄（表單與編輯視窗）`getAttribute('inputmode') === 'numeric'`；預算欄亦建議 numeric。
- 自動化：`parseFloat(getComputedStyle(el).fontSize) >= 16`。

#### T-11-06 提示訊息在底部、aria-live=polite、2 秒／5 秒
- 對應：F-11-3
- 步驟：新增一筆（2 秒）；刪除一筆（5 秒）。
- 預期：toast 容器 `getAttribute('aria-live') === 'polite'`（建議 `role="status"`）；`rec(toast).bottom ≥ innerHeight − 120`（在畫面底部）；一般提示 1.5s 在、2.6s 消失；刪除提示 4.5s 在、5.6s 消失。
- 自動化：時序輪詢 `toast()`。

#### T-11-07 每個輸入欄都有 label
- 對應：F-11-4
- 預期：所有 `input:not([type=hidden]), select, textarea`（含兩個視窗內）`el.labels.length > 0`（`<label for>` 或包裹）；規格明寫 `<label>`，`aria-label` 只做備援記錄。
- 自動化：列出 `labels.length === 0` 的元素，應為空。

#### T-11-08 純鍵盤操作：Tab 可達全部、Enter 送出、Esc 關閉
- 對應：F-11-4
- 步驟：從 `body` 開始連續 Tab（CDP `Input.dispatchKeyEvent`）記錄 `document.activeElement` 序列直到循環；在金額欄 Enter；Tab 到「編輯」按 Enter 開視窗 → Esc；Tab 到「關於」Enter → Esc。
- 預期：序列涵蓋：關於、‹、›（停用時可跳過）、支出／收入切換、金額、8 顆分類、今天／昨天、日期、備註、記一筆、三個篩選控制、每列的編輯／刪除、預算欄、匯出 CSV、匯出 JSON、匯入、載入範例、清除全部；視窗開啟時焦點在視窗內，Esc 後關閉且焦點回到觸發按鈕（加分）；無 `tabindex > 0`。
- 自動化：`[...document.querySelectorAll('[tabindex]')].every(e => +e.tabIndex <= 0)`；序列集合比對。

#### T-11-09 visibilitychange 跨午夜更新今天
- 對應：F-11-5
- 前置：`clock('2026-10-09T23:59')` 載入 FX-T2。
- 步驟：以 `Runtime.evaluate` 把假時鐘推到 `2026-10-10T00:01`（harness 暴露 `window.__setNow`）→ `Object.defineProperty(document,'visibilityState',{value:'visible',configurable:true}); document.dispatchEvent(new Event('visibilitychange'))`。
- 預期：日期欄變 `2026-10-10`；清單「今天」標籤移到 10/10（10/09 不再標今天）；預算卡「還有 22 天」、今天以前已花變 2,485、今日額度 ⌊5515/22⌋=250、今天還能花 250；長條圖出現 10/10 空長條（含今天外框）。
- 自動化：文字比對；跨月版本：`2026-10-31T23:59 → 2026-11-01T00:01`，預期月份列是否自動跳到 11 月為模糊點 A20，但不得出現 11/01 紀錄被歸入 10 月。

#### T-11-10 配色與系統字型（manual-visual + computed）
- 對應：§4 配色
- 預期：`body` 背景 `rgb(245, 247, 251)`；卡片 `rgb(255, 255, 255)`、邊框 `rgb(229, 231, 235)`；主按鈕背景 `rgb(79, 70, 229)`；`font-family` 含 `PingFang TC` 與 `Microsoft JhengHei`；無 `@font-face`、無 `fonts.googleapis`。
- 自動化：computed style 比對；`!/fonts\.googleapis|@font-face/.test(document.documentElement.outerHTML)`；整體觀感 manual-visual（截圖存 `screenshots/`）。

#### T-11-11 焦點可見（manual-visual）
- 對應：F-11-4
- 預期：Tab 到任一按鈕／輸入欄有可見 focus ring（不得 `outline:none` 又無替代）。
- 自動化：`getComputedStyle(el, ':focus-visible').outlineStyle !== 'none'` 不可靠 → manual-visual 截圖 2–3 個元素。

---

## 12. F-12 關於／版本紀錄

#### T-12-01 「關於」視窗內容
- 對應：F-12-1
- 步驟：點頁首「關於」。
- 預期：視窗含一句話說明（「幾秒記好一筆…」）、「規格 v1.0」字樣、版本紀錄清單；每筆含版本號（`v1.x`）、日期（`YYYY-MM-DD`）、回饋／問題、修改內容、對應 `F-xx`；順序由新到舊（版本號遞減）。
- 自動化：`[...dialog.querySelectorAll('li, tr')].map(e=>e.textContent)` → 正則擷取版本號序列，檢查遞減；每筆含 `/F-\d\d/`。

#### T-12-02 頁尾「v1.x · 最後更新 YYYY-MM-DD」且與版本紀錄一致
- 對應：F-12-2
- 預期：頁尾符合 `/v1\.\d+ · 最後更新 \d{4}-\d{2}-\d{2}/`；版本與日期等於版本紀錄最新一筆；日期 ≤ 測試當天。
- 自動化：正則擷取後比對。

#### T-12-03 「顯示功能編號」開關
- 對應：F-12-3
- 步驟：開啟關於視窗 → 切換開關 → 關閉視窗 → 檢查 → 再切回。
- 預期：開啟後每個 `[data-spec]` 區塊左上角出現 `F-xx` 標籤（`::before` 或子元素，文字等於 `data-spec` 值）；關閉即消失；`raw()` 前後相同（不影響資料）。
- 自動化：`[...document.querySelectorAll('[data-spec]')].every(e => getComputedStyle(e,'::before').content.includes(e.dataset.spec) || e.querySelector('.spec-tag')?.textContent === e.dataset.spec)`；`raw()` 相等。

#### T-12-04 data-spec 覆蓋 F-01～F-12
- 對應：F-12-3、§6
- 預期：`new Set([...document.querySelectorAll('[data-spec]')].flatMap(e => e.dataset.spec.split(/[\s,]+/)))` 包含 `F-01`…`F-12` 全部。
- 自動化：集合差集為空。

#### T-12-05 關於視窗鍵盤關閉與 aria
- 對應：F-12-1、F-11-4
- 預期：`role="dialog"`（或 `<dialog>`）、`aria-modal="true"`、有 `aria-labelledby`；Esc 關閉。
- 自動化：屬性比對。

---

## 13. T-S 技術規格靜態檢查（§6）

#### T-S-01 單檔、無外部資源、無網路
- 預期：僅 `index.html`；`<script src>`、`<link href>`（除 `rel=icon` data URI）、`@import`、`url(http` 皆不存在；載入後 `Network.requestWillBeSent` 只有 1 筆。
- 自動化：靜態 grep + CDP 計數。

#### T-S-02 `<html lang="zh-Hant-TW">`、viewport meta、charset
- 自動化：`document.documentElement.lang === 'zh-Hant-TW'`；`meta[name=viewport]` 含 `width=device-width`；`meta[charset]` 為 utf-8。

#### T-S-03 行數約 900–1,600（軟性）
- 自動化：`wc -l index.html`；超出範圍僅記錄，不判失敗。

#### T-S-04 `// [F-xx]` 註解覆蓋 F-01～F-12
- 自動化：`grep -o '\[F-[0-9][0-9]\]' index.html | sort -u` 含 12 個。

#### T-S-05 禁用模式靜態掃描
- 預期：原始碼不含 `toISOString().slice(0, 10)`、`toISOString().substring`、`new Date('20`、`new Date("20`、`.setMonth(`；`innerHTML` 若出現，右側不得拼接使用者字串（人工檢視每一處；建議 0 處或只用於固定模板）；`localStorage.getItem/setItem` 皆位於 `try` 區塊內（人工檢視）。
- 自動化：grep；`innerHTML` 出現處列出供 code review。

#### T-S-06 金額格式 `'NT$' + n.toLocaleString('zh-TW')`
- 預期：畫面上金額符合 `/NT\$\d{1,3}(,\d{3})*/`；`NT$1,200`、`NT$999,999`。
- 自動化：正則掃描 `body()` 中所有 `NT$` 後的數字皆有千分位。

#### T-S-07 state 集中、先 save 再 render（code review）
- 自動化：manual code review：找 `state` 物件、`save()`、`render()` 的呼叫順序。

---

## 14. T-D 日期與時區專項（彙整）

| ID | 固定時間（Asia/Taipei） | 檢查 | 對應 |
|---|---|---|---|
| T-D-01 | 2026-10-09 00:30（= 10-08 16:30Z） | 日期欄、今天標籤、剩餘天數 23、今天長條皆為 10/09 | F-01-1、§6 |
| T-D-02 | 2026-10-09 07:59 ／ 08:00 ／ 23:59 | 三者都是 10/09 | F-01-1 |
| T-D-03 | 2026-10-31 23:59 → 2026-11-01 00:01（visibilitychange） | 日期欄變 11/01；剩餘天數由 1 變 30 | F-11-5 |
| T-D-04 | 2026-01-01 09:00 | 「昨天」= 2025-12-31；「‹」= 2025 年 12 月 | F-01、F-04-2 |
| T-D-05 | 2028-02-10 | 2 月 29 天；長條 10 根；還有 20 天 | §6 |
| T-D-06 | 2026-03-31 | 連續 ‹ 14 次 › 14 次無漂移 | §6 setMonth |
| T-D-07 | 任一 | seed `date:'2026-10-09'` 的紀錄在 00:30 仍歸 10/09 組（不得用 `new Date('2026-10-09')` 解析成 10/08 16:00Z 再轉本地） | §6 |
| T-D-08 | 2026-10-09 | CSV 與 JSON 檔名用本地日期（00:30 仍為 10-09） | F-09-1、F-09-3 |

---

## 15. 規格第 8 節情境 T-1～T-6 對照

| 規格情境 | 本計畫案例 |
|---|---|
| T-1 第一次使用 | T-02-06、T-10-01、T-10-02、T-11-01、T-11-02、T-08-07 |
| T-2 驗算今天還能花 | T-10-06（清除）、T-01-14、T-04-05、T-05-02 |
| T-3 預算變色 | T-05-06、T-06-06 |
| T-4 輸入檢查、編輯與復原 | T-01-05、T-01-07、T-02-05、T-03-03、T-03-05、T-03-07 |
| T-5 篩選與圖表連動 | T-06-04、T-07-02、T-07-05、T-07-06、T-07-07 |
| T-6 儲存、匯出與匯入 | T-08-02、T-09-02、T-09-04、T-09-09、T-09-07、T-08-03 |

---

## 16. 評分對照表

| 評分項目（9–10 分） | 證明方式 | 對應案例／產出 |
|---|---|---|
| 規格書清楚具體 | 每條驗收標準都有具體數字與可機器判定的預期值；每條至少一個測試案例可追溯 | `spec.md` F-01～F-12 ↔ 本文件第 1～12 節全部案例；`process/01_規格討論紀錄.md` 說明為何改成可驗收的寫法；T-05-02／T-05-06／T-04-05 直接驗算規格給的數字 |
| 網頁有實際互動功能 | 新增／編輯／刪除／復原、月份切換、篩選搜尋、圖表連動、匯出匯入、範例資料、關於開關全部可操作，且資料持久化 | T-01-10、T-01-11、T-03-03、T-03-07、T-04-02、T-06-04、T-06-08、T-07-02、T-08-02、T-09-06、T-09-09、T-10-01、T-12-03；`screenshots/` 中 375px 與 1200px 截圖 |
| 實作與規格書對應 | 功能編號貫穿規格、程式註解、`data-spec` 標籤與測試 ID；關鍵數字逐一驗算；禁用模式靜態掃描 | T-S-04（`// [F-xx]`）、T-12-04（`data-spec` 覆蓋）、T-12-03（功能編號開關截圖）、T-S-05（禁用模式）、T-05-02／T-05-04／T-05-05／T-05-06（數字驗算）、T-09-03／T-09-04（CSV 規則）、T-08-03（壞資料處理） |
| 有透過 AI 反覆調整 | 版本紀錄只寫真實修改，每筆附 AI 測試回饋與對應 F 編號；測試計畫執行結果 → 修正 → 再測的紀錄 | T-12-01／T-12-02（版本紀錄與頁尾一致）、`process/01_規格討論紀錄.md`、本測試計畫 + 執行報告（建議存 `process/03_測試結果_v1.0.md`，記錄失敗案例 → v1.1 修正 → 重跑通過） |

---

## 17. 規格模糊點（建置者可能有不同解讀，測試前請確認）

| 編號 | 模糊點 | 建議解讀（測試預設） |
|---|---|---|
| A1 | 金額欄若用 `<input type="number">`，全形「１２０」與「1,200」無法輸入也讀不到值；`12.0`、`1e3`、` 90 `、`0090` 是否接受 | 用 `type="text" inputmode="numeric"`；正規化：全形→半形、去逗號、trim、去前導 0；`12.0`／`1e3` 拒絕 |
| A2 | 備註 31 字：`maxlength` 靜默截斷 vs. 顯示錯誤 | 兩者擇一但表單與編輯視窗一致；匯入時超過 30 字要判「格式不正確」或截斷，需一致 |
| A3 | 支出金額「−NT$90」的減號：U+2212 還是半形 `-` | 測試接受兩者；CSV 不含符號 |
| A4 | 組標題：只有收入的日子顯示什麼；「昨天」是否也有標籤；「今天」與「支出」間的空白字元 | 允許「支出 NT$0」或改顯示「收入 NT$X」；測試正規化空白 |
| A5 | 預算輸入何時生效（input／change／按鈕）；無效值的錯誤訊息文字 | 測試對 change 與按鈕都嘗試；訊息只要求存在 |
| A6 | 匯入錯誤「第 N 筆」：JSON 無法解析或 `records` 非陣列時 N 怎麼寫；N 是否 1-based；未來日期、31 字備註、缺 `settings`、重複 `id` 是否判錯 | 無法解析時可省略「（第 N 筆）」；N 1-based；未來日期建議拒絕（與 F-01-3 一致） |
| A7 | 主資料是合法 JSON 但結構錯（`[]`、`null`）是否視為「讀取失敗」；壞資料另存後主 key 何時被覆寫 | 一律視為讀取失敗並另存；主 key 在下一次 `save()` 才覆寫 |
| A8 | 類型「全部」時分類下拉顯示 12 個還是只有「全部分類」 | 顯示 12 個（分組亦可） |
| A9 | 篩選是否在切月時保留；「清除篩選」與統計列在無篩選時是否顯示 | 保留篩選；無篩選時隱藏統計列與清除鈕 |
| A10 | 匯出 CSV 的列順序（日期升序或清單順序） | 日期升序、同日 createdAt 升序（Excel 加總不受影響） |
| A11 | CSV「分類」欄含 emoji（🍱 餐飲）或只名稱 | 只名稱；測試接受兩者 |
| A12 | 「移除範例資料」按鈕在沒有 demo 紀錄時是否顯示／停用 | 無 demo 時隱藏或停用 |
| A13 | 切到「收入」時預設分類（第一個？上次用的收入分類？）；`lastCategory` 是否分支出／收入各記一個 | 第一次為 💰 生活費，之後沿用上次收入分類；`lastCategory` 只記一個時切型別取該型別第一個 |
| A14 | 編輯儲存後是否顯示提示、文字為何；編輯把日期改到其他月份時的提示 | 建議「已更新 🍱 餐飲 NT$120」／「已記到 2026 年 9 月」 |
| A15 | 清除全部後 `lastCategory` 是否重設；復原暫存是否清空 | 重設為 `food`；暫存清空 |
| A16 | 月底預估 ≤ 預算時的附註文字（規格只給「超出 NT$559 橘字」）；已花 0 時是否顯示預估 | 顯示「低於預算 NT$X」（綠或次要色）；已花 0 顯示 NT$0 |
| A17 | 「›」停用是 `disabled` 屬性還是只在視覺上停用 | `disabled` 屬性（鍵盤也不可觸發） |
| A18 | 日期欄若設 `max=今天`，UI 無法選未來日期，「不能記錄未來的日期」只在程式層出現 | 可同時設 `max` 與程式檢查；測試以 JS 設值 |
| A19 | 同時多個 toast（快速連續新增）是取代還是堆疊；新刪除是否重設 5 秒 | 取代；新刪除重設計時並覆蓋可復原目標 |
| A20 | 跨午夜進入新月份時（visibilitychange），檢視月份是否跟著跳到新月 | 若原本在「本月」則跟著跳；否則維持 |
| A21 | 排行點擊後的捲動是 smooth 還是 instant；捲到清單頂或篩選列 | 任一，測試等 800ms |
| A22 | 預算 0（未設定）時，長條圖是否仍畫（無虛線）；過去月份是否顯示「結算」或「尚未設定預算」 | 長條圖照畫、無虛線；過去月份顯示「尚未設定預算」 |
