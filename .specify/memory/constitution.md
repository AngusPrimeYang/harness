<!--
Sync Impact Report
Version change: 1.1.0 → 1.2.0
Modified principles:
  - none renamed
Added sections:
  - VI. API 開發規範 (API Development Standards)
Removed sections: none
Templates requiring updates:
  - .specify/templates/plan-template.md ✅ updated
  - .specify/templates/tasks-template.md ✅ updated
  - .specify/templates/spec-template.md ✅ updated (Assumptions note)
  - .specify/templates/constitution-template.md — generic placeholder retained
  - .cursor/rules/api-development.mdc ✅ added (agent long-term guidance)
Follow-up TODOs: none
-->

# System Constitution

## Core Principles

### I. 技術棧 (Technical Stack)

- **Framework**: MUST 使用 .NET 10 (LTS)，Target Framework `net10.0`
- **SDK**: MUST 使用 .NET 10 SDK（repo 根目錄 `global.json` 鎖定版本；目前 10.0.301）
- **套件版本**: Microsoft.* 與主要第三方套件版本 SHOULD 集中於 `Directory.Build.props`
- **ORM**: MUST 使用 Entity Framework Core 10.x（Code First 為目標規範；現況為 SSDT + EF Power Tools database-first）
- **Background Jobs**: CronJob MUST 使用 Quartz、`BackgroundService` 或 Hangfire（現況：Quartz + `System_Job`）
- **Testing**: MUST 使用 Reqnroll (BDD) + xUnit + FluentAssertions
- **部署目標**: Windows Server（`System.Drawing`、`DirectoryServices`、`Office.Interop` 等 Windows 專用 API）

**Rationale**: 統一技術選型以確保 LTS 支援、可維護性與團隊知識共享；.NET 10 為目前專案實際執行環境。

### II. 代碼架構規範 (Architecture)

- **Pattern**: MUST 遵循 Repository + Service Pattern
- **DI**: 依賴注入 MUST 一律使用介面 (Interface)；禁止直接實例化 concrete 類別
- **EF Core**:
  - 所有資料庫變更 MUST 透過 Migrations
  - 唯讀查詢 MUST 使用 `AsNoTracking()`
- **DTOs**: Controller 與 Service 之間 MUST 使用 DTO 傳遞資料；禁止直接回傳 EF Entities

**Rationale**: 分層架構與 DTO 隔離確保關注點分離、可測試性與 API 契約穩定。

### III. BDD 與測試規範

- **Feature Language**: MUST 使用 zh-TW (繁體中文)
- **Step Definitions**: MUST 放置於 `Tests/StepDefinitions` 資料夾
- **CronJob**: MUST 有對應的單元測試，模擬 `CancellationToken` 觸發後的停止邏輯

**Rationale**: BDD 以繁體中文描述業務行為，確保利害關係人可讀；背景工作需驗證優雅關閉。

### IV. 命名慣例 (Naming Conventions)

- **Interfaces**: MUST 以 `I` 開頭 (e.g., `IUserService`)
- **Async**: 所有異步方法 MUST 以 `Async` 結尾
- **Database Tables**: MUST 使用 PascalCase 複數形式

**Rationale**: 一致命名降低認知負擔，符合 .NET 社群慣例。

### V. 錯誤處理

- 全域例外處理 MUST 透過 Middleware 攔截
- 禁止空的 `try-catch` 區塊

**Rationale**: 集中式錯誤處理確保一致回應格式；空 catch 會吞沒錯誤、妨礙除錯。

### VI. API 開發規範 (API Development Standards)

- **新檔隔離**: 所有新建 API 相關程式碼 MUST 獨立存放於新檔案；禁止修改／汙染既有檔案內容以塞入新邏輯
- **既有邏輯複用**: 使用者驗證、紀錄查詢等行為 MUST 呼叫既有函式／方法；禁止重寫平行實作
- **資料庫不變**: API 開發 MUST NOT 變更資料庫規格（含 SSDT table、EF Entity、Migration、種子資料）；僅允許新建 API 接口，並以既有業務邏輯組建
- **ViewModel / DTO 沿用**: 若專案中已有可用的 ViewModel 或 DTO，MUST 優先沿用；僅在既有型別不足時才新增專用型別（仍須放在新檔案）
- **共通架構保護**: 若發現 `System_Common`、PortalApi、共享 Middleware／基礎建設等共通架構有誤或效率不佳，MUST NOT 直接修改；MUST 先詢問使用者是否同意調整後再動手

**Rationale**: API 為既有系統的對外薄封裝，隔離變更面、避免污染核心與 DB，並強制複用已驗證邏輯以降低回歸風險。

## 開發流程與品質閘

- 所有 PR MUST 通過 Constitution Check（見 `.specify/templates/plan-template.md`）
- 非 API 功能若涉及資料模型變更 MUST 包含 EF Migration；API 開發 MUST NOT 變更資料庫規格
- 涉及 API 端點 MUST 使用 DTO／ViewModel（優先沿用既有）、獨立新檔，並規劃對應測試覆蓋
- API 相關 PR MUST 驗證：未改既有檔業務內容、驗證／查詢走既有函式、未改 schema
- BackgroundService / CronJob 功能 MUST 包含 `CancellationToken` 停止邏輯單元測試
- Code review MUST 驗證 Repository + Service 分層與 DI 介面使用
- 共通架構疑慮 MUST 經使用者確認後方可修改

## 專案結構

Solution 預期結構：

```text
System/              # Web MVC 主專案 (net10.0)
System_Common/       # 共用程式庫 (net10.0)
System_Job/          # 背景工作專案 (net10.0)
System_DB/           # SSDT 資料庫專案（Visual Studio 建置）
global.json                        # .NET 10 SDK 版本鎖定
Directory.Build.props              # 集中管理 NuGet 套件版本
Tests/
├── Features/                      # Reqnroll .feature 檔案 (zh-TW)
├── StepDefinitions/               # BDD Step Definitions
├── Unit/                          # xUnit 單元測試
└── Integration/                   # 整合測試
```

## Governance

- 本憲法優先於所有其他開發慣例與個人偏好
- 修訂程序：提出修訂 PR → 團隊審核 → 更新版本號與 `LAST_AMENDED_DATE` → 同步相關模板
- 版本策略：MAJOR（移除或重新定義原則）、MINOR（新增原則或實質擴充）、PATCH（措辭澄清、錯字修正）
- 合規審查：每次 `/speckit-plan` 與 PR review MUST 對照本憲法驗證
- 執行指引：`.specify/memory/constitution.md` 為權威來源；實作細節見各 feature 的 `plan.md`

**Version**: 1.2.0 | **Ratified**: 2026-07-08 | **Last Amended**: 2026-08-07
