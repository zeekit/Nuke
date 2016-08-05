---
layout: page
title: Usage
permalink: /usage/
weight: 1
---

* Table of Contents
{:toc}

## Loading Images

Nuke allows for hassle-free image loading into image views (and other arbitrary targets).

{% highlight swift %}
Nuke.loadImage(with: URL(string: "http://...")!, into: imageView)
{% endhighlight %}

## Customizing Requests

Each image request is represented by `Request` struct which can be created with either `URL` or `URLRequest`.

You can add an arbitrary number of image processors to the request. One of the built-in processors is `Decompressor` which [decompresses](https://www.cocoanetics.com/2011/10/avoiding-image-decompression-sickness/) images in the background.

{% highlight swift %}
Nuke.loadImage(with: Request(url: url).process(with: Decompressor()), into: imageView)
{% endhighlight %}


## Processing Images

You can specify custom image processors using `Processing` protocol which consists of a single method `process(image: Image) -> Image?`. Here's an example of custom image filter that uses [Core Image](https://github.com/kean/Nuke/wiki/Core-Image-Integration-Guide):

{% highlight swift %}
struct GaussianBlur: Processing {
    var radius = 8

    func process(image: UIImage) -> UIImage? {
        return image.applyFilter(CIFilter(name: "CIGaussianBlur", withInputParameters: ["inputRadius" : self.radius]))
    }

    // `Processing` protocol inherits `Equatable` to identify cached images
    func ==(lhs: GaussianBlur, rhs: GaussianBlur) -> Bool {
        return lhs.radius == rhs.radius
    }
}
{% endhighlight %}


## Preheating Images

[Preheating](https://kean.github.io/blog/image-preheating) (prefetching) means loading images ahead of time in anticipation of its use. Nuke provides a `Preheater` class that does just that:

{% highlight swift %}
let preheater = Preheater()

// User enters the screen:
let requests = [Request(url: url1), Request(url: url2), ...]
preheater.startPreheating(for: requests)

// User leaves the screen:
preheater.stopPreheating(for: requests)
{% endhighlight %}


## Automating Preheating

You can use Nuke in combination with [Preheat](https://github.com/kean/Preheat) library which automates preheating of content in `UICollectionView` and `UITableView`.

{% highlight swift %}
let preheater = Preheater()
let controller = Preheat.Controller(view: collectionView)
controller.handler = { addedIndexPaths, removedIndexPaths in
    preheater.startPreheating(for: requests(for: addedIndexPaths))
    preheater.stopPreheating(for: requests(for: removedIndexPaths))
}
{% endhighlight %}


## Loading Images Directly

One of the Nuke's core classes is `Loader`. Its API and implementation is based on Promises. You can use it to load images directly.

{% highlight swift %}
let cts = CancellationTokenSource()
Loader.shared.loadImage(with: URL(string: "http://...")!, token: cts.token)
    .then { image in print("\(image) loaded") }
    .catch { error in print("catched \(error)") }
{% endhighlight %}
