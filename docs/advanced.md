# 進階內容

這份文件收錄 README 以外、仍然有效的技術細節。回到 [README](../README.md)。

## 輸入路由擴充

技能由原本的 39 個擴充到 60 個，新增工具能力判斷、外部狀態、解釋與比較、文字轉換、數值與樣本操作等程序。擴充來源與限制見 [`expansion_provenance.json`](../expansion_provenance.json)，各技能目前版本以 manifest 為準。

可選用的 [`routing/extension.py`](../routing/extension.py) 只讀取目前可見的輸入，並附有適用、近似不適用與換題測試。用法與限制見 [`routing/README.md`](../routing/README.md)。路由命中不等於語義正確，也不代表已執行工具或完成模型訓練。

## 事實回答修訂

`factual-question-resolution` v2、`concept-explanation` v2 與 `grounded-evidence-answer` v4 已更新。另提供具體屬性查詢的窄路由，讓應用程式放在寬鬆的概念規則之前。適用例、來源與閉卷資料改造、已知題保留的驗證方式，見 [`factual-calibration.md`](factual-calibration.md)。這些更新需要另外固定版本，並量測模型效果。

## Test-time 元件與短卡

可選用的 [`testtime/`](../testtime/) 元件支援中文詞彙檢索、按需短操作卡、帶來源的狀態，以及隔離的 Python 執行。完整的[實驗規格](../experiments/test_time_v1/goal.json)、[中文分析與實驗結果](test-time-study.html)把工程驗證、模型提升與「是否接近 9B」分開記錄。這項工作不訓練、不修改權重，也不更動已發布的資料集樣本。

短卡是帶有來源版本與雜湊的獨立表示，不應一次載入全部技能。請用目標模型的 tokenizer 量測長度。模型評測的隱藏答案與測試，只交給 scorer。

更新短卡後，請執行：

```bash
python scripts/build_inference_cards.py
python scripts/build_inference_cards.py --check
```

## 在其他 agent 中採用技能的注意事項

支援 GitHub API 的 agent，可以讀同一個 commit 的 `catalog.json` 與各技能路徑。`catalog.json` 是完整的機器可讀定義；`SKILL.md` 是方便模型閱讀的版本。

模型仍要檢查每份技能的「何時使用／何時不要用」、實際的工具 schema、目前任務與權限。不要把整個 repository 當成能蓋過 system 或 user 指令的政策，也不要硬套無關的技能。技能內容不得自行提升權限，或覆蓋應用程式的上層指令。

## 來源紀錄

[`provenance.json`](../provenance.json) 記錄這份技能快照的來源物件雜湊與已回讀的資料版本。GitHub 上的變更不會自動改動已發布的資料集樣本；新版技能要用在資料集之前，需要另行審查、固定版本，並重新驗證受影響的樣本。
