# SwiftUI學習筆記

## Observation Framework

- iOS 17+有 Observation framework：`@Observable`可以使用，故`@StateObject`, `@ObservedObject`, `@Published`, `@ObservableObject`, `@EnvironmentObject` 已經棄用
- Observation framework 有 `@ObservationIgnored`，可控制標示`@Observable` class 內哪些屬性不必監聽，一些情境可以配合`@Bindable`.
- `@Bindable`: 讓`@Observable`的物件生成`@Binding`提供修改，經常放在`some View`裡面
```swift
struct TitleEditView: View {
    @Environment(Book.self) private var book

    var body: some View {
        @Bindable var book = book
        TextField("Title", text: $book.title)
    }
}
```

### 參考資源
- [A Deep Dive Into Observation: A New Way to Boost SwiftUI Performance](https://fatbobman.com/en/posts/mastering-observation/)
- [Migrating from the Observable Object protocol to the Observable](https://developer.apple.com/documentation/swiftui/migrating-from-the-observable-object-protocol-to-the-observable-macro)

### 關鍵概念
- Owning the reference, not the data
- Only properties that appear in the body and are read will trigger a view update

## 集合與識別符

- 當struct或class實作Hashable時，使用List或Foreach時，就可以取代 \.id為 \.self
- (If your data object implements Hashable, you can also tell SwiftUI to use the entire object as the unique identifier.)

## Property Wrappers

### @State
- State是一個property wrapper類型，可以被SwiftUI讀取與寫入
- State是Value Type，若當參數傳送，是copy by value
- 若class宣告成State, 則不會有作用，故State不適用於宣告ViewModel
- 當變數宣告為State，是指該View擁有該變數Data的Reference(存在其他儲存空間，由SwiftUI管理), 而非直接存取Data,(因為宣告@State的地方為Struct View，如果要指定值，complier會報錯，狀態刷新:重建View時，為了保持狀態同步，必須額外儲存空間)，當state的值改變時，view會重新complier body(SwiftUI re-render), @State被用來當single source of truth
- 擁有@Published特性
- 當宣告@State var myVar時， 編輯器背後自動產生 var myVar = State<Int>(initialValue: 0), 可透過(有底線) _myVar.wrappedValue 取得該值; 若要將該值放到Binding<T>的變數時，可用myVar.projectedValue。但如果是手動宣告var myVar = State<Int>(initialValue: 0)，變數值前面不用加底線
- `@State var myIndex` 等於 `_myIndex.wrappedValue`
- `$myIndex` 等於 `_myIndex.projectedValue` : `Binding<T>`
- 若將一個struct(內含許多property)，宣告成State，可正常運作，但沒效率，因為其中一個property改變，就會將整個struct的實體換掉，所有相關聯的UI都會Trigger Refresh，影響效能，請小心使用struct配合State
- 將其他線程取得的值(常見如:API)指定給State變數, SwiftUI會自動處理跳至主線程，不需手動處理

### @Environment
- `@Environment`取得 App 的屬性，如`window`, `colorScheme`; `View` 也可以同樣的方式取得相關屬性，如`font`..等
- `.environment(\.keyPath, VALUE)` 設定該View底下所有environment的值
- 在Observation framework, `@Environment`除了透過 Key Path方式取得內建屬性:`EnvironmentValues`，現在也可以用於自訂類型，取代原有的`@EnvironmentObject`:

#### 簡易宣告方式
```swift
//需標記Observable
@Observable
class Members {
    var list: [String] = ["helper1", "helper2", "helper3"]
}

@main
struct MyVibeCodingApp: App {
    @State private var members = Members()
    
    var body: some Scene {
        WindowGroup {
            LoginView()
                .environment(members)
        }
    }
}

struct LoginView: View {
    @Environment(Members.self) private var members
    //...
}
```

#### KeyPath宣告方式
```swift
//接近內建取得方式
@Observable
class Members {
    var list: [String] = ["helper1", "helper2", "helper3"]
}

// MARK: - Environment Key
private struct MembersKey: EnvironmentKey {
    static let defaultValue = Members()
}

extension EnvironmentValues {
    var members: Members {
        get { self[MembersKey.self] }
        set { self[MembersKey.self] = newValue }
    }
}

@main
struct MyVibeCodingApp: App {
    @State private var members = Members()
    
    var body: some Scene {
        WindowGroup {
            LoginView(router: Router())
                .environment(\.members, members)
        }
    }
}

struct LoginView: View {
    @Environment(\.members) private var members
    //...
}
```

### @ObservedObject (已棄用)
- 擁有@Published特性
- 變數有@ObservedObject時，代表變數為外部所擁有，用binding的方式相關聯，來源可能為母層或Environment object，通常用來監聽屬性變化用
- 注意! view本身不create ObservedObject，如果view本身有create ObservedObject，該view UI重刷時, ObservedObject也會重新建立

```swift
struct ParentView: View {
    @ObservedObject var viewModel = ParentViewModel()

    var body: some View {
        ChildView(viewModel: viewModel)
    }
}

struct ChildView: View {
    @ObservedObject var viewModel: ParentViewModel

    var body: some View {
        // Use viewModel here
    }
}
```

- 當你使用class-instance當作model時，除非重新指定instance,不然修改裡面的變數並不會trigger UI-Refresh
- ObservedObject配合@Published，當class實作ObservableObject時，裡頭的@Published都會自動實作requires:objectWillChange，並將Class資料從View移至外部，不為View所有，只是添加訂閱關係，不影響儲存，@ObservedObject裡頭的@Published屬性，在view中的運作有如State

```swift
//@Published score等同於非@Published的nonPubishScore, 手動呼叫objectWillChange,即使View層沒有再監聽這個變數，View層依然會觸發一次re-render
class Model: ObservableObject {
    @Published var score: Int = 0
    
    var nonPubishScore: Int = 0 {
        didSet {
            objectWillChange.send()
        }
    }
    
    init() {
        print("Model created")
    }
}
```

- 因為Publisher運作有如State，若使用@Published的屬性必須value type: 基本型態或struct

### @StateObject (已棄用)
- StateObject是Reference type，用來宣告該屬性被View擁有，是@State的兄弟版(Value type)
- 有別於@ObservedObject，不會被UI重刷而重建，只會被建立一次
- Lifecycle與View綁定，適合用來當View的ViewModel
- @Published在哪個thread被建立，就只能用在該thread，除非搭配`receive(on:)`切換thread監聽

```swift
struct MyView: View {
    @StateObject private var viewModel = MyViewModel()

    var body: some View {
        // Use viewModel here
    }
}
```

#### @StateObject vs @ObservedObject 差別:

相同點:
- 兩者都必須實作ObservableObject, 並配合@Published

差異點:
- @ObservedObject: 會隨著View的更新多次建立, 主要發生在View與Model(external)之間的綁定(View向Model訂閱，Model向View通知)
- @StateObject: 生命週期不會follow該view，提供穩定性

### 其他Property Wrappers

- @AppStorage: 直接對UserDefaults讀寫並帶有觸發更新UI功能
- @EnvironmentObject: 將某個Object拉出環境並儲存為一個屬性
  - 必須是實作ObservableObject的Class
  - 對所有子View有效
  - 必須已經initialized完成
  - 使用@EnvironmentObject不需要再重新初始化

## Swift語法技巧

```swift
// case let範例:
guard
  let character = title.first,
  case let symbolName = "\(character.lowercased()).square",
...
```
## 佈局技巧

- 如果frame長寬設定為`.infinity`，將會撐滿父層
- 如果struct裡面的屬性都為Hashable的話，complier會認為該struct可實作hashable
- 如果使用NavigationView會出現breaking constraint之類的問題，可以加上以下，避免報錯:
```swift
NavigationView {
  // ...
}
.navigationViewStyle(.stack)
```
- `HStack` 與 `LazyHStack` 差異: HStack 以 `Child View` 大小為主；LazyHStack 不知道稍後讀取的Child View多大，則會撐滿`Parent View`，外層需有`ScrollView`才有效。

### List vs ForEach差別
- List: 針對特定情境使用：呈現一張直立表，但並不表示一定需要重覆的資料
  - List裡頭可以包ForEach與其他單項View
  - 採用lazy loading
  - 會重用child view
- ForEach: 需要用List或VStack包起來
  - 一次產生全部項目
  - 產生全部的child view

## UI元件技巧
- Button可點選範圍
```swift
//只有文字可點擊
Button {
    //...
} label: {
    Label("create New", systemImage: "plus")
}
.frame(maxWidth: .infinity)

//所有範圍可點擊
Button {
    //...
} label: {
    Label("create New", systemImage: "plus")
    .frame(maxWidth: .infinity)
}

```
- `ContentUnavailableView` 用來表示無效或空狀態的View，依使用者 Color Scheme 自動變色，內建 .search (空資料)，也可自訂標題圖案
- 使用 Date() 可建立簡易的倒數元件, 但無法控制，使用`TimelineView`較方便
```swift
// Date()
let interval: TimeInterval = 30
Text(Date().addingTimeInterval(interval), style: .timer) //0:29
Text(Date().addingTimeInterval(interval), style: .timer) //29 sec

// TimelineView
TimelineView(.animation(minimumInterval: 1.0,paused: timeRemaining <= 0)) { context in
        Text("\(timeRemaining) seconds")
        timeRemaining -= 1
    }
    .onChange(of: timeRemaining) {
      if timeRemaining < 1 {
        print("done")
      }
    }
```
- Button使用`.buttonStyle(.borderless)`可以讓裡面的View置中
- `.sheet(isPresented: $addingNewBook){ NewBookView()}`可以寫成`.sheet(isPresented: $addingNewBook, content: NewBookView.init)`
- `.sheet`的content可以配合`.presentationDetents`，控制sheet全營幕或半營幕
```swift
myButton
    .sheet(isPresented: $isShow) {
        SubView()
            .presentationDetents([.large, .medium])
    }

```
- 要把ToolBar秀出來，可以包在NavigationView裡面
- @Published someProperty可以用$存取, 即使該class沒有繼承ObservableObject
- 使用assign(to: \.someProperty, on: self)可以將發出的元素綁定在自身的@Published值(Republishing), 會回傳一個strong reference
- .onAppear {} - 當view出現時觸發
- .onReceive(_:perform:) - 當你需要publisher但不需要ObservableObject/ObservedObject時使用:
```swift
  struct ReceiverView: View {
      let timer = Timer.publish(every: 1.0, on: .main, in: .default).autoconnect()
      @State var time = ""

      var body: some View {
          Text(time)
              .onReceive(timer) { t in time = String(describing: t) }
      }
  }
```
- .onChange(of:) - 監聽@State值變化:
```swift
  struct PlayerView: View {
      var episode: Episode
      @State private var playState: PlayState = .paused

      var body: some View {
          Text(episode.title)
              .onChange(of: playState) { [playState] newState in
                  model.playStateDidChange(from: playState, to: newState)
              }
      }
  }
```
- 使用變數的方式安排小元件，加強易讀性:
```swift
struct CardsListView: View {  
  //變數
  private let content = RoundedRectangle(cornerRadius: 30)

  var body: some View {
    list 
      .fullScreenCover(isPresented: $isPresented) {
        ItemView()
      }
	  .foregroundStyle(color)
  }
  //變數
  var list: some View {
    ScrollView(showsIndicators: false) {
      VStack {
        ...
      }
    }
  }
	
}
```

## 非同步程式設計

- 當停止Task.Handle，並不會停止過程，只是標註"Task已取消"，必須自己實作取消過程
- iOS15 SwiftUI新增一個.task modifier, 當這個view出現時，執行非同步工作; 當View消失時，會cancel Task
- ContentView可設定Preview設備:
```swift
  struct ContentView_Previews: PreviewProvider {
    static var previews: some View {
      Group {
        CalculatorView()
          .previewDevice("iPhone Xs Max")
          .previewDisplayName("iPhone Xs Max")
        
        CalculatorView()
          .previewDevice("iPhone SE")
          .previewDisplayName("iPhone SE")
          .environment(\.colorScheme, .dark)
      }
    }
  }
```
[Reference](https://lawrey.medium.com/creating-a-swiftui-combine-alamofire-mvvm-login-example-df0bdc30ef25)

## 佈局差異

- ZStack & .Overlay差別: 
  - ZStack為動態Container, frame的尺寸完全基於內部內容，會由最大的子框架定義
  - overlay只會考慮第一個視圖定義的大小
- .task: 當SwiftUI出現後，執行非同步工作的modifier，適合用在非使用combine, 非使用MVVM的SwiftUI簡易情境

## 環境與狀態管理

- Environment的值通常用來讀取，不寫入值，宣告後，它就是該view的State屬性
- .EnvironmentObject(_:)用來創造自訂的Environment物件，將某個object注入至環境, 比較像是在SwiftUI層(View)的singleton模式
  - 注入的物件可以被該View與子View存取，但無法被母層存取
  - 特別注意，注入類別實體同時間只能存在一個，若其他地方再注入相同類別，便會取代上一個類別實體

## UI技巧與訣竅
- 使用 `.shadow`會把所有的子view都加上shadow, 可以加上`.background`之後再加`.shadow`顯示較正常
```swift
	MyView()
	.background(Color.primary.colorInvert().shadow(color: .primary.opacity(0.4), radius: 7))
```
- Ctrl + Option + Click點擊canvas畫面，出現元件設定選項
- 假若Object是一個@EnvironmentObject, 裡面有一個Compute Property: param, 若想要把這個param放到Binding<T>裡，可以用Binding的靜態方法.constant包起來，它可以由一個無法修改值初始化，ex:.constant(challengesViewModel.numberOfAnswered)
- [Environment值列表](https://developer.apple.com/documentation/swiftui/environmentvalues) - 可自訂environment屬性
- `@ViewBuilder`: 當view的body可能會產出多個view時使用:
```swift
  @ViewBuilder
  var body: some View {
      if verticalSizeClass == .compact {    
          VStack{...}
      } else {    
          VStack{...}
      }
  }
```
- `@ViewBuilder`: 自訂 Container View, `Group`, `HStack`, `VStack`...也是類似實作
```swift
    struct PaddingContainer<T: View>: View {
        
        let content: T
        
        init(@ViewBuilder content: () -> T) {
            self.content = content()
        }
        
        var body: some View {
            content
                .padding() //一率加padding
        }
    }

    PaddingContainer {
        Color.gray
    }

```
 - `@ViewBuilder`: 自訂產生 View 的 func
```swift
    struct MyView: View {
        
        var body: some View {
            colorShape()
        }
        
        @ViewBuilder
        func colorShape() -> some View {
            if true {
                Circle()
                    .fill(Color.green)
            }else{
                Circle()
                    .fill(Color.red)
            }
        }
    }
```

## 與UIKit整合

- 將SwiftUI包給UIKit的專案用: UIHostingController
- 將UIKit(UIView & UIViewController)包給SwiftUI專案用: UIViewRepresentable, UIViewControllerRepresentable
  - 加上Coordinator可以讓AppKit或UIKit回饋事件給SwiftUI
- AsyncImage不可以直接加modifer(例如.resizable()), 必須用closure後帶的Image才可以帶modifier
- @FocusState wrapper配合modifier: .focused，監聽View是否為Focus狀態
- modifer: .alert()裡面放按鈕，即使按鈕為空action, 按下後依然會關閉alert
- [SwiftUI易搞混類型比較](https://jaredsinclair.com/2020/05/07/swiftui-cheat-sheet.html)
- 當Class實作ObservableObject時，該Class擁有`Publisher`protocol功能，自動實作objectWillChange方法

## 混合UIKit與SwiftUI

```swift
guard
let window = UIApplication.shared.windows.first,
...
window.rootViewController?.present(alertController, animated: true)
```

- SwiftUI的View常透過Init()的方式new一個@ObservedObject，或是透過Init(:)將外面的Observable帶進來
- 在ObservableObject裡宣告的屬性(沒有@Published之類的Wrapper)，可以在前面加上$變成Binding，傳給View作綁定:
```swift
  class JobConnectionManager: ObservableObject {
      var isReceivingJobs: Bool = false
  }

  var headerView: some View {
     Toggle("Receive Jobs", isOn: $jobConnectionManager.isReceivingJobs)
  }
```

## 實用View與組件
- 使用 TabView 製作 Walkthrough 的頁面
- EmptyView(): 方便開發用
- Label: 有圖示(左)與文字(右)的View
- Button裡的Content: 可以有一個以上的View，會用一個隱藏的`HStack`包起來
```swift
   Button(action: () -> Void, label: () -> Label)
```
- 要在SwiftUI底下取得UIWindow, 要用`@Environment`:
```swift
  @Environment(\.window) var window: UIWindow?
```

## 線程管理

- .onAppear(perform:)裡頭run的是同步, 若要加非同步async/await，要用Task包起來，或用.task modifier取代.onAppear(perform:)
- 因為ViewModel多用來觸發UI更新，所以可以在Model的class前面加上@MainActor使之回到主線程:
```swift
  @MainActor class ViewModel: ObservableObject {
  // ...
  }
```
- Property或func前面也可以加`@MainActor`
- Task是一種top-level asynchronous task，表示可以從同步工作創建任一非同步工作，而且都為頂層Task
- 若是槽狀建立多重Task，運行時task間並不會有子母關係，都為頂層Task
- 若在UI層，透過Task執行async，只能確保出發時是在主線程，回來時可能就離開主線程了，因為Task裡頭不同的await，可能跑在不同的線程:
```
  Remember, you learned that every use of await is a suspension point, and your code might resume on a different thread. The first piece of your code runs on the main thread because the task initially runs on the main actor. But after the first await, your code could be running on any thread.
  You need to explicitly route any UI-driving code back to the main actor.
```
- 使用MainActor.run同等於DispatchQueue.main, 但如果用太多、會有太多的closure，不好閱讀；建議使用@MainActor在func上, 但該func就必須用await呼叫:
```swift
  await MainActor.run {
    // ... your UI code ...
  }
```

## 導航
- NavigationStack可使用兩種方式做Push View
```swift
//Pushed View包在`NavigationLink`裡
NavigationStack {
  List(store.objects) { object in
      NavigationLink(object.title) {
          MyView(object: object)
      }
  }
}

//監聽`NavigationLink`的值
NavigationStack {
  List(store.objects) { object in
      NavigationLink(value: object) {
        Text(object.title)
      }
  }
  .navigationTitle("The Met")
  .navigationDestination(for: Object.self) { object in
    ObjectView(object: object)
  }
}
```

- NavigationLink可單獨使用在ChildView中，如果母層有NavigationStack才會生效
- NavigationLink(destination:)在MVVM架構底下，不應該直接指定destination是哪個View，要透過一個中介的方式，讓ViewModel回傳目標View:
```swift
  var body: some View {
   NavigationLink(destination: viewModel.nextView) {
    Text("Next view")
   }
  }
  //viewModel
  class MyViewModel: ObservableObject {
   var nextView: some View {    
     return ViewBuilder.makeNextView()
    }
  }
  //view factory
  enum ViewBuilder {  
    static func makeNextView() -> some View {    
      let viewModel = NextViewModel()    
      return NextView(viewModel: viewModel)
    }  
  }
```

## 預覽
- 專案中有一個`Preview Content`資料夾，裡頭可以放XCode裡SwiftUI Preview會用到的腳本，不會包到App裡

- #Preview可以設定預覽大小，方便簡視子View，但預覽視窗要設定成Selectable，不是在device Mode底下
```swift
	#Preview(traits: .sizeThatFitsLayout) {
    	SubView()
	}
```


- Preview可以設定預覽大小, 有`.device`, `sizeThatFits`, `.fixed(width:, height:)`:
```swift
  struct MyView_Previews: PreviewProvider {
    static var previews: some View {
      JokeCardView()
        .previewLayout(.sizeThatFits)
    }
  }
```

## 初始化技巧

- 透過init初始化@Binding, @ObservedObject, @StateObject的參數, 變數前面要加底線:
```swift
  struct SomeView: View {
   @Binding var myIndex: Int
   @StateObject var vm: My_vm

   init(parentIndex: Binding<Int>) {
       _myIndex = parentIndex
       _vm = StateObject(wrappedValue: My_vm())
   }

    var body: some View {
      EmptyView()
    }
  }
```

## 渲染行為

- 如果有使用到GeometryReader並讀取到window的屬性，裡面的sub view會render兩次:
```swift
  struct SomeView: View {
    var body: some View {
      GeometryReader { window in
          //Init()觸發兩次 
          SubView()
            .frame(minWidth: window.size.width)
          //Init()觸發一次 
          SubView()
            .frame(minWidth: 100)
      }
    }
  }
```
- 如果有使用@State的變數，但view沒有其他的元素監聽該變數，即使改變該變數的值，畫面也不會重畫；即使有元素監聽該值，如果改變的值跟上一次相同，也不會重畫:
```swift
  struct TestContent_view: View {
      @State var param = "0"
      
      var body: some View {
          VStack {
              Button("set 0") {
                  param = "0"
              }
              Button("set 1") {
                  param = "1"
              }
              //Text(param) //加入Text後，按Button才會變色
          }
          .frame(width: 800, height: 600)
          .background(.random) //按Button不會變色
      }
  }
```

- `GeometryReader`:讀取 Parent View 的大小，用來設定 Child View 的大小，但會改變 Child View 對齊方式為`Left-Top`，另外有`.onGeometryChange` 監聽 view 的大小變化
- `.containerRelativeFrame(_:alignment:_:)`(iOS 17+) 也可基於 Parent View改變自身大小
```swift
	//GeometryReader
	GeometryReader { proxy in
    	Color.white
        	.frame(height: proxy.size.height * 0.5)
	}
	//containerRelativeFrame
	Color.white
    	.containerRelativeFrame(.vertical) { length, axis in
    	    	length * 0.5
    	}

```
- `ViewThatFits` 依據母空間，儘可能置入最合適的View，選擇最適方案 (iOS 17+)
```swift
    var body: some View {
            ViewThatFits(in: .horizontal) {
				// 方案 1
                HStack {
                    Text("\(uploadProgress.formatted(.percent))")
                    ProgressView(value: uploadProgress)
                        .frame(width: 100)
                }
				// 方案 2
                ProgressView(value: uploadProgress)
                    .frame(width: 100)
				// 方案 1
                Text("\(uploadProgress.formatted(.percent))")
            }
    }
```

## 綁定與選擇

- 若view端的List需要綁定selection在vm的selection上，vm的變數不需宣告成@Published, 避免view端多餘的UI-refresh:
```swift
  struct SideBar_view: View {
      @StateObject private var vm: SideBar_vm
      var body: some View {
          List(vm.projects, selection: $vm.selectedProjectID) { ... }
      }
  }

  class SideBar_vm: ObservableObject {
      var selectedProjectID: UUID?
  }
```

## View關閉與呈現

- 如果不使用代入isShow的Binding<Bool>給sheet的view做關閉的橋接，在sheet的view中可以使用`@Environment(\.dismiss) `private var dismiss直接關閉
- `@Environment(\.isPresented)` private var isPresented跟`onAppear(perform:)`類似，但isPresented會被呼叫多次
- Text預設字型大小為13.0，內建的字型表:
```
  .largeTitle    34.0
  .title1        28.0
  .title2        22.0
  .title3        20.0
  .headline      17.0
  .callout       16.0
  .subheadline   15.0
  .body          17.0
  .footnote      13.0
  .caption1      12.0
  .caption2      11.0
```
- `interactiveDismissDisabled`: 防止使用者用手勢下拉/按空白處的方式關閉sheet或popover
- 背景類的View可用`.ignoresSafeArea`填滿邊界，`.allowsHitTesting(false)`忽略點擊
- [`monospacedDigit`](https://developer.apple.com/documentation/swiftui/font/monospaceddigit()): 修改Text的字體，使用等寬的字元，適合用來控制日期顯示, 字串長度不會隨著數字字元變動
- 如果要顯示進度，可使用Text搭配formater，自動帶%符號:
```swift
  Text(progress, format: .percent.precision(.fractionLength(0)))
      .bold()
      .monospacedDigit()
```

## 自訂Modifier

- `ViewModifier` 客製`modifier`
```swift
    struct ResizableView: ViewModifier {
        
        @State private var transform = Transform()
        @State private var scale: CGFloat = 1
        
        func body(content: Content)-> some View {
            content
                .frame(width: transform.size.width,
                       height: transform.size.height)
                .rotationEffect(transform.rotation)
                .offset(transform.offset)
                .gesture(dragGesture)
                .gesture(SimultaneousGesture(rotationGesture, scaleGesture))
                .scaleEffect(scale)
        }
    }

    #Preview {
        RoundedRectangle(cornerRadius: 30)
            .foregroundStyle(.blue)
            .modifier(ResizableView())
    }

```
- 承上例，使用`extension`讓使用上更SwiftUI
```swift
    extension View {
      func resizableView() -> some View {
        modifier(ResizableView())
      }
    }

    #Preview {
        RoundedRectangle(cornerRadius: 30)
            .foregroundStyle(.blue)
            .resizeable()
    }
```
- 可自訂義`if` modifier，因應不同情境為view設定外型:
```swift
  private struct CreateButton: View {
      var isApplyBackground = false
      
      var body: some View {
          Button(action: {
              action()
          }, label: {
              Text(label)                
          })
          .if(isApplyBackground) { view in
              view.background(RoundedRectangle(cornerRadius: 16.0).fill(Color.blue))
          }
      }
  }

  //自訂義 if modifier
  extension View {
      @ViewBuilder func `if`<Content: View>(_ condition: Bool, transform: (Self) -> Content) -> some View {
          if condition {
              transform(self)
          } else {
              self
          }
      }
  }
```

## 圖片與文字技巧

- Image可用Text包起來，然後用font size指定圖像大小:
```swift
  Text(Image(systemName: "button.programmable"))
      .font(.largeTitle)
```
- `alignmentGuide`: 可自訂義對齊方式，例如A View的Bottom要對齊B View的Top, 可搭配`.offset(x:y:)`使用