## SwiftUI學習筆記

- Owning the reference, not the data

- List, ForEach
當struct或class實作Hashable時，使用List或Foreach時，就可以取代 \.id為 \.self
(If your data object implements Hashable, you can also tell SwiftUI to use the entire object as the unique identifier. )

@State
==
- State是一個property wrapper類型，可以被SwiftUI讀取與寫入
- State是Value Type，若當參數傳送，是copy by value
- 當變數宣告為State，是指該View擁有該變數Data的Reference(存在其他儲存空間，由SwiftUI管理), 而非直接存取Data,(因為宣告@State的地方為Struct View，如果要指定值，complier會報錯，狀態刷新:重建View時，為了保持狀態同步，必須額外儲存空間)，當state的值改變時，view會重新complier body(SwiftUI re-render), @State被用來當single source of truth 
- 擁有@Published特性
- 當宣告@State var myVar時， 編輯器背後自動產生 var myVar = State<Int>(initialValue: 0), 可透過(有底線) _myVar.wrappedValue 取得該值; 若要將該值放到Binding<T>的變數時，可用myVar.projectedValue。但如果是手動宣告var myVar = State<Int>(initialValue: 0)，變數值前面不用加底線
- `@State var myIndex` 等於 `_myIndex.wrappedValue`
- `$myIndex` 等於 `_myIndex.projectedValue` : `Binding<T>`
- 若將一個struct(內含許多property)，宣告成State，可正常運作，但沒效率，因為其中一個property改變，就會將整個struct的實體換掉，所有相關聯的UI都會Trigger Refresh，影響效能，請小心使用struct配合State。
- 若將一個class宣告成State, 則不會有作用. 
- 將其他線程取得的值(常見如:API)指定給State變數, SwiftUI會自動處理跳至主線程，不需手動處理

**@ObservedObject**
==
- 擁有@Published特性
- 變數有@ObservedObject時，代表變數為外部所擁有，用binding的方式相關聯，來源可能為母層或Environment object，通常用來監聽屬性變化用
- 注意! view本身不create ObservedObject，如果view本身有create ObservedObject，該view UI重刷時, ObservedObject也會重新建立
```
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
- OservedObject配合@Published，當class實作ObservableObject時，裡頭的@Published都會自動實作requires:objectWillChange，並將Class資料從View移至外部，不為View所有，只是添加訂閱關係，不影響儲存，@ObservedObject裡頭的@Published屬性，在view中的運作有如State
```
//@Published score等同於非@Published的nonPubishScore, 手動呼叫objectWillChange,即使View層沒有再監聽這個變數，View層依然會觸發一次re-render
class Model: ObservableObject{
    @Published var score: Int = 0
    
    var nonPubishScore: Int = 0{
        didSet{
            objectWillChange.send()
        }
    }
    
    init(){
        print("Model created")
    }
}

```
- 因為Publisher運作有如State，若使用@Published的屬性必須value type: 基本型態或struct

**@StateObject**
==
- StateObject是Reference type，用來宣告該屬性被View擁有，是@State的兄弟版(Value type)
- 有別於@ObservedObject，不會被UI重刷而重建，只會被建立一次
- Lifecycle與View綁定，適合用來當View的ViewModel
- @Published在哪個thread被建立，就只能用在該thread，除非搭配`receive(on:)`切換thread監聽
```
struct MyView: View {
    @StateObject private var viewModel = MyViewModel()

    var body: some View {
        // Use viewModel here
    }
}
```

- @StateObject vs @ObservedObject 差別:

相同點: 兩者都必須實作ObservableObject, 並配合@Published

差異點:
@ObservedObject:會隨著View的更新多次建立, 主要發生在View與Model(external)之間的綁定(View向Model訂閱，Model向View通知)

- @AppStorage 直接對UserDefaults讀寫並帶有觸發更新UI功能
- 使用@EnvironmentObject，將某個Object拉出環境?並儲存為一個屬性, 必須是 Class for ObservableObject, 全部的childView都有效, 並必須已經為initialized完成.使用@EnvironmentObject不需要再重新初始化
```
case let:
    guard
      let character = title.first,
      case let symbolName = "\(character.lowercased()).square",
    ...
```
- 如果frame 長寬設定為, 將會撐滿父層
- 如果struct裡面的屬性都為Hashable的話，complier會認為該struct可實作hashable
- 如果使用NavigationView會出現 breaking constraint 之類的問題，可以加上以下，避免報錯
```
 NavigationView {
      ...
    }
    .navigationViewStyle(.stack)
```
- List & ForEach 差別: 相較於ForEach, List為針對特定情境使用: 呈現一張直立表，但並不表示一定使用需要重覆的資料, *List裡頭可以包ForEach與其他單項View,
ForEach需要用List或VStack包起來

- Button 使用.buttonStyle(.borderless)可以讓裡面的View置中
```
.sheet(isPresented: $addingNewBook){ NewBookView()}
```
可以寫成
```
.sheet(isPresented: $addingNewBook, content: NewBookView.init)
```
- 要把ToolBar秀出來，可以包在NavigationView裡面
- @Published someProperty可以用$存取, 即使該class沒有繼承ObservableObject
- 使用 assign(to: \.someProperty, on: self)可以將發出的元素綁定在自身的@Published值(Republishing), 會回傳一個strong reference
- .onAppear {}
- 如果某些情境用不著宣告ObservableObject/ObservedObject，只想簡單用`publisher`, 可用.onReceive(_:perform:)，常用情境如 View收到ViewModel的`Publisher`發送訊息時，就執行特定action; 或是View自己宣告Timer, 自己發送更新.
```
struct ReceiverView: View {
    let timer = Timer.publish(every: 1.0, on: .main, in: .default).autoconnect()

    @State var time = ""

    var body: some View {
        Text(time)
            .onReceive(timer) { t in time = String(describing: t) }
    }
}
```
- 同上方狀況，若想監聽@State值變化觸發action，可用onChanged(_:)
```
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
- 當停止Task.Handle，並不會停止過程，只是標註 "Task已取消"，必須自己實作取消過程
- iOS15 SwiftUI 新增一個.task modifier, 當這個viewerOnAppear時，執行非同步工作; 當View消失時，會cancel Task
- ContentView可設定Preview設備
```
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

- .ZStack & .Overlay差別: ZStack為動態Container, frame的尺寸完全基於內部內容，會由最大的子框架定義；而overlay只會考慮第一個視圖定義的大小
- .task 當SwiftUI出現後，執行非同步工作的modifier，適合用在非使用combine, 非使用MVVM的SwiftUI簡易情境
- Environment的值通知用來讀取，不寫入值，宣告後，它就是該view的State屬性
- .EnvironmentObject(_:)用來創造自訂的Environment物件，將某個object注入至環境, 比較像是在SwiftUI層(View)的singleton模式, 注入的物件可以被該View與子View存取，但無法被母層存取。特別注意，注入類別實體同時間只能存在一個，若其他地方再注入相同類別，便會取代上一個類別實體.
- Ctrl + Option + Click 點擊canvas畫面，出現元件設定選項
- 假若Object是一個 @EnvironmentObject, 裡面有一個Compute Property: param, 若想要把這個param放到Binding<T>裡，可以用Binding的靜態方法.constant包起來，它可以由一個無法修改值初始化，ex:.constant(challengesViewModel.numberOfAnswered)
- 一般來說，在View中宣告ObservabedObject變數或藉由view-initializer傳遞Observable物件是有風險的，因為可能會有re-render就重建observable物件實體的問題。較好的做法是母層view建立observable，透過變數傳給子層較佳. SwiftUI2新增的 @StateObject可解決此問題, @StateObject的生命週期不會follow該view，詳情看第6點
```
//有風險: SomeView重render時就會重建一個UserManager()
struct SomeOtherView: View {
  var body: some View {
    SomeView(userManager: UserManager())
  }
}
```
```
//較安全
struct SomeOtherView: View {
  let userManager = UserManager()

  var body: some View {
    SomeView(userManager: userManager)
  }
}
```
```
//新做法: @StateObject變數不會隨著View重新render而重建，類似該View的靜態屬性
struct SomeView: View {
  @StateObject var userManager = UserManager()
  ...
}
```

- [environment列表](https://developer.apple.com/documentation/swiftui/environmentvalues), 可自訂environment屬性
- @ViewBuilder view的body可能會產出多個view時使用，如
```
@ViewBuilder
var body: some View {
    if verticalSizeClass == .compact {    
        VStack{...}
    }else{    
        VStack{...}
    }
}
```
- 將SwiftUI包給UIKit的專案用: UIHostingController
- 將UIKit(UIView & UIViewController)包給SwiftUI專案用: UIViewRepresentable, UIViewControllerRepresentable, 加上Coordinator可以讓AppKit或UIKit回饋事件給SwiftUI
- AsyncImage不可以直接加modifer(例如.resizable()), 必須用closure後帶的Image才可以帶modifier
- @FocusState wrapper配合modifier: .focused，監聽View是否為Focus狀態
- modifer: .alert()裡面放按鈕，即使按鈕為空action, 按下後依然會關閉alert
- [SwiftUI易搞混類型比較](https://jaredsinclair.com/2020/05/07/swiftui-cheat-sheet.html)
- 當Class實作ObservableObject時，該Class擁有`Publisher`protocol功能，自動實作objectWillChange方法
- SwiftUI配合UIApplication.shared.windows.first，可在ViewModel呼叫傳統的UIAlertController混合
```
guard
let window = UIApplication.shared.windows.first,
...
window.rootViewController?.present(alertController, animated: true)
```
- SwiftUI的View常透過Init()的方式new一個@ObservedObject，或是透過Init(:)將外面的Observable帶進來
- 在ObservableObject裡宣告的屬性(沒有@Published之類的Wrapper)，可以在前面加上$變成Binding，傳給View作綁定
```
class JobConnectionManager: ObservableObject {
	var isReceivingJobs: Bool = false
}

var headerView: some View {
   Toggle("Receive Jobs", isOn: $jobConnectionManager.isReceivingJobs)
}
```
- SwiftUI有一個EmptyView()方便開發用
- SwiftUI有一個View: Label，可以加上圖示
- 要在SwiftUI底下取得UIWindow, 要用`@Environment`
```
@Environment(\.window) var window: UIWindow?
```
- SwiftUI .onAppear(perform:)裡頭run的是同步, 若要加非同步 async/await，要用Task包起來，或用.task modifier取代.onAppear(perform:), .task也是view出現時就會觸發, task會處理view若被關掉時，取消所有的非同步Request
- 因為ViewModel多用來觸發UI更新，所以可以在Model的class前面加上@MainActor使之回到主線程
```
@MainActor class ViewModel: ObservableObject {
...
}
```
- Property或func前面也可以加`@MainActor`
- Task是一種top-level asynchronous task，表示可以從同步工作創建任一非同步工作，而且都為頂層Task
- 若是槽狀建立多重Task，運行時task間並不會有子母關係，都為頂層Task
- 若在UI層，透過Task執行async，只能確保出發時是在主線程，回來時可能就離開主線程了，因為Task裡頭不同的await，可能跑在不同的線程
```
Remember, you learned that every use of await is a suspension point, and your code might resume on a different thread. The first piece of your code runs on the main thread because the task initially runs on the main actor. But after the first await, your code could be running on any thread.
You need to explicitly route any UI-driving code back to the main actor.
```
- 使用MainActor.run同等於DispatchQueue.main, 但如果用太多、會有太多的closure，不好閱讀；建議使用@MainActor在func上, 但該func就必須用await呼叫
```
await MainActor.run {
  ... your UI code ...
}
```
- NavigationLink(destination:) 在MVVM架構底下，不應該直接指定destination是哪個View，要透過一個中介的方式，讓ViewModel回傳目標View.
```
var body: some View{
 NavigationLink(destination: viewModel.nextView) {
  Text("Next view")
 }
}
//viewModel
class MyViewModel: ObservableObject{
 var nextView: some View {    
   return ViewBuilder.makeNextView()
  }
}
//view factory
enum ViewBuilder{  
  static func makeNextView() -> some View{    
    let viewModel = NextViewModel()    
    return NextView(viewModel: viewModel)
  }  
}
```

- Preview可以設定預覽大小, 有`.device`, `sizeThatFits`, `.fixed(width:, height:)`
```
struct MyView_Previews: PreviewProvider {

  static var previews: some View {

    JokeCardView()
      .previewLayout(.sizeThatFits)

  }
}
```
- 透過init初始化@Binding, @ObservedObject, @StateObject的參數, 變數前面要加底線
```
struct SomeView: View {

 @Binding var myIndex: Int
 @StateObject var vm: My_vm

 init(parentIndex: Binding<Int>){
     _myIndex = parentIndex
     _vm = StateObject(wrappedValue:My_vm())
 }

  var body: some View {
    EmptyView()
  }
}
```
- 如果有使用到GeometryReader並讀取到window的屬性，裡面的sub view會render兩次:產生兩次Subview
```
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
- 如果有使用@State的變數，但view沒有其他的元素監聽該變數，即使改變該變數的值，畫面也不會重畫；即使有元素監聽該值，如果改變的值跟上一次相同，也不會重畫
```
struct TestContent_view: View {
    
    @State var param = "0"
    
    var body: some View {
        VStack{
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
- 若view端的List需要綁定selection在vm的selection上，vm的變數不需宣告成@Published, 避免view端多除的UI-refresh
```
struct SideBar_view: View {
	@StateObject private var vm: SideBar_vm
	var body: some View {
		List(vm.projects, selection: $vm.selectedProjectID){...}
	}
}

class SideBar_vm: ObservableObject{
	var selectedProjectID: UUID?
}
```
- 如果不使用代入isShow的Binding<Bool>給sheet的view做關閉的橋接，在sheet的view中可以使用`@Environment(\.dismiss) `private var dismiss直接關閉
- `@Environment(\.isPresented)` private var isPresented 跟  `onAppear(perform:)`類似，但isPresented會被呼叫多次
- Text預設字型大小為13.0，內建的字型表
```
.largeTitle 34.0
.title1	28.0
.title2	22.0
.title3	20.0
.headline 17.0
.callout 16.0
.subheadline 15.0
.body 17.0
.footnote 13.0
.caption1 12.0
.caption2 11.0
```
- `interactiveDismissDisabled`: 防止使用者用手勢下拉 / 按空白處的方式關閉sheet或popover
- 背景類的View可用`.ignoresSafeArea`填滿邊界，`.allowsHitTesting(false)`忽略點擊
- [`monospacedDigit`](https://developer.apple.com/documentation/swiftui/font/monospaceddigit()) 修改Text的字體，使用等寬的字元，適合用來控制日期顯示, 字串長度不會隨著數字字元變動
- 如果要顯示進度，可使用Text搭配formater，自動帶%符號
```
Text(progress, format: .percent.precision(.fractionLength(0)))
	.bold()
	.monospacedDigit()
```
- 可自訂義`if` modifier，因應不同情境為view設定外型
```
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
- Image可用Text包起來，然後用font size指定圖像大小
```
Text(Image(systemName: "button.programmable"))
	.font(.largeTitle)

```
- `alignmentGuide` 可自訂義對齊方式，例如A View的Bottom要對齊 B View的Top, 可搭配`.offset(x:y:)`使用