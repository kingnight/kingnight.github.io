---
title: "如何使用PHPicker在iOS系统无授权下获取资源"
description: "如何使用PHPicker在iOS系统无授权下获取资源"
category: programming
tags: PHPicker,PHPickerViewController,Limited Photo Library Access,NSItemProvider
---

自iOS14系统开始，苹果加强了用户隐私和安全功能。主要新增了“Limited Photo Library Access”模式，同时在授权弹窗中增加了“Select Photo”选项。这意味着用户可以在应用程序请求访问相册时选择部分照片供应用程序读取。从应用程序的角度来看，它只能访问到用户选择的这几张照片，无法得知其他照片的存在。然而，并非所有普通用户都能够正确理解这一机制，实际用户反馈中也反映出了一些误解。苹果推荐使用新的PHPicker来解决这个问题。在本篇文章中，我将详细介绍如何正确使用PHPicker以及何时应该使用PHPicker。我撰写这篇文章的原因是在尝试使用PHPicker访问资源库时遇到了一些问题。互联网上的许多文章提供的方法都是错误的，从而导致了对PHPicker和iOS权限的一些核心问题的误解。

# PHPicker是什么？

从iOS14开始，PHPicker是系统提供的Picker，它允许你从用户的照片库中访问照片和视频。新的 PHPicker 类不是在 UIKit 框架中的，而是位于 PhotosUI 框架中，包括：

* PHPickerViewController
* PHPickerConfiguration
* PHPickerFilter
* PHPickerResult

你呈现一个 PHPickerViewController，它有一个PHPickerConfiguration 配置来告诉它要选择多少个媒体项，以及需要选择的媒体类型。通过 PHPickerConfiguration 的 filter 属性配置可选择的媒体类型，它的选项可以是任意组合：图片、实况照片或视频。通过 PHPickerConfiguration 的 selectionLimit 属性来配置用户可以选择的媒体项数量。

```swift
let photoLibrary = PHPhotoLibrary.shared()
var config = PHPickerConfiguration(photoLibrary: photoLibrary)
                        
let selectedCount = self.albumViewModel.selectArray.count
let limited = min(4-selectedCount, 4)
config.selectionLimit = (type == .pic ? limited : 1)
config.filter = (type == .pic ? .images : .videos)
let pickerViewController = PHPickerViewController(configuration: config)
pickerViewController.delegate = self
self.viewController?.present(pickerViewController, animated: true)
```

![4678a001aeede788feb569b879e4c314](/assets/images/2648731-4937d706f1e5d232.png)

# 真的需要用户的授权吗？

当用户给予受限访问模式时，如果需要获得未授权的额外资源，网络上很多文章建议你使用PHAsset和PHPicker来获取额外的数据，这样做的问题是，你必须具有访问资源库的权限，这违背了苹果建议的使用PHPicker的初衷:在不请求权限的情况下使用的选择器。

我们来模拟一下流程：你的的应用程序请求访问用户资源库的权限，用户说:“我将只给这个应用程序有限的访问一些照片。” 此时，如果你的应用程序打开PHPicker并显示所有的照片；用户说:“奇怪，我以为我只给有限的访问权限，为什么所有照片都有？”；接下来，用户选择了一张他没有给我们访问权限的照片。应用程序现在需要什么都不做，为了使用PHAsset获得他选择的照片的元数据（metadata），他们必须再次更新他们的权限。用户感到困惑。

所以，如果你的目的非常明确，就是需要用户给予额外的资源授权来获得PHAsset对象，应该使用iOS14的新API `PHPhotoLibrary.shared().presentLimitedLibraryPicker(from: vc)`，反之，使用PHPicker不应该申请获得用户授权，正确的做法是使用PHPickerViewControllerDelegate返回的NSItemProvider获得元数据（metadata）信息，我将在这篇文章中后续详细介绍。

# 使用PHPicker的方式

## 错误方式

下面这段代码是网络上广泛被转载的一段错误代码：
```swift
import UIKit
import PhotosUI
class PhotoKitPickerViewController: UIViewController, PHPickerViewControllerDelegate {
    @IBAction func presentPicker(_ sender: Any) {
        let photoLibrary = PHPhotoLibrary.shared()
		let configuration = PHPickerConfiguration(photoLibrary: photoLibrary)
		let picker = PHPickerViewController(configuration: configuration)
		picker.delegate = self
		present(picker, animated: true)
	}
		
	func picker(_ picker: PHPickerViewController, didFinishPicking results: [PHPickerResult]) {
		picker.dismiss(animated: true)
		let identifiers = results.compactMap(\.assetIdentifier)
		let fetchResult = PHAsset.fetchAssets(withLocalIdentifiers: identifiers, options: nil)
				
		// TODO: Do something with the fetch result if you have Photos Library access
	}
}
```
在这段代码中，你使用 PHPickerViewController 选择了受限制的资源，但是在调用 PHAsset.fetchAssets 时返回了一个空结果。这是因为 fetchAssets 方法只能检索用户授权访问的所有资源，而在受限制模式下，只有最近的受限制资源可供访问，所以这种方式是错误的！

## 正确方式

PHAsset不应该与PHPicker一起使用，这不是使用PHPickeri的正确方法！应该使用NSItemProvider。NSItemProvider是一个项目提供程序，用于在拖放或复制/粘贴活动期间在进程之间传输数据或文件，或者从主机应用程序到应用程序扩展。
使用itemProvider，可以读取对象的类型，并根据它是照片、视频还是其他内容来处理它。比较合适的方式如下所示：
```swift
func picker(_ picker: PHPickerViewController, didFinishPicking results: [PHPickerResult]) {
    picker.dismiss(animated: true)
    for result in results {
        let itemProvider: NSItemProvider = result.itemProvider;
        if(itemProvider.hasItemConformingToTypeIdentifier(UTType.image.identifier)) {
            // 图片处理
        } else  if(itemProvider.hasItemConformingToTypeIdentifier(UTType.movie.identifier)) {
            // 视频处理
        } else {
            // 其他，暂时忽略
        }
    }
}
```

# 获得图片与图片元数据

通过前一步骤，我们已经知道了资源的类型。接下来通过NSItemProvider的API加载图片内容和获得元数据信息;查阅[NSItemProvider](https://developer.apple.com/documentation/foundation/nsitemprovider)文档，可以看到加载数据，主要提供了下面几种API：
* loadDataRepresentation：返回资源Data数据
* loadFileRepresentation：返回资源URL
* loadObject：指定资源类型返回

这里我推荐使用loadDataRepresentation，返回Data数据，方便下一步获得元数据信息；

```swift
func picker(_ picker: PHPickerViewController, didFinishPicking results: [PHPickerResult]) {
    picker.dismiss(animated: true)
    for result in results {
        let itemProvider: NSItemProvider = result.itemProvider;
        if(itemProvider.hasItemConformingToTypeIdentifier(UTType.image.identifier)) {
            // 图片处理
            itemProvider.loadDataRepresentation(forTypeIdentifier: UTType.image.identifier) { data, error in
                    //处理业务model转换
                    if let model = self.createPhotoResourcesModel(data: data, assetIdentifier: assetIdentifier){
                        self.albumViewModel.selectArrayAddObject(model)
                        
                        DispatchQueue.main.async {
                            //更新UI
                        }
                    }
              }
            
        } else  if(itemProvider.hasItemConformingToTypeIdentifier(UTType.movie.identifier)) {
            // 视频处理
        } else {
            // 其他，暂时忽略
        }
    }
}
```
在处理业务model转换函数中，由于Data类型很容易转换成UIImage，并且通过将Data转换为CFData类型，可以通过系统预设的key/value键值对获得元数据信息，
```swift
let imgSrc = CGImageSourceCreateWithData(data, options as CFDictionary)
let metadata = CGImageSourceCopyPropertiesAtIndex(imgSrc, 0, options as CFDictionary)
colorModel = metadata[kCGImagePropertyColorModel] as? String
pixelWidth = metadata[kCGImagePropertyPixelWidth] as? Double;
pixelHeight = metadata[kCGImagePropertyPixelHeight] as? Double;
```
这里我们使用了Github上开源的[ExifData](https://gist.github.com/lukebrandonfarrell/961a6dbc8367f0ac9cabc89b0052d1fe) 代码，完整实现了所有字段的获取封装，使用起来非常方便。

```swift
func createPhotoResourcesModel(data:Data?,
                            assetIdentifier:String?) -> SNSResourcesModel? {
        guard let imageData = data, let uiimage = UIImage(data: imageData) else {
            return nil
        }
        let model = SNSResourcesModel()
        model.assetLocalIdentifier = assetIdentifier
        model.assetType = .photo
        model.assetSource = .album
        model.originImage = uiimage
        model.bigPreviewImage = uiimage
        
        let exifData = ExifData(data: imageData)
        model.pixelWidth = Int(exifData.pixelWidth ?? 0)
        model.pixelHeight = Int(exifData.pixelHeight ?? 0)
        if imageData.imageType == .GIF{
            model.isGif = true
            model.gifData = imageData
        }
        return model
}
```

## 处理特殊格式图片

如果用户在资源库中选择了一张WebP格式图片或者GIF动图，由于展示所需代码和形式均不同，所以需要特别区分，那么如何来区别处理呢？

我们可以通过UTType来具体区分不同类型，UTType是Uniform Type Identifier的缩写，用于标识特定类型的文件或数据。在macOS和iOS等操作系统中，UTType通常用于识别文件类型、将文件分组到合适的应用程序中、在不同应用程序之间共享数据等。

UTType由两部分组成：类型标识符(type identifier)和类型标签(type tag)。类型标识符是一串唯一的字符串，用于标识特定类型的文件或数据，通常采用反向DNS风格的命名方式，如com.adobe.pdf、public.image等。类型标签是一个可选的字符串，用于描述特定类型的文件或数据，例如"PDF document"或"JPEG image"等。

```swift
if(itemProvider.hasItemConformingToTypeIdentifier(UTType.image.identifier)) {
    // 图片处理
    // 判断webp
    if itemProvider.hasItemConformingToTypeIdentifier(UTType.webP.identifier){
        //处理webp
    }
    //判断GIF
    if itemProvider.hasItemConformingToTypeIdentifier(UTType.gif.identifier){
        //处理GIF
    }        
}
```

# 获得视频

获得视频时，推荐使用loadFileRepresentation，返回URL，通过URL可以获得AVAsset
```swift
func picker(_ picker: PHPickerViewController, didFinishPicking results: [PHPickerResult]) {
    picker.dismiss(animated: true)
    for result in results {
        let itemProvider: NSItemProvider = result.itemProvider;
        if(itemProvider.hasItemConformingToTypeIdentifier(UTType.image.identifier)) {
            // 图片处理
        } else  if(itemProvider.hasItemConformingToTypeIdentifier(UTType.movie.identifier)) {
            // 视频处理
            itemProvider.loadFileRepresentation(forTypeIdentifier: UTType.movie.identifier) { url, error in
                    //业务model转换
                    if let model = self.createVideoResourcesModel(url: url, assetIdentifier: assetIdentifier){
                        self.albumViewModel.selectArrayAddObject(model)
                        
                        DispatchQueue.main.async {
                            //展示UI
                        }
                    }
            }
        } else {
            // 其他，暂时忽略
        }
    }
}
```

# 获取加载进度

但获取的资源文件较大时，我们系统获得加载数据的进度，此时可以使用NSItemProvider的加载数据函数提供的返回值NSProgress对象；

```swift
var progress:Progress?
progress = itemProvider.loadDataRepresentation(forTypeIdentifier: UTType.image.identifier) { data, error in
    //...
}
//添加观察者
progress?.addObserver(self, forKeyPath: "fractionCompleted", options: [.new], context: nil)

override func observeValue(forKeyPath keyPath: String?, of object: Any?, change: [NSKeyValueChangeKey : Any]?, context: UnsafeMutableRawPointer?) {
    if keyPath == "fractionCompleted" {
        print("fractionCompleted=\(self.progress?.fractionCompleted)")
    }
}
```
在上面的示例代码中，我们首先创建了一个NSItemProvider对象和一个指定类型标识符。然后，我们使用loadDataRepresentation(forTypeIdentifier:completionHandler:)方法加载数据，并返回一个NSProgress对象。我们可以将该对象添加为观察者，并在观察者的回调方法中更新进度条的值。

请注意，NSProgress对象是线程安全的，因此可以在不同的线程中使用。此外，如果你需要在多个地方使用同一个NSProgress对象；

NSProgress对象的fractionCompleted属性，该属性表示任务的完成度，其值在0.0和1.0之间。

# 总结
本文要点包含以下：

* PHPicker是iOS14开始引入的新组件,它允许在不需要用户授权的情况下访问照片库的所有资源。

* 使用PHPicker的正确方式是通过PHPickerViewControllerDelegate回调返回的NSItemProvider来获取所选资源,而不是通过PHAsset来获取,后者需要提前获取用户的相册访问授权。

* 通过NSItemProvider可以判断资源类型,加载资源数据或文件URL,获取图片、视频等多媒体资源。

* 对于图片,可以通过loadDataRepresentation获取Data,并利用该Data获取图片元数据信息。对于视频,可以通过loadFileRepresentation获取URL,并利用URL获取AVAsset。

* 通过UTType可以进一步判断特殊格式资源如webp、gif等进行不同处理。

* 可以通过NSProgress监听资源加载进度。

* 正确使用PHPicker可以避免引起用户疑惑,提高用户体验,是iOS14访问多媒体资源的推荐方式。

总之,本文详细介绍了在iOS14中如何正确使用PHPicker访问用户选择的部分照片资源,其要点是不需要提前获取授权,通过NSItemProvider处理多媒体资源,这是一种更符合系统设计初衷和提高用户体验的方式。

# 参考

* https://itnext.io/the-right-way-to-use-phpicker-and-retrieve-exif-data-without-requesting-library-permissions-in-336c13f87e3f
* https://swiftsenpai.com/development/webp-phpickerviewcontroller/
* https://gist.github.com/lukebrandonfarrell/961a6dbc8367f0ac9cabc89b0052d1fe
* https://www.jianshu.com/p/5e7aacfa4374
* https://developer.apple.com/forums/thread/650902