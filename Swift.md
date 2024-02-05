## Swift學習筆記
### General
- 當Class實作Hashable時，必當實作Equatable, 因為實作Equatable，因為不為valueType, 所以要用 === 判斷
- Class instance自動實作Identifiable
- 可以在Init裡面New實體
```
public init(jokesService: JokeServiceDataPublisher = JokesService()){...}
```
- `RelativeDateTimeFormatter()`可以回傳好閱讀的時間區間，如 “1 hour ago”, “in 2 weeks”, “yesterday”
- decode時轉換Key值型態
```
let decoder = JSONDecoder()
decoder.keyDecodingStrategy = .convertFromSnakeCase
```
- 可用extension的方式為protocol實現基本實作, 類似c#定義interface後，Abstract Class定義基本方法的概念
- `Result<Data, Error>` 代表一個結果，成功可帶值，失敗可帶error，常用來回傳api, 跟Combine裡面的Publisher<Int, Never>類似
```
func perform(_ request: URLRequest, completionHandler: @escaping (Result<Data, Error>) -> ())...
```
- 當要做 cache, cookie storage or credential storage時，不該使用URLSession.shared
```
switch case let .upc(let numberSystem, let product, let check):
//也可寫成
case let .upc(numberSystem, product, check):
```
- 當callback回傳的tuple是兩個option型態時，可以這樣判斷
```
SomeAPI.someFunc(){ value, error in
  switch (value, error){
  case (nil, let error?):
...
  case (let value?, nil):
...
  case (nil, nil):
...
  case let (address?, error?):
    ...
  }
}
```
- 可使用compare，結果是回傳比較的result:
```
 expiresAt.compare(Date()) == .orderedDescending
```
- 可用`compactMap`過濾項目
```
let list:[Any] = ["A", "B", 1, 2, true]
let numbers = list.compactMap({$0 as? Int})
```
- debug print時使用 `#file` 或 `#function`
- 隨機亂數 computeProperty:
```
var dialyGainLoss: Int{
  .random(in: -10...10)
}
```
- 檢查字串是否包含特定範圍, 可用`String.rangeOfCharacter:`
```
id.rangeOfCharacter(from: .letters) != nil
```
- `ObjectIdentifier`: 取得class instance的id, ObjectIdentifier(self)相等於id

- 半開放的range ...
```
case ...(-0.5):
  Action()
case 0.5...:
  Action()
default:
  backgroundColor = Color("Gray")
```            
- ProjectSetting內建變數: `$(SRCROOT)`, `$(PROJECT_DIR)`
- 如果要表示進度條資料，可用Class: [Progress](https://developer.apple.com/documentation/foundation/progress)，內建方法合併呈現多個工作進度, 可惜必須掛interval去持續詢問progress的fractionCompleted，無法像RxSwift主動發送
- 在DEBUG模式下，Variables View，不只可以看到變數，也選擇UIController的View，點選眼睛Icon，也看到App-UI畫面.
- 如果發生Crash時，只有跳到AppDelegate，無法正確的指出錯誤是哪一行，此時可加入Exception Breakpoint，IDE就會直接跳到問題行
- Assertions: EX: `assertionFailure`, 或 `assert(condition:)`, 手動觸發Crash，用來確保商業流程合乎期待，如果撰寫framework開放給外部使用者就會需要用到。**注意: Assertions在Release版本並不會發生作用**; 如果在Release發生作用，請用`preconditionFailure(_:file:line:)`或是`fatalError(_:file:line:)`
- `NSLog` & `print` 差別:NSLog會印出詳細時間，例如:
```
2022-12-30 15:56:35.233109+0800 GiftLister[9591:257040] hihi log!
```
- 搭配Breakpoint，配合Action: Log Message與下方的Automatically continue after evaluating actions，同等於使用print()，可避免寫太多print()在code裡，除了自訂訊息外，也可以寫%B印出該func名稱，寫%H可以印出總共觸發幾次。

- 搭配Breakpoint: 配合Action: Debugger Command，就可在runtime底下使用進階指令，例如印出帶有Timestamp的Log: expressing NSLog("hihi log!")
- 如果要模擬卡線程Block main thread，可用UNIX sleep command
```
sleep(10)
```
- 可使用`dispatchPrecondition(condition: .notOnQueue(.main))`或`Thread.isMainThread`檢查長時間task是否run在主線程
- 常用標籤: `MARK:`, `TODO:`, `FIXME:`, 如果空一個加橫槓，會在jump bar上，標籤上下各加一條橫線
- 可在BuildPhases新增Run Script，就可在編譯時，將TODO或FIXME以warning的方式呈現，幫助提醒
```
TAGS="TODO:|FIXME:"
echo "searching ${SRCROOT} for ${TAGS}"
find "${SRCROOT}" \( -name "*.swift" \) -print0 | xargs -0 egrep --with-filename --line-number --only-matching "($TAGS).*\$" | perl -p -e "s/($TAGS)/ warning: \$1/"
```
- **DEBUG小技巧**：如果想知道某個func回傳值，可以在return那行加上Breakpoint，然後RUN，執行該行hang住時，按step
- 取得月份: "January", "February", "March"...
```
Calendar.current.monthSymbols[month - 1]
```
- 如果要Loop所有的enum, 必須實作`CaseIterable`, 就可以用SomeEnum.allCases loop所有類型
- 如果自訂的Class想要使用官方的Delegate，必須要繼承`NSObject`
```
private let logger = Logger(subsystem: "com.apple.sample.photogrammetry", category: "HelloPhotogrammetry")
logger.log("Successfully...")
logger.error("Program terminated...")
```
- `autoreleasepool`: 每一輪 run loop 中，如果某些物件只有在這一輪 run loop 中有用，之後就應該釋放，我們可以先把物件放進 auto-release pool 裡頭，等到這一輪 run loop 的時候，再把 auto-release pool 倒空。
[Reference](https://zonble.gitbooks.io/kkbox-ios-dev/content/responder/run_loop.html)
- 計算func執行時間: 使用`CFAbsoluteTimeGetCurrent()`或`ContinuousClock`:
```
let start = CFAbsoluteTimeGetCurrent()
//some slowly work...
print("elapse time:\(CFAbsoluteTimeGetCurrent() - start)")
```
- closure宣告
```
(param) -> Void
//相等於
({ param in
})
```
- 如果要使用FileManager判斷檔案是否存在或移除檔案，URL不可使用File URL(開頭為file://)，否則會無效
- URLSession的timeout預設為60秒，若要修改時間可用:
```
  /// A URL session that lets requests run indefinitely so we can receive live updates from server.
  private lazy var liveURLSession: URLSession = {
    var configuration = URLSessionConfiguration.default
    configuration.timeoutIntervalForRequest = .infinity
    return URLSession(configuration: configuration)
  }()
```
- 如果要簡易快速的丢出錯誤，不用自訂enum Error，可以為String加上Extension
```
/// Easily throw generic errors with a text description.
extension String: Error { }
或
/// Easily throw generic errors with a text description.
extension String: LocalizedError {
  public var errorDescription: String? {
    return self
  }
}

do{
	throw "This is sample error"
}catch{
}

```
- 除了用Breakpoint，也可以用Watchpoint監聽觸發變化的時機
- `guard let` 除了常用的`return`, 也可以因應情境用`continue`
```
for imgUrl in imgUrls {
	guard let shotFileInfo = ShotFileInfo(url: imgUrl) 
	else {
		logger.error("Can't get shotId from url: \"\(imgUrl)\")")
		continue
	}

	newShots.append(shotFileInfo)
}

struct ShotFileInfo {
    let fileURL: URL

    init?(url: URL) {
        fileURL = url
        guard let shotID = CaptureFolderManager.parseShotId(url: url) else {
            return nil
        }
    }
}

```
- 透過設定[DateComponentsFormatter.ZeroFormattingBehavior](https://developer.apple.com/documentation/foundation/datecomponentsformatter/zeroformattingbehavior)，可以控制顯示時間若帶有0，該如何顯示，ex: 1點10秒可顯示為`1:00:10`或`1h 10s`
- 會throws的片段，除了可以用Do try catch，若不需知道Error類型，也可以用()?的方式，回傳結果
```
//常見
do{
  try FileManager.default.createDirectory(atPath: myPath, withIntermediateDirectories: true)
}catch{
  logger.error("Fail to create new project folder:\(myPath), error:\(error)")
}
//變化
let result: ()? = try? fileManager.createDirectory(atPath: myPath, withIntermediateDirectories: true)
guard result != nil else {
  return false
}
```
- Swift 5.9 新增 if, else的好用方式
```
//舊版
func setValue(index: String){
    let value: Int
    if index == "a"{
        value = 1
    }else if index == "b"{
        value = 2
    }else if index == "c"{
        value = 3
    }else{
        value = 0
    }
}

//新版
func setValue(index: String){
    let value =
    if index == "a"{ 1 }
    else if index == "b"{ 2 }
    else if index == "c"{ 3 }
    else{ 0 }
}

//舊版
func setValue(index: String){
    let value: Int
    switch index{
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

//新版
func setValue(index: String){
    let value: Int = switch index{
    case "a": 1
    case "b": 2
    case "c": 3
    default: 0
    }
}

//舊版
func getIndex(index: String) -> Int{
    if index == "a"{
        return 1
    }else if index == "b"{
        return 2
    }else if index == "c"{
        return 3
    }else{
        return 0
    }
}


//新版
func getIndex(index: String) -> Int{
    if index == "a"{ 1 }
    else if index == "b"{ 2 }
    else if index == "c"{ 3 }
    else{ 0 }
}

//舊版
func getIndex(index: String) -> Int{
    switch index{
    case "a":
        return 1
    case "b":
        return 2
    case "c":
        return 3
    default:
        return 0
    }
}


//新版(有一行限制)
func getIndex(index: String) -> Int{
    switch index{
    case "a": 1
    case "b": 2
    case "c": 3
    default: 0
    }
}
```
- `open`與`public`差別: `open`在模組內或外皆可以被override與繼承; `public`只限在模組內被override與
繼承
- Swift 5.9可傳不固定數量、不確定類型的泛型當參數: Func傳多個不確定類型數量參數
```
//舊的: 一定要明確定義泛型的數量
func oldGenericFunc<T, U>(a: T, b: U){
    //...
}
//新的: 不用確認指定
func newGenericFunc<each T>(_ p: repeat each T){
    //...
}
//可傳多個、各種類型參數
newGenericfunc("1", 1, "a", [1,2,3])

```
- class可掛@Observable，並配合async/await，而不需使用combine
- `addingPercentEncoding(withAllowedCharacters:)`用來把文字包成url可接收格式，類似url.encode，常用在URL Query(不知是否因為encode後的每個字元都帶%符號，所以func名稱取名為adding<b>Percent</b>Encoding)
```
let query = "Hello, World!"
if let encodedQuery = query.addingPercentEncoding(withAllowedCharacters: .urlQueryAllowed) {
    print("Encoded: \(encodedQuery)") //print Encoded: Hello%2C%20World%21
} else {
    print("Failed to encode query")
}

```
- 除了有自訂義CustomStringConvertible可用，也有CustomDebugStringConvertible可用，兩者的用法很類似，只是意圖不同
- 取得資料夾所有檔案URL的兩種方法
```
//FileManager.default.contentsOfDirectory
func getFiles(folder: URL) throws -> [URL]{
    guard let URLs = try? FileManager.default.contentsOfDirectory(at: folder,
                                                                   includingPropertiesForKeys: [],
                                                                   options: [ .skipsHiddenFiles])
    else { throw "Could not open the folder"}
    
    return URLs
}
//FileManager.default.enumerator
func getFiles(folder: URL) throws -> [URL]{
    var URLs = [URL]()
    guard let enumerator = FileManager.default.enumerator(at: folder,
                                                          includingPropertiesForKeys: [],
                                                          options: [.skipsHiddenFiles])
    else { throw "Could not open the folder"}
    
    for case let itemURL as URL in enumerator {
        URLs.append(itemURL)
    }
    return URLs
}
```
- 如果從Bundle.main.url讀取png檔，轉存到Document時，png檔容量會變大，Unity無法使用
### Error相關
- 如果error沒有實作equaltable, 要檢查error type, 可用 case MyError.someError = error
- 如果要用switch case判斷error類型，但error後面會附加一些資訊，無法使用傳統方式辨別，可用case _ where error.hasPrefix("can not find user_id"): 判斷
- 錯誤宣告為enum, 故過濾錯誤可用case .failure(let error) = $0, 只會有一個等號: 
```
guard case .failure(let error) = $0 
else {return}

//or 
enum MyError: Error{
    case ohNo
}

Just("Hello")
    .setFailureType(to: MyError.self)
    .eraseToAnyPublisher()
    .sink { completion in
        switch completion{
        case .failure(.ohNo):
            ...
        case .finished:
            ...
        }    
```
- 如果要使用保留字當作 enum的變數名稱，可用單引號
```
enum OnboardingUserInput{
    case `continue`(isFlippable: Bool) //continue為保留字,用單引號包著
    case skip(isFlippable: Bool)
    case finish
    case objectCannotBeFlipped
    case flipObjectAnyway
}
```

### UI相關
- 動態改變statusBar及homeIndicatior
```
// Hide the status bar to maximize immersion in AR experiences.
override var prefersStatusBarHidden: Bool {
    return !mHudVisible
}
mHudVisible.toggle()
self.setNeedsStatusBarAppearanceUpdate()
self.setNeedsUpdateOfHomeIndicatorAutoHidden()
```
- SafeArea
```
DEVICE_TOP_PADDING iPhone14Pro:59.0pt, iPhone11:48.0pt
DEVICE_BOTTOM_PADDING: 34.0
```
- UIViewerController.modalPresentationStyle類型
```
.fullScreen: 當 child-vc蓋上來時，會觸發viewDidDispear，然後view消失
.overFullScreen: 當 child-vc蓋上來時，不會觸發viewDidDispear，view不消失，適合用來modalView須要半透明背景的情境用
```
- UIViewerController可用 loadViewIfNeeded讓vc讀取view, 或用isViewLoaded檢查view是否已讀取
- 把vc嵌入另一個vc
```
parentView.addSubview(childVC.view))
//建立子母關系，確保功能都正常
parentVC.addChild(childVC)
//Notice child view controller added or removed from a container view controller is complete.
childVC.didMove(toParent: self)
```
- 觸發UIView(SCNView), 重新更新畫面: `setNeedsDisplay()`
- 動態產生vc測試效果, 使用preferredContentSize控制大小
```
let vc = ViewController()
vc.modalPresentationStyle = .formSheet
vc.preferredContentSize = .init(width: 500, height: 800)
present(vc, animated: true)
```
- 如果要使用MultipeerConnectivity，要在info.plist加上兩個宣告
```
KEY: Privacy - Local Network Usage Description
VALUE: APP_NAME needs to use your phone’s data to discover devices nearby
```
```
KEY: Bonjour services
VALUE: _USER_CUSTOM_SERVICE_NAME._tcp
```
## Formatter相關
```
extension NumberFormatter {
  static var currency: NumberFormatter = {
    let numberFormatter = NumberFormatter()
    numberFormatter.numberStyle = .currency
    return numberFormatter
  }()
}

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

let sizeFormatter: ByteCountFormatter = {
  let formatter = ByteCountFormatter()
  formatter.allowedUnits = [.useMB]
  formatter.isAdaptive = true
  return formatter
}()

let dateFormatter: DateFormatter = {
  let formatter = DateFormatter()
  formatter.dateStyle = .short
  formatter.timeStyle = .short
  return formatter
}()
```
### Grand Central Dispatch (GCD)
DispatchQueue是Swift的Grand Central Dispatch (GCD)
- `DispatchQueue.global(qos:)`: 內建的並行Queue(Concurrent): 適合用來避免同时派送多個请求，以免對接收端造成太大壓力
- `DispatchQueue(label:qos:)`: 自訂的序列Queue(Serial): 適合用來觀察多個長時間運行的工作，將所有結果結合在一起
- `DispatchQueue.main`: 與更新UI相關，預設的QoS為`.userInteractive`
```
DispatchQueue.global(qos: .background).async {
	// Perform a background task
}
DispatchQueue(label: "MyCustomQueue", qos: .userInitiated).async{
	// Perform a long time task
}
DispatchQueue.main.async {
	// Update the UI
}
```
QoS主要有分4個等級
```
.userInteractive: This is the highest priority QoS class and is used for tasks that require immediate user interaction, such as rendering animations or responding to user input.

.userInitiated: This QoS class is used for tasks initiated by the user, such as opening a file or initiating a network request.

.default: This is the default QoS class and is used for tasks that don't require a higher priority QoS class, such as performing background computations or I/O operations.

.utility: This QoS class is used for long-running tasks that are not time-critical, such as indexing files or performing backups.
```
- `@discardableResult`關閉func有回傳值時，但使用的地方沒有宣告變數接，造成的warning
- 可在func中放入`dispatchPrecondition(condition: .notOnQueue(DispatchQueue.main))`預防該func不會在指定queue中執行
- 承上，反之 `dispatchPrecondition(condition: .onQueue(.main))` 檢查是否在該queue執行
- `Task.sleep(for:)` 讓task暫停一段時間且不會卡線程，方便測試用?
- 若要使用Sleep, 可用ContinuousClock或SuspendingClock
```
```
計時相關
---
1. `ContinuousClock`: 即使系統觸發sleep也不會中斷，適合用來monitor，像是監聽func執行時間，loggin時間
1. `SuspendingClock`: 與ContinuousClock，系統sleep則會暫停
```
import Foundation
let clock = ContinuousClock()
let startTime = clock.now()
// Perform tasks, or wait; the clock keeps running even if the system sleeps
// ...
let endTime = clock.now()
let elapsedDuration = endTime - startTime
print(elapsedDuration)

```
- 暫停
```
let clock1 = ContinuousClock()
try? await clock1.sleep(for: .seconds(1))

let clock2 = SuspendingClock()
try? await clock2.sleep(for: .seconds(1))

//Concurrency
Task{
    try await Task.sleep(for: .seconds(1), clock: .suspending)
}

//Async
Task{
    try await Task.sleep(for: .seconds(1), clock: .continuous)
}

```

### Logger相關

**Log Levels:**
1. `Debug`: Useful only during debugging
1. `Info`: Helpful but not essential for troubleshooting
1. `Notice (Default)`: Essential for troubleshooting
1. `Error`: Error seen during execution
1. `Fault`: Bug in program

**Persistence:**
1. `Debug`: Not persisted
1. `Info`: Persisted only during log collect
1. `Notice`: Persisted up to a storage limit
1. `Error`: Persisted up to a storage limit
1. `Fault`: Persisted up to a storage limit


```
let logger = Logger(subsystem: "com.example.Wallet", category: "networking")
```
[Reference](https://medium.com/@MariamBabutsa/why-you-should-use-logger-over-print-for-logging-you-apps-data-6f4085500a76)
- 使用`Logger`類別，已打包的程式可在"系統監視程式"可見; 另外好處是，在loggerd印出自訂的值，為了安全，在已打包的版本，自訂的值會印出`<private>`



## Concurrency
- Task功能是在同步的環境中，產生一個非同步的closure供使用, 非必要, Task實體可用來與之非同步工作互動，如取消。 當你不再參照Task實體時，裡頭的非同步工作依然執行中
- Task會在調用它的actor執行，如果要建立不在該actor上的task, 可用`Task.detached(priority:operation:)`:帶有特權的Task，建立的task就不會繼承父層的優先權，但會對執行效率造成負面影響，不建議使用。
- 如果要將Task儲存在變數內，因為成功不會回傳值，有可能噴錯，故類型為
```
@State var downloadTask: Task<Void, Error>?
```
- 當我們使用For In時，其實背後是取得array的iterator 
```
let animals = ["Antelope", "Butterfly", "Camel", "Dolphin"]
for animal in animals { 
	print(animal)
}
```
背後:
```
var animalIterator = animals.makeIterator()
while let animal = animalIterator.next() {
	print(animal)
}
```
- AsyncSequence只定義取得Element的方式，本身並不產生Element，是由裡頭的Iterator產生Element
```
//Custom AsyncSequence
struct Counter: AsyncSequence {

    //Required
    typealias Element = Int
    //Required
    func makeAsyncIterator() -> AsyncIterator {
        return AsyncIterator(howHigh: howHigh)
    }

    private let howHigh: Int
    //custom AsyncIteratorProtocol
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

//呼叫
for await number in Counter(howHigh: 10) {
  print(number, terminator: " ")
}
// Prints "1 2 3 4 5 6 7 8 9 10 "
```
- async / await最大的好處，再也不用譫心 weak或strongly capture self, 因為不再使用escaping closures callback
- async / await Async的相關class大部分都可以 throws error，所以使用大部分都用try catch
- 每一個`await`字眼出現，表示線程有可能改變; 每個`await`都會透過系統來引導執行，該系統會對task優先排序、取消請求、或是往上回報錯誤
- Swift的Concurrency帶有子母(Hierarchy)概念: 允許母層task取消時，也取消所有子層task(task呼叫另一task時，形成子母關係); 或是等待子層所有task都完成後，再完成母層task; 高等級task優先執行低等級task
```
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
      let mediaResponse = try decoder.decode(MediaResponse.self, from:F data)

      return mediaResponse.results
    } else {
      print("task cancelled")
      return [MusicItem]()
    }
  }
```  
- async let 可以確保非同步時一定有值，概念類似第三方套件的promise，只能用await取其中的值，PS: files & status是同時發出要求，status並不會等files
```  
//status() & availableFiles()同時執行，並行執行
do {
  async let files = try model.availableFiles()
  async let status = try model.status()
  let (filesResult, statusResult) = try await (files, status)
} catch {
  lastErrorMessage = error.localizedDescription
}

//status()會等到 availableFiles()完成後才執行，線性執行
files = try await model.availableFiles()
status = try await model.status()
```  
- async let 在宣告時就會觸發，不會等宣告await時才觸發

- `@MainActor`會強制切回主線程存取, 若在非@MainActor或非主線程的區段，存取@MainActor的變數，就要用async/await的方式
```
class MyViewModel{
    @MainActor var myParam = false
    //主線程存取
    @MainActor func myViewModelFunc1(){
        myParam = true
    }
    //在非主線程，使用async/await存取
    func myViewModelFunc2() async{
        let localParam = await myParam
    }
}

//外部呼叫，假設在非主線
func fetchData(vm: MyViewModel) async{
    //Update param on the main thread
    await vm.myParam = true
}
```
- `@TaskLocal`用來宣告給Task當區域變數使用，只能宣告在static或global的變數，有幾個特點:
1. 獨立於同一個Task與子Task裡面，每個Task的TaskLocal變數，都是不同版本的存在
1. 封裝性, 子Task可以覆寫母Task的TaskLocal變數
1. 資料不共享，不會有衝突或race conditions的狀況.
```
struct MyContext {
    var userID: String
}

@TaskLocal //Must static or global
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
```
//使用.withValue()給予@TaskLocal值
class MyViewModel{
    @TaskLocal static var taskLocalParam: String?
    
    func someFunc(){
        print(Self.taskLocalParam)
    }
}

class Sample{
    let vm = MyViewModel()
    
    func someFunc(){
        Task{
            MyViewModel.$taskLocalParam.withValue(nil){
                vm.someFunc() //print "nil"
            }
        }
        
        Task{
            MyViewModel.$taskLocalParam.withValue("bingo"){
                vm.someFunc() //print "bingo"
            }
        }
        
        print(MyViewModel.taskLocalParam) //print "nil"
    }
}

let sample = Sample()
sample.someFunc()
```
- `withCheckedContinuation(function:_:)`&` withCheckedThrowingContinuation(function:_:)`用來介接callback base的func給async/await用，只會回傳一個結果，類似RxSwift的`Single`或Combine的`Future`
- 實作運用: 把`withCheckedContinuation`裡頭的`continuation<T, Error>`傳到另一個class裡面做運用，但記得一定要送出一個結果出來，不然task無法清空
```
func shareLocation() async throws {
  let location: CLLocation = try await withCheckedThrowingContinuation { [weak self] continuation in  
    self?.delegate = ChatLocationDelegate(manager: manager, continuation: continuation)
    ...
  }
  print(location.description)
  manager.stopUpdatingLocation()
  delegate = nil
}

class ChatLocationDelegate: NSObject, CLLocationManagerDelegate {

  init(manager: CLLocationManager, continuation: CheckedContinuation<CLLocation, Error>) {
    self.continuation = continuation
    super.init()
    manager.delegate = self    
  }
  ...
  //避免沒有關閉task，class清除前continuation手動送出一個結果.
  deinit {
    continuation?.resume(throwing: CancellationError())
  }
}
```
```
func shareLocation() async throws -> String{

  let location = CLLocation(latitude: 37.785834, longitude: -122.406417)
  
  let address: String = try await withCheckedThrowingContinuation { continuation in
    //callback base API
    AddressEncoder.addressFor(location: location) { address, error in
      switch (address, error){
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

`Task.sleep()` & `Thread.sleep()` 差別
1. **Concurrency Model**: `Task.sleep` 適用Swift並行機制(`async/await`)而且不卡線程; `Thread.sleep`為舊版卡線程的一種方法
1. **Bocking Behavior**: `Task.sleep`期間允許其他Task在相同的thread上執行; `Thread.sleep`則否
1. **Cancellation**: `Task.sleep`可以被取消而中斷，所以可能丟出`CancellationError`，所以要用`try`; `Thread.sleep`則否

- Compute Property也可以用`async/await`
```
var myProperty: String{
 get async{ ...}
}

print(await myProperty)
```
- 參數若為closure也可以用`async`
```
func myFunction(worker: (Int) async -> Int) -> Int {
...
}

myFunction{
 return await computeNumbers($0)
}
```
- 可將tuple的值直接抓出來用
```
let (filesResult, statusResult) = try await (files, status)
self.files = filesResult
self.status = statusResult
```

- URL有一個內建AsyncSequence屬性resourceBytes，以非同步的方式讀取檔案資料
```
let bytes = URL(fileURLWithPath: "myFile.txt").resourceBytes

for await character in bytes.characters {
  ...
}

for await line in bytes.lines {
  ...
}

```
- 使用async/await搭配Combine，建立AsyncStream，製作計時器Sample
```
let startTime = Date().timeIntervalSince1970
let timerSequence =
Timer.publish(every: 1, tolerance: 1, on: .current, in: .default)
  .autoconnect()
  .map { date in
    let duration = Int(date.timeIntervalSince1970 - startTime)
    return "\(duration)s"
  }
  .values

let timerTask = Task{
  for await duration in timerSequence{
    //print 1s
    //print 2s
    //print 3s
  }
}

//Clear timer
timerTask.cancel()
```
- `withTaskCancellationHandler(handler:operation:)`: 全域方法，用來宣告執行Task，附帶取消時，自訂onCanceling方式
```
func downloadData() async throws -> Data{
    for _ in 0...5{
        try await Task.sleep(nanoseconds: 1_000_000_000)
        print("downloading")
    }
    return Data("DummyData".utf8)
}

func startDownload() async throws -> Data?{
    return try await withTaskCancellationHandler {
        return try await downloadData()
    } onCancel: {
        print("download was cancel, clear up...")
    }
}

var task: Task<Void, Error>?

task = Task{
    if let data = try? await startDownload(){
        print("receive data:\(data)")
    }
}
//Cancel task after 2 seconds
Task.detached {
    try? await Task.sleep(nanoseconds: 2_000_000_000)
    print("Canceling...")
    task?.cancel()
}

//[sec 1]: print downloading
//[sec 2]: print downloading
//[sec 2]: print Canceling...
//[sec 2]: print download was cancel, clear up...
```
- 若要自訂非同步序列(Asynchronous sequence)， 除了要繼承`AsyncSequence`外，還要另外宣告iterator實作`AsyncIteratorProtocol`，稍嫌瑣碎，如果不用針對特定索引或數量元素做處理，可使用`AsyncStream`(closure-based asynchronous). PS: 例如`PhotogrammetrySession.Outputs`就是AsyncSequence
```
//使用AsyncSequence & AsyncIteratorProtocol
struct Typewriter: AsyncSequence{
    
    typealias Element = String
    
    let phrase: String
    
    func makeAsyncIterator() -> TypewriterIterator {
        return TypewriterIterator(phrase)
    }
}

struct TypewriterIterator: AsyncIteratorProtocol{
    
    typealias Element = String
    
    let phrase: String
    var index: String.Index
    
    init(_ phrase: String) {
        self.phrase = phrase
        self.index = phrase.startIndex
    }
    
    mutating func next() async throws -> String? {
        guard index < phrase.endIndex
        else{return nil}
        
        try await Task.sleep(nanoseconds: 1_000_000_000)
        let result = String(phrase[phrase.startIndex...index])
        index = phrase.index(after: index)
        
        return result
    }
}
//example usage
for try await item in Typewriter(phrase: "Hello, World!"){
    print(item)
}

```
```
//使用AsyncStream(unfolding:onCancel:)為例: closure base, return value or nil, 適合用在async code中
var phrase = "Hello, World!"
var index = phrase.startIndex
let stream = AsyncStream<String> {
    guard index < phrase.endIndex else{ return nil }
    
    do{
        try await Task.sleep(nanoseconds: 1_000_000_000)
    }catch{
        return nil
    }
    
    let result = String(phrase[phrase.startIndex...index])
    index = phrase.index(after: index)
    
    return result
}

for try await item in stream{
    print(item)
}
```
```
//補充案例-1: AsyncStream(_ build:)為例: 適合用在non-async code
func timerStream(interval: TimeInterval, times: Int) -> AsyncStream<Date> {
    var count = 0

    // Create an AsyncStream of Date values
    return AsyncStream(Date.self) { continuation in
        // Set up a timer that fires every `interval` seconds
        let timer = Timer.scheduledTimer(withTimeInterval: interval, repeats: true) { _ in
            count += 1
            
            // Yield the current date
            continuation.yield(Date())

            // Finish the stream after emitting `times` values
            if count == times {
                continuation.finish()
                timer.invalidate()
            }
        }

        // Handle cancellation
        continuation.onTermination = { _ in
            timer.invalidate()
        }
    }
}

// Example usage
func runTimerStream() async {
    for await date in timerStream(interval: 1, times: 5) {
        print("Received date: \(date)")
    }
    print("Timer stream ended")
}

// Run the async function
Task {
    await runTimerStream()
}

```
```
//補充案例-2: AsyncStream(_ build:)為例: 與withCheckedContinuation相同，Continuation常被取出來應用
actor ImageLoader: ObservableObject {
  //for UI Listen
  @MainActor private(set) var accessStream: AsyncStream<Int>?
  //hold continuation for emit element
  private var accessStreamContinuation: AsyncStream<Int>.Continuation?
  private var accessCounter = 0{
    didSet{
      inMemoryAccessContinuation?.yield(inMemoryAccessCounter)
    }
  }

  func setUp() async{
    let accessStream = AsyncStream<Int> { continuation in
      accessStreamContinuation = continuation
    }
    await MainActor.run {
      self.accessStream = accessStream
    }
  }

  func add() {
    accessCounter += 1
  }

  deinit{
    accessStreamContinuation?.finish()
  }	
}

struct SwiftUIView: View {

	@State var inMemoryAccessCount = 0

	var body: some View {
		Text("\(inMemoryAccessCount) in memory")
		    .task {
		      guard let accessStream = imageLoader.accessStream
		      else{return}
		      
		      for await count in accessStream{
		        inMemoryAccessCount = count
		      }
		    }
	}
	
}
```
- 如果一個async func裡頭有兩個以上的for await Loop, 記得每個Loop要用Task包起來, 以免第一個之後的for await不會執行
```
func observerAppState() async{
    Task{
    for await _ in asyncSequence{
      ...
    }
  }

  Task{
    for await _ in asyncSequence{
      ...
    }
  }
}
```
- AsyncSequence有類似Rx的操作元可用，[swift-async-algorithm](https://github.com/apple/swift-async-algorithm)
- `Task.checkCancellation()`扔出的`CancellationError()`是繼承Error的`struct`型態，跟常見使用`enum`繼承Error方式不同
- 宣告一個可供覆寫的等待closure
```
class MyClass{
  var sleep: (Int) async throws -> Void = {
    try await Task.sleep(for: .seconds($0))
  }

  func someFunc() async throws {
    let sleep = self.sleep
    try await sleep(1)
    ...
  }
}

let anotherInstance = MyClass()
//從seconds覆寫成nano, 方便快速測試
anotherInstance.sleep = { try await Task.sleep(for: .nanoseconds($0)) }
```
- 在使用`withTaskGroup`的情況下，如果要在for await裡頭動態加入task，區域變數要加入`[]`才會帶給addTask使用，原因未知
```
await withTaskGroup(of: String.self) { [unowned self] group in
  
  let batchSize = 2
  for index in 0..<batchSize{
    group.addTask {
      await self.worker(number: index)
    }
  }
  
  var index = batchSize
  for await result in group{
    print("Completed:\(result)")
    if index < total{
//這裡如果要帶入區域變數index, 要加入[]
      group.addTask { [index] in
        await self.worker(number: index)
      }
    }
    index += 1
  }
  
}
```
- `Actor`類型是一種通過編譯檢查，用來保護內部狀態不受並行程式存取的類型
- `Actor`允許狀態內部同步存取(sync access), 而編譯器會強制外部存取使用非同步存取(async access)
- `Actor`使用`serial executor`來呼叫方法&存取屬性
- `Sendable`是一個protocol, 在並行(concurrency)程式執行中是安全的，大部分的`Value Type`: Bool, Double, Int...都是Sendable的一員，`Actor`也是; class因為是`Reference Type`，不為Sendable的一員.
- 在actor的類別下，前面加上`nonisolated`的func，會被視為普通的class方法，並移除安全檢查，稍為提升運行速度。像一些靜態方法，例如 ProjectStorage的Convert系列，不會改變本身狀態；如遇到Compilier錯誤提示要加上await該方法，皆可在該方法標記`nonisolated`解除該警告
```
actor MyModel{
 nonisolated func convertSomeFunc() async throw{...}
}
```
- 考慮利用類似的概念:`Enum`呈現工作狀態 管理重建/下載駐列工作
```
actor ImageLoader: ObservableObject{
  
  enum DownloadState{
    case inProgress(Task<UIImage, Error>)
    case completed(UIImage)
    case failed
  }
  
  private(set) var cache: [String: DownloadState] = [:]
  
  func image(_ serverPath: String) async throws -> UIImage{
    if let cached = cache[serverPath]{
      switch cached {
      case .inProgress(let task):
        return try await task.value
      case .completed(let uIImage):
        return uIImage
      case .failed:
        throw "Download failed"
      }
    }
    
    let download: Task<UIImage, Error> = 
    Task.detached {
      guard let url = URL(string: "http://localhost:8080".appending(serverPath))
      else { throw "Could not create the download URL"}
      print("Download: \(url.absoluteString)")
      
      let data = try await URLSession.shared.data(from: url).0
      return try resize(data, to: CGSize(width: 200, height: 200))
    }
    
    cache[serverPath] = .inProgress(download)
    
    do{
      let result = try await download.value
      add(result, forKey: serverPath)
      return result
    }catch{
      cache[serverPath] = .failed
      throw error
    }
    
  }

  func add(_ image: UIImage, forKey key: String){
    cache[key] = .completed(image)
  }
  
  func clear(){
    cache.removeAll()
  }
  
}
```
- 使用`globalActor`可以確保目標class永遠執行在`actor`的`serial executor`上，可以避免concurrency問題以及actor間相互切換的性能浪費
```
@globalActor actor ImageDatabase{
  
  static let shared = ImageDatabase()
  
  private let storage = DiskStorage()
}

@ImageDatabase class DiskStorage {
//guarantee the code in DiskStorage always runs on ImageDatabase’s serial executor.
}
```
- 如果要存取actor的陣列，最好複製一份出來使用，避免因concurrency造成各種不可預期的情況發生
```
actor ImageLoader: ObservableObject {
  private(set) var cache: [String: DownloadState] = [:]
  ...
}

@globalActor actor ImageDatabase{
...
 func image(_ key: String) async throws -> UIImage{
   //make a copy for safe
   let keys = await imageLoader.cache.keys
   if keys.contains(key){
     ....
   } 
 }
}
```
### Camera相關(後鏡頭為例)
- 取得手機支援的Camera，這裡的Camera都是虛擬Camera概念(除了builtInWideAngleCamera，可由device.isVirtualDevicet查詢)，各個Camera都是由不同的實體Camera組成，詳情洽下方各設備資訊
```//順序依Camera新->舊排序
let deviceTypes: [AVCaptureDevice.DeviceType] = [
    //builtInTripleCamera: a system that switches automatically among the three lens, etc Pro
    .builtInTripleCamera,
    //builtInDualWideCamera: a system that switches between 0.5x and 1x as appropriate, etc: Pro, i11
    .builtInDualWideCamera, 
    //builtInDualCamera: a system that switches between 1.0x and telephoto lens (2x, 2.5x, or 3x depending on the phone model), etc: x, Pro
    .builtInDualCamera, 
    //builtInWideAngleCamera: 1x single-lens camera, etc: i6, i7, x, i11, 14, 14pro
    .builtInWideAngleCamera,
]
//Supported Device Types
discoverySession = AVCaptureDevice.DiscoverySession(deviceTypes: deviceTypes, mediaType: .video, position: .back)
//取得第一個(最佳)Camera，或依需求自行選擇組合
videoDevice = discoverySession.devices.first
```

- **Overview** 
```
                Triple Camera   Wide Dual Camera    Dual Camera     Wide Camera
iPhone 14 Pro   Yes             Yes                 Yes             Yes
iPhone 13                       Yes                                 Yes
iPhone 11                       Yes                                 Yes
iPhone X                                            Yes             Yes
iPhone 6s                                                           Yes

```
- **iPhone14 Pro:** 
```
Supported Types: 
=======================
[Back Triple Camera]
[Back Dual Wide Camera]
[Back Dual Camera]
[Back Camera]
=======================

builtInTripleCamera
=======================
//實體鏡頭組合
constituentDevices: [[Back Ultra Wide Camera], [Back Camera], [Back Telephoto Camera]]
//觸發切換鏡頭的Zoom門崁，第一個鏡頭門崁不會列入表內，故2: [Back Camera], 6: [Back Telephoto Camera]
virtualZoomFactors: [2, 6]

builtInDualWideCamera
=======================
constituentDevices: [[Back Ultra Wide Camera], [Back Camera]]
virtualZoomFactors: [2]
isVirtualDevice: true

Dual Camera
=======================
constituentDevices:  [[Back Camera], [Back Telephoto Camera]]
virtualZoomFactors: [3]
isVirtualDevice: true

builtInWideAngleCamera
=======================
constituentDevices: []
virtualZoomFactors: []
isVirtualDevice: false
```

- **iPhone11:**
```
deviceTypes: 
=======================
[Back Dual Wide Camera]
[Back Camera]
=======================

builtInDualWideCamera
=======================
constituentDevices: [[Back Ultra Wide Camera], [Back Camera]]
virtualZoomFactors: [2]
isVirtualDevice: true

builtInWideAngleCamera
=======================
constituentDevices: []
virtualZoomFactors: []
isVirtualDevice: false
```

- **iPhoneX:**
```
deviceTypes: 
=======================
[Back Dual Camera]
[Back Camera]
=======================

builtInDualWideCamera
=======================
constituentDevices: [[Back Camera], [Back Telephoto Camera]]
virtualZoomFactors: [2]
isVirtualDevice: true

builtInWideAngleCamera
=======================
constituentDevices: []
virtualZoomFactors: []
isVirtualDevice: false
```
- 首次啟動鏡頭，videoZoomFactor都為1x, 若是新型手機，取得的實體Camera可能為Ultra Wide，畫面會變魚眼，故需要依當下已選譯虛擬Camera，找出觸發WideCamera的zoomFactor並指定，畫面才會正常
```
//一定要等到加到Input，才會套用到新的設定
session.addInput(videoDeviceInput)
try? device!.lockForConfiguration()
//virtualZoomFactors的數量若大於0，代表有超過1個以上的實體camera, isVirtualDevice = true; 
//virtualZoomFactors若為空，代表只有1個實體camera, isVirtualDevice = false

//virtualZoomFactors的數量 = constituentDevices數量-1，因為第一個實體Camera的zoomFactor不會列在virtualZoomFactors裡面
//每一種機型的實體Camera陣列(constituentDevices)順序不相同，要找出目標實體Camera(WideCamera)的index並依該index取得觸發該Camera的值(zoomFactor)

guard let virtualZoomFactors = device?.virtualDeviceSwitchOverVideoZoomFactors,
      virtualZoomFactors.count > 0,
      //找出wideCamera的index
      let wideCameraIndex = device?.constituentDevices.firstIndex(where: {
          $0.deviceType == .builtInWideAngleCamera
      }),
      wideCameraIndex != 0
else{return}

//因為第一個實體Camera的觸發zoomFactor不會列在virtualZoomFactors裡面，所以wideCamera的camera index要減1
let triggerWideCameraFactor = device!.virtualDeviceSwitchOverVideoZoomFactors[wideCameraIndex - 1]
//設定zoomFactor => 觸發實體鏡頭為WideCamera => 畫面正常
device!.videoZoomFactor = CGFloat(truncating: triggerWideCameraFactor)

device!.unlockForConfiguration()
```