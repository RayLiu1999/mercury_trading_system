# copilot-instructions.md（擴充建議）

1. 註解都要用中文表示（程式碼內的註解與本專案內部文件皆以中文為主，若為公開 API 或開源包可加英文備註）。
2. 建立檔案都要確認格式正確（例如：Go 用 gofmt, Python 用 black, ts/js 用 prettier）。
3. 目錄與 Package 結構：
  - 遵循 Monorepo 結構：Go 服務置於 `services/`，Python 工具置於 `python/`，前端置於 `frontend/`。
  - Go 檔案 package 命名應與目錄一致，`cmd/` 目錄下為 `package main`。
4. 代碼格式化與靜態檢查：請在本地執行 `gofmt -s -w`、`go vet`、`golangci-lint run`（或等價工具）並將 CI 設為必過。
5. 測試規範：
  - 所有新增功能需包含單元測試（coverage 目標為 80% 以上，核心邏輯如匹配引擎覆蓋率需更高）。
  - 長期維護核心（matching）必須有模型測試、綜合測試，CI 執行 `go test -race ./...`。
  - 若有重要性能要求，請提供 benchmark 測試（`go test -bench`）。
6. commit/PR 與分支規範：
  - **分支策略**：嚴格遵循 Git Flow。新功能 (feat) 與修復 (fix) 請從 `develop` 分支切出，並發起 PR 合併回 `develop`。僅 Hotfix 可針對 `main`。
  - **Commit 訊息**：必須遵守 Conventional Commits (例：feat, fix, docs, chore)。
  - **PR 內容**：需包含改動說明、測試驗證截圖，並確認是否需更新 `docs/API_SPEC.md`，並指定審查者。
7. API / Schema / 事件契約：
  - `order.events`, `trade.events` 等事件有明確 schema（JSON Schema 或 Protobuf）並置於 `services/shared/events`。
  - 所有事件或 public API 更動需同步更新 `docs/API_SPEC.md`。
  - REST API 需支援 `X-API-Key` header 進行認證。
8. 安全：
  - 不得在 repo 中提交任何 secret 檔（如 `.pem`, `.key`）。請使用 `.env.example` 且 secrets 統一管理於 CI/CD secrets 或 Vault。
  - 使用 `git-secrets` 或相同工具在 CI 檢測是否有 secret commit。
9. 日誌與監控：
  - 日誌使用結構化 JSON 且包含 `timestamp`, `level`, `service`, `request_id`, `trace_id`。
  - 每個 Service 都需紀錄主要的 Prometheus 指標（例如：orders_created_total, matches_per_second）。
10. 本地開發 / Docker：

- 每個 service 需提供 `Dockerfile` 與 `docker-compose` 示範（local dev），以及簡易說明在 `README.md` 中。

11. 貢獻流程與文件：

- 建立 `CONTRIBUTING.md`（包含 PR 模板、Issue 範本、review 流程）。
- 公開文件（例如 README、API Spec）建議雙語（中文為主，英文摘要），方便對外展示。

12. 其他注意：

- 在 Go 中請使用 `context.Context` 作為第一個參數於處理函式，並在需要時採用 timeout/cancel。
- 盡量避免在工具/測試中硬編資料庫連線資訊；使用 DI 或 mock。
- code review 重點：有無單元測試、是否更新文件、是否包含安全影響、是否破壞相容性（breaking changes）。

13. 事件驅動與非同步處理：

- Redis Stream 事件消費需實現幂等性（idempotent），避免重複處理。
- 事件生產與消費應有錯誤處理與重試機制（exponential backoff）。
- Matching Engine 必須保持單執行緒確定性（deterministic），避免 race condition。

14. 資料庫操作規範：

- 使用 migration 工具管理 schema 變更（ex: golang-migrate for PostgreSQL）。
- 查詢需考慮索引優化，避免 N+1 問題；使用 prepared statements。
- MongoDB 集合設計應考慮分片與聚合管道效能。

15. 前端與 WebSocket 整合：

- React/TypeScript 組件應使用 hooks 管理狀態，WebSocket 連線需實現自動重連與心跳。
- 前端事件處理應防範 XSS/CSRF，WebSocket 訊息驗證 schema。

16. 性能與可擴展性：

- 關鍵路徑（如撮合）需定期 benchmark，並設定性能門檻（thresholds）。
- 考慮水平擴展：服務應 stateless，狀態存於 Redis/PostgreSQL。
- 資源限制：Docker 容器設定 CPU/memory limits，避免資源耗盡。

17. 錯誤處理與恢復：

- 使用 recover() 處理 panic，避免服務崩潰；實現 circuit breaker 模式。
- 網路呼叫（ex: Redis/PostgreSQL）應有 timeout 與 retry 邏輯。
- 錯誤分類：業務邏輯錯誤 vs 系統錯誤，分別處理。

18. 版本控制與發佈：

- 採用 Semantic Versioning (MAJOR.MINOR.PATCH)，tag 發佈。
- 每次發佈生成 CHANGELOG.md（使用 conventional-changelog）。
- API 版本化：v1, v2 等，確保向後相容。

19. 本地開發環境設置：

- 提供 Makefile 或 scripts 自動安裝工具（Go, Docker, pre-commit）。
- 環境變數統一管理於 .env 文件，開發/生產區分。
- 依賴安裝：go mod tidy, pip install -r requirements.txt。

20. 測試策略擴展：

- 除了單元測試，加入整合測試（integration tests）驗證服務間互動。
- 端到端測試（e2e）模擬完整用戶流程（下單 → 撮合 → 推播）。
- 測試資料使用 fixtures，避免硬編碼。

21. 監控與告警：

- Grafana dashboards 涵蓋關鍵指標（latency, throughput, error rate）。
- 設定告警規則：ex. 撮合延遲 > 100ms 或錯誤率 > 5%。
- 健康檢查端點（/health）供 K8s 或 load balancer 使用。

22. 專案特定規則：

- Orderbook 操作必須 thread-safe，使用 mutex 或 channel。
- 撮合邏輯遵循價格優先/時間優先，測試需涵蓋 edge cases（ex: 部分成交、取消訂單）。
- 交易數據敏感，確保加密傳輸（TLS）與審計日誌。
