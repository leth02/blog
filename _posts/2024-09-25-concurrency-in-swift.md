---
title: "CONCURRENCY IN SWIFT"
date: 2024-10-01
---
There are several ways to do concurrency in Swift/Objective-C. This blog post is my attempt to gather all relevant mechanisms and serves as a reference for myself. I hope it benefits you as well.
The choice of concurrency framework should depend on your specific use cases, codebases, etc. Any of the frameworks mentioned here can be used to achieve common concurrency needs.

---

# 1. GCD
Grand Central Dispatch (GCD) queues allows us to execute tasks in FIFO (First-In-First-Out) manner. A task is basically some work that our apps need to perform.

## Types of scheduling

We schedule tasks either synchronously (sync) or asynchronously (async) with respect to the caller.

Sync means the current thread of execution is blocked until the specified task finishes executing. This is useful when we want to avoid race conditions or other synchronization errors.

Async means we schedule the task and continue to do other work from the calling thread.

Generally, because there is no way to know when a scheduled task will execute, `async is preferred over sync`.

## Types of dispatch queues
A dispatch queue executes scheduled tasks either serially or concurrently.

| Type | Description |
| ----------- | ----------- |
| Serial | Serial queues execute one task at a time in the order in which they are added to the queue. |
| Concurrent | Concurrent queues (also known as a type of global dispatch queue) execute one or more tasks concurrently, but tasks are still started in the order in which they were added to the queue. The exact number of tasks executing at any given point is variable and depends on system conditions. |
| Main dispatch queue | The main dispatch queue is a globally available serial queue that executes tasks on the application’s main thread |

## Samples
### Dispatch a single task sync or async on a serial queue
```Swift
import Foundation
let serialQueue = DispatchQueue(label: "com.example.serialQueue")
concurrentQueue.async {
  print("Async task execution")
}

print("Async task may or may not be executed yet") // The async task is executed on another thread at a certain point of time in the future so this print line might be printed before the print line inside the async task

// Execution of the async task above must be done before this sync task can be executed because this is a serial queue
concurrentQueue.sync {
    print("Sync task execution")
}
```

### Perform a completion block when a task is done
Sometimes we want to know when the task is done along with its result. It can be done through a completion handler block.
```Swift
let serialQueue = DispatchQueue(label: "com.example.concurrentQueue")
func performAsyncTask(completion: @escaping () -> Void) {
    serialQueue.sync {
        let result = "Task completed"
        
        serialQueue.sync {
            completion()
        }
    }
}

performAsyncTask {
  print("Completion handler called after the task finished.")
}
```

### Race condition 1: Schedule a task from the task that is already executing on the same queue
#### a. Calling sync inside a sync on the same serial queue will cause a deadlock
```Swift
import Foundation
let serialQueue = DispatchQueue(label: "serialQueue")
serialQueue.sync {
  print("outer sync body")
  serialQueue.sync {
    print("inner sync body")
  }
}
```
Explanation: The outer sync task starts executing and the queue can't start another task until it completes. Inside its body, the outer task then calls a sync task, which means the outer can't complete until the inner sync task completes. However, the inner task can't even start, let alone complete, until the outer task completes (the basic characteristic of a serial queue).

#### b. Calling sync inside an async on the same serial queue will cause a deadlock
```Swift
import Foundation
let serialQueue = DispatchQueue(label: "serialQueue")
serialQueue.async {
  print("outer sync body")
  serialQueue.sync {
    print("inner sync body")
  }
}
print("after outer async")
```
Explanation: The outer async task is submitted to the queue and the calling threads move on to execute `print("after outer sync")`. At a certain time in the future, the outer task is executed on the queue by another thread. The task then calls a sync task, which means the outer can't complete until the inner sync task completes. However, the inner task can't even start, let alone complete, until the outer task completes (the basic characteristic of a serial queue).

<mark> Apple's recommendation: You should never call `sync` function(s) from a task that is executing on the same queue. This is particularly important for serial queues, which are guaranteed to deadlock, but should also be avoided for concurrent queues. If you need to dispatch to the current queue, do so asynchronously via `async` functions. </mark>


### Dispatch a group task
We might want to execute a bunch of tasks asynchronously, wait for all of their completions, then processing or aggregating their results. This can be done with a [dispatch group](https://developer.apple.com/documentation/dispatch/dispatchgroup).

```Swift
// Create a dispatch group
let dispatchGroup = DispatchGroup()

// Use a global queue (concurrent) to submit the tasks
let globalQueue = DispatchQueue.global(qos: .userInitiated)
let myQueue = DispatchQueue(label: "myQueue")

// Submit the first task
dispatchGroup.enter()
globalQueue.async {
    print("Task 1 started")
    Thread.sleep(forTimeInterval: 1) // Simulating a task
    print("Task 1 completed")
    dispatchGroup.leave()
}

// Submit the second task
dispatchGroup.enter()
globalQueue.async {
    print("Task 2 started")
    Thread.sleep(forTimeInterval: 2) // Simulating a task
    print("Task 2 completed")
    dispatchGroup.leave()
}

// Notify when all tasks are complete
dispatchGroup.notify(queue: myQueue) {
    print("All tasks have been completed.")
}
```


---
### 2. OperationQueue
GCD queues are great for scheduling tasks to be executed serially or concurrently. However, they do not have built-in support for several cases 

---
### 3. Async/Await & Task
Swift 5.5 introduces a new way to write concurrent programs, using a concept called <mark>structured concurrency</mark>.

Before this, we use GCD or OperationQueue along with completion handlers. This older approach has several drawbacks pointed out by Apple's presentation. Let's look at their example.

Example: Fetch a bunch of images and resize them to be thumbnails _sequentially_.
```Swift
// Asynchronous code with completion is unstructured.
func fetchThumbnails(
    for ids: [String],
    completion handler: @escaping ([String: UIImage]?, Error?) -> Void
) {
    guard let id = ids.first else { return handler([:], nil) }
    let request = thumbnailURLRequest(for: id)
    let dataTask = URLSession.shared.dataTask(with: request) { data, response, error in
        guard let response = response,
              let data = data
        else {
            return handler(nil, error)
        }
        // ... check response ...
        UIImage(data: data)?.prepareThumbnail(of: thumbSize) { image in
            guard let image = image else {
                return handler(nil, ThumbnailFailedError())
            }
            fetchThumbnails(for: Array(ids.dropFirst())) { thumbnails, error in
                // ... add image to thumbnails ...
            }
        }
    }
    dataTask.resume()
}
```
Using completion handler pattern, this code cannot use structured control-flow for error handling or a loop to process each thumbnail.

The code above can be rewritten to use `async/await` syntax. This results in much cleaner code.
```Swift
func fetchThumbnails(for ids: [String]) async throws -> [String: UIImage] {
    var thumbnails: [String: UIImage] = [:]
    for id in ids {
        let request = thumbnailURLRequest(for: id)
        let (data, response) = try await URLSession.shared.data(for: request)
        try validateResponse(response)
        guard let image = await UIImage(data: data)?.byPreparingThumbnail(ofSize: thumbSize) else {
            throw ThumbnailFailedError()
        }
        thumbnails[id] = image
    }
    return thumbnails
}
```

Note that both code samples are still fetching images sequentially.

Now, imagine that we need to fetch many images or items, we should do fetching operations concurrently with the usage of `Task`.
A task provides a new async context for executing code concurrently. In other words, a task provides an execution context to run asynchronous code, and each task can run concurrently with respect to other execution contexts. They will be automatically scheduled to run in parallel if it is safe and efficient for the system to do so.
Note that calling `async` function does NOT create a new Task for the call. We need to do it explicitly.

#### Types of Tasks and related concepts:
##### Type 1: Async-let
Let's first talk about sequential binding. Assume that we want to fetch an image from the network, this code will block on the initialzation of `data` variable until the image is fetched. Other work can only be started after the image value is assigned to `data`.
```Swift
let data = try await URLSession.shared.data(...)
print("other work")
```

If we do not want to wait for the image fetching, we can turn it into concurrent binding with `async-let` syntax. This creates a child task to fetch the image, while the current code, aka. the parent task, continue its execution. The parent task will only await the completion of the child task when it reaches an expression that needs the actual result of the value.
```Swift
async let data = URLSession.shared.data(...)
print("other work...")
try await data
```

We can update the thumbnail fetching code above to use `async let`. We also refactor the code to fetch a single thumbnail into its own function. Now:
With sequential binding, notice that `try await` is on the right hand side of the expression because that is where an error or suspension would be observed.
```Swift
func fetchOneThumbnail(withID id: String) async throws -> UIImage {
    let imageReq = imageRequest(for: id), metadataReq = metadataRequest(for: id)
    let (data, _) = try await URLSession.shared.data(for: imageReq)
    let (metadata, _) = try await URLSession.shared.data(for: metadataReq)
    guard let size = parseSize(from: metadata),
          let image = UIImage(data: data)?.byPreparingThumbnail(ofSize: size)
    else {
        throw ThumbnailFailedError()
    }
    return image
}
```

With concurrent binding, notice that now `async` is on the left side of the lets.
Explanation: Both downloads now happen concurrently on child tasks. Now, the parent task only observe those downloads' effects when it actually _use_ the variables that are concurrently bound.
```Swift
func fetchOneThumbnail(withID id: String) async throws -> UIImage {
    let imageReq = imageRequest(for: id), metadataReq = metadataRequest(for: id)
    async let (data, _) = URLSession.shared.data(for: imageReq)
    async let (metadata, _) = URLSession.shared.data(for: metadataReq)
    guard let size = parseSize(from: try await metadata),
          let image = try await UIImage(data: data)?.byPreparingThumbnail(ofSize: size)
    else {
        throw ThumbnailFailedError()
    }
    return image
}
```

#### Task tree
Whenever we make a call from one async function to another, the same task is used to execute the call. The function therefore inherits all attributes of that task. If we create a new structured task inside the function, it becomes the child of the task that the function is running on. Task are not the child of a function, but their lifetime may be scoped to it.

A task tree is made up of links between each parent and its child tasks. A link enforces that <mark>a parent task can only finish its work if all of its child tasks have finished, even if abnormal control-flow which would prevent a child task from being awaited.</mark> Swift will automatically mark the unwaited task as cancelled and await for it to finish before exiting the function. Marking a task as cancelled does not stop the task. If a task is cancelled, all of its subtasks are cancelled, too. The function only exits when all of the structured tasks it created directly or indirectly have finished. This is called structured task guarantee.



#### Cooperative cancelation
Since tasks are not stopped immediately when cancelled, we must check for the cancellation status of current task in an appropriate way.

Let's look at the fetching thumbnail code below which fetches many thumbnails. In the case that the task calling this function is cancelled, the code will throw an error and will not make unnecessary calls to fetch unused thumbnails.
```Swift
// Checking for cancellation by calling a method that throws
func fetchThumbnails(for ids: [String]) async throws -> [String: UIImage] {
    var thumbnails: [String: UIImage] = [:]
    for id in ids {
        try Task.checkCancellation()
        thumbnails[id] = try await fetchOneThumbnail(withID: id)
    }
    return thumbnails
}
```

If we still want to receive partial result instead of a thrown error, we can use `Task.isCancelled`.
```Swift
// Checking for cancellation by calling a method that throws
func fetchThumbnails(for ids: [String]) async throws -> [String: UIImage] {
    var thumbnails: [String: UIImage] = [:]
    for id in ids {
        if Task.isCancelled() { break }
        thumbnails[id] = try await fetchOneThumbnail(withID: id)
    }
    return thumbnails
}
```


#### Type 2: Task Group
`Async-let` as shown in the fetching thumbnail examples above is for concurrency with static width. In other words, each thumbnail ID in the for-loop will call `fetchOneThumbnail` which in turns create exactly two child tasks. It means that the two child tasks must complete before the next loop iteration begins.

If we want this loop to start fetching all of the thumbnails concurrently, the amount of concurrency is not known statically anymore because it now depends on the number of thumbnails to fetch. We can use a Task Group to achieve this.
```Swift
func fetchThumbnails(for ids: [String]) async throws -> [String: UIImage] {
    var thumbnails: [String: UIImage] = [:]
    try await withThrowingTaskGroup(of: Void.self) { group in
        for id in ids {
            group.async {
                // Error: Mutation of captured var 'thumbnails' in concurrently executing code
                thumbnails[id] = try await fetchOneThumbnail(withID: id)
            }
        }
    }
    return thumbnails
}
```
The code above uses `withThrowingTaskGroup` to create a Task Group that allow us to create child tasks that might throw errors. We create child tasks in a group by using its `async` method. Tasks added to a group cannot outlive the scope of the block in which the group is defined. Once added to a group, child tasks being executing immediately and in any order. <mark>When a group object goes out of scope, the completion of all tasks within it will be implicitly awaited.</mark> This is due to the task tree rule discussed above as group tasks are also structured. We can use async-let within group tasks or create task groups within async-let tasks.

Now, a new task is created for each call to `fetchOneThumbnail`, which in turns create two more child tasks. However, the code above has an error: The `thumbnails` dictionary cannot handle more than one access at a time. If two child tasks tried to insert thumbnails simultaneously, it could cause a crash or data corruption.

__Data-race Safety__: When a task is created, the work it performs is within a new closure type called a `@Sendable` closure, which is restricted from capturing mutable variables in its lexical context. It should only capture values that are safe to share: value types, actors, or classes that implement their own synchronization. Refer to `Protect mutable state with actors` for more info.

We can fix the issue above by having each child task return a value and let the parent task bear the responsibility of processing the result.
```Swift
func fetchThumbnails(for ids: [String]) async throws -> [String: UIImage] {
    var thumbnails: [String: UIImage] = [:]
    try await withThrowingTaskGroup(of: (String, UIImage).self) { group in
        for id in ids {
            group.async {
                return (id, try await fetchOneThumbnail(withID: id))
            }
        }
        // Obtain results from the child tasks, sequentially (which means we can safely add the thumbnail to the dictionary), in order of completion.
        // This is an example of accessing a asynchronous sequence of values. Refer to `AsyncSequence` for more info.
        for try await (id, thumbnail) in group {
            thumbnails[id] = thumbnail
        }
    }
    return thumbnails
}
```

Note: There is a small difference in how the task tree rule is implemented for group tasks vs async-let tasks. Suppose when iterating through the results of a task group, we encounter a child task that completed with an error. Because that error was thrown out of the group's block, all tasks in the group will then be implicitly cancelled and then awaited. This works the same as async-let. However, the difference comes when a group goes out of scope through a normal exit from the block. Then, cancellation is not implicit. This allows us to express the fork-join pattern easier with a task group.


#### Type 3: Unstructured Tasks
We can also create unstructured tasks when we don't need a hierarchy for our tasks. Some situations include:
- <mark>We don't have a parent task: The tasks need to launch from non-async contexts</mark>
- <mark>The lifetime we want for a task might extend beyond a single scope or even a function.</mark> For example, we might want to start a task in response to a method call that puts an object into an active state, and then cancel its execution in response to a different method call that deactivates the object. This comes up a lot when implementing delegate objects in AppKit and UIKit.
 

#### Type 4: Detached Tasks
Similar to unstructured tasks, but do not inherit origin context.


#### Protect mutable state with actors

#### AsyncSequence
A description of how to produce values over time, this new protocol type resembles the `Sequence` type - offering a list of values we can step through one at a time - and adds asynchronicity. Each element is delivered asynchronously. An AsyncSequence will suspend on each element and resume when the underlying iterator produces a value or throws an error. We use for await-in loop to iterate through an AsyncSequence. It completes when its iterator returns nil. When errors occur, they will return nil on subsequent calls to next on their iterator.

```Swift
// Asynchronously loop over an AsyncSequence
for await quake in quakes {
    if quake.magnitude > 3 {
        displayEarthquake(quake)
    }
}

// if an AsyncSequence can throw, use `for try await`
do {
    for try await quakes in quakeDownload {
        print(quakes)
    }
} catch {}
```

In the code sample above, the second iterarion runs after the first iteration completes. However, running code sequentially is not always what's desired. We can run an interarion concurrent to other things going on by creating a new async task that encapsulates the iteration. This can be useful when we know our async sequences may run indefinitely.
```Swift
Task {
    for await quake in quakes {}
}

Task {
    do {
        for try await quake in quakeDownload {}
    } catch {}
}
```

We can also cancel the iteration externally
```Swift
let iteration1 = Task {
    for await quake in quakes {}
}

let iteration2 = Task {
    do {
        for try await quake in quakeDownload {}
    } catch {}
}

// ... later on
iteration1.cancel()
iteration2.cancel()
```

**AsyncSequence APIs and use cases**
1/ Read bytes asynchronously from a `FileHandle`
```Swift
for try await line in FileHandle.standardInput.bytes.lines {}
```

2/ Read bytes or lines asynchronously from a URL (can be used for both file and network)
```Swift
let url = URL(fileURLWithPath: "/tmp/somefile.txt")
for try await line in url.lines {}
```

3/ Read byets asynchronously from `URLSession`
```Swift
let (bytes, response) = try await URLSession.shared.bytes(from: url)
guard let httpResponse = response as? HTTPURLResponse,
    httpResponse.statucCode == 200 else {
  throw MyNetworkingError.invalidServerResponse
}
for try await byte in bytes {}
```

4/ Await notifications asynchronously with a predicate
```Swift
let center = NotificationCenter.default
let notification = await center.notifications(named: .NSPersistentStoreRemoteChange).first {
    $0.userInfo[NSStoreUUIDKey] == storeUUID
}
```

**Create your own AsyncSequence with AsyncStream/AsyncThrowingStream**
```Swift
class QuakeMonitor {
    var quakeHandler: (Quake) -> Void
    func startMonitoring()
    func stopMonitoring()
}

let quakes = AsyncStream(Quake.self) { continuation in
    let monitor = QuakeMonitor()
    monitor.quakeHandler = { quake in
        continuation.yield(quake)
    }
    continuation.onTermination = { @Sendable _ in
        monitor.stopMonitoring()
    }
    monitor.startMonitoring()
}

let significantQuakes = quakes.filter { quake in
    quake.magnitude > 3
}

for await quake in significantQuakes {
    ...
}
```

---
### References
I appreciate all the resources I have read below

[Apple's archived Concurrency Programming Guide](https://developer.apple.com/library/archive/documentation/General/Conceptual/ConcurrencyProgrammingGuide/Introduction/Introduction.html#//apple_ref/doc/uid/TP40008091-CH1-SW1)

[Explore Structured Concurrency in Swift](https://developer.apple.com/videos/play/wwdc2021/10134/)


