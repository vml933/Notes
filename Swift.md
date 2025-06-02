# Swift 學習筆記

## 目錄
- [集合類型 (Collections)](#集合類型-collections)
- [基礎語法與概念](#基礎語法與概念)
- [陣列相關](#陣列相關)
- [錯誤處理](#錯誤處理)
- [UI 相關](#ui-相關)
- [格式化工具](#格式化工具)
- [Grand Central Dispatch (GCD)](#grand-central-dispatch-gcd)
- [日誌記錄](#日誌記錄)
- [(Concurrency)](#concurrency)
- [KVO (Key-Value Observing)](#kvo-key-value-observing)
- [相機功能](#相機功能)

---

## 集合類型 (Collections)

### Collection 與 BidirectionalCollection 差別
1. **Collection**: 僅支援 forward traversal: `index(after:)`
2. **BidirectionalCollection**: 支援雙向 traversal: `index(after:)` & `index(before:)`
3. **繼承關係**: `BidirectionalCollection` 繼承 `Collection`，`Collection` 繼承 `Sequence`

### index 方法差別
在 `BidirectionalCollection` 中：
- `index(before:)`: non-mutating 方法，回傳一個新的 index
- `formIndex(before:)`: mutating 方法，修改帶入的 index

### RandomAccessCollection 特性
1. 可以在 O(1) 時間內移動任意距離的索引
2. 提升索引移動或距離測量的操作效能
3. 讀取 `count` 所花的時間是 O(1)，而不是 iterate
4. 常用的 `Array` 屬於 RandomAccessCollection 的一種

### 冷門小知識
- Swift standard library 中，`sort` function 使用 `introsort`: 結合 `Insertion sort` & `heap sort`
- 對於數量小於 20 的陣列，使用 `Insertion sort`
- `Array.contains(_:)` 是 `O(n)` 操作

---

## 基礎語法與概念

### 迴圈技巧

#### 倒數的 Loop
```swift
let array = [1, 2, 3, 4]

// 方法一
for i in (0..<array.count).reversed() {
    print(i)
}

// 方法二
for i in stride(from: array.count - 1, through: 0, by: -1) {
    print(array[i])
}

// 方法三
for i in array.indices.reversed() {
    print(i)
}
```

### 函數與泛型

#### 類型限制
```swift
func task<T>(in collection: T) -> Int? where T: RandomAccessCollection {
    return nil
}
```

### Protocol 實用用法
適用於不同的 class 都有相同的方法，但 class 間又無法抽出 base 時：

```swift
// 宣告 protocol
protocol TraversableBinaryNode {
    associatedtype Element
    var value: Element { get }
    var leftChild: Self? { get }
    var rightChild: Self? { get }
    
    func traverseInOrder(visit: (Element) -> Void)
    func traversePreOrder(visit: (Element) -> Void)
    func traversePostOrder(visit: (Element) -> Void)
}

// 以 extension 方式提供實作
extension TraversableBinaryNode {
    func traverseInOrder(visit: (Element) -> Void) {
        self.leftChild?.traverseInOrder(visit: visit)
        visit(value)
        self.rightChild?.traverseInOrder(visit: visit)
    }
    
    func traversePreOrder(visit: (Element) -> Void) {
        visit(value)
        self.leftChild?.traverseInOrder(visit: visit)
        self.rightChild?.traverseInOrder(visit: visit)
    }
    
    func traversePostOrder(visit: (Element) -> Void) {
        self.leftChild?.traverseInOrder(visit: visit)
        self.rightChild?.traverseInOrder(visit: visit)
        visit(value)
    }
}

// 類別以 extension 宣告實作 protocol
extension AVLNode: TraversableBinaryNode {
}

// 即可馬上使用
tree.root?.traverseInOrder { print($0) }
```

### lazy var 使用時機
```swift
class SomeClass {
    var apiService: APIService!
    var projectStorage: RoomScanProjectsStorage!
    var dataStorage: DataStorage!
    var cacheService: CacheService!
    
    // 因為宣告為 lazy，所以可以在宣告處直接使用其他變數
    lazy var vm: Scan_vm? = Scan_vm(dependency: (
        apiService: apiService,
        projectsStorage: projectStorage,
        dataStorage: dataStorage,
        cacheService: cacheService
    ))
    
    func init() {
        // ...
    }
}
```

### 型態與協定

#### Hashable 與 Set
- 型態必須繼承 `Hashable` 才可以放到 `Set` 或 Dictionary 的 Key
- 當 Class 實作 Hashable 時，必須實作 Equatable
- Class instance 自動實作 Identifiable

#### 初始化技巧
```swift
public init(jokesService: JokeServiceDataPublisher = JokesService()) {
    // ...
}
```

### 實用工具

#### 時間格式化
- `RelativeDateTimeFormatter()` 可以回傳好閱讀的時間區間，如 "1 hour ago", "in 2 weeks", "yesterday"

#### JSON 解碼
```swift
let decoder = JSONDecoder()
decoder.keyDecodingStrategy = .convertFromSnakeCase
```

#### Result 類型
```swift
func perform(_ request: URLRequest, completionHandler: @escaping (Result<Data, Error>) -> ()) {
    // ...
}
```

### Switch 語句技巧

#### 解構賦值
```swift
switch case let .upc(let numberSystem, let product, let check):
// 也可寫成
case let .upc(numberSystem, product, check):
```

#### 雙值判斷
```swift
SomeAPI.someFunc() { value, error in
    switch (value, error) {
    case (nil, let error?):
        // error 有值
    case (let value?, nil):
        // value 有值
    case (nil, nil):
        // 都沒值
    case let (address?, error?):
        // 都有值
    }
}
```

### 實用方法

#### 比較方法
```swift
expiresAt.compare(Date()) == .orderedDescending
```

#### 過濾與轉換
```swift
let list: [Any] = ["A", "B", 1, 2, true]
let numbers = list.compactMap { $0 as? Int }
```

#### 除錯工具
- 使用 `#file` 或 `#function` 進行 debug print

#### 隨機數
```swift
var dailyGainLoss: Int {
    .random(in: -10...10)
}
```

#### 字串檢查
```swift
id.rangeOfCharacter(from: .letters) != nil
```

#### 物件識別
- `ObjectIdentifier`: 取得 class instance 的 id，`ObjectIdentifier(self)` 相等於 id

#### 範圍表達式
```swift
case ...(-0.5):
    Action()
case 0.5...:
    Action()
default:
    backgroundColor = Color("Gray")
```

### 專案設定
- ProjectSetting 內建變數: `$(SRCROOT)`, `$(PROJECT_DIR)`

### 進度追蹤
- 如果要表示進度條資料，可用 Class: [Progress](https://developer.apple.com/documentation/foundation/progress)

### 除錯技巧

#### Exception Breakpoint
- 如果發生 Crash 時只跳到 AppDelegate，可加入 Exception Breakpoint

#### Assertions
- `assertionFailure`, `assert(condition:)`: 手動觸發 Crash
- **注意**: Assertions 在 Release 版本並不會發生作用
- Release 版本請用 `preconditionFailure(_:file:line:)` 或 `fatalError(_:file:line:)`

#### assert vs fatalError 差別
1. `assert`: 只在 debug builds 中有效
2. `precondition`: 在 debug 和 release builds 中都有效
3. `fatalError`: 立即停止程式執行

#### 日誌差別
- `NSLog` vs `print`: NSLog 會印出詳細時間
```
2022-12-30 15:56:35.233109+0800 GiftLister[9591:257040] hihi log!
```

#### Breakpoint 技巧
- 配合 Action: Log Message 與 Automatically continue，等同於使用 print()
- 配合 Action: Debugger Command，可在 runtime 使用進階指令

#### 模擬卡線程
```swift
sleep(10)
```

#### 線程檢查
```swift
dispatchPrecondition(condition: .notOnQueue(.main))
Thread.isMainThread
```

### 程式碼標籤
- 常用標籤: `MARK:`, `TODO:`, `FIXME:`
- 空一個加橫槓，會在 jump bar 上標籤上下各加一條橫線

#### Build Script 警告
在 BuildPhases 新增 Run Script：
```bash
TAGS="TODO:|FIXME:"
echo "searching ${SRCROOT} for ${TAGS}"
find "${SRCROOT}" \( -name "*.swift" \) -print0 | xargs -0 egrep --with-filename --line-number --only-matching "($TAGS).*\$" | perl -p -e "s/($TAGS)/ warning: \$1/"
```

### 實用技巧

#### 月份名稱
```swift
Calendar.current.monthSymbols[month - 1]
```

#### 枚舉遍歷
- 實作 `CaseIterable`，就可以用 `SomeEnum.allCases` 遍歷所有類型

#### 自訂 Delegate
- 自訂的 Class 想要使用官方的 Delegate，必須繼承 `NSObject`

#### 日誌記錄
```swift
private let logger = Logger(subsystem: "com.apple.sample.photogrammetry", category: #fileID)
logger.log("Successfully...")
logger.error("Program terminated...")
```

#### 記憶體管理
- `autoreleasepool`: 管理 run loop 中的物件釋放

#### 執行時間測量
```swift
let start = CFAbsoluteTimeGetCurrent()
// some slowly work...
print("elapse time:\(CFAbsoluteTimeGetCurrent() - start)")
```

#### Closure 宣告
```swift
(param) -> Void
// 相等於
({ param in
})
```

#### 檔案操作
- FileManager 判斷檔案存在或移除檔案時，URL 不可使用 File URL (開頭為 file://)

#### URLSession 超時設定
```swift
private lazy var liveURLSession: URLSession = {
    var configuration = URLSessionConfiguration.default
    configuration.timeoutIntervalForRequest = .infinity
    return URLSession(configuration: configuration)
}()
```

#### 簡易錯誤處理
```swift
extension String: Error { }
// 或
extension String: LocalizedError {
    public var errorDescription: String? {
        return self
    }
}

do {
    throw "This is sample error"
} catch {
    // 處理錯誤
}
```

#### Guard Let 變化
```swift
for imgUrl in imgUrls {
    guard let shotFileInfo = ShotFileInfo(url: imgUrl) 
    else {
        logger.error("Can't get shotId from url: \"\(imgUrl)\")")
        continue
    }
    newShots.append(shotFileInfo)
}
```

#### 時間格式控制
- 透過設定 `DateComponentsFormatter.ZeroFormattingBehavior` 控制時間顯示

#### Try 的變化用法
```swift
// 常見
do {
    try FileManager.default.createDirectory(atPath: myPath, withIntermediateDirectories: true)
} catch {
    logger.error("Fail to create new project folder:\(myPath), error:\(error)")
}

// 變化
let result: ()? = try? fileManager.createDirectory(atPath: myPath, withIntermediateDirectories: true)
guard result != nil else {
    return false
}
```

### Swift 5.9 新功能

#### If/Else 表達式
```swift
// 舊版
func setValue(index: String) {
    let value: Int
    if index == "a" {
        value = 1
    } else if index == "b" {
        value = 2
    } else if index == "c" {
        value = 3
    } else {
        value = 0
    }
}

// 新版
func setValue(index: String) {
    let value =
    if index == "a" { 1 }
    else if index == "b" { 2 }
    else if index == "c" { 3 }
    else { 0 }
}
```

#### Switch 表達式
```swift
// 舊版
func setValue(index: String) {
    let value: Int
    switch index {
    case "a":
        value = 1
    case "b":
        value = 2
    case "c":
        value = 3
    default:
        value = 0
    }
}

// 新版
func setValue(index: String) {
    let value: Int = switch index {
    case "a": 1
    case "b": 2
    case "c": 3
    default: 0
    }
}
```

#### 可變參數泛型
```swift
// 舊的: 一定要明確定義泛型的數量
func oldGenericFunc<T, U>(a: T, b: U) {
    // ...
}

// 新的: 不用確認指定
func newGenericFunc<each T>(_ p: repeat each T) {
    // ...
}

// 可傳多個、各種類型參數
newGenericFunc("1", 1, "a", [1,2,3])
```

### 存取控制
- `open` vs `public`: 
  - `open`: 在模組內或外皆可以被 override 與繼承
  - `public`: 只限在模組內被 override 與繼承

### 新功能
- class 可掛 `@Observable`，並配合 async/await，而不需使用 combine

### URL 編碼
```swift
let query = "Hello, World!"
if let encodedQuery = query.addingPercentEncoding(withAllowedCharacters: .urlQueryAllowed) {
    print("Encoded: \(encodedQuery)") // print Encoded: Hello%2C%20World%21
} else {
    print("Failed to encode query")
}
```

### 自訂字串表示
- `CustomStringConvertible` 和 `CustomDebugStringConvertible`

### 檔案操作方法
```swift
// 方法一: FileManager.default.contentsOfDirectory
func getFiles(folder: URL) throws -> [URL] {
    guard let URLs = try? FileManager.default.contentsOfDirectory(
        at: folder,
        includingPropertiesForKeys: [],
        options: [.skipsHiddenFiles]
    ) else { 
        throw "Could not open the folder"
    }
    return URLs
}

// 方法二: FileManager.default.enumerator
func getFiles(folder: URL) throws -> [URL] {
    var URLs = [URL]()
    guard let enumerator = FileManager.default.enumerator(
        at: folder,
        includingPropertiesForKeys: [],
        options: [.skipsHiddenFiles]
    ) else { 
        throw "Could not open the folder"
    }
    
    for case let itemURL as URL in enumerator {
        URLs.append(itemURL)
    }
    return URLs
}
```

### 等待一個 Frame
```swift
// wait until 1 frame
DispatchQueue.main.async {
    // TODO
}

// RxSwift
self.isResetEnabled
    // wait until 1 frame
    .delay(.seconds(0), scheduler: MainScheduler.instance)
    .bind(to: btnReset.rx.isEnabled)
    .disposed(by: disposeBag)
```

### SafeArea 相關
```swift
var safeAreaSize: CGSize {
    let safeAreaInsets = self.view.safeAreaInsets
    let width = self.view.bounds.width - safeAreaInsets.left - safeAreaInsets.right
    let height = self.view.bounds.height - safeAreaInsets.top - safeAreaInsets.bottom
    return CGSize(width: width, height: height)
}
```

### 計算 ChildView 大小
```swift
parentView.layoutIfNeeded()
let contentRect: CGRect = self.parentView.subviews
    .reduce(into: .zero) { rect, view in
        rect = rect.union(view.frame)
    }
parentViewHeight = contentRect.height
```

### 陣列快速相加
```swift
(1...n).reduce(0, +)
```

### 高效能集合
- 大量陣列操作可使用 [官方 Swift Collections](https://github.com/apple/swift-collections): `Deque`, `OrderedDictionary`, `OrderedSet`

### ExpressibleByArrayLiteral
```swift
public struct Stack<T> {
    private var storage: [T] = []
    // ...
}

extension Stack: ExpressibleByArrayLiteral {
    public init(arrayLiteral elements: T...) {
        self.storage = elements
    }
}

let myStack: Stack = [1, 2, 3, 4]
```

### Defer 使用
```swift
public func pop() -> value? {
    defer {
        // TODO...	
    }
    return head?.value
}
```

### COW (Copy-on-Write)
```swift
let array1 = [1, 2]
var array2 = array1
// Before append, array1 & array2 指向同一個記憶體
array2.append(3)
// append 被呼叫後，array2 複製 array1 的值
// array1: [1, 2]
// array2: [1, 2, 3]
```

### 記憶體參照檢查
- `isKnownUniquelyReferenced`: 查詢 instance 是否只有一個 reference 指向它

---

## 陣列相關

### 記憶體管理
- 宣告 Array 時 Swift 會給一個預設空間
- 若添加的項目超過容量，會自動配給空間，為原來的**兩倍**

### 效能優化
```swift
let size = 1024
var values: [Int] = []

// 避免 Array 大量產生重新分配空間的問題
values.reserveCapacity(size)

for i in 0 ..< size {
    values.append(i)
}
```

### Sequence 與 Iterator
- `Sequence`, `AnySequence` 因為沒有足夠的方法可以供 Loop 用，需要使用老式的 iterator：

```swift
let iterator = sequence.makeIterator()
while let value = iterator.next() {
    print(value)
}
```

### For-In 迴圈的背後機制
```swift
let animals = ["Antelope", "Butterfly", "Camel", "Dolphin"]
for animal in animals { 
    print(animal)
}

// 背後實際執行：
var animalIterator = animals.makeIterator()
while let animal = animalIterator.next() {
    print(animal)
}
```

### 透過 indices 遍歷
```swift
let array = ["a", "b", "c"]
for index in array.indices {
    print(array[index]) // a, b, c
}
```

### Array Extension 限制
```swift
// 要撰寫 Array 的 Extension 並限制 Element 型態時，使用 == 而非 :
extension Array where Element == Int {
    // ...
}
```

### removeLast() vs popLast() 差別
1. **回傳值**: 
   - `popLast()` 回傳為 `optional` 值，可以為 `nil`
   - `removeLast()` 一定會有值
2. **安全性**: 
   - 空陣列呼叫 `popLast()` 正常使用
   - 空陣列呼叫 `removeLast()` 會 Crash

### 多維陣列宣告
```swift
// 正確
var buckets: [[Element]] = .init(repeating: [], count: 10)

// 錯誤
// var buckets: [[Element]] = Array(repeating: [Int], count: 10)
```

---

## 錯誤處理

### 錯誤類型檢查
- 如果 error 沒有實作 `Equatable`，要檢查 error type，可用：
```swift
case MyError.someError = error
```

### 錯誤前綴判斷
- 如果要用 switch case 判斷 error 類型，但 error 後面會附加一些資訊，無法使用傳統方式辨別：
```swift
case _ where error.hasPrefix("can not find user_id"):
```

### 錯誤過濾
- 錯誤宣告為 enum，故過濾錯誤可用：
```swift
guard case .failure(let error) = $0 
else { return }

// 或
enum MyError: Error {
    case ohNo
}

Just("Hello")
    .setFailureType(to: MyError.self)
    .eraseToAnyPublisher()
    .sink { completion in
        switch completion {
        case .failure(.ohNo):
            // ...
        case .finished:
            // ...
        }    
    }
```

### 保留字作為 enum 變數名稱
```swift
enum OnboardingUserInput {
    case `continue`(isFlippable: Bool) // continue 為保留字，用單引號包著
    case skip(isFlippable: Bool)
    case finish
    case objectCannotBeFlipped
    case flipObjectAnyway
}
```

---

## UI 相關

### 狀態列與主畫面指示器
```swift
// 動態改變 statusBar 及 homeIndicator
override var prefersStatusBarHidden: Bool {
    return !mHudVisible
}

mHudVisible.toggle()
self.setNeedsStatusBarAppearanceUpdate()
self.setNeedsUpdateOfHomeIndicatorAutoHidden()
```

### SafeArea 尺寸
```swift
// SafeArea 常見尺寸
// DEVICE_TOP_PADDING iPhone14Pro: 59.0pt, iPhone11: 48.0pt
// DEVICE_BOTTOM_PADDING: 34.0

var safeAreaSize: CGSize {
    let safeAreaInsets = self.view.safeAreaInsets
    let width = self.view.bounds.width - safeAreaInsets.left - safeAreaInsets.right
    let height = self.view.bounds.height - safeAreaInsets.top - safeAreaInsets.bottom
    return CGSize(width: width, height: height)
}
```

### Modal 呈現樣式
```swift
// UIViewController.modalPresentationStyle 類型
.fullScreen: // 當 child-vc 蓋上來時，會觸發 viewDidDisappear，然後 view 消失
.overFullScreen: // 當 child-vc 蓋上來時，不會觸發 viewDidDisappear，view 不消失，適合半透明背景
```

### ViewController 管理
```swift
// 檢查 view 是否已讀取
vc.loadViewIfNeeded()
vc.isViewLoaded

// 把 vc 嵌入另一個 vc
parentView.addSubview(childVC.view)
// 建立子母關係，確保功能都正常
parentVC.addChild(childVC)
// 通知 child view controller 已加入或移除完成
childVC.didMove(toParent: self)
```

### View 更新
```swift
// 觸發 UIView (SCNView) 重新更新畫面
setNeedsDisplay()
```

### 動態產生 ViewController
```swift
let vc = ViewController()
vc.modalPresentationStyle = .formSheet
vc.preferredContentSize = .init(width: 500, height: 800)
present(vc, animated: true)
```

### MultipeerConnectivity 設定
如果要使用 MultipeerConnectivity，要在 info.plist 加上兩個宣告：

```xml
<!-- 第一個宣告 -->
<key>Privacy - Local Network Usage Description</key>
<string>APP_NAME needs to use your phone's data to discover devices nearby</string>

<!-- 第二個宣告 -->
<key>Bonjour services</key>
<array>
    <string>_USER_CUSTOM_SERVICE_NAME._tcp</string>
</array>
```

### 計算 ChildView 大小
```swift
parentView.layoutIfNeeded()
let contentRect: CGRect = self.parentView.subviews
    .reduce(into: .zero) { rect, view in
        rect = rect.union(view.frame)
    }
parentViewHeight = contentRect.height
```

### 陣列快速相加
```swift
(1...n).reduce(0, +)
```

### 高效能集合
- 大量陣列操作可使用 [官方 Swift Collections](https://github.com/apple/swift-collections): `Deque`, `OrderedDictionary`, `OrderedSet`

### ExpressibleByArrayLiteral
```swift
public struct Stack<T> {
    private var storage: [T] = []
    // ...
}

extension Stack: ExpressibleByArrayLiteral {
    public init(arrayLiteral elements: T...) {
        self.storage = elements
    }
}

let myStack: Stack = [1, 2, 3, 4]
```

### Defer 使用
```swift
public func pop() -> value? {
    defer {
        // TODO...	
    }
    return head?.value
}
```

### COW (Copy-on-Write)
```swift
let array1 = [1, 2]
var array2 = array1
// Before append, array1 & array2 指向同一個記憶體
array2.append(3)
// append 被呼叫後，array2 複製 array1 的值
// array1: [1, 2]
// array2: [1, 2, 3]
```

### 記憶體參照檢查
- `isKnownUniquelyReferenced`: 查詢 instance 是否只有一個 reference 指向它

---

## 格式化工具

### NumberFormatter
```swift
extension NumberFormatter {
    static var currency: NumberFormatter = {
        let numberFormatter = NumberFormatter()
        numberFormatter.numberStyle = .currency
        return numberFormatter
    }()
}
```

### DateFormatter
```swift
extension DateFormatter {
    static var dueDateFormatter: DateFormatter = {
        let formatter = DateFormatter()
        formatter.dateStyle = .medium
        formatter.timeStyle = .none
        return formatter
    }()

    static var timestampFormatter: DateFormatter = {
        let formatter = DateFormatter()
        formatter.dateStyle = .none
        formatter.timeStyle = .short
        return formatter
    }()
}
```

### ByteCountFormatter
```swift
let sizeFormatter: ByteCountFormatter = {
    let formatter = ByteCountFormatter()
    formatter.allowedUnits = [.useMB]
    formatter.isAdaptive = true
    return formatter
}()
```

### 綜合範例
```swift
let dateFormatter: DateFormatter = {
    let formatter = DateFormatter()
    formatter.dateStyle = .short
    formatter.timeStyle = .short
    return formatter
}()
```

---

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

### 實用功能

#### @discardableResult
- 關閉 func 有回傳值時，但使用的地方沒有宣告變數接收造成的 warning

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
```

---

## 日誌記錄

### Logger 基礎
```swift
let logger = Logger(subsystem: "com.example.Wallet", category: #fileID)
```

### Log Levels (日誌等級)

1. **Debug**: 僅在除錯時有用
2. **Info**: 有幫助但對故障排除不是必需的
3. **Notice (預設)**: 對故障排除至關重要
4. **Error**: 執行期間看到的錯誤
5. **Fault**: 程式中的錯誤

### Persistence (持久性)

1. **Debug**: 不持久化
2. **Info**: 僅在日誌收集期間持久化
3. **Notice**: 持久化直到達到儲存限制
4. **Error**: 持久化直到達到儲存限制
5. **Fault**: 持久化直到達到儲存限制

### Logger vs Print
- 使用 `Logger` 類別，已打包的程式可在「系統監視程式」中可見
- 好處是在 logger 中印出自訂的值，為了安全，在已打包的版本中，自訂的值會印出 `<private>`

### 參考資料
[為什麼你應該使用 Logger 而不是 print 來記錄應用程式資料](https://medium.com/@MariamBabutsa/why-you-should-use-logger-over-print-for-logging-you-apps-data-6f4085500a76)
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
- `actor` 只是確保自訂類型線程安全

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
```

---

## KVO (Key-Value Observing)

### 新版 KVO
目前都使用 RxSwift、Combine 或 Delegate、Notification Center 代替

### 基本設定
被觀察的 Class 要：
- 繼承 `NSObject`
- 屬性要標示 `@objc`: exposes the property to Objective-C
- `dynamic`: allows KVO to dynamically track changes at runtime

```swift
class Child: NSObject {
    let name: String
    // KVO-enabled properties must be @objc dynamic
    @objc dynamic var age: Int
    
    init(name: String, age: Int) {
        self.name = name
        self.age = age
    }
    
    func birthDay() {
        self.age += 1
    }
}

let mia = Child(name: "Mia", age: 1)

let observation = mia.observe(\.age, options: [.initial, .old, .new, .prior]) { child, change in
    print("child init:\(child.age)")
    print("child old:\(change.oldValue)")
    print("child new:\(change.newValue)")
    // 發生變化前，KVO會觸發一次，isPrior為true; 發生變化後，KVO觸發一次，isPrior為false
    print("is prior:\(change.isPrior)") 
}

print("birthday")
mia.birthDay()
print("birthday")
mia.birthDay()

// 不使用時要記得清除
deinit {
    observation.invalidate()
}
```

---

## 相機功能

### 取得支援的相機設備
以後鏡頭為例，這裡的 Camera 都是虛擬 Camera 概念（除了 builtInWideAngleCamera），各個 Camera 都是由不同的實體 Camera 組成：

```swift
// 順序依 Camera 新->舊排序
let deviceTypes: [AVCaptureDevice.DeviceType] = [
    // builtInTripleCamera: a system that switches automatically among the three lens, etc Pro
    .builtInTripleCamera,
    // builtInDualWideCamera: a system that switches between 0.5x and 1x as appropriate, etc: Pro, i11
    .builtInDualWideCamera, 
    // builtInDualCamera: a system that switches between 1.0x and telephoto lens (2x, 2.5x, or 3x depending on the phone model), etc: x, Pro
    .builtInDualCamera, 
    // builtInWideAngleCamera: 1x single-lens camera, etc: i6, i7, x, i11, 14, 14pro
    .builtInWideAngleCamera,
]

// Supported Device Types
discoverySession = AVCaptureDevice.DiscoverySession(deviceTypes: deviceTypes, mediaType: .video, position: .back)
// 取得第一個(最佳)Camera，或依需求自行選擇組合
videoDevice = discoverySession.devices.first
```

### 設備支援概覽
```
                Triple Camera   Wide Dual Camera    Dual Camera     Wide Camera
iPhone 14 Pro   Yes             Yes                 Yes             Yes
iPhone 13                       Yes                                 Yes
iPhone 11                       Yes                                 Yes
iPhone X                                            Yes             Yes
iPhone 6s                                                           Yes
```

### iPhone 14 Pro 詳細規格
```swift
// Supported Types: 
// [Back Triple Camera]
// [Back Dual Wide Camera]
// [Back Dual Camera]
// [Back Camera]

// builtInTripleCamera
// 實體鏡頭組合
constituentDevices: [[Back Ultra Wide Camera], [Back Camera], [Back Telephoto Camera]]
// 觸發切換鏡頭的Zoom門檻，第一個鏡頭門檻不會列入表內，故2: [Back Camera], 6: [Back Telephoto Camera]
virtualZoomFactors: [2, 6]

// builtInDualWideCamera
constituentDevices: [[Back Ultra Wide Camera], [Back Camera]]
virtualZoomFactors: [2]
isVirtualDevice: true

// Dual Camera
constituentDevices:  [[Back Camera], [Back Telephoto Camera]]
virtualZoomFactors: [3]
isVirtualDevice: true

// builtInWideAngleCamera
constituentDevices: []
virtualZoomFactors: []
isVirtualDevice: false
```

### 相機初始化設定
首次啟動鏡頭，videoZoomFactor 都為 1x，若是新型手機，取得的實體 Camera 可能為 Ultra Wide，畫面會變魚眼，故需要依當下已選擇虛擬 Camera，找出觸發 WideCamera 的 zoomFactor 並指定，畫面才會正常：

```swift
// 一定要等到加到 Input，才會套用到新的設定
session.addInput(videoDeviceInput)
try? device!.lockForConfiguration()

// virtualZoomFactors的數量若大於0，代表有超過1個以上的實體camera, isVirtualDevice = true; 
// virtualZoomFactors若為空，代表只有1個實體camera, isVirtualDevice = false

// virtualZoomFactors的數量 = constituentDevices數量-1，因為第一個實體Camera的zoomFactor不會列在virtualZoomFactors裡面
// 每一種機型的實體Camera陣列(constituentDevices)順序不相同，要找出目標實體Camera(WideCamera)的index並依該index取得觸發該Camera的值(zoomFactor)

guard let virtualZoomFactors = device?.virtualDeviceSwitchOverVideoZoomFactors,
      virtualZoomFactors.count > 0,
      // 找出wideCamera的index
      let wideCameraIndex = device?.constituentDevices.firstIndex(where: {
          $0.deviceType == .builtInWideAngleCamera
      }),
      wideCameraIndex != 0
else { return }

// 因為第一個實體Camera的觸發zoomFactor不會列在virtualZoomFactors裡面，所以wideCamera的camera index要減1
let triggerWideCameraFactor = device!.virtualDeviceSwitchOverVideoZoomFactors[wideCameraIndex - 1]
// 設定zoomFactor => 觸發實體鏡頭為WideCamera => 畫面正常
device!.videoZoomFactor = CGFloat(truncating: triggerWideCameraFactor)

device!.unlockForConfiguration()
```

---

## 結語

這份 Swift 學習筆記涵蓋了從基礎語法到進階並發程式設計的各個面向，包括：

- **集合類型與陣列操作**：深入了解 Swift 的集合類型特性與效能優化
- **錯誤處理**：掌握 Swift 的錯誤處理機制與最佳實踐
- **UI 開發**：UIKit 相關的實用技巧與設定
- **並發程式設計**：現代 Swift 的 async/await、Actor 模型等
- **相機功能**：AVFoundation 相機開發的詳細指南

持續學習和實踐這些概念，將有助於成為更優秀的 Swift 開發者。
