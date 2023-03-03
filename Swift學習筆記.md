## Swift學習筆記

- 當Class實作Hashable時，必當實作Equatable, 因為實作Equatable，因為不為valueType, 所以要用 === 判斷
- Class instance自動實作Identifiable
- 可以在Init裡面New實體
```
public init(jokesService: JokeServiceDataPublisher = JokesService()){...}
```
- 如果要取用本身class的static 屬性，使用大寫的Self
- `RelativeDateTimeFormatter()`可以回傳好閱讀的時間區間，如 “1 hour ago”, “in 2 weeks”, “yesterday”
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
- async / await最大的好處，再也不用譫心 weak或strongly capture self
- async / await Async的相關class大部分都可以 throws error，所以使用大部分都用try catch
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
- decode時轉換Key值型態
```
let decoder = JSONDecoder()
decoder.keyDecodingStrategy = .convertFromSnakeCase
```
- 可用extension的方式為protocol實現基本實作, 類似c#定義interface後，Abstract Class定義基本方法的概念
- Task實體產生一個非同步的closure給async使用, 非必要, Task實體可用來與之非同步工作互動，如取消。 當你不再參照Task實體時，裡頭的非同步工作依然執行中
- `Result<Data, Error>` 代表一個結果，成功可帶值，失敗可帶error，常用來回傳api, 跟Combine裡面的Publisher<Int, Never>類似
```
func perform(_ request: URLRequest, completionHandler: @escaping (Result<Data, Error>) -> ())...
```
- 當要做 cache, cookie storage or credential storage時，不該使用URLSession.shared
- switch case let .upc(let numberSystem, let product, let check): 可改寫為 case let .upc(numberSystem, product, check)...

Error相關
---
- 如果error沒有實作equaltable, 要檢查error type, 可用 case MyError.someError = error
- 如果要用switch case判斷error類型，但error後面會附加一些資訊，無法使用傳統方式辨別，可用case _ where error.hasPrefix("can not find user_id"): 判斷
- 過濾出錯誤可用case .failure(let error) = $0, 只會有一個等號: 
```
guard case .failure(let error) = $0 
else {return}
```
- 使用`@MainActor`自动在主线程更新UI, 只能运行在async/await环境中
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
- 使用`Logger`類別，已打包的程式可在"系統監視程式"可見; 另外好處是，在loggerd印出自訂的值，為了安全，在已打包的版本，自訂的值會印出`<private>`
```
private let logger = Logger(subsystem: "com.apple.sample.photogrammetry", category: "HelloPhotogrammetry")
logger.log("Successfully...")
logger.error("Program terminated...")
```
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
- ProjectSetting內建變數: `$(SRCROOT)`, `$(PROJECT_DIR)`
- 把vc嵌入另一個vc
```
parentView.addSubview(childVC.view))
//建立子母關系，確保功能都正常
parentVC.addChild(childVC)
childVC.didMove(toParent: self)
```
- `autoreleasepool`: 每一輪 run loop 中，如果某些物件只有在這一輪 run loop 中有用，之後就應該釋放，我們可以先把物件放進 auto-release pool 裡頭，等到這一輪 run loop 的時候，再把 auto-release pool 倒空。
[Reference](https://zonble.gitbooks.io/kkbox-ios-dev/content/responder/run_loop.html)
- 計算func執行時間: 使用`CFAbsoluteTimeGetCurrent()`:
```
let start = CFAbsoluteTimeGetCurrent()
//some slowly work...
print("elapse time:\(CFAbsoluteTimeGetCurrent() - start)")
```
- 觸發UIView(SCNView), 重新更新畫面: `setNeedsDisplay()`

- 動態產生vc測試效果, 使用preferredContentSize控制大小
```
let vc = ViewController()
vc.modalPresentationStyle = .formSheet
vc.preferredContentSize = .init(width: 500, height: 800)
present(vc, animated: true)
```

RxSwift相關
---
- 新版Rx有這個回收方法
```
_ =
sequence
    .take(until: self.rx.deallocated)
    .subscribe {
        print($0)
    }
```

- 檢查MemoryLeak, 升級到6.2
```
在Pods -> RxSwift-> building setting -> Other Swift Flags -> Debug新增 -D TRACE_RESOURCES
可使用RxSwift.Resources.total
```
- URLSession extensions don't return result on MainScheduler by default.
- 新的decode: .decode(type: [User].self, decoder: JSONDecoder())
- observable:withUnretained(self), 但無法優雅處理onError & onComplete，因為還是要加[weak self]:
```
observable
        .withUnretained(self)
        .subscribe { (owner, string) in
            owner.doSomething(string)
        } onError: { [weak self] (error) in
            guard let self = self else { return }
            self.handleError(error)
        } onCompleted: { [weak self] () in
            guard let self = self else { return }
            owner.handleDone()
        }
        .disposed(by: bag)
```
此時可改用Subscribe(with:):
```
observable
        .subscribe(with: self) { (owner, string) in
            owner.doSomething(string)
        } onError: { (owner, error) in
            owner.handleError(error)
        } onCompleted: { (owner) in
            owner.handleDone()
        }
        .disposed(by: bag)
```
drive不適用, 可改用.subscribe(with: <#T##Object#>, onNext:...(observable & drive皆適用)
```
/*You can also use withUnretained with other RxSwift operators to avoid the weak self retain cycle issue:*/
func setupLogin(_ submit: Observable<(String?, String?)>) -> Observable<User> {
    return submit
        .withLatestFrom(credentials)
        .withUnretained(self)
        .flatMap { (model, creds) -> Observable<String> in
            model.authorize(username: creds.0, password: creds.1)
        }
        .withUnretained(self)
        .flatMap { (model, token) -> Observable<User> in
            model.user(authorization: token)
        }
}
```
- Infallible(不會出錯，只有.next(element)跟.completed), 跟Driver&Signal差別: Driver跟Signal只使用在MainScheduler，而且share元素(share()); Infallible為單純不會出錯observable
- bind & drive 可多重binding
```
viewModel.string.bind(to: input1, input2, input3)
viewModel.string.drive(input1, input2, input3)
viewModel.number.emit(input4, input5)
```
- distinctUntilChange 可用Key Paths
```
//origin:
myStream.distinctUntilChanged { $0.searchTerm == $1.searchTerm }

//new:
myStream.distinctUntilChanged(at: \.searchTerm)
```
- new ReplayRelay
```
ReplayRelay<Int>.create(bufferSize: 3)
```
- DisposeBag新的使用方式，用Wrapper的方式將subscriber包起來
```
var disposeBag = DisposeBag {
     observable1.bind(to: input1)
    observable2.drive(input2)
    observable3.subscribe(onNext: { val in 
        print("Got \(val)")
    })
}

// Also works for insertions
disposeBag.insert {
    observable4.subscribe()
    observable5.bind(to: input5)
}
```
- `Observable:Single`改回傳`Result<Element, Swift.Error>`
- Rx 6.5 可與async/await合用, [Reference](https://github.com/ReactiveX/RxSwift/blob/main/Documentation/SwiftConcurrency.md)
- [What's new in RxSwift 6 ?](https://dev.to/freak4pc/what-s-new-in-rxswift-6-2nog)
- [Using withUnretained in RxSwift 6.0](https://betterprogramming.pub/using-withunretained-in-rxswift-6-0-8e3e221b37ee)
- closure宣告
```
(param) -> Void
```
相等於
```
({ param in
})
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
- 不錯的Formatter
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

do{
	throw "This is sample error"
}catch{
}

```
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