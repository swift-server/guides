# Log Levels 

This guide serves as guidelines for library authors with regards to what [SwiftLog](https://github.com/apple/swift-log) log levels are apropriate for use in libraries, and in what siuations to use what level.

## Guidelines for Libraries

SwiftLog defines the following 7 log levels via the [`Logger.Level` enum](https://apple.github.io/swift-log/docs/current/Logging/Structs/Logger/Level.html), ordered from least to most severe: 

* `trace`
* `debug`
* `info`
* `notice`
* `warning`
* `error`
* `critical`

Out of those, only levels _less severe than_ info (exclusive) are generally okay to be used by libraries.
In the following section we'll explore how to use them in practice.

### Recommended log levels

It is always fine for a library to log at `trace` and `debug` levels, and these two should be the primary levels any library is logging at.

`trace` is the finest log level, and end-users of a library will not usually use it unless debugging very specific issues. You should consider it as a way for library developers to "log everything we could possibly need to diagnose a hard to reproduce bug". It is expected to take a toll on the performance of a system, and developers can assume trace level logging will not be used in production deployments, unless enabled specifically to locate some specific issue.

This is in contrast with `debug` which some users _may_ choose to run with enabled on their production systems.

> Debug level logging should be not "too" noisy. Developers should assume some production deployments may need to (or want to) run with debug level logging enabled. 
>
> Debug level logging should not completely undermine the performance of a production system.

As such, `debug` logging should not be overly noisy. It should provide a high value understanding of what is going on in the library for end users without diving deep into its internals. This is what `trace` is intended for.

Use `warning` level sparingly. Whenever possible, try to rather emit errors to end users that are descriptive enough so they can inspect, log them and figure out the issue. Potentially they may then enable debug logging to find out more about the issue.

It is okey to log a `warning` "once", for example on system startup. This may include some one off "more secure configuration is available, try upgrading to it!" log statement upon a servers startup. You may also log warnings from background processes, which otherwise have no other means of informing the end user about some issue.

Logging on `error` level is similar to warnings: prefer to avoid doing so whenever possible. Instead offer errors or other mechanicms to users of your library such that they are informed about some error ocurring. For example, it is _not_ a good idea to log "connection failed" from an http client. Perhaps the end-user intended to make this request to a known offline server to _confirm_ it is offline? From their perspective this connection error is not a "real" error, it is just what they expected -- as such the http client should return or throw such error, but _not_ log it.

Do also note that in situations when you decide to log an error, be mindful of error rates. Will this error potentially be logged for every single operation while some network failure is happening? Some teams and companies have alerting systems set up based on the rate of errors logged in a system, and if it exceeds some threshold it may start calling and paging people in the middle of the night. When logging at error level, consider if the issue indeed is something that should be waking up people at night. You may also want to consider offering configuration in your library: "at what log level should this issue be reported?" This can come in handy in clustered systems which may log network failures themselfes, or depend on external systems detecting and reporting this.

Logging `critical` logs is allowed for libraries, however as the name implies - only in the most critical situations. Most often this implies that the library will *stop functioning* after such log has been issued. End users are thought to expect that a logged critical error is _very_ important, and they may have set up their systems to page people in the middle of the night to investigate the production system _right now_ when such log statements are detected. So please be careful about logging these kinds of errors.

#### Examples

`trace` level logging:

- could include various additional information about a request, such as various diagnostics about created datastructures, the state of caches or similar, which are created in order to serve a request.
- could include "begin operation" and "end operation" logs, 
  - however please do consider using swift-distributed-tracing to instrument such "begin/end" events as tracing can be more efficient than logging which may need to create string representations of the durations and log levels etc.

`debug` level logging:

- may include a single log statement for opening a connection, or accepting a request etc,
- it can include a _high level_ overview of what is going on in the library "started work, processing step X, finished work X, result code 200"

### Log levels to avoid

All these rules are only _general_ guidelines, and as such may have exceptions.Consider the following examples and rationale for why logging at high log levels by a library may not be desirable:

It is generally _not acceptable_ for a service client (e.g. an http client) to log an `error` when a request has failed. End-users may be using the client to probe if an endpoint is even responsive or not, and a failure to respond may be _expected_ behavior. Logging errors would only confuse and pollute their logs.
Instead, libraries should either `throw`, or return an `Error` value that users of the library will have enough knowladge about if they should log or ignore it.

It is even less acceptable for a library to log any successful operations. This leads to flodding server side systems, especially 

#### Examples (of things to avoid)

Avoid using `info` or any higher log level for:

- "normal operation" of the library, i.e. there is no need to log on info level "accepted a request" as this is the normal operation of a web service.

Avoid using `error` or `warning`:

- to report errors which the end-user of the library has the means of logging themselfes, e.g. if a database driver fails to fetch all rows of a query, it should not log an error or warning, but instead e.g. error the stream of values (or function, async function, or even the async sequence) that was representing the returned values.
  - since the end-user is consuming these values, and has a mean of reporting (or swallowing) this error, the library should not log anything on their behalf.
- never report as warnings which is merely an information, e.g. "weird header detected" may look like a good idea to log as a warning at first sight, however if the "weird header" is simply a misconfigured client (or just a "weird browser") you may be accidentally completely flooding an end-users logs with these "weird header" warnings (!)
  - only log warnings about actionable things which the end-user of your library can do something about. Using the the "weird header detected" log statement as an example: it would not be a good candidate to log as a warning because the server developer has no way to fix the users of their service to stop sending weird headers, so the server should not be logging this information as a warning.

Exceptions to the "avoid logging warnings" rule:

- "background processes" such as tasks scheduled on a periodic timer, may not have any other means of communicating a failure or warning to the end user of the library other than through logging.
  - consider offering an API that would collect errors at runtime, and then you can avoid logging errors manually. This can often take the form of a customizable "on error" hook that the library accepts when constructing the scheduled job. If the handler is not customized, we can log the errors, but if it was, it again is up to the end-user of the library to decide what to do with them.

### Suggested logging style

While libraries are free to use whichever logging message style they choose, here are some best practices to follow if you want users of your libraries *love* the logs your library produces.

And a minor yet important hint: avoid inserting newlines and other control characters into log statements (!). Many log aggregation systems assume that a single line in a logged output is specifically "one log statement" which can accidentally break if we log not sanitized, potentially multi-line, strings. This isn't a problem for _all_ log backends, e.g. some will automatically sanitize and form a JSON payload with `{message: "..."}` before emitting it to a backend service collecting the logs, but plain old stream (or file) loggers usually assume that one line equals one log statement - it also makes grepping through logs more reliable.

#### Structured Logging

Libraries may want to embrace the structured logging style. 

It is a very useful pattern which makes consuming logs in automated systems, and through grepping and other means much easier and future proof.

Consider the folowing not structured log statement:

```swift
// NOT structured logging style
log.info("Accepted connection \(connection.id) from \(connection.peer), total: \(connections.count)")
```

It contains 4 peieces of information:

- we accepted a connection,
- this is it's string representation,
- it is from this peer,
- we currently have `connections.count` active connections.

While this log statement contains all useful information that we meant to relay to end users, it is hard to visually and mechanically parse the detailed information it contains. For example, if we know connections start failing around the time when we reach a total of 100 concurrent connections, it is not trivial to find the specific log statement at which we hit this number. We would have to `grep 'total: 100'` for example, however perhaps there are many other `"total: "` strings present in all of our log systems.

Instead, we can express the same information using the structured logging pattern, as follows:

```swift
log.info("Accepted connection", metadata: [
  "connection.id": "\(connnection.id)",
  "connection.peer": "\(connection.peer)",
  "connections.total": "\(connections.count)"
])

// e.g. output:
// <date> info [connection.id:?,connection.peer:?, connections.total:?] Accepted connection
```

which can be formatted, depending on the logging backend, slightly differently on various systems, but even in the simple string representation of such log, we'd be able to grep for `connections.total: 100` rather than having to guess the right string to grep for.

Also, since the message now does not contain all that much "human readable wording", it is less prone to randomly change from "Accepted" to "We have accepted" or vice versa which could break alerting systems which are set up to parse and alert on specific log messages.

Needless to say, structured logs are very useful in combination with [swift-distributed-tracing](https://github.com/apple/swift-distributed-tracing)'s `LoggingContext`, which automatically populates the metadata with any present trace information. Thanks for this all logs made in response to some specific request will automatically carry the same TraceID.

### Exceptions to the rules

These are only general guidelines, and there always will be exceptions to these rules and other situations where these suggestions will be broken, for good reason. Please use your best judgement, and always consider the end-user of a system, and how they'll be interacting with yout library and decide case-by-case depending on the library and situation at hand how to handle each situation.

Here are a few examples of situations when logging a message on a relatively high level might still be tolerable for a library.

It's permittable for a library to log at `critical` level right before a _hard crash_, as a last resort of informing the log collection systems or end-user about additional information detailing the reason for the crash. This should be _in addition to_ the message from a `fatalError` and can lead to an improved diagnosis/debugging experience for end users.

Sometimes libraries may be able to detect a harmful misconfiguration of the library. For example, selecting deprecated protocol versions. In such situations it may be useful to inform users in production by issuing a `warning`. However you should ensure that the warning is not logged repeatedly! For example, it is not acceptable for an HTTP client to log a warning on every single http request using some misconfiguration of the client. It _may_ be acceptable however for the client to log such warning e.g. _once_ at configuration time, if the library has a good way to do this.

Some libraries may implement a "log this warning only once" or "log this warning only at startup", or "log this error only once an hour" or similar tricks to keep the noise level low but still informative enough to not be missed. This is usually a pattern reserved for stateful long running libraries, rather than clients of databases etc however.