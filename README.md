# JTCG Shop CRM AI Agent（AI 工程測驗）

本專案是一個模擬電商客服場景的 RAG（Retrieval-Augmented Generation）型 AI Agent，目標是協助 JTCG Shop 回答常見 FAQ、商品推薦、訂單查詢與真人客服轉接等情境，在小規模資料與有限開發時間下，兼顧「搜尋準確度」、「回覆品質」與「系統可維護性」。

## 專案概要

- 執行環境：Vertex AI Workbench（Notebook）
- 雲端服務：BigQuery、Cloud Storage、Vertex AI Gemini
- 主要功能：
  - FAQ / 商品知識庫檢索與回答（RAG）
  - 商品向量搜尋與推薦（螢幕支架相關）
  - 訂單查詢流程（依 `user_id` / `order_id` 查詢）
  - 真人客服轉接（模擬 API）
- 資料來源：
  - FAQ：`ai-eng-test-sample-knowledges.csv`
  - 商品：`ai-eng-test-sample-products.csv`
  - 對話樣本：`ai-eng-test-sample-conversations.json`
  - 訂單：Notebook 內建 JSON（模擬小型 Orders DB）

## 架構與流程總覽

整體流程可分為五個層次：

1. 資料載入與標準化  
   - 從 Cloud Storage 讀取 FAQ / 商品 CSV。  
   - 標準化欄位名稱（FAQ：`id`, `title`, `question`, `answer`, `url`, `lang`；商品：`sku`, `name`, `description`, `spec`, `product_url`, `image_url`）。  
   - 將整理後的 DataFrame 寫入 BigQuery 資料表 `knowledges` 與 `products`。

2. 向量嵌入與索引  
   - 在 BigQuery 建立 Remote Model，端點採用 Gemini 多語言向量模型：`ENDPOINT = 'gemini-embedding-001'`。  
   - FAQ：
     - 使用 `question/title + answer` 串接成一段文字做 embedding，寫入 `answer_embedding`。  
   - 商品：
     - 使用 `description`（若無則退回 `name`）做 embedding，寫入 `description_embedding`。  
   - 之後透過 BigQuery `VECTOR_SEARCH` 進行語意相似度搜尋。

3. 意圖判斷（Intent Detection）  
   - 使用規則將使用者問題分類為：
     - `FAQ`：一般客服、政策說明  
     - `PRODUCT`：商品規格 / 推薦 / VESA / 支架 / 走線等  
     - `ORDER`：查詢訂單狀態  
     - `HANDOFF`：要求真人客服  
   - 規則以關鍵字與簡單字串匹配為主，避免過度複雜的意圖模型。

4. 各 Intent Handler  
   - FAQ Handler：
     - 呼叫 BigQuery 向量搜尋取得 Top-K FAQ。  
     - 若最相近 FAQ 題目/內容與問題主題高度吻合（搭配關鍵字比對），直接以該 FAQ answer 回覆。  
     - 若信心不足，將多筆 FAQ 組成 context blocks，交由 LLM 統整回答。  
   - PRODUCT Handler：
     - 透過商品向量搜尋列出最相近商品，回覆名稱、簡要描述與商品連結。  
   - ORDER Handler：
     - 從文字中解析 `user_id`（例如 `u_123456`）與 `order_id`（例如 `JTCG-202508-10001`）。  
       - 沒有任何 ID：要求先提供 `user_id`。  
       - 只有 `user_id`：列出該使用者底下所有訂單摘要。  
       - 有 `user_id + order_id`：回傳該訂單詳細狀態、物流資訊與連結。  
       - 只有 `order_id`：基於資訊安全要求補上 `user_id`。  
   - HANDOFF Handler：
     - 嘗試從訊息中抓出 Email，格式正確則呼叫模擬 `handover_simple`。  
     - 若未偵測到 Email，請先提供 Email 再轉接。

5. LLM 回覆層（RAG Answer Layer）  
   - 使用 Vertex AI Gemini 2.5 Pro 作為最後一層回覆生成。  
   - 將「使用者問題」與「RAG 檢索到的 FAQ / 商品內容」包裝成 prompt，請模型在嚴格規則下作答：  
     - 只能依賴檢索到的內容說明政策，不得自行發明條款。  
     - 有明確 FAQ 時，直接用該 FAQ 資訊整理出具體答案。  
     - 所有資料都無關或沒有有用線索時，才回覆「目前無法根據現有資料確認」，並建議改由真人客服協助。

## GCP 環境與前置設定

### 基本專案與區域

- 專案 ID：`willy-poc`（示意，可依實際調整）  
- BigQuery 位置：`US`（多區域）  
- Vertex AI 區域：`us-central1`  
- Cloud Storage Bucket：`willy-poc-crm-agent`  

### BigQuery Dataset 與資料表

- Dataset：`jtcg`  
- 資料表：
  - `jtcg.knowledges`  
    - `id`, `title`, `question`, `answer`, `url`, `lang`  
    - 向量欄位：`answer_embedding`  
  - `jtcg.products`  
    - `sku`, `name`, `description`, `spec`, `product_url`, `image_url`  
    - 向量欄位：`description_embedding`  

### Remote Model 與 Connection

- BigQuery Connection 範例：`us.jtcg`  
- Remote Model 指向 Vertex AI 向量模型：`ENDPOINT = 'gemini-embedding-001'`  
- 提供 `ML.GENERATE_EMBEDDING` 與 `VECTOR_SEARCH` 使用。

### 權限與 API（概略）

- 需啟用：
  - BigQuery API  
  - BigQuery Connection API  
  - Vertex AI API  
  - Cloud Storage API  
- Service Account 需具備：
  - BigQuery Dataset 讀寫  
  - Cloud Storage Bucket 讀取  
  - 透過 BigQuery Connection 呼叫 Vertex AI Remote Model 的權限  

## 專案檔案與資料說明

Repo 主要包含：

- `README.md`：專案說明（本文件）  
- Notebook 檔（例如）：`jtcg_crm_ai_agent.ipynb`  
  - 包含資料載入、向量建模、Agent Handler、批次模擬與輸出等流程。  

實際資料放在 GCS，repo 中只描述路徑：

- FAQ CSV：`gs://willy-poc-crm-agent/ai-eng-test-sample-knowledges.csv`  
- 商品 CSV：`gs://willy-poc-crm-agent/ai-eng-test-sample-products.csv`  
- 對話 JSON：`gs://willy-poc-crm-agent/ai-eng-test-sample-conversations.json`  

## 解決的問題與優化說明

### 1. FAQ 搜尋不準：向量模型與內容調整

問題：  
- 初版使用 `text-embedding-004` 並僅對 `answer` 做 embedding。  
- 中文問句如「保固多久？維修怎麼申請？」常檢索到優惠券或其他不相關 FAQ。

解法：  
- 改用多語言向量模型 `gemini-embedding-001`，較適合繁體中文。  
- Embedding 內容改為 `question/title + answer` 串接，讓模型同時看到「問題」與「答案」，強化語意對齊。

### 2. 檢索結果主題偏移：關鍵字輔助比對

問題：  
- 單純看向量距離排序，有時 Top-1 FAQ 主題仍與問題略有偏移。  

解法：  
- 在 FAQ Handler 中加入主題關鍵字比對，尤其針對：  
  - 「保固」、「維修」、「發票」、「三聯」、「統編」、「出貨」、「免運」等。  
- 雙向檢查：關鍵字同時出現在「使用者問題」與「候選 FAQ 的標題/內容」，才視為主題對齊。  
- 若距離合理且關鍵字對齊 → 直接採用該 FAQ answer 回覆。  
- 若無明顯對齊 → 交由 LLM 在多筆候選中權衡整理。

### 3. LLM 過度保守：常說「資料不足」

問題：  
- 即使已檢索到明確 FAQ（例如保固 1 年、三聯式發票規則），模型仍可能回答「目前無法確認」。

解法：  
- 強化 Prompt 規則：  
  - 只要檢索結果中有清楚且具體的答案，就必須使用該資料給出自信回答。  
  - 只有所有資料都無關或沒有有用線索時，才可以回答「無法確認」。  
- 明確限制：  
  - 不得自行新增退換貨、保固、發票、出貨、運費、優惠券等政策條款，必須以檢索到的內容為準。

### 4. 降低幻覺與不必要延伸

- Prompt 中強調：
  - 只能根據檢索資料說明政策，不能臆測或延伸規則。  
  - 不需重複貼出原文，不需解釋思考過程。  
- 實測結果顯示，回答風格
