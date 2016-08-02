---
layout: page
title: Usage
permalink: /usage/
weight: 1
---

* Table of Contents
{:toc}

## Loading Images

{% highlight swift %}
Nuke.loadImage(with: URL(string: "http://...")!).then { image in
    print("\(image) loaded")
}
{% endhighlight %}

## Customizing Requests

Each image request is represented by `Request` struct which can be initialized either with `URL` or `URLRequest`.

{% highlight swift %}
Nuke.loadImage(with: Request(urlRequest: URLRequest(url: (URL: "http://...")!)))
{% endhighlight %}

## Using Response

Each of the methods from `loadImage(with:...)` family returns a `Promise<Image>` with expected methods like `then`, `catch`, etc.

{% highlight swift %}
// The closures get called on the main thread by default.
Nuke.loadImage(with: URL(string: "http://...")!)
    .then { image in print("\(image) loaded") }
    .catch { error in print("catched \(error)") }
{% endhighlight %}

It also has a more conventional in iOS `completion` method:

{% highlight swift %}
Nuke.loadImage(with: URL(string: "http://...")!).completion { resolution in
    switch resolution {
    case let .fulfilled(image): print("\(image) loaded")
    case let .rejected(error): print("catched \(error)")
    }
}
{% endhighlight %}

## Cancelling Request

If you need to cancel your requests you should create them with a [`CancellationToken`](https://msdn.microsoft.com/en-us/library/system.threading.cancellationtokensource(v=vs.110).aspx).
{% highlight swift %}
let cts = CancellationTokenSource()
Nuke.loadImage(with: URL(string: "http://...")!, token: cts.token).then { image in
    print("got \(image)")
}
cts.cancel()
{% endhighlight %}

This pattern provides a simple and reliable model for cooperative cancellation of asynchronous operations.

## Using UI Extensions

Nuke provides UI extensions to make image loading as simple as possible.

{% highlight swift %}
let imageView = UIImageView()

// Loads and displays an image for the given URL. Previously started request is cancelled.
imageView.nk_setImage(with: URL(string: "http://...")!)
{% endhighlight %}

## Adding UI Extensions

It's also extremely easy to add image loading capabilities (trait) to custom UI components. All you need is to implement `ResponseHandling` protocol in your view which consists of a single method `nk_handle(response:isFromMemoryCache:)`.

{% highlight swift %}
extension MKAnnotationView: ResponseHandling {
    public func nk_handle(response: PromiseResolution<Image>, isFromMemoryCache: Bool) {
        // display image, handle error, etc
    }
}
{% endhighlight %}

Each view that implements `ResponseHandling` gets a bunch of method for loading images.

{% highlight swift %}
let view = MKAnnotationView()
view.nk_setImage(with: Request(urlRequest: <#request#>))

{% endhighlight %}

## Customizing UI Extensions

Each view with image loading trait also get and an associated `ViewContext` object which is your primary interface for customizing image loading.

{% highlight swift %}
let view = UIImageView()
view.nk_context.loader = <#loader#> // `Loader.shared` by default.
view.nk_context.cache = <#cache#> // `Cache.shared` by default.
view.nk_context.handler = { _ in // Overwrite deafult handler.
    // Handler response
}

{% endhighlight %}

## UICollection(Table)View

When you display a collection of images it becomes quite tedious to manage tasks associated with image cells. Nuke takes care of all that complexity for you:

{% highlight swift %}
func collectionView(collectionView: UICollectionView, cellForItemAtIndexPath indexPath: NSIndexPath) -> UICollectionViewCell {
    let cell = collectionView.dequeueReusableCellWithReuseIdentifier(cellReuseID, forIndexPath: indexPath)
    let imageView: ImageView = <#view#>
    imageView.image = nil
    imageView.nk_setImage(with: imageURL)
    return cell
}
{% endhighlight %}

## Applying Filters

Nuke defines a simple `Processing` protocol that represents image filters. It takes just a couple line of code to create your own filters. You can apply filters by adding them to the `Request`.

{% highlight swift %}
let filter1: Processing = <#filter#>
let filter2: Processing = <#filter#>

var request = Request(url: <#image_url#>)
request.add(processor: filter1)
request.add(processor: filter2)

Nuke.loadImage(with: request).then { image in
    // Filters are applied, processed image
}.resume()
{% endhighlight %}

## Creating Filters

`Processing` protocol consists of a single method `process(image: Image) -> Image?`. Here's an example of custom image filter that uses [Core Image](https://developer.apple.com/library/mac/documentation/GraphicsImaging/Conceptual/CoreImaging/ci_intro/ci_intro.html). For more info see [Core Image Integration Guide](https://github.com/kean/Nuke/wiki/Core-Image-Integration-Guide).

{% highlight swift %}
struct ImageFilterGaussianBlur: Processing {
    private let radius: Int
    init(radius: Int = 8) {
        self.radius = radius
    }

    func process(image: UIImage) -> UIImage? {
        // The `applyFilter` function is not shipped with Nuke.
        return image.applyFilter(CIFilter(name: "CIGaussianBlur", withInputParameters: ["inputRadius" : self.radius]))
    }

    // `Processing` protocol also requires filters to be `Hashable`.
    // Nuke compares filters to be able to identify cached images and deduplicate equivalent requests.
    func ==(lhs: ImageFilterGaussianBlur, rhs: ImageFilterGaussianBlur) -> Bool {
        return lhs.radius == rhs.radius
    }
}
{% endhighlight %}

## Preheating Images

[Preheating](https://kean.github.io/blog/image-preheating) means loading and caching images ahead of time in anticipation of its use. Nuke provides a `Preheater` class with a set of self-explanatory methods for image preheating which were inspired by [PHImageManager](https://developer.apple.com/library/prerelease/ios/documentation/Photos/Reference/PHImageManager_Class/index.html):

{% highlight swift %}
let preheater = Preheater(loader: Loader.shared)

// User enters the screen:
let requests = [Request(url: imageURL1), Request(url: imageURL2), ...]
preheater.startPreheating(for: requests)

// User leaves the screen:
preheater.stopPreheating(for: requests)
{% endhighlight %}

## Automating Preheating

You can use Nuke with [Preheat](https://github.com/kean/Preheat) library which automates preheating of content in `UICollectionView` and `UITableView`. For more info see [Image Preheating Guide](https://kean.github.io/blog/image-preheating), Nuke's demo project, and [Preheat](https://github.com/kean/Preheat) documentation.

{% highlight swift %}
let preheater = Nuke.Preheater(loader: Loader.shared)
let controller = Preheat.Controller(view: <#collectionView#>)
controller.handler = { addedIndexPaths, removedIndexPaths in
    preheater.startPreheating(for: requests(for: addedIndexPaths))
    preheater.stopPreheating(for: requests(for: removedIndexPaths))
}
{% endhighlight %}

## Caching Images

Nuke provides both on-disk and in-memory caching.

For on-disk caching it relies on `URLCache`. The `URLCache` is used to cache original image data downloaded from the server. This class a part of the URL Loading System's cache management, which relies on HTTP cache.

As an alternative to `URLCache` `Nuke` provides a `DataCaching` protocol that allows you to easily integrate any third-party caching library.

For on-memory caching Nuke provides `Caching` protocol and its implementation in `Cache` class built on top of `Foundation.Cache`. The `Cache` is used for fast access to processed images that are ready for display.

The combination of two cache layers results in a high performance caching system. For more info see [Image Caching Guide](https://kean.github.io/blog/image-caching) which provides a comprehensive look at HTTP cache, URL Loading System and NSCache.
