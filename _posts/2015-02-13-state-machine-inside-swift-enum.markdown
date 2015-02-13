---
layout: post
title: Swift's enum as a DataSource state machine
date: 2015-02-13 16:36
categories: Swift iOS enum
featured_image: /images/featured/highvoltage.jpg
photo_attribution: "creative commons licensed (BY-NC-SA) flickr photo by _boris"
photo_attribution_link: http://flickr.com/photos/_boris/311189091
excerpt: Last week at dotSwift.io, two of the presentations   contained a version of the following example of using associated values of Swift enumerations…
---
Last week at [dotSwift.io](http://www.dotswift.io), two of the presentations contained a version of the following example of using associated values of Swift enumerations:

```
enum State<D> {
  case Empty
  case Loading
  case Ready(D)
  case Error(NSError)
}

struct DataSource<T> {
  var state: State<T>
}
```

<!--more-->

This is nice[^1], because it can replace the separate variables for keeping state and the data associated with it:

```
struct DataSource<T> {
  var loading: Bool
  var data: T?
  var error: NSError?
}
```

This is of course a great example for a presentation, but it makes the assumption that you only want to retrieve that data in the `.Ready` state. But what if you want to reload the data? You will have to switch to the `.Loading` state, but because the data is associated with the `.Ready` state, you can't access the previously loaded data anymore. And even worse: if the reloading fails and you end up in the `.Error` state, you can't even show the data that was shown before we reloaded!

![State machine](/images/state_machine.svg){: .centered .full-width .max-width-500 }

If we look at the above image, we see that we need a representation of data in three of the four states. We can model that with optional associated values as follows:

```
enum State<D> {
    case Empty
    case Loading(D?)
    case Ready(D)
    case Error(NSError,D?)
}
```

Ouch. That suddenly doesn't look as nice anymore, does it? But bear with me, we can fix this. We can define a computed property on the state enum that provides us with the data or error, no matter what the current state is:

```
extension State {
    var data: D? {
        switch self {
        case .Empty:
            return nil
        case .Ready(let data):
            return data
        case .Loading(let data):
            return data
        case .Error(_, let data):
            return data
        }
    }

    var error: NSError? {
        switch self {
        case .Error(let error, _):
            return error
        default:
            return nil
        }
    }  
}
```

Ok, now we can retrieve the data in a better way, but what about associating the data with the enum? That can also be handled by defining some mutating functions on the enum:

```
extension State {
    mutating func toLoading() {
        switch self {
        case .Ready(let oldData):
            self = .Loading(oldData)
        default:
            self = .Loading(nil)
        }
    }

    mutating func toError(error:NSError) {
        switch self {
        case .Loading(let oldData):
            self = .Error(error,oldData)
        default:
            assert(false, "Invalid state transition to .Error from other than .Loading")
        }
    }

    mutating func toReady(data: D) {
        switch self {
        case .Loading:
            self = .Ready(data)
        default:
            assert(false, "Invalid state transition to .Ready from other than .Loading")
        }
    }
}
```

Using these functions to switch state serves three purposes:
- Abstracting away the implementation details of where and how the data is stored in the enum
- Making sure there can be no invalid state transitions
- Enforcing the associated data to be present at the moment you switch to a state

This state machine is actually pretty nice to use now:

```
class DataSource<T> {
  var state: State<[T]> = .Empty

  func load() {
    state.toLoading()
    requestData({ data,error in
      if error != nil {
        self.state.toError(error)
      } else {
        self.state.toReady(data)
      }
    })
  }

  func numberOfItems() -> Int {
    return state.data?.count ?? 0
  }
}

```

This enum could be further extended to have callbacks when changing the state, so it is much easier implement behaviour upon state change. The full code for this post can be found in [this gist](https://gist.github.com/nvh/1b09b32a335be28e47e8)

---
[^1]: The actual enum would look a bit less nice, because of limitations in Swift. You'll have to [box](https://github.com/robrix/Box) the generic value, but we'll ignore that for readability in this post
