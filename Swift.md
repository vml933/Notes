# Swift 學習筆記

## 目錄
- [集合類型 (Collections)](#集合類型-collections)
- [基礎語法與概念](#基礎語法與概念)
- [陣列相關](#陣列相關)
- [錯誤處理](#錯誤處理)
- [UI 相關](#ui-相關)
- [格式化工具](#格式化工具)
- [日誌記錄](#日誌記錄)
- [KVO (Key-Value Observing)](#kvo-key-value-observing)

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

## 語法
- 顯示Sandbox使用者資料夾路徑`Documents`，其他常用路徑如`Library`、`Library\Caches`，iOS 備份包含`Document`與`Libray`，不含`Caches`。取用方式參考 [URL](https://developer.apple.com/documentation/foundation/url) 的 Accessing common directories (iOS 16.0+)
```
print(URL.documentsDirectory) 
// file:///Users/mark/Library/Developer/.../Documents/
```
- UserDefault 為 plist 檔，儲存在Sandbox中`Library`->`Preferences`

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
### `some protocol` 與 `any Protocol` 比較
- `some`的速度較快
- 與 `any` 類似，目的都是在隱藏實作，不希望 API 使用者依賴具體型別
- `some`目的在於簡化API，為了使用者手叫方便
- `any` 目的在於提供`多種`類型實作；`some`的背後只限制一種類似，否則 Compiler 出錯
```swift
protocol Animal {
    func speak() -> String
}

struct Dog: Animal {
    func speak() -> String { "Woof" }
}

struct Cat: Animal {
    func speak() -> String { "Meow" }
}
// some protocol
func makeAnimal(useDog: Bool) -> some Animal {
    if useDog {
        return Dog()
    } else {
        return Cat() // ❌ 編譯錯誤：return type of function must be the same
    }
}
// any protocol
func makeAnimal(useDog: Bool) -> any Animal {
    if useDog {
        return Dog()
    } else {
        return Cat() // ✅ OK：都符合 Animal 協議
    }
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
- `enum` 實作 `Identifiable`方式，直接加上computer property id
```swift
enum ToolbarSelection: Identifiable {
    
    var id: Int{
        hashValue
    }
    
    case photoModel
    ...
}
swift

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
// 或
struct MyError: Error{}


do {
    throw "This is sample error"
	throw MyError()
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
- 以下是同樣的事，第一行是sugar
```swift
var cards: [Card] = []
var cards: Array<Card> = []
```

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
### 實用功能

#### Operator Overloading，可用來新增支援格式, 相加`CGSize`為例
```swift
func +(lhs: CGSize, rhs: CGSize) -> CGSize {
    CGSize(
       width: lhs.width + rhs.width,
       height: lhs.height + rhs.height)
}
```

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

