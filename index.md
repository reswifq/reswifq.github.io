---
layout: default
title: Reswifq - Simple, reliable, background processing for Swift
---

# Overview

Reswifq (pronounced like _re - swif - queue_) is a Redis-backed package for creating background jobs, and processing them later. It has an **at least once** delivery guarantee and it strictly implements the **reliable queue pattern**.

Background jobs can be any Swift type that conforms to the `Job` protocol. Your existing types can easily be converted to background jobs or you can create new types specifically to do work. Or, you can do both.

Reswifq, comprises different parts:

- A Swift package for creating, querying, and processing jobs
- A generic [abstract module](https://github.com/reswifq/redis-client) to interface with any Swift Redis client
- A [Kitura Redis client](https://github.com/reswifq/redis-client-kitura) adapter to use out of the box
- A [Vapor Redis client](https://github.com/reswifq/redis-client-vapor) adapter to use out of the box
- A Dashboard app for monitoring queues, jobs, and workers. (🚧 work in progress 🚧)

Reswifq workers can be distributed between multiple machines, are resilient to memory bloat / leaks, tell you what they're doing, and expect failure, unfortunately, [priorities are not supported](https://github.com/antirez/redis/issues/1785) at the moment.

Reswifq queues are persistent; support constant time, atomic push and pop (thanks to Redis); provide visibility into their contents; and store jobs as simple JSON packages.

This repository is the framework's source code. To view some sample projects, check out our demo [here](https://github.com/reswifq/demo).

# Getting Started

Reswifq allows you to create jobs and place them on a queue, then, later, pull those jobs off the queue and process them.

Reswifq jobs are Swift classes (or struct) which conforms to the `Job` protocol. Here's an example:

```swift
struct SendEmail: Job {

    let recipient: String
    let subject: String
    let body: String

    // MARK: DataEncodable

    func data() throws -> Data {

        let object: [String: Any] = [
            "recipient": self.recipient,
            "subject": self.subject,
            "body": self.body
        ]

        return try JSONSerialization.data(withJSONObject: object)
    }


    // MARK: DataDecodable

    init(data: Data) throws {

        let object = try JSONSerialization.jsonObject(with: data)

        guard let dictionary = object as? Dictionary<String, Any> else {
            throw DataDecodableError.invalidData(data)
        }

        guard let recipient = dictionary["recipient"] as? String,
            let subject = dictionary["subject"] as? String,
            let body = dictionary["body"] as? String
            else {
                throw DataDecodableError.invalidData(data)
        }

        self.recipient = recipient
        self.subject = subject
        self.body = body
    }

    // MARK: Job

    func perform() throws {

        try SMTPClient.send(to: recipient, subject: subject, body: body)
    }
}

```

In order to conform to the `Job` protocol we have to also conform to the `DataEncodable` and `DataDecodable` protocols. In other words, we have to tell Reswifq how to serialize and deserialize your job, a process you could already be familiar with, if you've used `NSCoding` before. In this example we are using JSON, but any format can be used as long as it's possible to convert the job from and to `Data`.

In addition to the above we may want to also set the following properties (optional):

- **type** - the job type as stored in the `jobMap`, default to the type name
- **timeToLive** -  the maximum time a job can stay in the processing queue, default to 1 hour

To place a `SendEmail` job on our queue, we might add this to our pre-existing application's queue:

```swift
let job = SendEmail(recipient: "john.appleseed@apple.com", subject: "Reswifq", body: "A simple reliable background processing for Swift.")
queue.enqueue(job)
```

If we prefer to schedule the job for a delayed execution we can instead enqueue it specifying a schedule time:

```swift
queue.enqueue(job, scheduleAt: Date(timeIntervalSinceNow: 3600.0)) // Send the email in an hour from now
```

Either way, a job will be created and placed on a queue.

Later, a worker will run something like this code to process the job:

```swift
let job = try queue.dequeue()
job.perform()
```

### Workers

Reswifq workers are swift processes that run forever and don't know about your app environment, in fact we suggest you to group your jobs into a module that can be shared between your app and workers.

Once a Reswifq worker starts, it'll try to run the `SendEmail.perform()` code snippet above and process jobs until it can't find any more, at which point it will sleep for a small period and repeatedly poll the queue for more jobs.

Workers can run on multiple machines. In fact they can be run anywhere with network access to the Redis server.

For an example on how to start a worker and for a list of configurable settings available, please follow the documentation on our [demo project](https://github.com/reswifq/demo).

### Failure

If a job raises an exception, it is logged and handed off to the `Reswifc.Monitor` module.
`Reswifc` will periodically check for expired / failed jobs and it will re-enqueue them until the maximum retry attempt limit is reached, once that happens the job will be completely removed from the processing queue.

# Recommended Setup

A typical project is composed by four targets:

- **Scheduler** - enqueues and schedules jobs, usually your webservice
- **Worker** - dequeues and processes pending jobs
- **Clock** - monitor expired jobs and schedules delayed jobs for execution
- **Jobs** - shared module containing the jobs definitions

This is the recommended and most common setup for [Reswifq](https://github.com/reswifq/reswifq). Although, it's not the only way and nothing should stop you from using a different structure if it makes sense for your project.

**Note:** For a working example refer to our [demo project](https://github.com/reswifq/demo).

## Scheduler

A simple jobs scheduler implementation that enqueues a `DemoJob` every second could look like this. In a real use case this would probably be your webservice or a dedicated cronjob.

```swift
let client = Redis()
client.connect(host: "localhost", port: 6379, callback: { _ in })

let queue = Reswifq(client: client)

while true {

    do {
        try queue.enqueue(DemoJob())
    } catch let error {
        print(error.localizedDescription)
    }

    sleep(1)
}
```

## Worker

An example of how to instantiate and run a worker process.

```swift
let client = RedisClientPool(maxElementCount: 10) { () -> RedisClient in

    let client = Redis()
    client.connect(host: "localhost", port: 6379, callback: { _ in })

    return client
}

let queue = Reswifq(client: client)
queue.jobMap[String(describing: DemoJob.self)] = DemoJob.self

let worker = Worker(queue: queue, maxConcurrentJobs: 4, averagePollingInterval: 0)

worker.run()
```

Apart from a standard `Worker` configuration, the `jobMap` is what requires some major attention.

`queue.jobMap[String(describing: DemoJob.self)] = DemoJob.self`

This is telling the queue, that whenever a job of type `DemoJob` is dequeued, it has to create an instance of type `DemoJob`.

**Notes:**

 - The job's type is simply a `String` and mapping it to a concrete type is **mandatory**.
 - The `jobMap`'s keys must correspond to the `Job`'s type attribute.
 - Note the use of a `RedisClientPool`, to ensure the same client is not used by different threads at the same time.
 - A `RedisClientPool` is not required when `maxConcurrentJobs` is `1`.

## Clock

An instance of `Reswifc`, configured with both a `Monitor` and a `Scheduler` process.

- **Monitor** - analyzes the processing queue for expired jobs
- **Scheduler** - analyzes the delayed queue for overdue jobs and schedules them for execution

```swift
let client = RedisClientPool(maxElementCount: 2) { () -> RedisClient in

    let client = Redis()
    client.connect(host: "localhost", port: 6379, callback: { _ in })

    return client
}

let queue = Reswifq(client: client)

let reswifc = Reswifc(processes: [
    Reswifc.Monitor(queue: queue, interval: 5, maxRetryAttempts: 3),
    Reswifc.Scheduler(queue: queue, interval: 1)
    ])

reswifc.run()
```

**Notes:**

- As each process runs in its own thread we are using a `RedisClientPool` of size `2` which is shared across both because we are passing to each process the same queue instance.
- Based on your project, you may need or not, any combination of the above processes running.

### Do I need a Monitor process?

Yes, if you need a reliable queue, otherwise expired jobs will simply fail without any retry attempt.

### Do I need a Scheduler process?

Yes, if you need to schedule jobs for delayed execution. For example: run a job on a specific date and time.

### Do I need a Clock process at all?

Yes, if you need either a `Monitor` or a `Scheduler`.

# Compatibility

This package has been tested on macOS and Ubuntu.

# License

Created by [Valerio Mazzeo](https://github.com/valeriomazzeo).

Copyright © 2017 [VMLabs Limited](https://www.vmlabs.it). All rights reserved.

This program is free software: you can redistribute it and/or modify
it under the terms of the GNU Lesser General Public License as published by
the Free Software Foundation, either version 3 of the License, or
(at your option) any later version.

This program is distributed in the hope that it will be useful,
but WITHOUT ANY WARRANTY; without even the implied warranty of
MERCHANTABILITY or FITNESS FOR A PARTICULAR PURPOSE.

See the [GNU Lesser General Public License](http://www.gnu.org/licenses) for more details.
