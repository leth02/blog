---
title: "CONCURRENCY IN SWIFT"
date: 2024-09-25
---
There are several ways to do concurrency in Swift/Objective-C. This blog post is my attempt to gather all relevant mechanisms and serves as a reference for myself. I hope it benefits you as well.

---

### 1. GCD
Grand Central Dispatch (GCD) queues allows us to execute tasks in FIFO (First-In-First-Out) manner. A task is basically some work that our apps need to perform.

#### Types of scheduling

We schedule tasks either synchronously (sync) or asynchronously (async) with respect to the caller.

Sync means that the current thread of execution is blocked until the specified task finishes executing. This is useful when we want to avoid race conditions or other synchronization errors.

Async means that we schedule the execution of the task and continue to do other work from the calling thread.

Generally, because there is no way to know when a scheduled task will execute, `async is preferred over sync`.

#### Types of dispatch queues
A dispatch queue executes scheduled tasks either serially or concurrently.

| Type | Description |
| ----------- | ----------- |
| Serial | Serial queues execute one task at a time in the order in which they are added to the queue. |
| Concurrent | Concurrent queues (also known as a type of global dispatch queue) execute one or more tasks concurrently, but tasks are still started in the order in which they were added to the queue. The exact number of tasks executing at any given point is variable and depends on system conditions. |
| Main dispatch queue | The main dispatch queue is a globally available serial queue that executes tasks on the application’s main thread |

#### Samples
##### Dispatch a single task sync and async on a serial queue
```Swift
import Foundation
let concurrentQueue = DispatchQueue(label: "com.example.concurrentQueue")
concurrentQueue.async {
  print("Async task execution")
}

print("Async task may or may not be executed yet") // The async task is executed on another thread at a certain point of time in the future so this print line might be printed before the print line inside the async task

// The async task must be started before this sync task can be executed because this is a serial queue
concurrentQueue.sync {
    print("Sync task execution")
}
```

#### Perform a completion block when a task is done
Sometimes we want to know when the task is done along with its result. It can be done through a completion block.
TODO: add code sample here

#### Race condition 1: Schedule a task from the task that is executing in the same queue
This race condition is usually caused by a circular dependency of tasks submitted to the queue: Task A is executed and enqueue task B. In order for task A to complete, task B must complete first. But since this is a queue, task A must complete before task B. This leads to a crash.

This issue is always reproduced with sync calls to a serial queue.
```Swift
let serialQueue = DispatchQueue(label: "com.example.concurrentQueue")
func performAsyncTask(completion: @escaping (Result<String, Error>) -> Void) {
    serialQueue.sync {
        let result = "Task completed"
        
        serialQueue.sync {
            completion(.success(result))
        }
    }
}

performAsyncTask { result in
    switch result {
    case .success(let value):
        print("Success: \(value)")
    case .failure(let error):
        print("Error: \(error)")
    }
}
```

Sample crash log
```
Stack dump:
0.      Program arguments: /usr/bin/swift-frontend -frontend -interpret - -disable-objc-i
nterop -color-diagnostics -D DEBUG -new-driver-path /usr/bin/swift-driver -empty-abi-descriptor -resource-dir /usr/lib/swift -module-name main -plugin-path /usr/lib/swift/host/plugins -plugin-path /usr/local/lib/swift/host/plugins
1.      Swift version 5.10.1 (swift-5.10.1-RELEASE)
2.      Compiling with the current language version
3.      While running user code "<stdin>"
Stack dump without symbol names (ensure you have llvm-symbolizer in your PATH or set the environment var `LLVM_SYMBOLIZER_PATH` to point to it):
/usr/bin/swift-frontend(+0x61b7463)[0x55b9b662b463]
/usr/bin/swift-frontend(+0x61b541e)[0x55b9b662941e]
/usr/bin/swift-frontend(+0x61b77da)[0x55b9b662b7da]
/lib/x86_64-linux-gnu/libc.so.6(+0x42520)[0x7f42701cf520]
/usr/lib/swift/linux/libdispatch.so(+0x34928)[0x7f42700cb928]
/usr/lib/swift/linux/libdispatch.so(+0x343e1)[0x7f42700cb3e1]
/usr/lib/swift/linux/libswiftDispatch.so($s8Dispatch0A5QueueC4sync7executeyyyXE_tF+0x92)[0x7f426e009892]
[0x7f4270743f20]
/usr/lib/swift/linux/libswiftDispatch.so(+0x2091c)[0x7f426e00991c]
/usr/lib/swift/linux/libswiftDispatch.so(+0x156c9)[0x7f426dffe6c9]
/usr/lib/swift/linux/libdispatch.so(+0x345de)[0x7f42700cb5de]
/usr/lib/swift/linux/libswiftDispatch.so($s8Dispatch0A5QueueC4sync7executeyyyXE_tF+0x92)[0x7f426e009892]
[0x7f4270743815]
[0x7f427074316d]
/usr/bin/swift-frontend(+0xdd331d)[0x55b9b124731d]
/usr/bin/swift-frontend(+0xcb93f8)[0x55b9b112d3f8]
/usr/bin/swift-frontend(+0xcb7223)[0x55b9b112b223]
/usr/bin/swift-frontend(+0xc63ea6)[0x55b9b10d7ea6]
/usr/bin/swift-frontend(+0xc5f102)[0x55b9b10d3102]
/usr/bin/swift-frontend(+0xc5e18b)[0x55b9b10d218b]
/usr/bin/swift-frontend(+0xc714ba)[0x55b9b10e54ba]
/usr/bin/swift-frontend(+0xc624f8)[0x55b9b10d64f8]
/usr/bin/swift-frontend(+0xc5ff7d)[0x55b9b10d3f7d]
/usr/bin/swift-frontend(+0xaf92c0)[0x55b9b0f6d2c0]
/lib/x86_64-linux-gnu/libc.so.6(+0x29d90)[0x7f42701b6d90]
/lib/x86_64-linux-gnu/libc.so.6(__libc_start_main+0x80)[0x7f42701b6e40]
/usr/bin/swift-frontend(+0xaf83f5)[0x55b9b0f6c3f5]
💣 Program crashed: Signal 4: Backtracing from 0x7f42700cb928...
💣 Program crashed: Illegal instruction at 0x00007f42700cb928
Thread 0 "swift-frontend" crashed:
0 0x00007f42700cb928 __DISPATCH_WAIT_FOR_QUEUE__ + 360 in libdispatch.so
Backtrace took 0.00s
```

A race condition may or may not happen with async calls to a serial queue. Let's look at the code below
```Swift
let serialQueue = DispatchQueue(label: "com.example.serialQueue")
func performAsyncTask(completion: @escaping (Result<String, Error>) -> Void) {
    serialQueue.async {
        let result = "Task completed"
        
        serialQueue.async {
            completion(.success(result))
        }
    }
}

performAsyncTask { result in
    switch result {
    case .success(let value):
        print("Success: \(value)")
    case .failure(let error):
        print("Error: \(error)")
    }
}
```
The program works most of the time, but sometimes will hang or crash.
TODO: Investigate the root causes of the hang and the crash. They might have different root causes.

#### Dispatch a group task
We might want to execute a bunch of tasks and wait for all of their completions before processing their results. This can be done with a dispatch group.

---
### 2. OperationQueue

---
### 3. Async/Await

---
### 4. Others
