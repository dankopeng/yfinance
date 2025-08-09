## yfinance 系統架構說明（中文）

### 目的與範圍
- 本文件說明 `yfinance` 專案的核心架構、主要元件與模組職責、資料流、快取策略、即時資料（WebSocket）、篩選器（Screener）與搜尋，以及可擴充點。
- 對應原始碼請參考 `yfinance/` 目錄與測試於 `tests/`。

### 整體架構概覽
- **公共 API 層（Public API）**：集中於 `yfinance/__init__.py`，對外暴露：
  - **資料物件與操作**：`Ticker`、`Tickers`、`download`
  - **市場與領域模型**：`Market`、`Sector`、`Industry`
  - **即時資料**：`WebSocket`、`AsyncWebSocket`
  - **資訊探索**：`Search`、`Lookup`
  - **篩選器**：`EquityQuery`、`FundQuery`、`screen`、`PREDEFINED_SCREENER_QUERIES`
  - **設定與工具**：`set_config`（設定 proxy）、`enable_debug_mode`、`set_tz_cache_location`
- **資料取用層（Data Access & Transport）**：
  - `yfinance/data.py` 定義 `YfData`（Singleton）：集中處理 HTTP session、Proxy、Cookie/Crumb 取得與切換策略、請求重試與節流例外、LRU 快取封裝。
  - `yfinance/const.py`：端點與鍵值常數（例如 `_BASE_URL_`、`_QUERY1_URL_`、`_ROOT_URL_`、`fundamentals_keys`）。
  - `yfinance/cache.py`：持久化快取（SQLite via `peewee`）與檔案路徑管理（via `platformdirs`）。
- **資料擷取與彙整（Scrapers & Aggregation）**：
  - `yfinance/scrapers/`：功能模組化的擷取器：`quote.py`、`history.py`、`analysis.py`、`fundamentals.py`、`funds.py`、`holders.py` 等，封裝 Yahoo Finance 的 REST/HTML 端點解析與資料整理。
  - `yfinance/base.py` 與 `yfinance/ticker.py`：`TickerBase` 與 `Ticker` 封裝單一代碼（symbol）的多種資料讀取操作，內部以 lazy-load 方式組合各 `scrapers`。
  - `yfinance/multi.py`：`download` 實作，支援多代碼並行下載、錯誤彙整、時區處理與結果整併。
  - `yfinance/tickers.py`：多代碼介面（`Tickers`）。
- **領域模型（Domain）**：
  - `yfinance/domain/`：`Market`、`Sector`、`Industry` 與相關型別封裝。
- **即時資料層（Realtime Streaming）**：
  - `yfinance/live.py`：`WebSocket`、`AsyncWebSocket` 封裝 Yahoo Finance streamer（`wss://streamer.finance.yahoo.com`）。
  - `yfinance/pricing.proto`／`pricing_pb2.py`：即時報價 Protobuf schema 與對應生成之 Python 類別。
- **工具與共用**：
  - `yfinance/utils.py`：日志、時間與時區處理、資料整形、進度條、各種工具方法。
  - `yfinance/shared.py`：批次下載共享狀態（例如 `_DFS`、錯誤與 traceback 收集、ISIN 映射）。
  - `yfinance/exceptions.py`：自訂例外（如 `YFRateLimitError`、`YFTzMissingError` 等）。

### 目錄結構（關鍵節選）
- `yfinance/`
  - `__init__.py`：公共 API 匯出
  - `data.py`：HTTP/快取核心（Singleton `YfData`）
  - `base.py`、`ticker.py`、`tickers.py`：單/多代碼操作
  - `multi.py`：批次 `download`
  - `scrapers/`：各資料面向的擷取器（`quote`, `history`, `analysis`, `fundamentals`, `funds`, `holders`）
  - `domain/`：`Market`、`Sector`、`Industry`
  - `live.py`、`pricing.proto`、`pricing_pb2.py`：即時資料
  - `cache.py`、`utils.py`、`const.py`、`exceptions.py`、`shared.py`
- `tests/`：對應功能測試（快取、即時、歷史、搜尋、篩選、Ticker 等）

### 核心元件與責任
- **YfData（單例）**：
  - 管理 `curl_cffi.requests.Session(impersonate="chrome")`，集中處理 Cookie、Crumb。
  - 兩段 Cookie 策略：`basic` 與 `csrf`，遇到 4xx/429 會切換策略並重試。
  - 對外提供 `get/post/cache_get/get_raw_json`，並以 `lru_cache` 對 `cache_get` 進行記憶化。
  - 支援設定 Proxy，並與全域設定 `set_config(proxy=...)` 對應。
- **Ticker/TickerBase**：
  - 封裝對單一代碼的操作：歷史價（`history`）、快資訊、基本面、持股、分析、建議、行事曆等。
  - 以 lazy-load 建立對應 scraper（例如首次呼叫 `history()` 才初始化 `PriceHistory`）。
  - 自動 ISIN→Ticker 對照、時區快取、例外與回退處理。
- **download（multi.py）**：
  - 支援多代碼、平行（`multitasking`）下載，提供進度列與錯誤彙整。
  - 依據區間自動決定時區忽略策略（分鐘/小時等級不忽略；日以上忽略）。
  - 輸出 `pandas.DataFrame`，支援 MultiIndex 與欄位分組（by `group_by`）。
- **Scrapers**：
  - 各自負責特定資料面向的端點存取與轉換，回傳 `pandas` 或結構化資料。
  - `history.py` 處理 K 線/分時、`quote.py` 處理報價/快資訊、`fundamentals.py`/`analysis.py`/`holders.py` 對應基本面與持有者資料、`funds.py` 對 ETF/基金。
- **WebSocket（live.py）**：
  - 提供同步與非同步兩種客戶端，負責訂閱與解析即時報價（透過 `pricing_pb2.py`）。
  - 對外提供事件/回呼型使用方式（message handler）。
- **Domain**：
  - `Market/Sector/Industry` 型別封裝市場維度；`const.SECTOR_INDUSTY_MAPPING` 提供對照。

### 資料流與互動
- **單一代碼歷史資料（`Ticker.history()`）**：
  1) `Ticker.history()` 透過 `PriceHistory` 組出 REST 請求與參數
  2) 呼叫 `YfData.get/cache_get` → 取得/更新 Cookie/Crumb → 發送請求
  3) 解析 Yahoo JSON → 轉為 `pandas.DataFrame` → 回傳
- **多代碼下載（`download()`）**：
  1) 將輸入字串/清單規整化、處理 ISIN → Ticker 映射
  2) 視需求開啟多執行緒並行請求（DEBUG 模式則關閉）
  3) 匯總各代碼資料、補處理索引時區、建 MultiIndex 或依 `group_by` 整欄
  4) 彙整錯誤/traceback 進日志
- **Cookie/Crumb 與重試**：
  - `YfData` 先嘗試當前策略（`basic` 或 `csrf`），4xx/429 時自動切換策略再試一次；429 會顯式拋出 `YFRateLimitError`。

### 快取策略
- **應用層記憶化**：`YfData.cache_get` 使用 `lru_cache`（參數以 `frozendict/tuple` 固化以支援雜湊）。
- **持久層（SQLite via peewee）**：
  - 時區快取：Ticker → TimeZone，集中於 `cache.py`（路徑管理用 `platformdirs`）。
  - Cookie 快取：持久化 `curl_cffi` cookie jar（過期檢查與再用）。
- **其他共用狀態**：下載過程的中介狀態在 `shared.py`（結果、錯誤、traceback）。

### Screener 與搜尋
- **搜尋（`Search`）**：基於查詢字串取得代碼、報價與新聞摘要（`yfinance/search.py`）。
- **篩選（`screener`）**：
  - `EquityQuery`/`FundQuery` 以組合式查詢條件（EQ、IS-IN、BTWN、GT、LT、GTE、LTE）構建過濾樹。
  - `screen()` 執行查詢，回傳符合條件的標的清單與屬性（詳見 `yfinance/screener/`）。

### 例外處理與穩定性
- 主要例外：`YFRateLimitError`（429/節流）、`YFDataException`（session/快取不相容）、`YFTickerMissingError`/`YFTzMissingError`/`YFPricesMissingError` 等。
- Cookie 策略自動切換、發生錯誤時的錯誤彙整與日志輸出，提升可觀測性。

### 設定與環境
- `set_config(proxy=...)`：設定全域 Proxy（會更新 `YfData` 的 session proxies）。
- `set_tz_cache_location(path)`：變更時區快取 SQLite 存放路徑。
- `enable_debug_mode()`：啟用更詳細的結構化日志（`utils.py`）。

### 依賴與版本（節選）
- 主要：`pandas`、`numpy`、`curl_cffi`、`beautifulsoup4`、`peewee`、`platformdirs`、`pytz`、`frozendict`、`multitasking`、`protobuf`、`websockets`
- 額外（選配）：`requests_cache`、`requests_ratelimiter`、`scipy`
- 參考 `setup.py` 與 `requirements.txt`。

### 測試
- `tests/` 涵蓋：快取與無權限情境、即時、搜尋、Ticker API、價格修復與歷史資料等，作為行為規格與回歸保障。

### 可擴充點與建議
- 新增資料面向：
  - 在 `scrapers/` 新增模組，封裝端點與解析；於 `TickerBase` 以 lazy-load 方式掛載。
- 增補公共 API：
  - 在 `yfinance/__init__.py` 匯出新元件，並在 `__all__` 更新。
- 請求/效能最佳化：
  - 盡量透過 `YfData.cache_get` 與批次 `download()`；謹慎調整 `threads`、`progress`、`timeout`。
- 注意事項：
  - 不要手動向 `params` 注入 `crumb`（`YfData` 會自動處理）。
  - 自訂 session 應使用 `curl_cffi.requests.Session`，避免與 `requests_cache` 不相容。

### 法律與使用限制
- 專案與 Yahoo, Inc. 無附屬/授權/背書關係，僅供研究與教育用途。
- 請遵循 Yahoo! 服務條款與使用規範。

### 參考檔案（關鍵）
- `yfinance/__init__.py`、`yfinance/data.py`、`yfinance/base.py`、`yfinance/multi.py`
- `yfinance/scrapers/*`、`yfinance/live.py`、`yfinance/pricing.proto`
- `yfinance/cache.py`、`yfinance/utils.py`、`yfinance/const.py`、`yfinance/exceptions.py`
- `tests/*`