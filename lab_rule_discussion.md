# 實驗室 fMRIPrep 運算環境架構與資料規範說明會議

**會議目的：** 與大家同步實驗室伺服器在執行 fMRIPrep 神經影像前處理流程時的標準作業模式，期望在確保系統底層安全性的同時，也能協助大家建立統一的資料格式，讓整體的運算流程更加順暢。

---

## 1. 容器化環境之安全考量與政策宣導

在查閱 fMRIPrep 官方文獻或網路教學時，大家最常看到的執行指令通常是 `fmriprep-docker`。然而，在實驗室多用戶共享的伺服器架構下，我們必須避免採用此種單機運作模式，以免帶來系統風險。

* **風險說明：** Docker Daemon 在 Linux 系統底層預設是以 `root` 權限持續運行的。若開放使用 `fmriprep-docker`，系統在架構上必須給予所有使用者較高的管理員權限。這不符合資訊安全的「最小權限原則」，若操作不慎，極易影響伺服器的整體穩定性與其他同學的運算作業。
* **業界規範參考：** 為了系統安全，國際大型運算中心通常會嚴格限制 Docker，並採用更安全的替代方案。正如 fMRIPrep 官方文件 (v0.5.1) 所述：
    > *"For security reasons, many HPCs (e.g., TACC) do not allow Docker containers, but do allow Singularity containers."*
    > （官方文件參考：https://fmriprep.org/en/0.5.1/installation.html ）

基於上述維運與安全考量，實驗室伺服器上將不開放透過 Docker 執行 fMRIPrep，請大家在查閱網路教學時特別留意此環境差異。

---

## 2. 實驗室標準方案：Apptainer (原 Singularity)

為了解決大家的運算需求並兼顧伺服器安全，實驗室全面採用 **Apptainer**（前身為 Singularity）作為標準的容器執行環境。這不僅是為了安全性，更能為我們的科學研究帶來實質好處：

### Apptainer 的架構與學術優勢：
1. **無特權執行 (Rootless Execution)：** 專為高效能運算 (HPC) 設計，不需背景 root daemon 即可運行，安全性極高，且會自動對應大家原本的使用者身份存取檔案。
2. **避免污染系統環境 (Environment Isolation)：** fMRIPrep 底層依賴 FSL, FreeSurfer, AFNI 等龐大且版本敏感的神經影像軟體。透過容器化，我們不會在伺服器全域安裝這些軟體，也不會引發 Python 套件的相容性衝突，保持伺服器環境的純淨。
3. **延續並確保實驗結果的絕對重現性 (Scientific Reproducibility)：** Apptainer 將運算環境封裝成單一不可變的 `.sif` 檔。這確保了各位的研究結果不會因為伺服器底層軟體的日後更新而產生偏差，保證了科學實驗的絕對可重現性。

### 使用 Apptainer 的注意事項與學習重點：
初次接觸 fMRIPrep 時，大家若參考網路上的 Docker 教學，會發現其指令相對簡短，因為 `fmriprep-docker` 是一個會自動處理路徑的包裝程式 (Wrapper)。在我們的 Apptainer 環境中，大家需要建立更扎實的系統路徑概念：

* **需手動設定掛載路徑 (Manual Bind Mounting)：** 大家需要學習使用 `--bind` 參數，手動將本機的 BIDS 資料夾、輸出目錄以及 FreeSurfer License 映射進容器中。指令雖然會比網路上看到的稍長，但能幫助大家更清晰地掌握資料流向。
* **映像檔的統一管理：** fMRIPrep 的 `.sif` 映像檔體積龐大（通常超過 10GB）。為避免佔用過多硬碟空間，實驗室將會在共用目錄統一放置一份映像檔供大家呼叫，請大家不要各自重複下載。

---

## 3. 原始資料 (Raw Data) 的 BIDS 結構化建議與轉換工具

fMRIPrep 本身被設計為一個 "BIDS App"，這代表它在架構上高度依賴且強制要求輸入的檔案結構必須符合 BIDS (Brain Imaging Data Structure) 規範。 

### BIDS 目錄結構概覽：
一個標準的 BIDS 資料夾，除了將 DICOM 轉換為 NIfTI (`.nii.gz`) 格式外，還必須具備嚴謹的階層命名與對應的 JSON Metadata 檔案。其基礎架構大致如下：

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
