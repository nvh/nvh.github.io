---
layout: post
title: Nesting Stacks in Spine.js
date: 2012-04-17 23:55
categories: CoffeeScript Spine
featured_image: /images/featured/library.jpg
photo_attribution: creative commons licensed (BY-SA) flickr photo by joiseyshowaa
photo_attribution_link: http://flickr.com/photos/joiseyshowaa/4874702533
excerpt: >
    [Spine.js](http://spinejs.com) is a simple and lightweight MVC framework in CoffeeScript. It has a nifty feature called 'Stacks', which I can't explain better than the documentation: "Stacks are a way of grouping controllers, ensuring that only one controller is activated and displayed at any one time."
---
[Spine.js](http://spinejs.com) is a simple and lightweight MVC framework in CoffeeScript. It has a nifty feature called 'Stacks', which I can't explain better than the documentation:

> Stacks are a way of grouping controllers, ensuring that only one controller is activated and displayed at any one time.


If you aren't up to speed about Stacks and how to use them, I encourage you to read the [Spine documentation](http://spinejs.com/docs/stacks), it's really good.

When I tried nesting Stacks, I ran into the problem that the correct controllers would not always activate. In this post I will discuss the setup in which this problem arises and provide my solution to it.

<!--more-->

I used a `Root`-controller that would activate the separate sections of my application represented by their own controllers, in this example a `Resources` and a `Users` controller:

{% gist 1852497 example_root.js.coffee %}

Each controller in this Stack contains another (sub)stack which controls the different functions of that sub-controller. For example, the `Users` controller has separate controllers and routes for searching users, showing a user, and editing its profile:

{% gist 1852497 example_users.js.coffee %}

When a controller of the substack is activated (by navigating to one of the associated routes), the containing controller will not automatically activate itself. This results the right controller in the substack to be activated, but the substack itself could remain hidden because it's controller is not activated.

Usually the key in the routes hash is a string referencing a controller, but the key can also be a callback function. This can be used to add extra functionality to the activation of a route.

{% gist 1852497 substack.js.coffee %}

In this subclass of `Spine.Stack` the `@routes` hash is rewritten in the constructor. Instead of using the controller names as values, a function is defined for each route. This function will first call `@active()`, activating the controller and displaying its contents in the `Root` stack. After that, the correct sub-controller of the substack is activated by calling `@[value].active(arguments...)`. After rewriting the `@routes` hash, the constructor of `Spine.Stack` is called to setup the routing (among other things).

Note that this method relies on the fact that in the constructor of `Spine.Stack`, the controllers are instantiated and stored as properties in the class, as can be seen in this (incomplete) snippet:

```coffeescript
class Spine.Stack extends Spine.Controller
  …
  constructor: ->
    …
    for key, value of @controllers
      @[key] = new value(stack: @)
      @add(@[key])
    …
…
```

Therefore you can only define the routes to your controller with strings, but if you would want to use a function, you could call `@active()` in that function anyway.
