# ftxqz-future-arbitrage
FTX  Quant Zone Rule Set 量化空間的期現套利策略組

## 功能 ##
- 自動建倉現貨+永續合約
- 自動減倉現貨+永續合約
- 會在一定的點差值 (spread) 範圍內才會發出訂單
- 可設定每次建/減倉的量 (position batch size) 
- 建倉與減倉的點差值可分開設定

## 架構 ##
整個 FTX 量化空間策略組共有三個 rules：
1. SET CONSTANT - 設定各項參數
2. OPEN - 會判斷在相對有利的價格時下訂單買現貨並做空
3. CLOSE - 會判斷在相對有利的價格時下訂單賣現貨並減倉空單

在建倉時，要啟用：
1. SET CONSTANT
2. OPEN

在減倉時要啟用：
1. SET CONSTANT
3. CLOSE

## 策略1 - SET CONSTANT ##
```
策略命名: SET CONSTANT
觸發邏輯: balance("USD") > 0
```
觸發邏輯的條件只是隨便設一個總是能觸發的邏輯，因為接下來要執行的動作就是設定參數而已。這裡的策略範例會用 LINK/USD 以及 LINK-PERP 為例子

接下來在 `執行邏輯` 要新增四個執行邏輯如下：
```
type: 設定變量
變量名稱: target_size
變量值： 2
```
`target size` 是用來設定要建倉的量有多少，假設要建倉 100 單位的 SOL 現貨以及 做空 100 單位 SOL 永續合約。那就設定變量值為 `100`，這邊設定是 `2` 
```
type: 設定變量
變量名稱: batch_size
變量值： 0.2
```
`batch_size` 是每次下單的量，這邊設定是 `0.2`  也就是總共要建 10 次才會到達 `target_size = 2` 
```
type: 設定變量
變量名稱: spread_on_open
變量值： 1.001
```
```
type: 設定變量
變量名稱: spread_on_close
變量值： 1.0012
```
這邊是在設定建倉 (open) 跟減倉 (close) 的點差範圍。
![image](https://user-images.githubusercontent.com/102121/115504072-ef20a980-a2a9-11eb-8a80-741828a89131.png)
![image](https://user-images.githubusercontent.com/102121/115504261-3dce4380-a2aa-11eb-8495-2f46c4c56e7d.png)


## 策略2 - OPEN ##
```
觸發邏輯 Trigger
position("LINK-PERP")  <= get_variable("target_size")  and 
balance("LINK") <= get_variable("target_size")  and
abs(balance("LINK") - position("LINK-PERP")) < (get_variable("batch_size")*0.3) and
bid_price("LINK-PERP") / offer_price("LINK/USD") > get_variable("spread_on_open") 
```

```
執行邏輯 Action 1
type : 下自定義訂單
限價委託 : 買入 : LINK/USD
訂單數量 : get_variable("batch_size")
限價 : offer_price("LINK/USD")
如果已經有一個委託存在 : 取消並下新訂單
[V] Cancel order if rule is no longer triggered
```
```
執行邏輯 Action 2
type : 下自定義訂單
限價委託 : 賣出 : LINK-PERP
訂單數量 : get_variable("batch_size")
限價 : bid_price("LINK-PERP") 
如果已經有一個委託存在 : 取消並下新訂單
[V] Cancel order if rule is no longer triggered
```
![image](https://user-images.githubusercontent.com/102121/115504504-a7e6e880-a2aa-11eb-9928-42ef54108508.png)
![image](https://user-images.githubusercontent.com/102121/115503996-db754300-a2a9-11eb-8535-ca995c3b5ede.png)


## 策略3 - CLOSE ##
```
觸發邏輯 Trigger
balance("LINK")>=get_variable("batch_size") and 
position("LINK-PERP")>=get_variable("batch_size") and
offer_price("LINK-PERP")/bid_price("LINK/USD") < get_variable("spread_on_close") 
```

```
執行邏輯 Action 1
type : 下自定義訂單
限價委託 : 賣出 : LINK/USD
訂單數量 : get_variable("batch_size")
限價 : bid_price("LINK/USD")
如果已經有一個委託存在 : 取消並下新訂單
[V] Cancel order if rule is no longer triggered
```
```
執行邏輯 Action 2
type : 下自定義訂單
限價委託 : 買入 : LINK-PERP
訂單數量 : get_variable("batch_size")
限價 : offer_price("LINK-PERP") 
[V] 僅減少
如果已經有一個委託存在 : 取消並下新訂單
[V] Cancel order if rule is no longer triggered
```
![image](https://user-images.githubusercontent.com/102121/115503893-b5e83980-a2a9-11eb-85a7-50b9986a7a68.png)
![image](https://user-images.githubusercontent.com/102121/115503914-bda7de00-a2a9-11eb-9de9-3f5d56ba148c.png)



