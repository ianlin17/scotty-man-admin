# scotty-man 後台管理系統規格（Admin Console）

> 專案代號：`scotty-man-admin`
> 目錄：`admin/`
> 角色：給營運、客服、行銷人員使用的後台管理系統。透過 Medusa Admin API 操作商品、訂單、客戶，並提供 Medusa 內建 Admin 沒有的營運功能（儀表板、內容管理、評論審核、客服工具）。

---

## 1. 目標與範圍

### 1.1 目標
- 提供一個符合 scotty-man 營運流程、繁體中文介面的後台，取代直接使用 Medusa 內建 Admin 的日常操作。
- 所有商務資料一律經由後端 Medusa Admin API（見 `../backend/spec.md`），本專案不直接連資料庫。
- 依角色控制功能可見性（管理者 / 營運 / 客服 / 行銷）。

### 1.2 定位（與 Medusa 內建 Admin 的分工）

| 項目 | 本系統 | Medusa 內建 Admin（`/app`） |
| --- | --- | --- |
| 日常訂單處理、出貨、退款 | ✔ 主要入口 | 備援 |
| 商品 / 庫存 / 價格維護 | ✔ | 備援 |
| 客戶查詢與客服備註 | ✔ | |
| 促銷活動與折扣碼 | ✔ | 備援 |
| 營運儀表板與報表 | ✔ | ✘ |
| 首頁 Banner / 內容管理 | ✔ | ✘ |
| 商品評論審核 | ✔ | ✘ |
| 系統設定（Region、稅、Provider、API Key） | ✘ | ✔ 只在這裡做 |
| 自訂模組 / 開發者設定 | ✘ | ✔ |

### 1.3 不在範圍內
- 消費者端頁面（見 `../frontend/spec.md`）。
- 後端商業邏輯與資料庫 schema。

---

## 2. 技術棧

| 項目 | 選擇 | 備註 |
| --- | --- | --- |
| Framework | Next.js 15+（App Router） | 與前端一致，降低維護成本 |
| 語言 | TypeScript（strict） | |
| UI | Tailwind CSS + shadcn/ui | 表格用 TanStack Table，圖表用 Recharts |
| API Client | `@medusajs/js-sdk`（admin 端） | Bearer JWT |
| 資料抓取 | TanStack Query | 後台以 Client Components 為主，互動多 |
| 表單 | react-hook-form + zod | |
| 測試 | Vitest + React Testing Library；Playwright（E2E） | |
| Node | 22 LTS | |
| 套件管理 | pnpm | |

---

## 3. 環境變數

檔案：`.env.local`（不進版控），範本：`.env.template`

| 變數 | 說明 | 範例 |
| --- | --- | --- |
| `NEXT_PUBLIC_MEDUSA_BACKEND_URL` | Medusa 後端 URL | `http://localhost:9000` |
| `NEXT_PUBLIC_STOREFRONT_URL` | 前台網址（預覽商品用） | `http://localhost:8000` |
| `SESSION_SECRET` | 加密 session cookie 用 | 隨機 32+ 字元 |

後端 `ADMIN_CORS` 與 `AUTH_CORS` 必須加入本系統的網址（開發：`http://localhost:3001`）。

---

## 4. 認證與權限

### 4.1 登入
- 使用 Medusa Auth 的 `/auth/user/emailpass` 取得 JWT。
- JWT 存於 httpOnly cookie，由 Next.js Route Handler 代理附加到 Admin API 請求，瀏覽器不直接持有 token。
- 閒置 8 小時自動登出；支援「忘記密碼」（呼叫 Medusa reset-password 流程）。

### 4.2 角色（RBAC）
Medusa 使用者本身沒有細分角色，本系統以使用者 `metadata.role` 儲存角色，由前端控制功能可見性，並在 Route Handler 層再次檢查。

| 角色 | 權限 |
| --- | --- |
| `admin` | 全部功能，含使用者管理 |
| `ops` | 商品、庫存、訂單、出貨、退款 |
| `cs` | 訂單查詢、客戶查詢、客服備註、評論審核；不可退款 |
| `marketing` | 促銷、Banner / 內容、報表 |

---

## 5. 功能模組

### 5.1 儀表板（`/`）
- 今日 / 本週 / 本月：訂單數、營業額、平均客單價、新客數。
- 待處理項目：待出貨訂單、待審核評論、低庫存商品。
- 近 30 天營業額與訂單數趨勢圖。
- 資料來源：Admin API 的 orders / customers / inventory，統計於本系統 Route Handler 彙整並快取 5 分鐘。

### 5.2 訂單管理（`/orders`）
- 列表：依狀態 / 付款狀態 / 出貨狀態 / 日期 / 關鍵字篩選；可匯出 CSV。
- 詳情：商品明細、金額、客戶與地址、付款紀錄、出貨紀錄、時間軸。
- 操作：建立出貨（填物流單號）、標記送達、取消訂單、退款、部分退貨、加註內部備註。
- 批次操作：批次列印揀貨單、批次標記出貨。

### 5.3 商品管理（`/products`）
- 列表、搜尋、依分類 / 系列 / 狀態篩選。
- 新增 / 編輯：基本資料、圖片上傳、選項與變體、價格（依 Region）、庫存、分類、系列、品牌、SEO 欄位。
- 推桿規格區塊（對應後端 `product.metadata` 固定 key，見 `../backend/spec.md` §6.1）：Loft、Lie、頸部設計（下拉）、桿頭形狀（下拉）、材質、趾部下垂、桿頭重量；表單以與後端共用的 zod schema 驗證。
- 變體產生器：勾選長度（33" / 34" / 35"）與左右手（RH / LH）後自動產生 Length × Dexterity 的 variants 與 SKU。
- 前台篩選預覽：儲存後顯示此商品會出現在哪些 facets（呼叫 `/store/catalog/facets`），確認規格填寫完整。
- 上下架、複製商品、批次調整價格。
- 分類（`/categories`）、系列（`/collections`）、品牌（`/brands`）維護。

### 5.4 庫存管理（`/inventory`）
- 依倉庫查看庫存與保留量；手動調整並記錄原因。
- 低庫存門檻設定與提醒。

### 5.5 客戶管理（`/customers`）
- 列表與搜尋；詳情含訂單歷史、地址、客群、累計消費。
- 客服備註（存於 customer `metadata.notes`，含操作者與時間）。
- 客群（`/customer-groups`）維護。

### 5.6 促銷管理（`/promotions`）
- 折扣碼與自動促銷的建立 / 編輯 / 停用。
- 條件：金額門檻、指定商品 / 分類、客群、使用次數、有效期間。
- 使用成效統計。

### 5.7 內容管理（`/content`）
- 首頁 Banner（圖片、連結、排序、排程上下架）。
- 公告列、頁尾連結、靜態頁（關於我們、退換貨政策）。
- 資料存放：後端自訂 `cms` 模組（需在 `backend/` 新增，列入該專案 P1）。

### 5.8 評論審核（`/reviews`）
- 待審核 / 已通過 / 已拒絕列表，可回覆評論。
- 對應後端 `review` 模組的 Admin 端點。

### 5.9 報表（`/reports`）
- 銷售報表（依日 / 週 / 月、依分類、依商品）。
- 客戶報表（新客 / 回購率）。
- 皆可匯出 CSV。

### 5.10 系統（`/settings`）
- 使用者與角色管理（僅 `admin`）。
- 個人設定：改密碼。
- 其他系統設定連結至 Medusa 內建 Admin。

---

## 6. 路由結構

```
app/
├─ (auth)/
│  ├─ login/
│  └─ reset-password/
├─ (dashboard)/               # 需登入，含側邊欄 layout
│  ├─ page.tsx                # 儀表板
│  ├─ orders/[id]/
│  ├─ products/[id]/ 、 categories/ 、 collections/ 、 brands/
│  ├─ inventory/
│  ├─ customers/[id]/ 、 customer-groups/
│  ├─ promotions/[id]/
│  ├─ content/banners/ 、 content/pages/
│  ├─ reviews/
│  ├─ reports/
│  └─ settings/users/ 、 settings/profile/
├─ api/
│  ├─ auth/                   # login / logout，處理 cookie
│  ├─ medusa/[...path]/       # 代理 Admin API，附加 JWT 與角色檢查
│  └─ stats/                  # 儀表板統計彙整
└─ middleware.ts              # 未登入導向 /login；角色不足回 403
```

---

## 7. 專案結構

```
src/
├─ app/
├─ components/
│  ├─ ui/                     # shadcn/ui
│  ├─ data-table/             # 通用表格（分頁、排序、篩選、匯出）
│  ├─ forms/
│  └─ layout/                 # Sidebar、Topbar、Breadcrumb
├─ lib/
│  ├─ medusa.ts               # js-sdk admin client
│  ├─ auth.ts                 # session、角色檢查
│  ├─ permissions.ts          # 角色 → 功能對照表
│  └─ queries/                # TanStack Query hooks（orders、products …）
├─ hooks/
└─ i18n/                      # zh-TW 文字
```

---

## 8. UI / UX 規範
- 桌機優先（最小寬度 1280px），平板可用；不支援手機。
- 側邊欄導覽 + 頂部麵包屑；所有列表頁支援 URL query 保存篩選狀態。
- 表格：預設每頁 25 筆，可切換 50 / 100；欄位可排序；支援多選批次操作。
- 破壞性操作（取消訂單、退款、刪除）一律二次確認。
- 所有操作結果以 toast 回饋；錯誤訊息顯示後端回傳的 message。
- 深色模式支援。

---

## 9. 本機開發

```bash
pnpm install
cp .env.template .env.local
pnpm dev                     # http://localhost:3001
```

需先啟動 `database/` 與 `backend/`，並用後端 seed 建立的 Admin 帳號登入。

| 指令 | 說明 |
| --- | --- |
| `pnpm dev` | 開發伺服器（port 3001） |
| `pnpm build` / `pnpm start` | 正式建置與啟動 |
| `pnpm lint` / `pnpm typecheck` | 靜態檢查 |
| `pnpm test` / `pnpm test:e2e` | 測試 |

---

## 10. 安全性
- 只允許 HTTPS；正式環境建議加 IP 白名單或 VPN。
- JWT 只存 httpOnly cookie，所有 Admin API 呼叫經由本系統伺服器端代理。
- 角色檢查在前端與 Route Handler 各做一次；敏感操作（退款、刪除使用者）記錄操作者與時間。
- 上傳圖片限制格式（jpg / png / webp）與大小（5MB）。

---

## 11. 部署
- Vercel 或容器（提供 `Dockerfile`，Node 22 alpine）。
- CI：GitHub Actions，PR 觸發 lint → typecheck → test → build。
- 正式環境網址需加入後端 `ADMIN_CORS` / `AUTH_CORS`。

---

## 12. 驗收標準
- [ ] Admin 帳號可登入，`cs` 角色看不到退款按鈕且 API 代理層會拒絕退款請求。
- [ ] 可完成「建立商品 → 前台可見 → 下單 → 後台出貨 → 訂單狀態更新」流程。
- [ ] 儀表板數字與 Medusa 內建 Admin 的訂單資料一致。
- [ ] Banner 排程上下架能正確反映在前台首頁。
- [ ] Lighthouse Accessibility ≥ 90。
