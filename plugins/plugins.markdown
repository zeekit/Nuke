---
layout: page
title: Plugins
permalink: /plugins/
weight: 2
---

## [Nuke Alamofire Plugin](https://github.com/kean/Nuke-Alamofire-Plugin)

[Alamofire](https://github.com/Alamofire/Alamofire) plugin for Nuke that allows you to use Alamofire for networking.

{% highlight swift %}
import Alamofire
import Nuke

let dataLoader: ImageDataLoading = AlamofireImageDataLoader(manager: <#AlamofireManager#>)
let decoder: ImageDecoding = <#decoder#>
let cache: ImageMemoryCaching = <#cache#>

let configuration = ImageManagerConfiguration(dataLoader: dataLoader, decoder: decoder, cache: cache)
ImageManager.shared = ImageManager(configuration: configuration)
{% endhighlight %}


## [Nuke AnimatedImage Plugin](https://github.com/kean/Nuke-AnimatedImage-Plugin)

[FLAnimatedImage](https://github.com/Flipboard/FLAnimatedImage) plugin for [Nuke](https://github.com/kean/Nuke) that allows you to load and display animated GIFs with a silky smooth scrolling performance and extremely low memory footprint.

<div class="video-container">
  <iframe width="640" height="360" src="https://www.youtube.com/embed/_8X7ip0Apx8?rel=0" frameborder="0" allowfullscreen></iframe>
</div>

Footage from iPod Touch 5G (slightly slower than iPhone 5)

- `AnimatedImageDecoder` creates `AnimatedImages` from received data
- `AnimatedImageLoaderDelegate` prevents `ImageLoader` from processing `AnimatedImages`
- `AnimatedImageMemoryCache` calculates proper cost for animated images, can also be used to disable animated images storage all together

{% highlight swift %}
let decoder = ImageDecoderComposition(decoders: [AnimatedImageDecoder(), ImageDecoder()])
let loader = ImageLoader(configuration: ImageLoaderConfiguration(dataLoader: <#dataLoader#>, decoder: decoder), delegate: AnimatedImageLoaderDelegate())
let cache = AnimatedImageMemoryCache()

ImageManager.shared = ImageManager(configuration: ImageManagerConfiguration(loader: loader, cahce: cache))
{% endhighlight %}

Nuke adds full-featured image loading extension to FLAnimatedImageView

{% highlight swift %}
let imageView = FLAnimatedImageView()
imageView.nk_setImageWith(<#imageRequest#>) // Loads animated image and starts playback
{% endhighlight %}
