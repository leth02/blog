---
title: "Should the caller need to provide a completion handler?"
date: 2024-10-01
---
I recently came across a small piece of code at work that has this function signature
```Swift
public class AClass {
    private var asyncBlock: (( @escaping () -> Void) -> Void)?

    public init() { }
    public func start(asyncBlock: @escaping ( @escaping () -> Void) -> Void) {
      self.asyncBlock = asyncBlock
    }

     public func addAttribute(_ name: String, value: Any?) {
         asyncBlock? {
             addAttribute(name, withValue: value)
         }
     }
}
```
Basically, this piece of code allow the caller of `start` to specify the completion handler, or what I usually call an asynchronous execution context, that will execute some internal operation of the class.
Depending on your use case, this pattern might not work for you. Does it make sense to leave the caller the responsibility of deciding the execution context of some operation that has nothing to do with the caller?

The fix I saw was quite simple. Just let the class itself decide the asynchronous execution of its own internal operation.
```Swift
public func start() {
    Task.detached {
        addAttribute(name, withValue: value)
    }
}
```
