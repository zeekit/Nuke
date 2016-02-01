---
layout: page
title: Usage
permalink: /usage/
weight: 2
---

## Zero Config

Loading an image is as simple as creating and resuming an `ImageTask`. Nuke is thread safe, you can freely create and resume tasks in any thread.

{% highlight swift %}
Nuke.taskWith(NSURL(URL: "http://...")!) {
    let image = $0.image
}.resume()
{% endhighlight %}

## Adding Request Options

Each `ImageTask` is created with an `ImageRequest` which contains request parameters. An `ImageRequest` can be initialized with either `NSURL` or `NSURLRequest`.

{% highlight swift %}
var request = ImageRequest(URLRequest: NSURLRequest(NSURL(URL: "http://...")!))

// Set target size (in pixels) and content mode that describe how to resize loaded image
// Images are resized during decompression which save resources
request.targetSize = CGSize(width: 300.0, height: 400.0) // Set target size in pixels
request.contentMode = .AspectFill

// Set filters (more on filters later)
request.processor = YourBlurImageFilter()

// Control memory caching
request.memoryCacheStorageAllowed = true // true is default
request.memoryCachePolicy = .ReloadIgnoringCachedImage // Force reload

// Change the priority of the underlying NSURLSessionTask
request.priority = NSURLSessionTaskPriorityHigh

Nuke.taskWith(request) {
    // - Image is resized to fill target size
    // - Blur is applied
    let image = $0.image
}.resume()
{% endhighlight %}

Processed images are stored into memory cache for fast access. Next time you start the request, the completion closure will be called synchronously on the callers thread.

## Using Image Response

The response passed into the completion closure is represented by `ImageResponse` enum. It has two states: `Success` and `Failure`.

{% highlight swift %}
Nuke.taskWith(request) { response in
    switch response {
    case let .Success(image, info):
        // Use image and inspect info
        if (info.isFastResponse) {
            // Image was returned from the memory cache
        }
    case let .Failure(error):
        // Handle error
    }
}.resume()
{% endhighlight %}

## Using Image Task

`ImageTask` is your primary interface for controlling the image load. Task is always in one of four states: `Suspended`, `Running`, `Cancelled` or `Completed`. The task is always created in a `Suspended` state. You can use the corresponding `resume()`, `suspend()` and `cancel()` methods to control task's state. It's always safe to call these methods, no matter in which state task currently is.

{% highlight swift %}
let task = Nuke.taskWith(imageURL).resume()
print(task.state) // Prints "Running"

// Cancels the image load, task completes with an error ImageManagerErrorCode.Cancelled
task.cancel()
{% endhighlight %}

You can also use `ImageTask` to monitor load progress.

{% highlight swift %}
let task = Nuke.taskWith(imageURL).resume()
print(task.progress) // The initial progress is (completed: 0, total: 0)

// Add progress handler which gets called periodically on the main thread
task.progressHandler = { progress in
   // Update progress
}

// Task also allows you to add multiple completion handlers, even for completed task
task.completion {
    let image = $0.image
}
{% endhighlight %}

## Using UI Extensions

Nuke provides full-featured UI extensions, which are implemented in `ImageLoadingView` protocol (trait).

{% highlight swift %}
let imageView = UIImageView()

// Loads and displays an image for the given URL
// Uses ImageContentMode.AspectFill and current view size as a target size
let task = imageView.nk_setImageWith(NSURL(URL: "http://...")!)

// Previously started requests are cancelled
{% endhighlight %}

You have some extra control over loading via `ImageViewLoadingOptions`. If allows you to provide custom `animations`, or completely override completion `handler`.

{% highlight swift %}
let imageView = UIImageView()
let request = ImageRequest(URLRequest: NSURLRequest(NSURL(URL: "http://...")!))

var options = ImageViewLoadingOptions()
options.handler = {
    // The `ImageViewLoading` trait controls the task
    // You handle its completion
}
let task = imageView.nk_setImageWith(request, options: )
{% endhighlight %}

## Adding UI Extensions

Nuke makes it extremely easy to add full-featured image loading extensions to custom UI components. You can do so by either implementing `ImageDisplayingView` protocol:

{% highlight swift %}
extension MKAnnotationView: ImageDisplayingView, ImageLoadingView {
    // That's it, you get default implementation of all methods in ImageLoadingView protocol
    public var nk_image: UIImage? {
        get { return self.image }
        set { self.image = newValue }
    }
}
{% endhighlight %}

Or providing an implementation for unimplemented `ImageLoadingView` methods:

{% highlight swift %}
extension MKAnnotationView: ImageLoadingView {
    public func nk_imageTask(task: ImageTask, didFinishWithResponse response: ImageResponse, options: ImageViewLoadingOptions) {
        // Handle task completion
    }
}
{% endhighlight %}

## UICollectionView

Sometimes you need to display a collection of images, which is a quite complex task when it comes to actually loading the images. Nuke takes care of all the complexity for you.

{% highlight swift %}
func collectionView(collectionView: UICollectionView, cellForItemAtIndexPath indexPath: NSIndexPath) -> UICollectionViewCell {
    let cell = collectionView.dequeueReusableCellWithReuseIdentifier(cellReuseID, forIndexPath: indexPath)
    let imageView: ImageView = <#view#>
    imageView.image = nil
    imageView.nk_setImageWith(imageURL)
    return cell
}
{% endhighlight %}

You can also cancel image task as soon as the cell goes offscreen (optional):

{% highlight swift %}
func collectionView(collectionView: UICollectionView, didEndDisplayingCell cell: UICollectionViewCell, forItemAtIndexPath indexPath: NSIndexPath) {
    let imageView: ImageView = <#view#>
    imageView.nk_cancelLoading()
}
{% endhighlight %}

## Applying Filters

{% highlight swift %}
let filter1: ImageProcessing = <#filter#>
let filter2: ImageProcessing = <#filter#>
let filterComposition = ImageProcessorComposition(processors: [filter1, filter2])

var request = ImageRequest(URL: <#image_url#>)
request.processor = filterComposition

Nuke.taskWith(request) {
    // Filters are applied, filtered image is stored in memory cache
    let image = $0.image
}.resume()
{% endhighlight %}

## Composing Filters

{% highlight swift %}
let processor1: ImageProcessing = <#processor#>
let processor2: ImageProcessing = <#processor#>
let composition = ImageProcessorComposition(processors: [processor1, processor2])
{% endhighlight %}

## Preheating Images

{% highlight swift %}
let requests = [ImageRequest(URL: imageURL1), ImageRequest(URL: imageURL2)]
Nuke.startPreheatingImages(requests: requests)
Nuke.stopPreheatingImages(requests: requests)
{% endhighlight %}

## Automate Preheating

{% highlight swift %}
let preheater = ImagePreheatingControllerForCollectionView(collectionView: <#collectionView#>)
preheater.delegate = self // Signals when preheat window changes
{% endhighlight %}

## Customizing Image Manager

{% highlight swift %}
let dataLoader: ImageDataLoading = <#dataLoader#>
let decoder: ImageDecoding = <#decoder#>
let cache: ImageMemoryCaching = <#cache#>

let configuration = ImageManagerConfiguration(dataLoader: dataLoader, decoder: decoder, cache: cache)
ImageManager.shared = ImageManager(configuration: configuration)
{% endhighlight %}
