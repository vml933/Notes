# Swift - Camera 學習筆記

## 相機功能

### 取得支援的相機設備
以後鏡頭為例，這裡的 Camera 都是虛擬 Camera 概念（除了 builtInWideAngleCamera），各個 Camera 都是由不同的實體 Camera 組成：

```swift
// 順序依 Camera 新->舊排序
let deviceTypes: [AVCaptureDevice.DeviceType] = [
    // builtInTripleCamera: a system that switches automatically among the three lens, etc Pro
    .builtInTripleCamera,
    // builtInDualWideCamera: a system that switches between 0.5x and 1x as appropriate, etc: Pro, i11
    .builtInDualWideCamera, 
    // builtInDualCamera: a system that switches between 1.0x and telephoto lens (2x, 2.5x, or 3x depending on the phone model), etc: x, Pro
    .builtInDualCamera, 
    // builtInWideAngleCamera: 1x single-lens camera, etc: i6, i7, x, i11, 14, 14pro
    .builtInWideAngleCamera,
]

// Supported Device Types
discoverySession = AVCaptureDevice.DiscoverySession(deviceTypes: deviceTypes, mediaType: .video, position: .back)
// 取得第一個(最佳)Camera，或依需求自行選擇組合
videoDevice = discoverySession.devices.first
```

### 設備支援概覽
```
                Triple Camera   Wide Dual Camera    Dual Camera     Wide Camera
iPhone 14 Pro   Yes             Yes                 Yes             Yes
iPhone 13                       Yes                                 Yes
iPhone 11                       Yes                                 Yes
iPhone X                                            Yes             Yes
iPhone 6s                                                           Yes
```

### iPhone 14 Pro 詳細規格
```swift
// Supported Types: 
// [Back Triple Camera]
// [Back Dual Wide Camera]
// [Back Dual Camera]
// [Back Camera]

// builtInTripleCamera
// 實體鏡頭組合
constituentDevices: [[Back Ultra Wide Camera], [Back Camera], [Back Telephoto Camera]]
// 觸發切換鏡頭的Zoom門檻，第一個鏡頭門檻不會列入表內，故2: [Back Camera], 6: [Back Telephoto Camera]
virtualZoomFactors: [2, 6]

// builtInDualWideCamera
constituentDevices: [[Back Ultra Wide Camera], [Back Camera]]
virtualZoomFactors: [2]
isVirtualDevice: true

// Dual Camera
constituentDevices:  [[Back Camera], [Back Telephoto Camera]]
virtualZoomFactors: [3]
isVirtualDevice: true

// builtInWideAngleCamera
constituentDevices: []
virtualZoomFactors: []
isVirtualDevice: false
```

### 相機初始化設定
首次啟動鏡頭，videoZoomFactor 都為 1x，若是新型手機，取得的實體 Camera 可能為 Ultra Wide，畫面會變魚眼，故需要依當下已選擇虛擬 Camera，找出觸發 WideCamera 的 zoomFactor 並指定，畫面才會正常：

```swift
// 一定要等到加到 Input，才會套用到新的設定
session.addInput(videoDeviceInput)
try? device!.lockForConfiguration()

// virtualZoomFactors的數量若大於0，代表有超過1個以上的實體camera, isVirtualDevice = true; 
// virtualZoomFactors若為空，代表只有1個實體camera, isVirtualDevice = false

// virtualZoomFactors的數量 = constituentDevices數量-1，因為第一個實體Camera的zoomFactor不會列在virtualZoomFactors裡面
// 每一種機型的實體Camera陣列(constituentDevices)順序不相同，要找出目標實體Camera(WideCamera)的index並依該index取得觸發該Camera的值(zoomFactor)

guard let virtualZoomFactors = device?.virtualDeviceSwitchOverVideoZoomFactors,
      virtualZoomFactors.count > 0,
      // 找出wideCamera的index
      let wideCameraIndex = device?.constituentDevices.firstIndex(where: {
          $0.deviceType == .builtInWideAngleCamera
      }),
      wideCameraIndex != 0
else { return }

// 因為第一個實體Camera的觸發zoomFactor不會列在virtualZoomFactors裡面，所以wideCamera的camera index要減1
let triggerWideCameraFactor = device!.virtualDeviceSwitchOverVideoZoomFactors[wideCameraIndex - 1]
// 設定zoomFactor => 觸發實體鏡頭為WideCamera => 畫面正常
device!.videoZoomFactor = CGFloat(truncating: triggerWideCameraFactor)

device!.unlockForConfiguration()
```