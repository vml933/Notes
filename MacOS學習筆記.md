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