# Swift - Concurrency 學習筆記

## Grand Central Dispatch (GCD)

### DispatchQueue 基礎
DispatchQueue 是 Swift 的 Grand Central Dispatch (GCD)

### 主要 Queue 類型

#### Global Queue (並行)
```swift
DispatchQueue.global(qos: .background).async {
    // 執行背景任務
}
```
- 內建的並行 Queue (Concurrent)
- 適合避免同時派送多個請求，以免對接收端造成太大壓力
- 適合不用照順序、長時間運行的工作：下載、影像處理等

#### 自訂 Queue
```swift
DispatchQueue(label: "MyCustomQueue", qos: .userInitiated).async {
    // 執行長時間任務
}
```
```swift
// 利用 Queue Serial的特性，避免 Data race，但容易造成 Thread explosion
let queue: DispatchQueue = DispatchQueue(label: "queue")

API.checkPermission(completion: { allowed in
    if allowed {
        // 加入排隊
        queue.async {
            store.increment(1000)
        }
    }
})


API.checkPermission(completion: { allowed in
    if allowed {
        // 加入排隊
        queue.async {
            store.increment(1000)
        }
    }
})}
```

- 可為序列 (Serial)(預設) 或並行 (Concurrent)
- 適合處理多個照順序運行的工作

#### Main Queue
```swift
DispatchQueue.main.async {
    // 更新 UI
}
```
- 與更新 UI 相關
- 預設的 QoS 為 `.userInteractive`

### QoS (Quality of Service) 等級

```swift
.userInteractive: // 最高優先級，用於需要立即用戶互動的任務，如渲染動畫或響應用戶輸入

.userInitiated: // 用於用戶發起的任務，如打開文件或發起網路請求

.default: // 預設 QoS 等級，用於不需要更高優先級的任務，如背景計算或 I/O 操作

.utility: // 用於長時間運行且不緊急的任務，如索引文件或執行備份
```

#### 線程檢查
```swift
// 預防該 func 不會在指定 queue 中執行
dispatchPrecondition(condition: .notOnQueue(DispatchQueue.main))

// 檢查是否在該 queue 執行
dispatchPrecondition(condition: .onQueue(.main))
```

#### Task Sleep
```swift
// 讓 task 暫停一段時間且不會卡線程
Task.sleep(for:)
```

### 計時相關

#### ContinuousClock vs SuspendingClock
1. **ContinuousClock**: 即使系統觸發 sleep 也不會中斷，適合用來 monitor，像是監聽 func 執行時間、logging 時間
2. **SuspendingClock**: 與 ContinuousClock 相反，系統 sleep 則會暫停

```swift
import Foundation
let clock = ContinuousClock()
let startTime = clock.now()
// 執行任務，或等待；即使系統睡眠，時鐘也會繼續運行
// ...
let endTime = clock.now()
let elapsedDuration = endTime - startTime
print(elapsedDuration)
```

#### 暫停方法
```swift
let clock1 = ContinuousClock()
try? await clock1.sleep(for: .seconds(1))

let clock2 = SuspendingClock()
try? await clock2.sleep(for: .seconds(1))

// Concurrency
Task {
    try await Task.sleep(for: .seconds(1), clock: .suspending)
}

// Async
Task {
    try await Task.sleep(for: .seconds(1), clock: .continuous)
}
```

---

## Concurrency

### Task 基礎概念
- Task 功能是在同步的環境中，產生一個非同步的 closure 供使用
- Task 實體可用來與非同步工作互動，如取消
- 當你不再參照 Task 實體時，裡頭的非同步工作依然執行中

### Task 執行環境
- Task 會在調用它的 actor 執行
- 如果要建立不在該 actor 上的 task，可用 `Task.detached(priority:operation:)`
- 帶有特權的 Task，建立的 task 就不會繼承父層的優先權，但會對執行效率造成負面影響，不建議使用

### Task 儲存
```swift
// 如果要將 Task 儲存在變數內，因為成功不會回傳值，有可能噴錯
@State var downloadTask: Task<Void, Error>?
```

### AsyncSequence
- AsyncSequence 只定義取得 Element 的方式，本身並不產生 Element
- 是由裡頭的 Iterator 產生 Element

```swift
// Custom AsyncSequence
struct Counter: AsyncSequence {
    // Required
    typealias Element = Int
    
    // Required
    func makeAsyncIterator() -> AsyncIterator {
        return AsyncIterator(howHigh: howHigh)
    }

    private let howHigh: Int
    
    // custom AsyncIteratorProtocol
    struct AsyncIterator: AsyncIteratorProtocol {
        let howHigh: Int
        var current = 1

        mutating func next() async -> Int? {
            // A genuinely asynchronous implementation uses the `Task`
            // API to check for cancellation here and return early.
            guard current <= howHigh else {
                return nil
            }

            let result = current
            current += 1
            return result
        }
    }
}

// 呼叫
for await number in Counter(howHigh: 10) {
    print(number, terminator: " ")
}
// Prints "1 2 3 4 5 6 7 8 9 10 "
```

### Async/Await 優點
- 最大的好處：再也不用擔心 weak 或 strongly capture self
- 因為不再使用 escaping closures callback
- Async 的相關 class 大部分都可以 throws error，所以使用大部分都用 try catch
- 也有類似 Rx 或 Combine 的 modifier 可使用：[swift async algorithms](https://github.com/apple/swift-async-algorithms)

### Await 關鍵字
- 每一個 `await` 字眼出現，表示線程有可能改變
- 每個 `await` 都會透過系統來引導執行，該系統會對 task 優先排序、取消請求、或是往上回報錯誤

### 階層概念 (Hierarchy)
Swift 的 Concurrency 帶有子母 (Hierarchy) 概念：
- 允許母層 task 取消時，也取消所有子層 task
- 等待子層所有 task 都完成後，再完成母層 task
- 高等級 task 優先執行低等級 task

### 實際應用範例
```swift
func fetchSongs(for artist: String) async throws -> [MusicItem] {
    guard let url = URL(string: "https://itunes.apple.com/search?media=music&entity=song&term=\(artist)") else {
        fatalError()
    }
    
    let (data, response) = try await URLSession.shared.data(from: url)

    if !Task.isCancelled {
        guard let httpResponse = response as? HTTPURLResponse else {
            throw FetchError.urlResponse
        }
        guard (200..<300).contains(httpResponse.statusCode) else {
            throw FetchError.statusCode(httpResponse.statusCode)
        }
        
        let decoder = JSONDecoder()
        let mediaResponse = try decoder.decode(MediaResponse.self, from: data)
        return mediaResponse.results
    } else {
        print("task cancelled")
        return [MusicItem]()
    }
}
```

### Async Let
- `async let` 可以確保非同步時一定有值，概念類似第三方套件的 promise
- 只能用 await 取其中的值
- 使用 async let 時，後面的 try 不用接 await

```swift
// status() & availableFiles() 同時執行，並行執行
do {
    async let files = try model.availableFiles()
    async let status = try model.status()
    let (filesResult, statusResult) = try await (files, status)
} catch {
    lastErrorMessage = error.localizedDescription
}

// status() 會等到 availableFiles() 完成後才執行，線性執行
files = try await model.availableFiles()
status = try await model.status()
```

- `async let` 在宣告時就會觸發，不會等宣告 await 時才觸發

### @MainActor
- `@MainActor` 會強制切回主線程存取
- 若在非 @MainActor 或非主線程的區段，存取 @MainActor 的變數，就要用 async/await 的方式
- `@MainActor` 常用來搭配 `UIKit/SwiftUI`
### Actor
- `actor` 確保類型線程安全，一率在自己的`serial executor`下，確保來存取資源的人都能夠依序執行
- 以下面的片段為例，在 Actor model 的概念上，對 actor 來說，呼叫 increment 這個動作被稱為送一個 “message” 進去 actor，所以你不是直接呼叫 increment 這個 method，而是送一個 “message” 告訴 actor “你需要做 increment 這個動作”。actor 收到這樣的訊息就會開始做事，而送訊息進去的人就要在外面等它完成

```swift
// 因為外面世界是沒有辦法對 actor 做直接的存取，actor 裡面的資源 (properties, methods) 只能被自己人使用，也就是這些資源是被隔離 (isolated)的。
let store = BalanceStore()
store.increment(100) // Compilier Error

// actor 以外安全地呼叫 increment，我們可以利用 Task 產生一個獨立的環境，也就是一個新的 context （或 concurrency domain），被包在 closure 裡面的程式碼有它自己的排程，你可以在這個 closure 裡面運用 async/await，而不用擔心佔用目前的 thread。

let store = BalanceStore()
Task {
    await store.increment(100) // Correct
}
``` 
- 如果讀取的屬性本身是不能被修改的 ，那當然也不會有上面描述的非同步問題，所以你可以直接讀取這個資源。
```swift
actor BalanceStore {
    let accountName: String
}

let store = BalanceStore()
print(store.accountName)   // Okay!
``` 
- 非隔離 (non-isolated) 就表示你不需要準備 concurrency domain（Task {} 或 Task.detached {}，或 await) 也能夠隨意地讀取。隔離 (isolated) 指的就是這個東西是受到保護的，需要標注 await 才能夠安全地讀取。
```swift
actor BalanceStore {
    let accountName: String              // non-isolated
    var accountNickName: String          // isolated
}
``` 

```swift
@MainActor class MyViewModel {}

class MyViewModel {
    @MainActor var myParam = false
    
    // 主線程存取
    @MainActor func myViewModelFunc1() {
        myParam = true
    }
    
    // 在非主線程，使用 async/await 存取
    func myViewModelFunc2() async {
        let localParam = await myParam
    }
}

// 外部呼叫，假設在非主線程
func fetchData(vm: MyViewModel) async {
    // Update param on the main thread
    await vm.myParam = true
}
```
- [AppCoda: Swift 5.5 的新語法和機制 讓我們用最直觀的方式撰寫非同步程式](https://www.appcoda.com.tw/swift-concurrency-actor/)
### @TaskLocal
- `@TaskLocal` 實務上不常使用
- 用來宣告給 Task 當區域變數使用，只能宣告在 static 或 global 的變數

特點：
1. 獨立於同一個 Task 與子 Task 裡面，每個 Task 的 TaskLocal 變數，都是不同版本的存在
2. 封裝性，子 Task 可以覆寫母 Task 的 TaskLocal 變數
3. 資料不共享，不會有衝突或 race conditions 的狀況

```swift
struct MyContext {
    var userID: String
}

@TaskLocal // Must static or global
static var currentContext: MyContext?

func logMessage(_ message: String) {
    if let context = currentContext {
        print("\(context.userID):\(message)")
    } else {
        print(message)
    }
}

Task {
    currentContext = MyContext(userID: "12345")
    logMessage("This is a log message") // Prints: 12345: This is a log message

    // Child task inherits the context
    Task {
        logMessage("This is another log message") // Prints: 12345: This is another log message
    }
}

Task {
    // Not set currentContent
    logMessage("This message has no user context") // Prints: This message has no user context
}
```

### withCheckedContinuation
- `withCheckedContinuation(function:_:)` & `withCheckedThrowingContinuation(function:_:)` 用來介接 callback base 的 func 給 async/await 用
- 只會回傳一個結果，類似 RxSwift 的 `Single` 或 Combine 的 `Future`

```swift
func shareLocation() async throws -> String {
    let location = CLLocation(latitude: 37.785834, longitude: -122.406417)
    
    let address: String = try await withCheckedThrowingContinuation { continuation in
        // callback base API
        AddressEncoder.addressFor(location: location) { address, error in
            switch (address, error) {
            case (nil, let error?):
                continuation.resume(throwing: error)
            case (let address?, nil):
                continuation.resume(returning: address)
            case (nil, nil):
                continuation.resume(throwing: "Address encoding failed")
            case let (address?, error?):
                continuation.resume(returning: address)
                print(error)
            }
        }
    }
    return address
}

let myLocation = try await shareLocation()
```

### Task.sleep() vs Thread.sleep() 差別
1. **Concurrency Model**: `Task.sleep` 適用 Swift 並行機制 (`async/await`) 而且不卡線程；`Thread.sleep` 為舊版卡線程的一種方法
2. **Blocking Behavior**: `Task.sleep` 期間允許其他 Task 在相同的 thread 上執行；`Thread.sleep` 則否
3. **Cancellation**: `Task.sleep` 可以被取消而中斷，所以可能丟出 `CancellationError`，所以要用 `try`；`Thread.sleep` 則否

### Async Properties 與 Functions
```swift
// Computed Property 也可以用 async/await
var myProperty: String {
    get async { ... }
}

print(await myProperty)

// 參數若為 closure 也可以用 async
func myFunction(worker: (Int) async -> Int) -> Int {
    // ...
}

myFunction {
    return await computeNumbers($0)
}

// 可將 tuple 的值直接抓出來用
let (filesResult, statusResult) = try await (files, status)
self.files = filesResult
self.status = statusResult
```

### URL AsyncSequence
```swift
// URL 有一個內建 AsyncSequence 屬性 resourceBytes，以非同步的方式讀取檔案資料
let bytes = URL(fileURLWithPath: "myFile.txt").resourceBytes

for await character in bytes.characters {
    // ...
}

for await line in bytes.lines {
    // ...
}
```
