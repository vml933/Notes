## Combine學習筆記

- `assign(to:on:)`可用來binding UIKIT(但支援並不全面)，因為可以用來bind keypath, 只能用來處理Failure = Never的情形
- `assign(to:)`可用來binding @Published，使該value重新轉送出去。如果是自身的publish變數記得前綴有兩個符號 &$param, 如果是別人的，記得 &object.$value
- `assign(to:on:)`會回傳一個Anycancel, assign(to:)不會，因為`assign(to:)` binding的@Published值會自己處理Life Cycle; There is one tricky part about `assign(to:on:)` — It’ll strongly capture the object provided to the on argument.
- 下方案例會造成StrongReference, 改成 .assign(to) 就可解決
```
class MyObject {
  @Published var word: String = ""
  var subscriptions = Set<AnyCancellable>()

  init() {
    ["A", "B", "C"].publisher
      .assign(to: \.word, on: self)
      .store(in: &subscriptions)
  }
}
```
- 目前未知使用subscriber或custom subscriber的時機為何，已知可以.subscribe(subject)
- 已知手動publisher.subscribe(subscriber)，沒有回傳值；跟publisher.sink的回傳值為AnyCancellable不同
- Futrue是產生一個值後馬上結束或fail, 但api或許不適用，因為一創建就執行(greedy)，視情況使用
- (備註)future is greedy, Future當它創建完後馬上執行，不用像publisher(lazy)需要subscriber, 如果有兩個以上的subscribe，它只會執行一次，並且share結果
- Subscriber的.max(2)，是指加上可接受兩個元素
- publisher, futures, subjects 可配合async/await!
- map可直接map keypath直接使用，最多三個，例如輸入coordinate, 使用.map(\.x, \.y)接; 但tryMap沒有keypath可接!
- prepend除了可以接元素，也可以接publisher
- switchToLatest主要使用在發送publisher的publisher, 主要用的情境為，使用者點擊後打API, API尚未回應，使用者又點擊了，即可取消上一次完成的request, 開始新的request,類似RxSwift的flatmapLatest
- 點擊的訊號用 let taps = PassthroughSubject<Void, Never>()
- merge後裡面，所有的publish必須finished，merge才會complete, combineLatest同樣
- Timer.publish 為 connectable屬性，故後面要接autoconnect()才會啟動
- collect可以填數字外，也可以填 .byTime 或 .byTimeOrCount
- debounce(防抖)是該元素發出後，超過t秒內，未有新元素發出，就送出，使用場景: 搜尋輸入框的推薦關鍵字功能
- throttle(節流)第一次觸發後，在t秒內的所有觸發都不算，適合每隔一段時間才能被執行一次
- debug時除了用print, handleEvents也是好方法，專業一點就用breakpointOnError()
- 大部分publisher都是struct，但如果使用share(), 會回傳Publishers.Share，是class(PassthroughSubject & CurrentValueSubject, Future也是class)，就可以存取ref共享值；第一個訂閱者會觸發啟動，後續的訂閱者只是connect進來；若太晚訂閱，但不會收到上層已發射的值，頂多只會收到completion.除非使用RxSwift的ShareReplay()
按照doc顯示，share()是PassthroughSubject結合multicast，autoconnect()的結合體. 思考使用CurrentValueSubject搭配autoconnect&multicast，是否可達到ShareReplay的目的。
- 如果publisher的Failure為Never的話, Sink可以有只處理元素.sink(receiveValue:而不用管complete的方法，不然都一定要處理complete
- assertNoFailure 可以開發用
- map會攜帶明確error type,在sink裡面可以清楚的處理completion的failure type, 但tryMap會清掉error type, 變成單純的error,只好搭配mapError用 error as? SomeErrow 處理，或 switch case: case is SomeError: ..., mapError要放在最尾端, mapError的功能是把不同類型的error轉為同一種類型的error後發出.
- combine也有支援NSObject的KVO監聽
- ObservableObject裡面的published屬性改變，居然有objectWillChange.sink可以用, 只是element是void, failure是Never, 適合用來觸發swift更新

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
- 串聯如果中間出錯，即使用.replaceError(with處理，串聯會改為正常結束; 使用replaceError(with:), Failure會變Never,待測使用CombineExt的ignoreFailure(completeImmediately:)是否遇到錯誤可避免中斷, 如果改用cache, 可以改丟一個Empty(): 一個不會送出元素，也不會結束(可設定)的publisher
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
- Combine如果要若扔出error, 可用Fail<Output, Failure>,例如: Fail<Joke, Error>(error: .jokeDoesntExist(id: id)).eraseToAnyPublisher()
- retry()是嘗試重新resubscribe, 重建subscription
- 有些運算元只可用在不會失敗的publisher, 如 sink(receiveValue:), setFailureType, assign(to:on:)
- API實務中，儘可能將外部可能發生的錯誤全部統一為單一自訂錯誤, 一是使用方便，二為隱藏內部的實作方式
- ObservableObject是讓普通的ViewModel, 轉化成SwiftUI-View可以讀取更新的protocol
- CurrentValueSubject適合用來表示狀態的情境，PassthroughSubject適合用來表示事件的情境
- 若要存取@Published的publisher型式的值，前面要加$






