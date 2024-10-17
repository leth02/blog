---
title: "GCD vs Swift Concurrency"
date: 2024-10-17
---

In 2021, Apple introduces a new way to add concurrency to applications with a language-level model called Swift Concurrency. This newer approach allows developers to write more straightforward concurrent code and avoid hidden performance issues when using GCD such as thread explosion.

Here is the use case example and code sample from Apple talk: https://developer.apple.com/videos/play/wwdc2021/10254?time=297
The use case is to fetch news feeds.

With GCD, the model is:
One serial queue to process tasks that interact with the database. This ensures that access to the database is protected as serial queues ensure mutual exclusion. Let's call it the database queue.

One concurrent queue to process network tasks. Individual network tasks are independent from each other. Let's call it the network queue.

- User touch a button on UI to start the whole fetching experience
- The event submit a task on the serial queue to fetch several feed items from a database
- For each feed's URL, create a URL data task to fetch the actual content of that feed
- For each data task's result, process the data on the concurrent queue by parsing it and submitting a task to the serial queue to update the database with the result

```Swift
func deserializeArticles(from data: Data) throws -> [Article] { /* ... */ }
func updateDatabase(with articles: [Article], for feed: Feed) { /* ... */ }

let urlSession = URLSession(configuration: .default, delegate: self, delegateQueue: concurrentQueue)

for feed in feedsToUpdate {
    let dataTask = urlSession.dataTask(with: feed.url) { data, response, error in
        // ...
        guard let data = data else { return }
        do {
            let articles = try deserializeArticles(from: data)
            databaseQueue.sync {
                updateDatabase(with: articles, for: feed)
            }
        } catch { /* ... */ }
    }
    dataTask.resume()
}
```

This model however would lead to thread explosion.
When a data task completes, its result is processed on the network queue and saved to the database via a `sync` call to the serial database queue. This effectively blocks the current thread that is running on the network queue. Since this network queue is concurrent, it will try to process subsequent pending tasks by spawning more threads. If this situation continues (imagine we fetch hundreds of feeds), the app creates too many threads for its operations, and the costs of storing and switching threads between memory and CPU cores outweigh the benefits of added concurrency.


With Swift Concurrency, the threading model is different. Instead of switching threads on and off a CPU core, Swift Concurrency maintains one thread on CPU core, and that thread switches actual work items. The cost is now only associated with function calls, which is better than switching threads.

This nature is achived by two new language features:
- `await` and non-blocking of threads
- Tracking of task dependencies

These features allow Swift to make a runtime contract:
- Threads are always able to make forward progress

This contact allows integrated OS support for Swift Concurrency. This is in a new form of Cooperative thread pool:
- Default executor for Swift
- Only create as many threads as there are CPU cores
- Controlled granularity of concurrency: worker threads don't block; avoid thread explosion and context switching.




```Swift
func deserializeArticles(from data: Data) -> [Article] { /*...*/ }
func updateDatabase(with articles: [Article], for feed: Feed) async { /*...*/ }

// `group` is a continuation object
await withThrowingTaskGroup(of [Article].self) { group in
  for feed in feedsToUpdate {
    group.async {
      let (data, response) = try await URLSession.shared.data(from: feed.url)
      // ...
      let articles = try deserializeArticles(from: data)
      await updateDabase(with: articles, for: feed)
      return articles
    }
  }
}

```

### Performance of Concurrency
Tips
- Ensure that benefits of concurrency outweigh the costs of managing it
- Profile code with Instruments system trace to understand it's performance characteristics

### await and atomicity
- Cannot hold lock across await
- Thread specific data is not preserved across an `await

### Preserve the runtime contract
- Use Swift concurrency primitives: await, Actors, Task groups
