## Combine學習筆記
- Combine基本元素Publisher Protocol, 與RxSwift基本元素Event<Element>不同
```
//Combine
protocol Publisher<Output, Failure>

//RxSwift
public enum Event<Element> {
    case next(Element)
    case error(Swift.Error)
    case completed
}
```
- Publishers's `completion`事件可能為`successful`或`failure`
- `assign(to:on:)`可用來binding UIKIT(但支援並不全面)，因為可以用來bind keypath, 只能用來處理Failure = Never的情形
- `assign(to:)`可用來binding @Published。
```
class SomeObject {
   @Published var value = 0
}
    
let object = SomeObject()
//使用前綴字元:$ 取得@Publisher屬性底下的publisher
object.$value.sink {
    print($0)
}
    
(0..<10).publisher
//前綴字元&表示inout reference
   .assign(to: &object.$value)
```
- `assign(to:on:)`會回傳一個Anycancel, assign(to:)不會，因為`assign(to:)` binding的@Published變數, 當該變數deinitialized時，subscription會自己取消， There is one tricky part about `assign(to:on:)` — It’ll strongly capture the object provided to the on argument.
- 下方案例會造成StrongReference, 改成 assign(to: &$word) 就可解決
```
//Strong Reference
class MyObject {
  @Published var word: String = ""
  var subscriptions = Set<AnyCancellable>()

  init() {
    ["A", "B", "C"].publisher
      .assign(to: \.word, on: self)
      .store(in: &subscriptions)
  }
}

//Fix
class MyObject {
  @Published var word: String = ""
  var subscriptions = Set<AnyCancellable>()

  init() {
    ["A", "B", "C"].publisher
      .assign(to: &$word)
  }
}
```
- 目前未知使用subscriber或custom subscriber的時機為何，已知可以.subscribe(subject)
```
let sourcePublisher = PassthroughSubject<Date, Never>()
let subscription = Timer
  .publish(every: 1.0 / valuesPerSecond, on: .main, in: .common)
  .autoconnect()
  .subscribe(sourcePublisher)
```
- 已知手動publisher.subscribe(subscriber)，沒有回傳值；跟publisher.sink的回傳值為AnyCancellable不同
- `Future`是產生一個值後馬上結束或fail, 但api或許不適用，因為一創建就執行，不須等人訂閱
```
func createFuture() -> Future<Int, Never> {
  return Future { promise in
    print("Closure executed")
    promise(.success(42))
  }
}

let future = createFuture()
// prints "Closure executed
```
搭配`Deferred`可以達成等到有人訂閱才執行；Deferred是struct, Future是class，會造成有人訂閱後，就產生新的Future
```
func createFuture() -> AnyPublisher<Int, Never> {
  return Deferred {
    Future { promise in
      print("Closure executed")
      promise(.success(42))
    }
  }.eraseToAnyPublisher()
}

let future = createFuture()  // nothing happens yet

let sub1 = future.sink(receiveValue: { value in 
  print("sub1: \(value)")
}) // the Future executes because it has a subscriber

let sub2 = future.sink(receiveValue: { value in 
  print("sub2: \(value)")
}) // the Future executes again because it received another subscriber
```
- Future如果有兩個以上的subscribe，它只會執行一次，並且share結果，跟RxSwift的Single很像，但有下方幾點不同. 
```
A Future will begin executing immediately when you create it.
A Future will only run its supplied closure once.
Subscribing to the same Future multiple times will yield in the same result being returned.
A Future in Combine serves a similar purpose as RxSwift's Single but they behave differently.
```
- Subscriber的`.max(2)`，是指加上可接受兩個元素
- `publisher`, `subject`, `future`, 可配合`async/await`，監聽`values`(publisher, subject), `value(future)`或屬性
```
func generateAsyncRandomNumberFromFuture() -> Future <Int, Never> {
    return Future() { promise in
        DispatchQueue.main.asyncAfter(deadline: .now() + 2) {
            let number = Int.random(in: 1...10)
            promise(Result.success(number))
        }
    }
}

//use await
let number = await generateAsyncRandomNumberFromFuture().value

```

```
let subject = PassthroughSubject<Int, Never>()

Task{
//values returns an asynchronous sequence
    for await element in subject.values{
        print("element:\(element)")
    }
    print("complete")
}

subject.send(1)
subject.send(2)
subject.send(3)
subject.send(completion: .finished)

```
- async-await有一個語法類似Combine的Future,  `withCheckedContinuation(function:_:)` & ` withCheckedThrowingContinuation(function:_:)`
```
func generateAsyncRandomNumberFromContinuation() async -> Int {
    return await withCheckedContinuation { continuation in
        DispatchQueue.main.asyncAfter(deadline: .now() + 2) {
            let number = Int.random(in: 1...10)
            continuation.resume(returning: number)
        }
    }
}

let asyncRandom = await generateAsyncRandomNumberFromContinuation()
```
- `map`可直接map keypath直接使用，最多三個，例如輸入coordinate, 使用.map(\.x, \.y)接; 但`tryMap`沒有keypath可接!
- `prepend`除了可以接元素，也可以接publisher
- `switchToLatest`主要使用在發送publisher的publisher, 主要用的情境為，使用者點擊後打API, API尚未回應，使用者又點擊了，即可取消上一次完成的request, 開始新的request,類似RxSwift的flatmapLatest
- 點擊的訊號 
```
let taps = PassthroughSubject<Void, Never>()
```
- `merge`裡面，所有的publish必須finished，merge才會`complete`, `combineLatest`同樣
- `Timer.publish`為`connectable`屬性，故後面要接`autoconnect()`才會啟動
- `collect`可以填數字外，也可以填 `.byTime` 或 `.byTimeOrCount`
- `debounce`(防抖)是該元素發出後，超過t秒內，未有新元素發出，就送出，使用場景: 搜尋輸入框的推薦關鍵字功能
- `throttle`(節流)第一次觸發後，在t秒內的所有觸發都不算，適合每隔一段時間才能被執行一次
- debug時除了用print,`handleEvents`也是好方法，專業一點就用`breakpointOnError()`，會直接觸發XCode的中斷處理，也要條件中斷點的`breakpoint(receiveOutput:)`可用
- 大部分`publisher`都是`struct`，但如果使用`share()`, 會回傳`Publishers.Share`；反之`PassthroughSubject`, `CurrentValueSubject`, `Future`為`Class`，就可以存取ref共享值；第一個訂閱者會觸發啟動，後續的訂閱者只是connect進來；若太晚訂閱，但不會收到上層已發射的值，頂多只會收到completion.除非使用RxSwift的ShareReplay(), 或搭配轉換成`makeConnectable()`
按照doc顯示，share()是PassthroughSubject結合multicast，autoconnect()的結合體. 思考使用CurrentValueSubject搭配autoconnect&multicast，是否可達到ShareReplay的目的。
- 如果publisher的Failure為Never的話, Sink可以有只處理元素.sink(receiveValue:而不用管complete的方法，不然都一定要處理complete
- assertNoFailure 可以開發用, 若有錯誤扔出，即丟出fatal Error
- map會攜帶明確error type,在sink裡面可以清楚的處理completion的failure type, 但tryMap會清掉error type, 變成單純的error,只好搭配mapError用 error as? SomeErrow 處理，或 switch case: case is SomeError: ..., mapError要放在最尾端, mapError的功能是把不同類型的error轉為同一種類型的error後發出.
- combine也有支援NSObject的KVO監聽, 前提 1. 須為class，2. 須繼承`NSObject` 3. 屬性(簡易屬性、陣列、字典等必須能跟ObjectC呈現)要加上`@objc dynamic`(While the Swift language doesn’t directly support KVO, marking your properties @objc dynamic forces the compiler to generate hidden methods that trigger the KVO machinery.)
```
class TestObject: NSObject {
  @objc dynamic var integerProperty: Int = 0
}

let obj = TestObject()
let subscription = obj.publisher(for: \.integerProperty)
  .sink {
    print("integerProperty changes to \($0)")
  }

obj.integerProperty = 100
obj.integerProperty = 200
```
- ObservableObject會自動產生objectWillChange方法, 可以給combinen用，其中的@published屬性改變，可以收到通知, element是void, failure是Never,可惜無法知道是哪個屬性更新，適合用來觸發swift更新，
```
class MonitorObject: ObservableObject {
  @Published var someProperty = false
  @Published var someOtherProperty = ""
}

let object = MonitorObject()
let subscription = object.objectWillChange.sink {
  print("object will change")
}

object.someProperty = true
object.someOtherProperty = "Hello world"
```

**CombineExt(CombineCommunity/CombineEx)**
- 可以用陣列的方式使用merge, zip, combinelatest，
- 改善subject.assign會有strong ref的缺點，
- 不使錯誤中斷的materialize，可用value或failure過濾結果，會改送Event(但event是combine內建的還是自訂的?待求證)，mapToResult可達到相同目的?，
- 處理元素是陣列的mapMany & zipMany & filterMany, 
- 使用ignoreOutput後, 元素會轉型為NEVER(實證後好像不會?), 但使用setOutputType，可在轉回來(是好是壞?)有方便的ignoreOutput(setOutputType:)可用,
- 移除元素內所有相同的元素(已完結的publisher)removeAllDuplicates
- 好用的share(replay:)可用
- 有自訂的AnyPublisher.create可用
- 有flatMapLatest，與switchToLatest類似，只是不用_接一個值，小方便
- 不會發生錯誤會不需要帶錯誤型態的CurrentValueRelay & PassthroughRelay, 除非被deinit, 不然無法手動發出complete(所以無法送出錯)
- 有replaySubject可用
- 串聯如果中間出錯，即使用.replaceError(with處理，串聯會改為正常結束; 使用replaceError(with:), Failure會變Never,待測使用CombineExt的ignoreFailure(completeImmediately:)是否遇到錯誤可避免中斷, 如果改用cache:(重發替代Publisher), 可以改丟一個Empty(): 一個不會送出元素，也不會結束(可設定)的publisher
- collect(count)不須等到publisher結束就會發出
- 在API實務上, 常常會遇到api回的錯誤訊息，不會送出URLError的404, 而是送出自訂的錯誤Json, 這時在combine可用tryMap, 裡頭用JSONSerialization.jsonObject(with: data)(或SwifyJson)解析dic自訂錯誤訊息Json，並送出自訂API的錯誤；實務2: 如果要在api裡處理內建error或未知error，然後發出自訂error, 可使用is搭配default, ex:
```
.mapError{ error -> DadJokes.Error in
    switch error {
    case is URLError:
        return .network
    case is DecodingError:
        return .parsing
    default:
        return error as? DadJokes.Error ?? .unknown
    }
}
```
- Combine如果要若扔出error, 可用Fail<Output, Failure>,例如: Fail<Joke, Error>(error: .jokeDoesntExist(id: id))
- retry()是嘗試重新resubscribe, 重建subscription
- 有些運算元只可用在不會失敗的publisher, 如 sink(receiveValue:), setFailureType, assign(to:on:)
- API實務中，儘可能將外部可能發生的錯誤全部統一為單一自訂錯誤, 一是使用方便，二為隱藏內部的實作方式
- ObservableObject是讓普通的ViewModel, 轉化成SwiftUI-View可以讀取更新的protocol
- CurrentValueSubject適合用來表示狀態的情境，PassthroughSubject適合用來表示事件的情境
- 若要存取@Published的publisher型式的值，前面要加$
- Combine的timeout運作方式與常用的timeout不同：若超過時間沒有收到元素，則會強制完成(無錯誤)；若要噴出錯誤，必須帶自訂的Error參數 
```
//3秒後完成
srcPublisher
    .timeout(.seconds(3), scheduler: DispatchQueue.main)
    .sink { completion in
        print("completion:\(completion)")
    } receiveValue: { _ in ()}
    .store(in: &subscriptions)


//3秒後噴錯
enum APIError: Error{
    case timeout
}

srcPublisher
	.timeout(.seconds(3), scheduler: DispatchQueue.main, customError: {.timeout})
    .sink { completion in
        print("completion:\(completion)")
    } receiveValue: { _ in ()}
    .store(in: &subscriptions)

```
- .measureInterval用來衡量元素之間的時間, value值本為nanoseconds, 轉為seconds較好閱讀
```
let srcPublisher = PassthroughSubject<Void, Never>()
let measureSubject = srcPublisher.measureInterval(using: DispatchQueue.main)

measureSubject
    .sink { value in
        print((Double(value.magnitude) / 1_000_000_000.0))
    }
    .store(in: &subscriptions)

srcPublisher.send()
DispatchQueue.main.asyncAfter(deadline: .now() + 3){
    srcPublisher.send()
}
```
- combine獨有: `output(at:)`，取得特定索引的值發出
- combine獨有: `output(in:)`，取出特定範圍的值，並逐一發出元素，非打包發出
- `contains(` 找到符合值即馬上結束並回傳true值
- CurrentValueSubject的value值可以直接被修改，不局限只能用subject.send()更新值
- `.decode`可直接用來解析API的JSON值, 可丟出一個錯誤, 搭配mapError
```
//without .decode
URLSession.shared
  .dataTaskPublisher(for: url)
  .tryMap { data, _ in
    try JSONDecoder().decode(MyType.self, from: data)
  }

//with built-in decode
URLSession.shared
  .dataTaskPublisher(for: url)
  .map(\.data)
  .decode(type: MyType.self, decoder: JSONDecoder()
  .mapError{error -> API.Error in
       switch error{
          case is URLError:
	         return Error.addressUnreachable(EndPoint.stories.url)
 			default:
             return Error.invalidResponse
           }
   }


```
- print可加入時間點，方便debug, 在print參數加入TextOutputStream協定
```
class TimeLogger: TextOutputStream {
  private var previous = Date()
  private let formatter = NumberFormatter()

  init() {
    formatter.maximumFractionDigits = 5
    formatter.minimumFractionDigits = 5
  }

  func write(_ string: String) {
    let trimmed = string.trimmingCharacters(in: .whitespacesAndNewlines)
    guard !trimmed.isEmpty else { return }
    let now = Date()
    print("+\(formatter.string(for: now.timeIntervalSince(previous))!)s: \(string)")
    previous = now
  }
}

let subscription = (1...3).publisher
  .print("publisher", to: TimeLogger())
  .sink { _ in }

//+0.00111s: publisher: receive subscription: (1...3)
//+0.03485s: publisher: request unlimited
//+0.00035s: publisher: receive value: (1)
//+0.00025s: publisher: receive value: (2)
//+0.00027s: publisher: receive value: (3)
//+0.00024s: publisher: receive finished

```
- publisher大都是struct，如果使用Share()，則以傳遞Reference取代傳遞value，如果搭配CurrentValueSubject可以達到shareReplay效果
- 過濾資料群中，是否有符合關鍵字群的聰明作法
```
var allStories = [Story]()
var filter = [String]()

var stories: [Story] {
    guard !filter.isEmpty else {
        return allStories
    }
    return allStories
        .filter { story -> Bool in
            return filter.reduce(false) { isMatch, keyword -> Bool in
                return isMatch || story.title.lowercased().contains(keyword)
            }
        }
}
}
```
- `sink(receiveValue:)`, `setFailureType`, `assign(to:on:)`只適用在 Failure 為 Never的前提
- $號前後的差別
```
class WeeklyWeatherViewModel: ObservableObject{
	@Published var city: String = ""
}

$viewModel.city // Binding<String>
viewModel.$city // Published<String>.Publisher

```