---
title: "Event Streaming"
date: 2024-10-11
---

Traditionally, a client-server interaction consists of a single request and a single response.
Event streaming generalizes this pattern by allowing clients and servers to send each other streams of structured messages, or events, which can be incrementally processed as they arrive.

An event stream may be present in the request, the response, neither, or both. This gives rise to four basic types of operations:

**Unary operations**

These consist of a single request and a single response. On the server, no request processing can occur until the entire request has been buffered and deserialized; similarly, on the client side, the server’s response must be fully buffered and deserialized before it can be handled. This type of operation is sometimes also referred to as buffered request-response.

**Client streaming operations**
These operations consist of a streaming request and a single response. The server processes messages from the client as they are received, without waiting for the end of the stream. Only when the client finishes sending its stream (or an error occurs) does the server transmit its response.

**Server streaming operations**
These operations consist of a single request and a streaming response. This is the mirror image of client streaming: the client sends its request and then processes events from the server as they are received. Examples of server streaming operations include [S3 Select](https://aws.amazon.com/blogs/aws/s3-glacier-select/) and [Kinesis SubscribeToShard](https://docs.aws.amazon.com/streams/latest/dev/building-enhanced-consumers-api.html).

**Bidirectional streaming operations**
These operations consist of a streaming request and a streaming response. Unlike all other operation types, bidirectional streaming operations allow the client and the server to send each other messages concurrently. This capability can be used to implement a long-lived connection between a client and a specific host. This type of operation is sometimes also referred to as full-duplex. Examples of bidirectional streaming operations include [Transcribe streaming transcription](https://docs.aws.amazon.com/transcribe/latest/dg/streaming.html) and [Kinesis Video Streams PutMedia](https://docs.aws.amazon.com/kinesisvideostreams/latest/dg/API_dataplane_PutMedia.html).

An event stream always consists of an initial event, called the initial request or initial response, followed by a stream of zero or more events; these events may be of different types. The stream can terminate normally or exceptionally.

In Java, event streams are represented using the [Reactive Streams API](https://www.reactive-streams.org/). This API consists of interfaces like Publisher, Subscriber, and Subscription. These interfaces provide a standard way for different reactive libraries to interoperate. We can use RxJava to work with event streams. Other reactive streaming integrations include Project Reactor, Akka Streams, and Kotlin Asynchronous Flow.
[You should not implement the Reactive Streams API directly](https://www.youtube.com/watch?v=_stAxdjx8qk&t=1370s). Always reuse a well-established implementation of these interfaces.


