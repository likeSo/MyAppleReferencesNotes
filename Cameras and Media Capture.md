# 摄像头与媒体捕捉

## 请求媒体捕捉访问权限

##### 如果你的App要使用摄像头，需要在Info.plist文件的NSCameraUsageDescription指定使用摄像头的原因。

##### 如果你的App要使用麦克风，需要在Info.plist文件的NSMicrophoneUsageDescription指定使用麦克风的原因。

请注意，如果没有指定对应API的使用原因，那么你在调用相关API时，系统会将你的App终结。

#### 检查授权情况

##### 苹果官方文档建议我们在使用以上这些API之前首先使用 [AVCaptureDevice](https://developer.apple.com/documentation/avfoundation/avcapturedevice).[authorizationStatus(for:)](https://developer.apple.com/documentation/avfoundation/avcapturedevice/1624613-authorizationstatus) 检查应用权限的授予情况，如果没有被授予使用权限，需要使用[AVCaptureDevice](https://developer.apple.com/documentation/avfoundation/avcapturedevice).[requestAccess(for:completionHandler:)](https://developer.apple.com/documentation/avfoundation/avcapturedevice/1624584-requestaccess)来显示弹框，请求权限。

```swift
switch AVCaptureDevice.authorizationStatus(for: .video) {
    case .authorized: // 用户之前已经授予了使用权限
        self.setupCaptureSession()
    
    case .notDetermined: // 还没有授予权限
    	// requestAccess方法将会显示系统的权限请求弹窗
    	// 要先配置Info.plist文件，添加NSCameraUsageDescription字段
    	// 先停止sessionQueue的进行
    	self.sessionQueue.suspend()
        AVCaptureDevice.requestAccess(for: .video) { granted in
            if granted {
                self.sessionQueue.resume()
                self.setupCaptureSession()
            }
        }
    
    case .denied: // 被拒绝
       	self.sessionQueue.suspend()
        return
    case .restricted: // 因为某些原因无法授予权限
       	self.sessionQueue.suspend()
        return
}
```

请注意，如果摄像或者拍照不是你的App的主要功能，那么你只能在会使用到相关功能的时候才可以请求权限。

#### 请求保存媒体权限

如果是使用上述API拍摄的视频或者照片，请使用[PHPhotoLibrary](https://developer.apple.com/documentation/photokit/phphotolibrary)以及[PHAssetCreationRequest](https://developer.apple.com/documentation/photokit/phassetcreationrequest)类。这些类会使用到相册的读写权限，所以需要指定NSPhotoLibraryUsageDescription字段。

如果只是想保存UIImage对象，请使用 [UIImageWriteToSavedPhotosAlbum(_:_:_:_:)](https://developer.apple.com/documentation/uikit/1619125-uiimagewritetosavedphotosalbum) 方法，此方法会使用到相册的写入权限。请注意，如果图片对象来自AVFoundation（ [AVCapturePhotoOutput](https://developer.apple.com/documentation/avfoundation/avcapturephotooutput)），不推荐使用此方法写入，因为UIImage中不会包含图片中的全部信息。

如果想保存一段视频，请使用 [UISaveVideoAtPathToSavedPhotosAlbum(_:_:_:_:)](https://developer.apple.com/documentation/uikit/1619162-uisavevideoatpathtosavedphotosal) 方法，此方法同样会使用到相册的写入权限。

以上两个方法都需要指定NSPhotoLibraryAddUsageDescription字段。



## 配置媒体捕获与输出设备

[AVCaptureSession](https://developer.apple.com/documentation/avfoundation/avcapturesession)类用于整合输入流（媒体输入设备，比如摄像头，麦克风）和输出流（媒体输出设备，比如相机预览，扬声器等），是输入流与输出流交互的管道。开发者通过`AVCaptureConnection`类将输入流额输出流绑定。

[AVCaptureDevice](https://developer.apple.com/documentation/avfoundation/avcapturedevice)类代表了物理媒体捕获设备，比如相机以及麦克风。

[AVCaptureDeviceInput](https://developer.apple.com/documentation/avfoundation/avcapturedeviceinput)类代表一个输入流，它基于一个captureDevice，比如前置相机，话筒等。负责与captureSession交互。

[AVCaptureDeviceOutput](https://developer.apple.com/documentation/avfoundation/avcaptureoutput)类代表了一个输出流，这是一个抽象类，它描述的是媒体输出的方式，比如图片输出：[AVCapturePhotoOutput](https://developer.apple.com/documentation/avfoundation/avcapturephotooutput)，用于描述的是静态图像，Live Photo等媒体输出方式。

所以初始化session之后，就是开始配置：

```swift
// 固定语法，开始AVCaptureSession类的配置
captureSession.beginConfiguration()
// 获取后置的摄像头
let videoDevice = AVCaptureDevice.default(.builtInWideAngleCamera,
                                          for: .video, position: .unspecified)
guard
    let videoDeviceInput = try? AVCaptureDeviceInput(device: videoDevice!),
	// 在添加输出或者输入流之前必须检查是否可以添加
    captureSession.canAddInput(videoDeviceInput)
    else { return }
captureSession.addInput(videoDeviceInput)
```

`AVCaptureDevice`的可选值有这些：

```swift
// 内置麦克风
static let builtInMicrophone: AVCaptureDevice.DeviceType
// 对于iPhone来说，指的是后置的摄像头
static let builtInWideAngleCamera: AVCaptureDevice.DeviceType
// 后置双摄
static let builtInDualCamera: AVCaptureDevice.DeviceType
// 这个就不用说了，粪叉以后带出来的TrueDepth摄像头
static let builtInTrueDepthCamera: AVCaptureDevice.DeviceType

/// ... 
```

一个captureSession可以配置多个输入以及输出流，比如需求是需要同时调用后置摄像头以及麦克风，你可以这样写：

```swift
captureSession.beginConfiguration()
let photoDevice = AVCaptureDevice.default(.builtInWideAngleCamera, for: .video, position: .back)
let microphoneDevice = AVCaptureDevice.default(.builtInMicrophone, for: .audio, position: .unspecified)
guard let photoDeviceInput = try? AVCaptureDeviceInput(device: photoDevice!), 
	  let microDeviceInput = try? AVCaptureDeviceInput(device: microphoneDevice!),
	  captureSession.canAddInput(photoDeviceInput),
	  captureSession.canAddInput(microDeviceInput) else {return}
captureSession.addInput(photoDeviceInput)
captureSession.addInput(microDeviceInput)
```

下一步，添加输出流：

```swift
let photoOutput = AVCapturePhotoOutput()
// photoOutput.isLivePhotoCaptureEnabled = photoOutput.isLivePhotoCaptureSupported
// photoOutput.isDepthDataDeliveryEnabled = photoOutput.isDepthDataDeliverySupported
// photoOutput.isPortraitEffectsMatteDeliveryEnabled = photoOutput.isPortraitEffectsMatteDeliverySupported
// 需要注意的是，如果你的应用支持切换前后摄像头的功能，以上这些属性都需要重新设置一次。
guard captureSession.canAddOutput(photoOutput) else { return }
captureSession.sessionPreset = .photo
captureSession.addOutput(photoOutput)
captureSession.commitConfiguration()
```

输出并不依赖于硬件，所以只需要添加一个输出流，配置完成之后调用`commitConfiguration`。这里只是进行了一个最基础的captureSession的配置，一般来说，只需要一个输入以及输出流，一个流程就算是走通了。

视频格式的媒体捕获，对应的输出类型是[`AVCaptureMovieFileOutput`](https://developer.apple.com/documentation/avfoundation/avcapturemoviefileoutput)。

####显示相机预览 

实时预览到相机捕获到的画面，是基于[AVCaptureVideoPreviewLayer](https://developer.apple.com/documentation/avfoundation/avcapturevideopreviewlayer)类，它是`CALayer`的子类。它通过`session`属性与captureSession交互。

你可以以直接操作`CALayer`的方式，初始化previewLayer，设置`session`属性，然后`self.view.layer.addSubLayer(previewLayer)`的方式使用，也可以：

```swift
// 苹果官方Demo给出的方案
class PreviewView: UIView {
    override class var layerClass: AnyClass {
        return AVCaptureVideoPreviewLayer.self
    }
    
    var videoPreviewLayer: AVCaptureVideoPreviewLayer {
        return layer as! AVCaptureVideoPreviewLayer
    }
}

class TakePhotoViewController: UIViewController {
    var videoPreviewView: PreviewView {
        return self.view as! PreviewView
    }
    
    override func loadView() {
        let previewView = PreviewView(frame: CGRect.zero)
        // 配置预览页面的显示模式，类似于UIView的ContentMode
        previewView.videoGravity = .resize
        self.view = view
    }
}
```

然后是设置session：

```swift
self.videoPreviewView.videoPreviewLayer.session = self.captureSession
```

> 请注意，如果你的拍照页面支持横向拍摄的话，请使用`AVCaptureVideoPreviewLayer`的connection属性来获得当前设备拍摄朝向。
>
> ```swift
> // 在进入sessionQueue之前取得这个属性，是为了确保UI级别的操作都在主线程进行，session的操作都在sessionQueue进行。
> let videoPreviewLayerOrientation = videoPreviewView.videoPreviewLayer.connection?.videoOrientation
> sessionQueue.async {
>     if let photoOutputConnection = self.photoOutput.connection(with: .video) {
>                 photoOutputConnection.videoOrientation = videoPreviewLayerOrientation!
>             }
> }
> ```



#### 跑起来吧

至此，基本的配置工作已经完成了，接下来就是让数据流在输入以及输出流中跑起来。

```swift
self.captureSession.startRunning()
// 调用了startRunning()方法之后，即可以在设置的videoPreviewLayer上预览到实时的相机效果。
// startRunning()以及stopRunning()方法都会阻塞当前线程，请确保是在sessionQueue里进行。
```

#### 处理错误

退出页面时，请记得调用captureSession的`stopRunning()`方法，因为应用在后台时调用相机，是被苹果禁止的，这也是未越狱的苹果设备上，没有“偷拍应用”的原因。

另外需要注意的是，硬件的调用有可能被其他系统行为打断（比如有电话进来）。

`AVCaptureSession`类提供了一系列的`NotificationName`，其中包括session开始运行，结束运行，或者session被打断：

```swift
// startRunning()调用完成之后发送此通知
static let AVCaptureSessionDidStartRunning: NSNotification.Name

// stopRunning() 调用完成之后发送
static let AVCaptureSessionDidStopRunning: NSNotification.Name

// session被打断，通知内的`userInfo`会带有AVCaptureSessionInterruptionReasonKey字段，其值是枚举值的rawValue，标识着被打断的原因。
@available(iOS 9.0, *)
    public enum InterruptionReason : Int {
		// 当前应用被退出到后台
        case videoDeviceNotAvailableInBackground
		// 被其他应用占用了音配输入设备，比如来电话了，或者系统闹铃响了
        case audioDeviceInUseByAnotherClient
		// 被其他captureSession占用了输入设备
        case videoDeviceInUseByAnotherClient
		// "MultipleForegroundApps"一般指的是iPad或者macOS上的分屏功能
        case videoDeviceNotAvailableWithMultipleForegroundApps
		// 被系统强制关闭
        @available(iOS 11.1, *)
        case videoDeviceNotAvailableDueToSystemPressure
    }
static let AVCaptureSessionWasInterrupted: NSNotification.Name

// “打断”结束了
static let AVCaptureSessionInterruptionEnded: NSNotification.Name

NotificationCenter.default.addObserver(self, 
                                       selector: #selector(self.sessionWasInterrupted), 
                                       name: .AVCaptureSessionWasInterrupted, 
                                       object: self.captureSession)

...

@objc func sessionWasInterrupted(_ note: Notification) {
    // 
    if let reasonIntegerValue = notification.userInfo?[AVCaptureSessionInterruptionReasonKey] as? Int,
    let reason = AVCaptureSession.InterruptionReason(rawValue: reasonIntegerValue) {
        switch reason {
            ...
        }
    }
}
```



### 拍摄

#####  准备工作

创建一个`AVCapturePhotoSettings`类，并配置相关设置。这些设置将在拍照时使用，比如拍照时是否打开闪光灯等属性；

```swift
let capturePhotoSettings = AVCapturePhotoSettings(format: [AVVideoCodecKey: AVVideoCodecType.jpeg])
capturePhotoSettings.isAutoRedEyeReductionEnabled = true
capturePhotoSettings.isHighResolutionPhotoEnabled = true
...

// 接下来应该是调用AVCapturePhotoOutput的capturePhoto方法
photoOutput.capturePhoto(with: capturePhotoSettings, delegate: self)
```



##### Next Step，使用NSData或者将照片保存到相册

遵循`AVCapturePhotoCaptureDelegate`并实现相关代理方法。一般来说我们只需要关心拍照结果：

```swift
func photoOutput(_ output: AVCapturePhotoOutput, didFinishProcessingPhoto photo: AVCapturePhoto, error: Error?) {
    	
		// photo.fileDataRepresentation()可用于获得拍照得到的NSData对象，可用于转化UIImage
    // 请求相册权限，前文说了我们应该只在使用到相关功能的时候才会请求相关权限

    guard error == nil else { print("Error capturing photo: \(error!)"); return }

    PHPhotoLibrary.requestAuthorization { status in
        guard status == .authorized else { return }
        
        PHPhotoLibrary.shared().performChanges({
            // Add the captured photo's file data as the main resource for the Photos asset.
            let creationRequest = PHAssetCreationRequest.forAsset()
            creationRequest.addResource(with: .photo, data: photo.fileDataRepresentation()!, options: nil)
        }, completionHandler: self.handlePhotoLibraryError)
    }
    	
}
```

`AVCapturePhotoCaptureDelegate`的其他代理方法是用来追踪拍照进度的，分别代表 开始曝光，结束曝光，开始处理，结束处理等时间点。



若要拍摄Live Photo，请查看：[**官方文档**](https://developer.apple.com/documentation/avfoundation/cameras_and_media_capture/capturing_still_and_live_photos/capturing_and_saving_live_photos)。