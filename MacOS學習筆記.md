## MacOS學習筆記

- Application的Main Menu的Menu Item只能在AppDelegate建立@IBOutlet, 但可以拖曳至First Responder: NSViewController 或 NSWindowsController建立@IBAction
- 尋找Terminal Command位置
```
which whomai
```
則會顯示
```
/usr/bin/whoami
```
- 如果要衡量視窗大小，擷圖後，若是Retina營幕，要將Pixels值/2，才是SwiftUI的單位
- 可使用Command-click取消選取選項
- 使用Environment Overrides修正Light-Mode或Dark-Mode
- @AppStorage is a property wrapper that gives easy access to UserDefaults
- NSApp是NSApplication.shared縮寫
- `@SceneStorage`和 `@AppStorage`類似，操作UserDefaults的Wrapper，但是可以儲存每個window的settings狀態
- TableColumn簡化: value: keyPath屬性除了可以用來指定sort的欄位，不用特別指定cell的內容，keyPath值自動轉化成TextView
```
TableColumn("Title") {
   Text($0.text)
}
```
簡化成
```
TableColumn("Title", value: \.text)
```
- TableView中的每筆資料會使用UUID來辨別每列的資料, 故資料必須要有UUID
- List加上選取功能，如果List裡頭有一個以上的ForEach,會依據selection類型對相同類型的ForEach加上選取功能
```
List(selection: $selection)
```
- MacOS App的icon不會自動切圓角
- 設定keyboardShortcut時，因為Command Key是預設行為，不用特別指定
- NSWorkspace可以存取內建的app或服務：打開Finder於特定資料夾、打開預設瀏覽器...等
- 呼叫Process試圖讀取/寫入檔案，在Playground下沒有讀取/寫入限制，但若在App裡面讀取/寫入非container的位置就會遇到Mac Sandbox的限制
- 呼叫Process無法下達萬用字元參數，如 sips --resampleHeight 600 *.png --out resized_images
- 要在SwiftUI下取得AppDelegate要使用NSApplicationDelegateAdaptor