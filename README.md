# Bosch Production Line Performance, NO.74/top 6%
 [Bosch Production Line Performance](https://www.kaggle.com/c/bosch-production-line-performance )<br>
 結論 : 原始變數超過 4000 種，而我們的 Fitted model 只使用 50 個變數，
 即可達到 top 6% ，因此這 50 個變數是重要變數，對於提高良率方面，可以先從這些變數下手。
 
 # 緒論
 在現今的產業上，對於產品製造，大多都傾向自動化，因此我選擇該問題進行製程分析。 
 製程分析與我過去處理過的資料---銷售/庫存預測非常不同，90%以上都是NA，
 而在生產線上，這是合理的，我們必須找出哪個製程出錯，即重要變數。
 問題是，NA過多，一般統計方法 AIC/BIC/lasso 無法處理missing value，
 因此轉往使用 XGB 內建的 importance 找出重要變數。
 並參考kernel中[Daniel FG](https://www.kaggle.com/danielfg/xgboost-reg-linear-lb-0-485)的作法，
 進行 Feature Engineering 。
 
 # 資料介紹
 Bosch Production Line Performance 是關於生產線( 製程 )的分析，
 在自動化製造產品的過程中，可能由於設備老舊或人為疏失，導致不良品的產生，
 但是我們不可能單純利用人工檢驗哪個環節出錯，因為整個製程中超過 4000 道程序。
 因此，我們希望藉由製程分析，找出導致不良品產生的因素。
 
 由於我沒有相關製程經驗，也無實際接觸生產線，所以參考kernel中
 [Daniel FG](https://www.kaggle.com/danielfg/xgboost-reg-linear-lb-0-485)
 的 code ，並加入我的想法，主要重點在於 --- Feature Engineering 。

 ### 資料準備 
 Kaggle 所提供的資料，可以分為以下六種 :
 
|data|size|n (資料筆數)|p (變數數量)|
|----|----|-----------|-----------|
|train_numeric|2.1GB|100萬筆|970個|
|train_date|2.9GB|100萬筆|1157個|
|train_categorical|2.7GB|100萬筆|2141個|
|test_numeric|2.1GB|100萬筆|969個|
|test_date|2.9GB|100萬筆|1157個|
|test_categorical|2.7GB|100萬筆|2141個|

主要變數如下 :

|變數名稱|意義|
|-------|----|
|Response|目標值, 0 : 良品, 1 : 不良品|
|Id|產品代號|
|Lx_Sx_Fx|L : line，S : station，F : feature number|

舉例來說，L3_S36_F3939，代表第 3 條生產線上，第 36 個設備中的第 3939 個特徵值，
而 numeric 中代表的是，產品在該設備中收集到的值， date 代表產品經過該設備的時間點 ，
我們並沒有使用 categorical，因此並不清楚該變數意義。

由於變數過多，如何找到重要變的方法，將在稍後的章節中提到。
該問題的 evaluation 是 [MCC](https://en.wikipedia.org/wiki/Matthews_correlation_coefficient) 。

# 特徵製造
### feature engineering 1 ( 特徵工程 1 )

在生產線上，可能在某一時段機器故障，導致產品出現問題，所以對 date data 進行特徵工程。
 
|feature|解釋|
|-------|---|
|first|進入該製成時間點|
|min|製成時間點最小值|
|last|離開該製成時間點|
|max|製成時間點最大值|
|class.amount|該製成時間點種類數量|
|na.amount|na數量，如果時間點完全記錄，那該變數代表未經過製程的數量|

以上分別對 所有生產線、L0、L1、L2、L3 進行特徵工程，製造特徵變數。
ex : all_first, L0_first, L1_first, L2_first, L3_first <br>
由於 data 過大，進行分段處理，每次讀取 100 萬筆資料，製造 feature ，
在 feature engineering 1 階段， kaggle rank 約在 50% ，
結果不夠好，因此將進行，feature engineering 2。

### feature engineering 2 ( 特徵工程 2 )

在生產線上，同一時間製造多個產品，它們的表現可能有高度相關，因此進行以下特徵工程。<br>
注意：first[is.na(first)] = 0，否則對其他的 feature 製造會產生na。

|feature|解釋|code|
|-------|----|----|
|next|下一個產品是否為同時製造的產品||
|prev|上一個產品是否為同時製造的產品||
|total|total|next+prev|
|same.time|是否同時製造|total>0|
|order.same.time|在同時製造的產品中，該產品是第幾個|(cumsum(prev)+1) * same.time|
|group|第幾群同時製造的產品，同時製造代表可能有相同表現||
|group.length|該群數量|table(group)|
|cost.time|該製程耗時，耗時過長或過短，可能是因為產品出問題|max-min|
|pcost.time|與上一個產品相比，製程耗時差距，差距過大，可能是因為產品出問題|cost.time-c(NA,cost.time[1:length(cost.time)-1])|
|ncost.time|與下一個產品相比，製程耗時差距，差距過大，可能是因為產品出問題|cost.time-c(cost.time[2:length(cost.time)],NA)|
|pna.amount|與上一個產品相比，na數量，差距過大，可能是因為產品出問題|na.amount--c(NA,na.amount[1:length(na.amount)-1])|
|nna.amount|與下一個產品相比，na數量，差距過大，可能是因為產品出問題|na.amount--c(na.amount[2:length(na.amount)],NA)|
|prev.target|上一個產品表現，彼此間可能相關|c(NA,target[1:(nrow(target)-1)])|
|next.target|下一個產品表現，彼此間可能相關|c(target[2:nrow(target)],NA)|

以上變數均對 所有生產線 進行特徵工程，並沒有對 L0~L3 進行特徵工程。

### 變數選擇

由於原始資料變數過多，資料龐大，不易建模，而在實際製程方面，
出問題的設備只佔極少數，因此，
我們對於 train_numeric 中的所有變數，進行變數選擇。

由於每個產品經過的製程不同，某些製程可能導致不良品產生，
因此我們對於所有製程，計算 不良品 比率，並進行排序，
選擇該比例中，前 100 個變數與後 100 個變數，作為特徵變數。

前 100 代表產出不良品比率高，有助於預測不良品，
而後 100 代表產出良品比例高，有助於預測良品。
實際上，不良品佔所有產品中約 0.058，
而不良品比率最高的設備，其不良品佔所有產品中約 0.045，
設備 ID 為 L3_S32_F3850。

### feature selection
藉由 feature engineering 1、feature engineering 2 與變數選擇，
製造約 450 個變數。我們利用這些變數進行 XGBoost 建模，
主要利用 xgb.cv 找出 bset nrounds ， 並利用 bset nrounds 在進行建模，
在模型建立後，
使用 xgb.importance 函數找出前 50 個重要變數，
選擇這 50 個變數作為 fitted model 的 feature。

其中一點需要注意的是， xgb.cv 的 bset nrounds 並不代表最好的 nrounds，
我們藉由觀察 xgb.cv ，調整最後的 nrounds 。

# Fitted model
該問題是有關二元分類問題，unblance 問題非常嚴重，
而且此問題的 evaluation --- MCC 並沒有在 XGBoost 的 evaluation 中，因此我們進行以下處理：

1. 使用 XGBoost 內建的 rmse 逼近 MCC，rmse 是常見的準則























