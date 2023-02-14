## SwiftUI學習筆記

- Owning the reference, not the data

- List, ForEach
當struct或class實作Hashable時，使用List或Foreach時，就可以取代 \.id為 \.self
(If your data object implements Hashable, you can also tell SwiftUI to use the entire object as the unique identifier. )

**@State**
- State是一個property wrapper類型，可以被SwiftUI讀取與寫入
- 當變數宣告為State，是指該View擁有該變數Data的Reference(存在其他儲存空間，由SwiftUI管理), 而非直接存取Data,(因為宣告@State的地方為Struct View，如果要指定值，complier會報錯，狀態刷新:重建View時，為了保持狀態同步，必須額外儲存空間)，當state的值改變時，view會重新complier body(SwiftUI re-render), @sState被用來當single source of truth 
- 擁有@Published特性
- 當宣告@State var myVar時， 編輯器背後自動產生 var myVar = State<Int>(initialValue: 0), 可透過(有底線) _myVar.wrappedValue 取得該值; 若要將該值放到Binding<T>的變數時，可用myVar.projectedValue。但如果是手動宣告var myVar = State<Int>(initialValue: 0)，變數值前面不用加底線
- `@State var myIndex` 等於 `_myIndex.wrappedValue`
- `$myIndex` 等於 `_myIndex.projectedValue` : `Binding<T>`
- State是Value Type，若當參數傳送，是copy by value
- 若將一個struct(內含許多property)，宣告成State，可正常運作，但沒效率，因為其中一個property改變，就會將整個struct的實體換掉，所有相關聯的UI都會Trigger Refresh，影響效能，請小心使用struct配合State。
- 若將一個class宣告成State, 則不會有作用. 

**@ObservedObject**
- 變數宣告為ObservedObject時，代表變數被移除View的變數空間，並用binding的方式相關聯
- 表示該變數為外部儲存，換句說話，該資料並不屬於View
- 擁有@Published特性
- 當你使用class-instance當作model時，除非重新指定instance,不然修改裡面的變數並不會trigger UI-Refresh
- OservedObject配合@Published，當class實作ObservableObject時，裡頭的@Published都會自動實作requires:objectWillChange，並將Class資料從View移至外部，不為View所有，@ObservedObject裡頭的@Published屬性，在view中的運作有如State
- 因為Publisher運作有如State，若使用@Published的屬性必須value type: 基本型態或struct
- @StateObject vs @ObservedObject 差別:

相同點: 兩者都必須實作ObservableObject, 並配合@Published

差異點:
@ObservedObject:會隨著View的更新多次建立, 主要發生在View與Model(external)之間的綁定(View向Model訂閱，Model向View通知)
@StateObject:只會被建立一次，是@State(value for ObservableObject)的兄弟版(Reference for ObservableObject)

- @AppStorage 直接對UserDefaults讀寫
- 使用@EnvironmentObject，將某個Object拉出環境?並儲存為一個屬性, 必須是 Class for ObservableObject, 全部的childView都有效
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
- 如果不想使用ObservableObject/ObservedObject，只想簡單用publisher, 可用.onReceive(_:perform:) 當該View收到指定Publisher發送訊息時，就執行特定action
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
- .EnvironmentObject(_:)用來創造自訂的Environment物件，將某個object注入至環境, 比較像是在SwiftUI層(View)的singleton模式, 注入的物件可以被該View與子View存取，但無法被母層存取。特別注意，注入類別實體只能存在一個，若其他地方再注入相同類別，便會取代上一個類別實體.
- Ctrl + Option + Click 點擊canvas畫面，出現元件設定選項
- 假若Object是一個 @EnvironmentObject, 裡面有一個Compute Property: param, 若想要把這個param放到Binding<T>裡，可以用Binding的靜態方法.constant包起來，它可以由一個無法修改值初始化，ex:.constant(challengesViewModel.numberOfAnswered)
- 一般來說，藉由view-initializer傳遞Observable物件是不明智的，因為可能會有re-render就重建observable物件實體的問題。較好的做法是母層view建立observable，透過變數傳給子層較佳. SwiftUI2新增的 @StateObject可解決此問題, @StateObject的生命週期不會follow該view，詳情看第6點
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
- [SwiftUI易搞混類型比較](bit.ly/35Xt7eU)
- 當Class實作ObservableObject時，該Class擁有`Publisher`protocol功能，自動實作objectWillChange方法

