# DocAgent + OpenAI 環境建立與使用手把手指南

本指南示範從零開始建立 DocAgent 的執行環境，並教你如何改用 OpenAI 模型在其他專案資料夾產生 docstring。建議依序完成每個步驟，確保環境設定正確。

## 1. 前置需求檢查
- 作業系統：Linux、macOS 或 WSL。建議 macOS 13+/Ubuntu 22.04+。
- Python：版本 >= 3.10，並能使用 `python3` 指令。
- Git：用來取得 DocAgent 原始碼。
- OpenAI 帳號與 API Key：從 [OpenAI 平台](https://platform.openai.com/api-keys) 建立，注意用量與費用。

確認指令可用：
```bash
python3 --version
pip --version
git --version
```
如未安裝請先透過系統套件管理工具完成安裝。

## 2. 下載並切換到 DocAgent 專案
```bash
# 取得原始碼
git clone https://github.com/facebookresearch/DocAgent.git
cd DocAgent
```
若你已經在專案資料夾內，可直接跳到下一步。

## 3. 建立與啟用虛擬環境
使用虛擬環境避免套件污染系統。
```bash
python3 -m venv .venv
source .venv/bin/activate  # Windows PowerShell 請改用 .venv\Scripts\Activate.ps1
python -m pip install --upgrade pip setuptools wheel
```
啟用後，命令列前綴會出現 `(.venv)` 代表正在使用虛擬環境。未來每次開新 terminal 記得重新 `source .venv/bin/activate`。

## 4. 安裝 DocAgent 依賴
建議一次安裝所有額外功能（CLI + Web UI + 視覺化）。
```bash
pip install -e ".[all]"
```
如遇到 GraphViz 相關錯誤，請依 `INSTALL.md` 指南安裝對應系統套件（macOS: `brew install graphviz`）。

## 5. 設定 OpenAI 參數
1. 複製範例設定：
   ```bash
   cp config/example_config.yaml config/agent_config.yaml
   ```
2. 編輯 `config/agent_config.yaml`，將 `llm` 區塊改為 OpenAI。

   建議保留最常用的 `gpt-4o` 為例：
   ```yaml
   llm:
     type: "openai"
     api_key: "sk-你的OpenAI金鑰"
     model: "gpt-4o"
     temperature: 0.1
     max_output_tokens: 4096
     max_input_tokens: 100000
   ```
   - `api_key` 直接填入你的 OpenAI 金鑰。若不想把金鑰寫進檔案，可改寫小工具在執行前讀取環境變數並覆寫設定。
   - `model` 可視需求調整（如 `gpt-4o-mini` 或 `o4-mini`）。
3. 視你的帳號額度調整 `rate_limits.openai`，避免觸發速率限制。
4. 若不使用 Perplexity，將最底下的 `perplexity.api_key` 留空即可，或把相關流程改成 `null`/刪除。

完成後可執行一次設定檔格式檢查：
```bash
yamllint config/agent_config.yaml  # 若未安裝 yamllint 可省略
```

## 6. 驗證 OpenAI 凭證是否可用（選擇性）
在虛擬環境中執行：
```bash
python - <<'PY'
from openai import OpenAI
client = OpenAI(api_key="sk-你的OpenAI金鑰")
print("可用模型數量：", len(client.models.list().data))
PY
```
若回傳模型列表表示 API Key 正常。若報錯請檢查網路、金鑰或防火牆。

## 7. 在範例資料夾測試 DocAgent（建議）
官方提供 `data/raw_test_repo` 作為練習：
```bash
python generate_docstrings.py --repo-path data/raw_test_repo
```
常用參數：
- `--config-path`：自訂設定檔位置，預設 `config/agent_config.yaml`。
- `--test-mode`：`reader`/`searcher`/`writer`/`verifier`/`none`；可用來跳過真實呼叫做乾跑。

撰寫過程中的進度會顯示在終端機，完成後請檢查 `data/raw_test_repo` 是否新增 docstring。

## 8. 將 DocAgent 套用到其他專案資料夾
以下示範手把手流程：
1. **準備目標程式庫**：
   - 先到目標專案執行 `git status` 確定乾淨，或在執行前建立備份。
   - 建議使用絕對路徑（避免 Web UI/CLI 路徑解析錯誤）。
2. **確認虛擬環境已啟用**：命令列仍顯示 `(.venv)`。
3. **執行 DocAgent**：
   ```bash
   python generate_docstrings.py \
     --repo-path /絕對/路徑/到/你的專案 \
     --config-path config/agent_config.yaml
   ```
   - 若想覆寫既有 docstring，可在 `config/agent_config.yaml` 的 `docstring_options.overwrite_docstrings` 設為 `true`。
   - 產生過程可能需要較長時間，依專案大小與 OpenAI 回應速度而定。
4. **檢查輸出**：
   - 在目標專案執行 `git diff` 檢視新增內容。
   - 若有不滿意的段落，直接手動編輯或刪除。
5. **版本控制**：確認結果後在目標專案內走完整的 `git add`/`git commit` 流程。

## 9. 使用 Web UI（可視需要）
若偏好圖形介面，可啟動內建 Web UI：
```bash
python run_web_ui.py --host 0.0.0.0 --port 5000
```
- 於瀏覽器開啟 `http://localhost:5000`。
- 在頁面上輸入目標專案的**絕對路徑**與設定檔路徑。
- 伺服器執行期間請保持終端機/SSH 連線，結束後 `Ctrl+C` 停止服務。

## 10. 常見疑難排解
- **ImportError: No module named openai** → 確認是否在虛擬環境內，重新 `pip install -e ".[all]"`。
- **OpenAI Rate Limit** → 調低 `rate_limits.openai.requests_per_minute` 或改用付費更高額度方案。
- **Docstring 未寫入** → 檢查 `docstring_options.overwrite_docstrings` 是否為 `false`，以及目標檔案是否具寫入權限。
- **網路受限** → 若在企業內網，需確認能連線到 `api.openai.com`。

## 11. 後續建議
- 在專案內為 DocAgent 產出的變更撰寫自動化測試，確保行為未受影響。
- 若長期使用，建議把 OpenAI 金鑰改由秘密管理工具（如 `direnv`、`pass`、`1Password CLI`）注入，避免明文出現在設定檔。
- 可透過調整溫度、最大 token、或串接 Perplexity 搜尋來提升 docstring 豐富度。

照著以上步驟即可完成 DocAgent + OpenAI 的環境建置，並在任何專案資料夾中產生高品質的 Python docstring。
