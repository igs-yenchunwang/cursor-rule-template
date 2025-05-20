# Go 程式碼風格規範

## 目錄結構

最小Topic範例

```
example/
├── Base/                      # 基礎定義資料夾 
│   ├── Define.go 			  # 在此定義常數、介面、型別 (只能引用內層的Base/Define 不能引用上層或平級的go檔案
│   └── Var.go 			  	  # 在此放此topic下共用的變數
│
├── Router.go				  # 該主題的程式碼實現
├── Handler.go				  # 該主題的程式碼實現
├── GlobalVar.go			  # 對外公開的變數
└── GlobalFunc.go  			   # 對外公開函數

```

Topic下的子Topic範例

```
AIDBTool/
├── Base/                      # 基礎定義
│   └── Define.go              # 常數、介面、型別定義
│
├── Config/                    # (設定主題) 設定檔和配置相關
│   └── Config.go              # 配置實作
│
├── Mgrs/                      # (main Topic) 
│   └── AIMgr/                 # AI 管理器    (sub Topic)
│       ├── Base/              # AI 管理器的基礎定義
│       │   └── Define.go      # AI 相關常數、介面、型別定義
│       │
│       ├── GPTMgr/            # GPT 管理器   (sub topic's sub topic)
│       │   ├── Base/          # GPT 管理器的基礎定義
│       │   │	├── Define.go  # GPTMgr 相關常數、介面、型別定義
│       │   │	└── Var.go     # GPTMgr 相關變數
│       │   ├── GPTMgr.go      # GPT 管理器主要實作
│       │   └── GlobalFunc.go  # 對外公開函數
│       │
│       ├── LangChainMgr/      # LangChain 管理器
│       │   ├── Base/          # LangChain 管理器的基礎定義
│       │   └── LangChainMgr.go # LangChain 管理器主要實作
│       │
│       └── AIMgr.go           # AI 管理器主要實作
│
└── Server/                     # 伺服器相關程式碼
    ├── Server.go              # 伺服器主要實作
    ├── Router.go              # 路由處理
    ├── DBHelper.go            # 資料庫輔助函數
    └── StringHelper.go        # 字串處理輔助函數
```

## 命名規則

### 命名規則

- **全域變數**：以 `G_` 開頭

  ```go
  var G_LogDB *gorm.DB
  ```
- **常數**：以 `C_` 開頭

  ```go
  const C_DefaultTimeout = 30
  ```
- **成員變數**：以 `m_` 開頭

  ```go
  type Client struct {
      m_Connection *Connection
      m_Timeout    int
  }
  ```
- **receiver**：統一使用 `sel`

  ```go
  func (sel *Client) Connect() error
  ```
- **型別**：以 `T` 開頭

```go
  type T_StringMap map[string]string
```

- **建構函數**：以 `New` 開頭

  ```go
  func NewServer() *Server
  ```
- **介面**：以 `I_` 開頭

  ```go
  type I_AITopic interface {
      SendMessage(clearLastMsg bool, firstInfo string, input string) (error, string)
  }
  ```
- **其他規範**：

  首字母大小寫以是否要對外公開決定

  Define.go引用的方向為外層可以引用內層；內層不能引用外層

## 最佳實踐

### ======== 避免使用魔術數字 (Magic Numbers) ========

避免在程式碼中直接使用沒有明確含義的數字常量。這類「魔術數字」會使程式碼難以理解和維護。

#### 不良示範

```go
if status == 1 {
    // 啟動伺服器
}

timeout := 30 * 24 * 60 * 60 // 有效期限
db.SetConnMaxLifetime(300 * time.Second) // 資料庫連接超時
```

#### 良好示範

```go
// 使用 iota 建立列舉常量
const (
    C_StatusStopped = iota
    C_StatusRunning
    C_StatusPaused
    C_StatusError
)

// 時間常數定義清晰且有說明
const (
    C_OneMonth = 30 * 24 * 60 * 60 // 以秒為單位
    C_DBConnMaxLifetime = 300      // 資料庫連接最大生命週期，單位：秒
)

if status == C_StatusRunning {
    // 啟動伺服器
}

// 使用具名常數提高可讀性
db.SetConnMaxLifetime(time.Duration(C_DBConnMaxLifetime) * time.Second)
```

#### 規則說明

- 將具有特定含義的數字定義為常數，並賦予有意義的名稱
- 對於複雜計算的數字，提供清晰的註解說明其含義
- 使用常數組（`const`）來組織相關的常數值
- 使用 `iota` 處理狀態列舉，避免手動賦值可能導致的錯誤和維護問題

### ======== 處理型別斷言失敗 ========

型別斷言的單一回傳值形式在型別不正確時會引發 panic。因此，務必使用「逗號 ok」慣用法。

#### 不良示範

```go
t := i.(string)
```

#### 良好示範

```go
t, ok := i.(string)
if !ok {
    // 優雅地處理錯誤
}
```

#### 規則說明

- 總是使用兩個回傳值的型別斷言形式
- 明確處理型別斷言失敗的情況
- 避免在生產環境中發生不預期的 panic

### ======== 減少嵌套 ========

**減少嵌套**：通過提前返回來減少程式碼的嵌套層級，提高可讀性。

#### 不良示範

```go
func processData(data []int) (int, error) {
    if len(data) != 0 {
        if data[0] > 0 {
            sum := 0
            for i := 0; i < len(data); i++ {
                if data[i] > 0 {
                    sum += data[i]
                } else {
                    return 0, errors.New("發現負數")
                }
            }
            return sum, nil
        } else {
            return 0, errors.New("第一個元素不是正數")
        }
    } else {
        return 0, errors.New("空的數據集")
    }
}
```

#### 良好示範

```go
func processData(data []int) (int, error) {
    if len(data) == 0 {
        return 0, errors.New("空的數據集")
    }
  
    if data[0] <= 0 {
        return 0, errors.New("第一個元素不是正數")
    }
  
    sum := 0
    for _, val := range data {
        if val <= 0 {
            return 0, errors.New("發現負數")
        }
        sum += val
    }
  
    return sum, nil
}
```

#### 規則說明

- 特殊情況提前判斷並return
- 避免不必要的 else 區塊
- 減少縮排提高程式碼可讀性

### ======== 指定容器容量 ========

在創建Slice和Map時預先指定容量，避免添加元素時進行多次記憶體重新分配。

#### 不良示範

```go
func createUserIDs(users []*User) []string {
    ids := []string{}
    for _, user := range users {
        ids = append(ids, user.ID)  // 每次 append 可能導致重新分配記憶體
    }
    return ids
}

// 建立大型映射但未預分配空間
func buildLookupTable(values []string) map[string]bool {
    lookupTable := map[string]bool{} // 在添加元素時會多次擴容
    for _, v := range values {
        lookupTable[v] = true
    }
    return lookupTable
}
```

#### 良好示範

```go
func createUserIDs(users []*User) []string {
    ids := make([]string, 0, len(users)) // 預先分配足夠容量
    for _, user := range users {
        ids = append(ids, user.ID)  // 不需要重新分配內存
    }
    return ids
}

// 預分配合適容量的映射
func buildLookupTable(values []string) map[string]bool {
    lookupTable := make(map[string]bool, len(values)) // 預分配空間
    for _, v := range values {
        lookupTable[v] = true
    }
    return lookupTable
}
```

#### 規則說明

- 若提前知道大約的元素量，使用 `make()` 函數指定容器初始容量
  - Slice語法為 `make([]Type, 0, capacity)`
  - map語法為 `make(map[KeyType]ValueType, capacity)`

### ======== 使用 init() 的注意事項 ========

使用 `init()` 函數時，需特別留意下列重點。`init()` 函數執行時應該：具有完全確定性、不依賴其他 `init()` 執行順序、避免 I/O 操作。

#### 不良示範

```go
func init() {
    // 從環境變數讀取設定
    os.Getenv("APP_ENV")
  
    // 建立資料庫連線（不良做法：無法處理錯誤、無法控制連線生命週期、難以進行單元測試）
    db, err := sql.Open("mysql", os.Getenv("DB_CONNECTION"))
    if err != nil {
        log.Fatal(err)
    }
  
    // 設定全域變數
    G_Config = loadConfig()
    G_LogDB = db
}
```

#### 良好示範

```go
// 範例：設定靜態路由
import "net/http"

func init() {
    // 設定靜態路由，此為確定性操作
    http.HandleFunc("/healthcheck", healthCheckHandler)
    http.HandleFunc("/ping", pingHandler)
  
    // 靜態資源處理
    staticFiles := http.FileServer(http.Dir("static"))
    http.Handle("/static/", http.StripPrefix("/static/", staticFiles))
}

func healthCheckHandler(w http.ResponseWriter, r *http.Request) {
    w.Write([]byte("OK"))
}

func pingHandler(w http.ResponseWriter, r *http.Request) {
    w.Write([]byte("pong"))
}
```

#### 規則說明

- 使用 `init()` 時應限於確定性操作和無外部依賴的情境，
- 避免執行需要錯誤處理或受環境影響的操作（如讀取環境變數、建立資料庫連線等），這類操作應改為明確呼叫的函式

### ======== Struct中操作 Slice 和 Map 的注意事項 ========

Slice 和 Map 都包含指向底層資料的指標，在函式的參數傳遞或回傳值時需特別小心，以避免非預期的資料變動。

#### 不良示範

```go
// 直接儲存接收到的 slice，容易被外部修改影響
type T_Driver struct {
    m_ID    string
    m_Trips []T_Trip
}

func (sel *T_Driver) SetTrips(trips []T_Trip) {
    sel.m_Trips = trips    // 直接儲存引用，外部修改會影響內部狀態
}





// 直接回傳內部 map，可能讓內部狀態被外部意外修改
type T_Stats struct {
    m_Mutex    sync.Mutex
    m_Counters map[string]int
}

func (sel *T_Stats) GetCounters() map[string]int {
    sel.m_Mutex.Lock()
    defer sel.m_Mutex.Unlock()
  
    return sel.m_Counters  // 回傳內部 map 引用，外部可直接修改內部狀態
}
```

#### 良好示範

```go
// 複製接收到的 slice 以防止外部修改影響內部狀態
type T_Driver struct {
    m_ID    string
    m_Trips []T_Trip
}

func (sel *T_Driver) SetTrips(trips []T_Trip) {
    // 建立完整複本
    sel.m_Trips = make([]T_Trip, len(trips))
    deepCopyTrips(sel.m_Trips, trips)   // deepCopyTrips為自己實作的深拷貝
}




// 回傳 map 的複本以保護內部狀態
type T_Stats struct {
    m_Mutex    sync.Mutex
    m_Counters map[string]int
}

func (sel *T_Stats) GetCounters() map[string]int {
    sel.m_Mutex.Lock()
    defer sel.m_Mutex.Unlock()
  
    // 建立並回傳複本
    result := make(map[string]int, len(sel.m_Counters))
    for k, v := range sel.m_Counters {
        result[k] = v
    }
    return result
}
```

#### 規則說明

- 當函式需要保存傳入的 slice 或 map 時，應建立複本以防止受外部修改影響
- 當函式需要回傳內部的 slice 或 map 時，應回傳複本以保護內部狀態
- 注意 slice 和 map 中可能存在的巢狀引用型別，必要時進行深層複製

### ======== 使用 for range 操作 Slice 和 Map 的注意事項 ========

在 Go 中使用 for range 迭代 slice 和 map 時，需要注意迭代變數的重用、集合修改帶來的影響以及迭代特性等問題。

#### 不良示範

```go
// 問題1: 在for range中使用迭代變數
func processList(items []string) []*string {
    var results []*string
    for _, item := range items {
        results = append(results, &item)  // item 在每次迭代被重用，所有指標指向同一個地址
    }
    return results  // 所有結果指向相同的值（最後一個元素）
    
    // 例如: 對於輸入 []string{"a", "b", "c"}
    // 預期輸出: 指向 "a", "b", "c" 的指標陣列
    // 實際輸出: 全部指向 "c" 的指標陣列 ["c", "c", "c"]
}

// 問題2: 在迴圈中修改 slice 可能的錯誤
func processSlice() {
    // 例1: 在遍歷時刪除元素的錯誤做法
    numbers := []int{1, 2, 3, 4, 5, 6}
    for i, num := range numbers {
        if num%2 == 0 { // 刪除偶數
            numbers = append(numbers[:i], numbers[i+1:]...) // 刪除元素後，索引位置改變，可能跳過元素或造成越界錯誤
        }
    }
    
    // 例2: 在遍歷時追加元素的錯誤做法
    items := []string{"a", "b", "c"}
    for i, item := range items {
        if item == "b" {
            items = append(items, item+"_new") // 可能造成非預期的結果
        }
    }
}

// 問題3: map 迭代中依賴順序
func printTopThree(scores map[string]int) {
    count := 0
    for name, score := range scores {
        fmt.Printf("%s: %d\n", name, score)
        count++
        if count == 3 {
            break  // 假設這裡想取前三名，但 map 的迭代順序不固定
        }
    }
}
```

#### 良好示範

```go
// 解決問題1: 正確處理迭代變數
func processList(items []string) []*string {
    results := make([]*string, 0, len(items))
    
    // 方法一: 創建局部變數複製
    for _, item := range items {
        itemCopy := item           // 每次創建新的局部變數
        results = append(results, &itemCopy)
    }
    
    // 方法二: 直接使用索引獲取地址
    results = make([]*string, len(items))
    for i := range items {
        results[i] = &items[i]     // 直接使用原始 slice 中元素的地址
    }
    
    return results
}

// 解決問題2: 安全地在循環中修改 slice
func processSliceSafely() {
    // 例1: 安全地過濾元素 - 使用新的slice收集結果
    numbers := []int{1, 2, 3, 4, 5, 6}
    
    // 方法一: 創建新的slice保存需要的元素
    result := make([]int, 0, len(numbers))
    for _, num := range numbers {
        if num%2 != 0 { // 只保留奇數
            result = append(result, num)
        }
    }
    // 現在 result 中只有奇數: [1, 3, 5]
    
    // 方法二: 收集需要添加的元素，在遍歷結束後一次性添加
    source := []int{1, 2, 3}
    var toAdd []int
    
    for _, v := range source {
        if v%2 != 0 { // 對於奇數，添加其平方
            toAdd = append(toAdd, v*v)
        }
    }
    // 遍歷完成後一次性添加新元素
    source = append(source, toAdd...)
    // 結果: source = [1, 2, 3, 1, 9]
}

// 解決問題3: 不依賴 map 迭代順序 (若map元素不大)
func printTopThree(scores map[string]int) {
    // 將 map 鍵收集到 slice
    names := make([]string, 0, len(scores))
    for name := range scores {
        names = append(names, name)
    }
    
    // 根據分數排序
    sort.Slice(names, func(i, j int) bool {
        return scores[names[i]] > scores[names[j]]  // 降序排列
    })
    
    // 使用排序後的 slice 按順序訪問 map
    for i := 0; i < len(names) && i < 3; i++ {
        name := names[i]
        fmt.Printf("%s: %d\n", name, scores[name])
    }
}
```

#### 規則說明

- **迭代變數重用**: `for range` 使用同一變數進行每次迭代，若要保存指標應該複製該值或直接使用索引取原始元素的地址
- **集合修改**: 在迭代過程中修改正在迭代的集合可能導致非預期行為
  - 如需修改，優先考慮使用索引迭代而非 `range`
  - 或在迭代前複製原始集合，在副本上進行修改
- **Map 迭代順序**: Go 中 map 的迭代順序是隨機的，且可能在每次執行時變化
  - 若需依賴順序，應先將鍵收集到 slice 中並進行排序
  - 然後使用排序後的 slice 按需求順序訪問 map 元素