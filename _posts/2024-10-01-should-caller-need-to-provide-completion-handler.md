---
title: "Should the caller need to provide a completion handler?"
date: 2024-10-01
---
I recently reviewed a small piece of code at work that has this function signature
```Swift
public class AClass {
    private var asyncBlock: (( @escaping () -> Void) -> Void)?

    public init() { }
    public func start(asyncBlock: @escaping ( @escaping () -> Void) -> Void) {
      self.asyncBlock = asyncBlock
    }

     public func addAttribute(_ name: String, value: Any?, tab: String) {
         asyncBlock? {
             Bugsnag.addAttribute(name, withValue: value, toTabWithName: tab)
         }
     }
}
```
And here is the unit test
```Swift
import XCTest
class AClassTests: XCTestCase {
    var subject: AClass
    var asyncBlockCalledCount = 0
    var asyncBlock: ((@escaping () -> Void) -> Void)!
    override func setUp() {
        super.setUp()
        asyncBlockCalledCount = 0
        subject = AClass()
        asyncBlock = { [weak self] work in
            self?.asyncBlockCalledCount += 1
            work()
        }
    }
    func testAddAttribute() {
        subject.start(asyncBlock: asyncBlock)
        subject.addAttribute("attributeName", value: "attributeValue", tab: "tab")
        XCTAssertEqual(asyncBlockCalledCount, 1)
    }
}
```
Basically, this piece of code allow the caller of `start` to specify a completion handler that will execute some internal operation of the class. I thought to myself: "Does it make sense to leave the caller the responsibility of deciding the execution context of some operation that has nothing to do with the caller"?
And the unit test is not really helpful as well. It only checks that the passed completion handler is called, but what about the actual effect of calling `addAtribute`?

The fix should be simple. Just let the class itself decide the asynchronous execution of its own internal operation.
```Swift
public func start() {
    Task.detached {
        addAttribute(name, withValue: value)
    }
}
```
And the unit test can be updated to really test the effect of setting an attribute: the attribute value itself should be checked.

```Swift
func testAddAttributeV2() {
    subject.start()
    subject.addAttribute("attributeName", value: "attributeValue", tab: "tab")
    let result = XCTWaiter.wait(for: [expectation(description: "Add attribute happened")], timeout: 0.5)
    // Artifically wait for .5 seconds and check the set attribute in Bugsnag
    if result == XCTWaiter.Result.timedOut {
        let value = Bugsnag.getAttribute(...) as! String
        XCTAssertEqual(value, "attributeValue")
    }
}
```
