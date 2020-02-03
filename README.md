# laravel 三層式架構 開發規範
 + 提供一些開發規範，讓大家在各個功能撰寫使用上可以有個區隔的判斷
 + 此規範以可維護性，與可讀性為基礎構思設計
 + 可以依專案需求進行變形
 + [hackmd](https://hackmd.io/OrQOCsQWRp-emBgPYGXF8g?both)

## 三層式架構
+ 商業邏輯層 BLL ( Business Logic Layer )
    將商業邏輯或是商業流程放置Service

+ 資料存取層 DAL ( Data Access Layer )
    將查詢與DB操作單獨抽出來，放到Repository
    
+ 表現層 USL ( User Show Layer 或 UI 或 Presentation layer)
    將外觀顯示邏輯放置Presenter

## 開發原則
+ 職責分離 (Segregation of Duties, SOD)
    職責分離是指遵循不相容職責相分離的原則，實現合理的組織分工。
    例如一個公司的授權、簽發、核准、執行、記錄工作，不應該由一個人擔任。

+ 關注點分離 (Separation of Concerns, SOC)
    就是將交錯著各種目的的複雜程式碼，依程式碼的概念、目的，進行分類、篩檢、整理。讓高複雜性程式碼切割、轉換為簡潔易懂的單純性程式碼。

## 架構圖
![](https://i.imgur.com/e4IO4WV.png)

## 開發重點與規範
+ 主要針對Controller、Service、Repository與Presenter比較容易有爭議的部分進行規範
+ 其餘部分Route、Middleware、View、Model等，按照原先系統規劃的方式撰寫即可

### ***Controllers***
+ **接收 HTTP request，調用其他 service**

#### 資料預處理
+ request資料處理，request不能直接帶入service
+ service有使用session也請這在處理帶入service
```
    $a = $request["a"];
    $b = $request["b"];
    $c = $request["c"];
    $d = $session->get['a'];
```

#### 清楚的service傳入參數
+ 決定使用哪些service，並將處理過的資料帶入，只能將會使用的資料帶入
+ 不能帶入request
```
    $service->onefunction(
        $a,
        $b,
        $c
    );
```

#### 回傳類別
+ response
    + 定義return訊息
        + 不直接回傳service return的data
        + 決定return的格式，與訊息
        ```
            return response()->json([
                'status' => false,
                'data' => [
                    'a' => '範例',
                    'b' => [
                        'type' => 'good',
                        'text' => 'ABC'
                    ],
                    'c' => ['c1' => true]
                ]
            ]);
        ```
+ view
    + 決定view與帶入資料
        + 指定輸出的view，資料帶入部分，不同service回傳資料，使用不同變數名稱帶入view
        ```
            return view('view.view',[
                'data1' => $serviceResult1,
                'data2' => $serviceResult2
            ]);
        ```

### ***Service***
  + **輔助 controller，處理商業邏輯，然後注入到 controller**

#### 傳入資料項目明確
 + 請明確定義有哪些資料會使用，不能傳入request不明確內容的變數
```
    public function onefunction(
        $a,
        $b,
        $c = null
    ){
        $d = $a + $b;
        if($c != null)
            $d = $d - $c;
        return $d;
    }
```

#### service事件盡量單一功能化
+ 事件單一，如果有複雜事件，拆成多個service
+ 單一職責
+ 盡量採取物件導向開發原則
```
舉例
    預約訂單功能:
        1.檢查訂單
        2.新增使用者(新使用者)
        3.新增訂單
        4.發信給管理者
        5.完成預約

    public function 新增預約訂單(
        $user,
        $order
    ){
        
        $user_id = $this->新增使用者($user);
        
        $order_id = $this->新增訂單(
            $user_id, 
            $order
        );
        
        if($order_id){
            $state = $this->發信給管理者($order_id);
            if($state)
                return 成功;
            else
                return 失敗; 
        }
    }
    
    public function 新增使用者($user){
        $id = $userRepository->selectUser($user);
        if($id){
            $id = $userRepository->creatUser($user);
        }
        return $id;
    }
    
    public function 新增訂單(
        $user_id,
        $order
    ){
        $order_id = $orderRepository->creatOrder(
            $user_id,
            $order
        );
        return $order_id;
    
    }
    
    public function 發信給管理者($order_id){
        $data = $orderRepository->selectOrder($order_id);
        $eamilService->sendMail($data);
        return true;
    }
    
```

#### 固定參數使用config or env
+ 防止太多地方要使用同一參數，修改時遺漏修改
+ 讓程式保持一定程度的修改彈性
+ 不同機器，請用env，例如:DB連線資訊、發信信箱資訊
+ 不同環境情況時，請用config，例如:預設語系、預設時區

#### 不直接使用DB
+ 避免閱讀混亂
+ DB指令的集中管理
+ 在簡單的DB指令也要寫Repository

#### 不用global變數
+ 避免service或者其他功能相互影響
+ 方便抓錯與閱讀理解程式
+ 如有需要共用變數請善用session，且請於controller帶入

#### function 內變數獨立
+ 增加功能的可讀性
+ 修改可以保證不會影響其他function功能
+ 方便進行遷移或移轉至其他的service使用

#### 註解請寫在該行程式碼上方，並且與程式縮排相同
+ 加速程式閱讀，快速理解該行為模式
+ 請說明該步驟是要做什麼功能或效果
```
    註解範例
    
        //解查XXX後，新增訂單
        public function orderNew(
            $user,
            $order
        ){
            //判斷並新增用戶
            $this->userNew($user);
            //新增訂單
            $state = $this->新增訂單(
                $user, 
                $order
            );
            if($state){
                //發信給管理者
                $this->postEmail($order);
                //發信成功回傳成果
                return 成功;
            }
            //訂單新增失敗
            return 失敗;
        }
```

### ***Repository***
+ **輔助 model，處理資料庫邏輯，然後注入到 service**

#### 不能丟陣列進insert或update直接跑DB
+ 直接使用array新增很方便，但是不可讀
+ 可以保持SQL指令的乾淨，以及修改性
+ 同一個table，如果有多種insert的情況，可以參考下列作法
    + 把重要的變數往前擺放，其餘使用funcction預設值帶入，如舉例1
    + 將insert分成不同的function，如舉例2
    ```
    舉例1

        //有三筆資料
        $Repository->selectDB(
            1,
            2,
            3
        );
        
        //有四筆資料
        $Repository->selectDB(
            1,
            2,
            3,
            4
        );
        
        //有五筆資料
        $Repository->selectDB(
            1,
            2,
            3,
            4,
            5
        );
        
        
        public function selectDB(
            $a,
            $b,
            $c,
            $d = 0,
            $e = null
        )
        {
            $data = Company::where('id', function ($query) use ($a) {
                            $query->select('XXX')
                                    ->from('XXX')
                                    ->where('id', $a);
                        })
                        ->first();
    
            return $data;
        }
    ```

    ```
    舉例2
    
        //只有三筆資料
        $Repository->selectDB1(
            1,
            2,
            3
        );
        
        //只有五筆資料
        $Repository->selectDB2(
            1,
            2,
            3,
            4,
            5
        );
        
        
        public function selectDB1(
            $a,
            $b,
            $c
        )
        {
            $data = Company::where('id', function ($query) use ($a) {
                            $query->select('XXX')
                                    ->from('XXX')
                                    ->where('id', $a);
                        })
                        ->first();
    
            return $data;
        }
        
        public function selectDB2(
            $a,
            $b,
            $c,
            $d,
            $e
        )
        {
            $data = Company::where('id', function ($query) use ($a) {
                            $query->select('XXX')
                                    ->from('XXX')
                                    ->where('id', $a);
                        })
                        ->first();
    
            return $data;
        }
    ```

#### 不呼叫其他的Repository function
+ 增加可更動性，Repository function之間不互相干擾
+ 如果需要呼叫其他DB，請用join或service多步驟

#### 盡量不使用if else
+ 讓DB指令單一化，清楚知道該SQL指令的行為
+ 商業行為的條件判斷應該主要給service進行
+ 不影響輸出結果或不涉及商業邏輯，可以使用if else
    + 資料排序
    + 欄位有無

### ***Presenter***
+ **處理顯示邏輯，然後注入到view**
+ **這盡量少使用，如果view的切版有切好，應該不會增加太多可讀性，幫助不大(個人怨念)**

#### 不能連結任何後台的程式
+ 所有邏輯使用，應該於service處理完成

#### 變數只能使用view傳進來
+ 所有參數，應該於controller獲得
+ 在Controllers取值，一併帶入view

#### 不涉及JS行為操作
+ JS不應該使用Presenter內部定義的name或id
+ 嚴重增加可讀性跟Debug困難
+ 如果需要使用JS，可以使用以下方式
    + name或id是由view帶入Presenter，JS在取用時閱讀上直接在blade上可以找到
    + 將JS寫包含在Presenter的輸出，一併輸出至view

#### Presenter只能回傳html或html的內容，不能是參數
+ 資料單向性
+ 程式閱讀上為blade去找Presenter，Presenter不可以回到blade進行其餘操作

#### 不判斷blade區塊顯示與否
+ 不應該利用Presenter的回傳參數決定blade的顯示區塊
+ Presenter為單向由blade傳入，在由Presenter決定或判斷顯示區塊的內容

#### function互相不影響
+ 可讀性
+ 重複使用性
+ 移動性
