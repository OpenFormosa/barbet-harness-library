# Barbet Harness Library

給語言模型用的「工作手冊」與「執行框架」。

- **64 份技能手冊**：每份寫清楚某類任務什麼時候適用、步驟、出錯怎麼辦、怎麼驗收。
- **`barbet_inference` 執行框架**：幫模型挑手冊、讀文件、跑公開測試，失敗時有限次修復。

兩者合起來，就是一套 harness：不改模型權重，只改模型「看到什麼、怎麼被檢查」。

> 本專案原名 Barbet Skill Library。舊網址會自動轉到這裡。
>
> **English.** A library of 64 versioned procedure guides (skills) for language models, plus `barbet_inference`, a runtime that selects skills, reads documents, runs public tests, and repairs failures. Together they form a harness that changes what a model sees and how its output is checked, without changing its weights. Formerly the Barbet Skill Library.

## 目錄

- [給誰用](#給誰用)
- [30 秒看懂一份技能](#30-秒看懂一份技能)
- [快速開始](#快速開始)
- [為什麼叫 harness](#為什麼叫-harness)
- [瀏覽技能](#瀏覽技能)
- [給其他語言模型使用](#給其他語言模型使用)
- [實驗結果與限制](#實驗結果與限制)
- [如何貢獻](#如何貢獻)
- [進階內容](#進階內容)
- [來源、界線與授權](#來源界線與授權)

## 給誰用

| 你是 | 你可以 |
| --- | --- |
| 想讓小模型更守規矩的開發者 | 安裝 `barbet-infer`，直接用技能與測試包住模型 |
| 用其他模型或 agent 框架的人 | 只下載需要的 `SKILL.md`，放進模型的前文 |
| 研究人員 | 重現固定權重的 test-time 實驗，看每個數字的條件 |
| 想參與的人 | 提議新技能，或改善既有技能。這是 OpenFormosa 最容易上手的入口 |

## 30 秒看懂一份技能

以 [`arithmetic-modeling`](skills/arithmetic-modeling/SKILL.md) 為例，每份技能都有同樣的欄位：

| 欄位 | 內容（節錄） |
| --- | --- |
| 何時使用 | 使用者用文字描述數量關係，需要算出結果 |
| 何時不要用 | 問題只有定性描述，或使用者已給完整算式 |
| 步驟 | 抽出數值與單位 → 找出運算關係 → 檢查分母 → 組出算式 → 求解 → 依格式輸出 |
| 失敗處理 | 缺資訊時標出缺什麼；能給符號解就不拒答 |
| 驗收 | 輸出符合使用者指定的格式 |

技能不含任何訓練題目的答案、隱藏測試或工具結果。

## 快速開始

在本 repo 目錄內、獨立的 Python 3.12 環境中安裝。本機生成需要相容的 GPU 與 CUDA。

```bash
python -m venv .venv
source .venv/bin/activate
pip install -e '.[vllm]'
```

問一個問題：

```bash
barbet-infer run --family mini \
  --prompt '每箱 12 瓶，買 3 箱共有多少瓶？' --contract number \
  --output outputs/answer.jsonl --trace outputs/calls.jsonl
```

常用指令：

| 指令 | 用途 |
| --- | --- |
| `barbet-infer list-skills` | 列出可用技能 |
| `barbet-infer prepare ...` | 先看 prompt、來源與 token 預算，不生成 |
| `--skill NAME` | 指定一份完整技能；不指定時使用按需短卡 |
| `--contract text` | 寫作、翻譯等自由文字，不套用短答案限制 |

其他安裝方式：

- 只連遠端 vLLM 相容服務：`pip install -e '.[tokenizer]'`。
- 只讀技能，或接自己的生成後端：`pip install -e .`。

目前內建 MiniCPM5-1B（`--family mini`）與 Qwen3.5-9B（`--family qwen`）的固定版本設定。完整的安裝、HTTP 服務、Python API、輸入格式與輸出意義見[使用說明](docs/inference.md)。五種任務的自訂輸入範例見 [`examples/inference/`](examples/inference/)。

## 為什麼叫 harness

只有手冊，模型不一定會用，也不一定用對。只有執行框架，模型不知道每類任務該怎麼做。

`barbet_inference` 把這兩件事接在一起：

1. **選手冊**：依輸出契約（例如 `number`、`python`、`tool_call`）挑出適用的技能或短卡。
2. **給資料**：檢索使用者提供的文件，附上來源。
3. **生成**：呼叫本機 vLLM 或相容的 HTTP 服務。
4. **檢查**：程式題在隔離環境執行公開測試，工具呼叫檢查 schema。
5. **修復**：檢查失敗時，帶著錯誤訊息重試，次數有上限。

整個過程不訓練模型，也不會改用大模型代答。

## 瀏覽技能

64 份技能分成 7 個領域：

| 領域 | 數量 | 例子 |
| --- | ---: | --- |
| `general` 一般指令 | 20 | `clarify-missing-inputs`、`arithmetic-modeling`、`tool-selection-and-arguments` |
| `coding` 程式 | 13 | `code-task-framing`、`execution-feedback-repair` |
| `math` 數學 | 10 | `descriptive-statistics`、`successive-relative-growth` |
| `agent` 工具與代理 | 7 | `tool-capability-triage`、`initial-action-planning` |
| `reading` 閱讀 | 6 | `passage-fact-binding`、`unsupported-question-calibration` |
| `writing` 寫作 | 5 | `constrained-creative-writing`、`translation-alignment` |
| `science` 科學 | 3 | `bilingual-science-terms`、`hypothesis-and-evidence` |

三種瀏覽方式：

- **讀給人看**：打開 [`skills/`](skills/) 底下任一目錄的 `SKILL.md`。
- **讀給程式看**：[`catalog.json`](catalog.json) 是全部技能的機器可讀定義。
- **固定版本**：[`manifest.json`](manifest.json) 記錄每份技能的路徑、版本、範圍與 SHA-256。

使用時，先放入適用的 `general` 技能，再依任務加入其他領域的技能。模型仍要檢查每份技能的「何時使用／何時不要用」、實際的工具 schema 與權限。不要把整個 repo 當成能蓋過 system 或 user 指令的政策，也不要硬套無關的技能。

## 給其他語言模型使用

不需要 Barbet 專用工具。挑好技能，把 `SKILL.md` 的文字放進模型前文即可。

正式使用時，請固定到一個審查過的 Git commit，並用 manifest 裡的 `markdown_sha256` 核對下載內容。`main` 會隨社群維護持續更新。

```bash
COMMIT="replace-with-reviewed-commit-sha"
BASE="https://raw.githubusercontent.com/OpenFormosa/barbet-harness-library/${COMMIT}"
curl -fLsS "${BASE}/manifest.json" -o manifest.json
curl -fLsS "${BASE}/skills/tool-selection-and-arguments/SKILL.md" -o SKILL.md
```

支援 GitHub API 的 agent，也可以讀同一個 commit 的 `catalog.json`。請保留技能的原始名稱與版本，再和任務、實際工具契約一起提供。

注意：技能是程序提示，不會自動安裝到其他模型。公開讀得到，也不代表它在你的環境裡有更高的指令優先權。請自行審查後再採用。

## 實驗結果與限制

**做法。** 固定模型權重，只改變模型看到的技能與工具。事先封存 646 題測試，跑 3 個 seed、比較 10 組設定。

**結果。** MiniCPM5-1B 的四領域主指標從 38.02% 提升到 50.39%（+12.37 個百分點，配對 95% CI +9.90 至 +14.97）。同一套 harness 下，Qwen3.5-9B 是 76.24%，1B 模型仍有明顯差距。

**這不代表什麼。**

- 主評分包含輸出格式是否正確，所以有一部分提升來自格式。
- 數學題的事後格式診斷，沒有看到數值推理變好。
- 來源引句吻合、schema 合法或公開測試通過，都不等於整題答對。

完整的消融、成本、失敗分類與限制，見[中文分析與實驗結果](docs/test-time-study.html)與[重現說明](experiments/test_time_v1/README.md)。實驗規格見 [`goal.json`](experiments/test_time_v1/goal.json)。這項工作沒有訓練或修改權重，也沒有更動已發布的 HF 樣本。

## 如何貢獻

任何人都可以 fork 並送 PR，或用 [Issues](https://github.com/OpenFormosa/barbet-harness-library/issues) 提議新技能、回報問題。

1. 編輯 `skills/<name>/skill.json`。`SKILL.md` 是自動生成的，不要只改它。
2. 有語義變更時，更新技能的 `version`。
3. 重新產生並檢查：

   ```bash
   python scripts/build.py
   python scripts/build.py --check
   ```

   這會更新 `SKILL.md`、`catalog.json` 與 `manifest.json`。
4. 在 PR 附上一個適用例、一個相似但不適用的例子，以及驗證方法。

細節見 [`CONTRIBUTING.md`](CONTRIBUTING.md)。PR 會經過程式碼擁有人審核與自動檢查，維護者也會審查語義、來源與安全性。

GitHub 上的變更**不會自動改動**已發布的 HF 訓練樣本。新版技能要用在資料集之前，需要另行審查、固定版本，並重新驗證受影響的樣本。

## 進階內容

### 輸入路由擴充

技能由原本的 39 個擴充到 60 個，新增工具能力判斷、外部狀態、解釋與比較、文字轉換、數值與樣本操作等程序。擴充來源與限制見 [`expansion_provenance.json`](expansion_provenance.json)，各技能目前版本以 manifest 為準。

可選用的 [`routing/extension.py`](routing/extension.py) 只讀取目前可見的輸入，並附有適用、近似不適用與換題測試。用法與限制見 [`routing/README.md`](routing/README.md)。路由命中不等於語義正確，也不代表已執行工具或完成模型訓練。

### 事實回答修訂

`factual-question-resolution` v2、`concept-explanation` v2 與 `grounded-evidence-answer` v4 已更新。另提供具體屬性查詢的窄路由，讓應用程式放在寬鬆的概念規則之前。適用例、來源與閉卷資料改造、已知題保留的驗證方式，見 [`docs/factual-calibration.md`](docs/factual-calibration.md)。這些更新需要另外固定版本，並量測模型效果。

### Test-time 元件與短卡

可選用的 [`testtime/`](testtime/) 元件支援中文詞彙檢索、按需短操作卡、帶來源的狀態，以及隔離的 Python 執行。

短卡是帶有來源版本與雜湊的獨立表示，不應一次載入全部技能。請用目標模型的 tokenizer 量測長度。模型評測的隱藏答案與測試，只交給 scorer。

更新短卡後，請執行：

```bash
python scripts/build_inference_cards.py
python scripts/build_inference_cards.py --check
```

### SFT 資料改造提案

[v26 診斷與 skill-aware SFT 設計](docs/skill-aware-sft-v1.html)，把輸入介面、數學、工具語意與資料覆蓋的問題，對應到技能使用、RAG 與真實回饋修復的學習視圖。[版本化配方](docs/configs/skill-aware-sft-v1.json)區分三件事：目前 harness 已支援的流程、尚需實作的 adapter，以及訓練後需要驗證的技能增益。

這是尚未採用的資料方案，沒有啟動訓練，也沒有修改 HF release。完整彙總與來源雜湊見 [v26 分析](docs/results/v26-sft-diagnosis.json)，可用[唯讀腳本](scripts/analyze_v26_for_sft.py)重現。

### 給 1B 模型的技能指引

針對 Barbet 1B 的技能與資料修訂，見 [1B 技能與公開解法指引](docs/student-1b-guidance.md)。

## 來源、界線與授權

- **來源。** 這份快照匯出自 Barbet `skill-prefill-v1` 資料工程中已接受的技能，對應 [Barbet long-context SFT 資料集](https://huggingface.co/datasets/OpenFormosa/barbet-long-context-sft)。[`provenance.json`](provenance.json) 記錄來源物件的雜湊，以及一個已回讀的 HF 資料版本。
- **界線。** 部分技能由本地 Qwen 模型生成，只經過有限的測試與模型／工程審查，**不是 human-verified**。它們也沒有證明 Barbet 已學會路由、組合技能或使用 1M 上下文。實際能力需要另外量測。
- **授權。** 原創技能文字與工具採 [Apache License 2.0](LICENSE)。引用或衍生自其他來源的內容，仍受各自條款約束。HF 資料集的多來源授權，不因本 repo 的授權而改變。
