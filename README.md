# 113-2-DeepLearning-HW3-Shared-bicycle-demand-forecast
About 此為 113-2 工工所的深度學習的第三份作業，利用LSTM模型去預測一段時間的共享腳踏車需求

## Author：國立陽明交通大學 資訊管理與財務金融學系財務金融所碩一 313707043 翁智宏

本次是深度學習課程的第三份作業，利用 LSTM 模型去預測一段時間的共享腳踏車需求。

**Questions**

隨著共享交通逐漸普及，如何預測未來的租借需求成為智慧城市、營運管理的重要議題。透過精準的需求預測，營運方可以更有效地調度腳踏車、優化資源配置，減少閒置與缺車的問題，提升使用者體驗與營運效率。本作業以華盛頓特區的共享腳踏車數據 (Bike Sharing Dataset) 為基礎，
要我們利用時間序列預測模型（LSTM），進行短期需求預測。 

根據附件提供的共享租借腳踏車資料集，利用時間序列預測方法，預測 2012/11/1 ~ 2012/11/20 二十天的腳踏車租借量。本作業將著重於實作 LSTM 模型進行需求預測，並藉由參數調整、特徵選取來優化模型表現。

**Datasets**
train.csv：2011/01/01 ~ 2012/10/31 每小時腳踏車租借數據 

test.csv：2012/11/1 ~ 2012/11/20 每小時資料（不含租借量）

sample_submission.csv：作業繳交格式範例（預設為 0，請更新預測值後上傳kaggle） 

Readme.txt：相關參數說明供同學參考

注意：雖然 test 資料是以每小時（hourly）為單位進行預測，但在提交作業時，請將「同一天」的24小時預測結果加總，轉換為「每日（daily）預測值」後，再填寫到 sample_submission.csv 中。

---

## 實作流程與思路說明

### 模型架構設計

本次我採用 單向多層 LSTM模型 搭配全連接層進行時間序列的回歸預測，架構如下：

•	輸入維度：每筆樣本為 (24, F)，其中 24 為時間步長（代表過去 24 小時，1天），F 為特徵數量 (33個)

•	LSTM 模組：
  o	hidden_size = 64 
  o	num_layers = 2
  o	dropout = 0.4
  
•	輸出模組：
  o	最後時間步的隱藏狀態 → 接全連接層 → 輸出預測值（log1p(cnt)）
  o	使用 F.softplus() 確保預測值非負
  
•	損失函數：以 MSE為主，評估預測與實際租借量（cnt）間的距離

### Sliding Window 設計

為了建構時間序列預測輸入，我使用 固定長度滑動視窗（Sliding Window）設計方式：

•	每筆樣本由 過去 24 小時的連續資料 組成，視窗大小為 24

•	每個視窗的目標為當下時間點的租借數量（cnt），轉換為 log1p(cnt) 作為回歸目標

•	模型學習方式為：根據過去 N 小時的特徵序列，預測當下租借數

•	驗證與預測皆採用相同的滑窗邏輯，但推理時逐步往後遞推預測

### 輸入特徵選擇

本次實作初期共設計 33 個輸入特徵欄位，涵蓋原始資料欄位、週期性變數、以及手動設計的交互特徵。透過SHAP分析，將目標設為：

•	強化時間序列建模中對「週期性」與「交互影響」的掌握

•	增強模型對天氣、時間與使用情境變化的理解能力

原始欄位（氣象 + 類別）：

•	包含 temp, atemp, hum, windspeed 4 個連續欄位，以及 6 個類別欄（如 season, weathersit, weekday 等）做 one-hot 編碼

週期性欄位：

•	使用 hr_sin, hr_cos 表示一天中的小時週期（0~23）

•	mnth_sin, mnth_cos 表示月份週期性（1~12）

人工交互特徵：

•	hr_x_weekday：小時 × 星期幾（工作/休假節奏）

•	hr_x_workingday：小時 × 是否工作日

•	temp_x_hum、atemp_x_wind：氣象交互特徵，模擬體感天氣的組合因素

![Confusion matrix](Images/confusion_matrix.png)

### 預測流程說明

預測流程分為以下幾步：

1.	前處理：
   
o	針對 test 資料加上所有與訓練相同的時間與交互特徵

o	使用相同的 MinMaxScaler 對數值欄位正規化

o	進行 one-hot 編碼後補齊欄位，與訓練資料欄位完全對齊

3.	Sliding windows建立：
   
o	建構過去 24 小時的Sliding windows序列，視窗從訓練尾端接到 test 起點

o	每個時間步使用目前特徵組預測當下租借數（log1p(cnt)）

5.	推理與還原：
   
o	預測值為 log1p(cnt)，最後透過 expm1() 進行還原

o	逐步更新 test 資料，實現「自回歸預測」過程

7.	輸出結果：
   
o	將每筆 test 資料依據日期彙總，產出每日租借總量預測值

o	輸出為符合指定格式的 submission.csv。

---

## 參數比較與討論

### window size

| window size | Training Loss | Validation Loss |
|------------|----------------|------------------|
| 24 (一天)   | **0.3165**       | **0.2831**        |
| 48 (兩天)   | 0.3293       | 0.2878          |
| 72 (三天)   | 0.3234         | 0.2974        |
| 168 (一周)  | 0.3460 | 0.2833     |

### LSTM 層數 (基於 window size = 24)

| 層數 | Training Loss | Validation Loss |
|------|----------------|------------------|
| 2    | **0.3165**     | **0.2831**       |
| 3    | 0.3525         | 0.3028           |
| 4    | 0.3739         | 0.3104           |


### hidden_size 數量 (基於 window size = 24、Layer = 2)

| hidden_size | Training Loss | Validation Loss |
|-------------|----------------|------------------|
| 64          | **0.3165**     | **0.2831**       |
| 96          | 0.3352         | 0.2797           |
| 128         | 0.3401         | 0.2696           |


### hidden_size 數量 (基於 window size = 24、Layer = 3)

| hidden_size | Training Loss | Validation Loss |
|-------------|----------------|------------------|
| 64          | 0.3525         | 0.3028           |
| 96          | 0.3353         | 0.3010           |
| 128         | 0.2870         | **0.2803**       |

### 分析：

表一得知window_size = 24 表現最佳，能抓住一天的租借週期性，且訓練與驗證差距小。

表二，兩層 LSTM 足夠學到時序資訊，繼續疊層反而造成過度擬合。

表三，hidden_size = 128 拿到最低驗證 loss，但 64 結構更簡單且差距不大。

表四得知，若使用 3 層，必須配大 hidden_size 才能穩定，但效果仍不如 2 層。

---

## 其他有助於說明的分析(資料前分析)

![Data descripts](Images/data_descripts.png)

![Hour analysis](Images/hour_analysis.png)

![Weather analysis](Images/weather_analysis.png)

![Days analysis](Images/days_analysis.png)

![SHAP analysis](Images/SHAP_analysis.png)
