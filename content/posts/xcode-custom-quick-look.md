---
title: "How to make debugQuickLookObject work on Mac Catalyst"
date: 2023-03-15T22:44:35+03:00
draft: false
---

In [Vectornator](https://vectornator.io/) we have lots of different types of elements that has to be rendered on the screen. E.g., shapes, artboards, layers and so on. And even if they are using CoreGraphics and `CGPath` under the hood there was still no convinient way to visualize elements during debugging. So we decided to add Quick look support for Xcode debugger. 

It's possible to do that with implementing an objc function - [`debugQuickLookObject`](https://developer.apple.com/library/archive/documentation/IDEs/Conceptual/CustomClassDisplay_in_QuickLook/CH01-quick_look_for_custom_objects/CH01-quick_look_for_custom_objects.html). And we've been thinking that we already have CoreGraphics-based rendering of our custom elements into bitmap, so why not just return `UIImage` backend by `CGImage` bitmap? 

In normal iOS/macOS application this will work (`UIImage` for iOS and `NSImage` for macOS). But since our macOS aplication is done via Mac Catalyst we have to deal with UIKit classes only. And we did something like this

```
extension Artboard {
    @objc
    public func debugQuickLookObject() -> AnyObject {
        guard let image = BitmapRepresentor.image(for: self, includeBackground: true) else {
            let msg = "BitmapRepresentor returned a nil image for \(debugDescription)"
                + " (artboard styleBounds: \(styleBounds))"
            return NSString(string: msg)
        }

        return image
    }
}
```

And on the iOS everything looks good!
![iOS example](../images/xcode-custom-quick-look/layer_preview.jpg)

But if we trying to check the same `Layer` with Quick look we see this
![macOS failed preview](../images/xcode-custom-quick-look/layer_failed_preview.png)

Especially interesting that it says something about `Optional`, but `debugQuickLookObject` clearly return non-optional value (and it works on iOS). After some research we found that it's common problem for Mac Catalyst to work with `UIImage` preview.

But after some time we noticed that quick look for `CIImage` works correctly. And we gave it a try. We just took bitmap and wrap it into `CIImage(cgImage: cgImage)` and check the quick look. Turned out it's working. But having preview with additional CoreImage graph is not so convinient. But now we know that the problem is only with `UIImage` and not all bitmaps preview.


![ciimage preview](../images/xcode-custom-quick-look/ciimage_preview.png)

Another possible class that represents only bitmap data is `SKTexture`. We tried to use it and that was the best workaround for us!

```
extension Artboard {
    @objc
    public func debugQuickLookObject() -> AnyObject {
        guard let image = BitmapRepresentor.image(for: self, includeBackground: true) else {
            let msg = "BitmapRepresentor returned a nil image for \(debugDescription)"
                + " (artboard styleBounds: \(styleBounds))"
            return NSString(string: msg)
        }

        #if targetEnvironment(macCatalyst)
        return SKTexture(cgImage: image.cgImage!)
        #else
        return image
        #endif
    }
}
```

Recompile & run to see the result.

![result image](../images/xcode-custom-quick-look/quick-look-result.png)

Profit.
