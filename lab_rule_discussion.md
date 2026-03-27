# 實驗室 fMRIPrep 運算環境架構與資料規範說明會議

**會議目的：** 與大家同步實驗室伺服器在執行 fMRIPrep 神經影像前處理流程時的標準作業模式，期望在確保系統底層安全性的同時，也能協助大家建立統一的資料格式，讓整體的運算流程更加順暢。

---

## 1. 容器化環境之安全考量與政策宣導

在查閱 fMRIPrep 官方文獻或網路教學時，大家最常看到的執行指令通常是 `fmriprep-docker`。然而，在實驗室多用戶共享的伺服器架構下，我們必須避免採用此種單機運作模式，以免帶來系統風險。

* **風險說明：** Docker Daemon 在 Linux 系統底層預設是以 `root` 權限持續運行的。若開放使用 `fmriprep-docker`，系統在架構上必須給予所有使用者較高的管理員權限。這不符合資訊安全的「最小權限原則」，若操作不慎，極易影響伺服器的整體穩定性與其他同學的運算作業。

* **業界規範參考：** 為了系統安全，國際大型運算中心通常會嚴格限制 Docker，並採用更安全的替代方案。正如 fMRIPrep 官方文件 (v0.5.1) 所述：

  > *"For security reasons, many HPCs (e.g., TACC) do not allow Docker containers, but do allow Singularity containers."*
  > （官方文件參考：https://fmriprep.org/en/0.5.1/installation.html ）

---

## 2. 實驗室標準方案：Apptainer (原 Singularity)

為了解決大家的運算需求並兼顧伺服器安全，實驗室全面採用 **Apptainer**（前身為 Singularity）作為標準的容器執行環境。

### Apptainer 的學術優勢：

1. **無特權執行 (Rootless Execution)：** 專為高效能運算 (HPC) 設計，不需 root 權限即可運行，且自動對應原本的使用者身份存取檔案。

2. **避免污染系統環境 (Environment Isolation)：** 所有龐大的依賴軟體（FSL, FreeSurfer 等）皆封裝在容器內，保持伺服器全域環境的純淨。

3. **延續並確保實驗結果的絕對重現性 (Scientific Reproducibility)：** 透過不可變的 `.sif` 映像檔，確保各位的研究結果不會因為伺服器軟體更新而產生偏差，保證科學實驗的可重現性。

### 使用注意事項與後續行動：

* **需手動設定掛載路徑 (Manual Bind Mounting)：** 大家需要學習使用 `--bind` 參數，手動將 BIDS 資料夾、輸出目錄以及 FreeSurfer License 映射進容器中。指令雖然會比網路上看到的稍長，但能幫助大家更清晰地掌握資料流向。

* **映像檔的統一管理：** 實驗室在共用目錄提供統一的映像檔，請勿重複下載以節省硬碟空間。

* **[後續行動] 自動化腳本支援：** 為了降低配置門檻並避免路徑設定錯誤，**實驗室後續將會撰寫並提供標準化的執行腳本 (Standardized Wrapper Script)**。該腳本將自動處理大部分的 `--bind` 掛載參數，方便大家一鍵呼叫 `.sif` 映像檔進行運算。

---

## 3. 原始資料 (Raw Data) 的 BIDS 結構化建議與轉換工具

fMRIPrep 強制要求輸入的檔案結構必須符合 **BIDS** 規範，否則管線將無法運作。這是一項基於官方架構的硬性要求：

> **"The fMRIPrep workflow takes as principal input the path of the dataset that is to be processed. The input dataset is required to be in valid BIDS format, and it must include at least one T1w structural image and (unless disabled with a flag) a BOLD series."**
> — *Extract from fMRIPrep Usage Documentation (https://fmriprep.org/en/stable/usage.html)*

### 參考依據與實務建議：
* **官方文件出處：** https://fmriprep.org/en/stable/usage.html
* **學術實務建議 (JNeuroLab)：** https://www.jneurolab.org/fmriprep-bids

### BIDS 目錄結構概覽：

```text
my_dataset/
├── dataset_description.json
├── participants.tsv
├── sub-01/
│   ├── anat/
│   │   ├── sub-01_T1w.nii.gz
│   │   └── sub-01_T1w.json
│   └── func/
│       ├── sub-01_task-rest_bold.nii.gz
│       └── sub-01_task-rest_bold.json
```

---

## 4. 運算前置作業：BIDS 資料標準驗證流程

### 未使用 BIDS 格式的潛在後果：

1. **管線啟動失敗 (Execution Failure)：** 解析器會直接報錯並終止程序，浪費排隊運算時間。
2. **演算法套用錯誤 (Resource Waste)：** 資料命名模糊可能導致系統誤判影像類型，消耗數十小時產生無效數據。
3. **後續機器學習訓練瓶頸：** 缺乏統一結構會導致後端的資料讀取程式無法自動化處理。

### 標準驗證流程建議：

為了確保資料品質並節省運算資源，官方強烈建議在啟動管線前進行驗證：

> **"We highly recommend that you validate your dataset with the free, online BIDS Validator."**
> — *Official Recommendation from fMRIPrep (https://fmriprep.org/en/stable/usage.html)*

**選項 A：網頁端檢核工具 (適合初學者)**
* **工具網址：** https://bids.neuroimaging.io/tools/validator.html
* **說明：** 直接在瀏覽器開啟。此工具僅解析「檔名」，絕對不會上傳影像檔案。

**選項 B：命令列檢核工具 CLI (適合進階與自動化)**
* **官方 Github：** https://github.com/bids-standard/bids-validator
* **優勢：** 直接在伺服器端對已上傳的資料集進行快速驗證，大幅提升效率。

**最終放行標準：** 確認驗證結果顯示 **"0 Errors"** 後再啟動運算管線。
