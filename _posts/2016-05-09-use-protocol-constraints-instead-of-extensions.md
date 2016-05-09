---
layout: post
title: Use Protocol Constraints Instead Of Extensions
date: 2016-05-09 22:35
categories: Swift networking protocols
featured_image: /images/featured/networking.jpg
photo_attribution: "creative commons licensed (BY-NC-ND) flickr photo by Air Force One"
photo_attribution_link: https://flickr.com/photos/airforceone/3035915842
excerpt: >
  In his great post “[Doubling Down on Protocol-Oriented Programming](http://khanlou.com/2016/05/protocol-oriented-programming/)”, Soroush Khanlou describes an improvement of his earlier Protocol-Oriented Networking approach. While this is a great technique, I think there's a small but significant way in which his approach could be improved.
---
In his great post “[Doubling Down on Protocol-Oriented Programming](http://khanlou.com/2016/05/protocol-oriented-programming/)”, [Soroush Khanlou](http://www.twitter.com/khanlou) describes an improvement of his earlier [Protocol-Oriented Networking](http://khanlou.com/2015/06/protocol-oriented-networking/) approach. While this is a great technique, I think there's a small but significant way in which his approach could be improved.

Let's look at his definition of the `SendableRequest` protocol:

```
protocol SendableRequest: ConstructableRequest { }
```

This inheritance enforces that every `SendableRequest` should also be a `ConstructableRequest`. But what if we don’t care about the request being constructable, but do want to take advantage of the handy `ResultParsing` functionality described in the second half of the post?

With the current implementation, you’re out of luck. You have to implement `buildRequest()` (or use it’s default implementation) if you want to conform to `SendableRequest`.

Luckily there’s a good solution for this: by defining `SendableRequest` without inheritance and adding the restriction to the extension:

```
protocol SendableRequest {
	func sendRequest(success success: (string: String) -> (), failure: (error: ErrorType) -> ())
}

extension SendableRequest where Self: ConstructableRequest {
    func sendRequest(success success: (string: String) -> (), failure: (error: ErrorType) -> ()) {
        // send the request
        // parse the result
        // fire the blocks on success and failure
    }
}
```

The key part here is the restriction of the extension:

```
where Self: ConstructableRequest
```

This defines a constraint on the protocol extension, such that _if_ your implementation conforms to both `SendableRequest` _and_ `ConstructableRequest`, the default implementation in the extension is available. Because this extension is limited to the conformance of both protocols, it’s possible to use the `buildRequest()` function inside `sendRequest()`, even though the `SendableRequest` protocol doesn’t enforce it itself. However, it's perfectly possible conform to `SendableRequest` without implementing `ConstructableRequest` methods.

This solution improves the composability and reusability of protocols and their extensions. This makes it easier to use only parts of a framework or implementation, and enforces you to think about the separate concerns different protocols should have.
