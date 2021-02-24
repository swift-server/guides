# Log Levels 

This guide serves as guideline for library authors with regards to what [SwiftLog](https://github.com/apple/swift-log) log levels are apropriate for use in libraries, and in what siuations to use what level.

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

`trace` is the finest log level, and end-users of a library will not usually use it unless debugging very specific issues. You should consider it as a way for library developers to "log everything we could possibly need to diagnose a hard to reproduce bug".

**// TODO: should we suggest that a system should be able to survive with debug logging turned on in production (ktoso: I'd suggest that, yes).**

This is in contrast with `debug` which some users _may_ choose to run with enabled on their production systems. As such, debug logging should not be overly noisy. It should provide a high value understanding of what is going on in the library for end users without diving deep into its internals. This is what `trace` is intended for.

### Log levels to avoid

All these rules are only _general_ guidelines, and as such may have exceptions.Consider the following examples and rationale for why logging at high log levels by a library may not be desirable:

It is generally _not acceptable_ for a service client (e.g. an http client) to log an `error` when a request has failed. End-users may be using the client to probe if an endpoint is even responsive or not, and a failure to respond may be _expected_ behavior. Logging errors would only confuse and pollute their logs.
Instead, libraries should either `throw`, or return an `Error` value that users of the library will have enough knowladge about if they should log or ignore it.

It is even less acceptable for a library to log any successful operations. This leads to flodding server side systems, especially 

### Suggested logging style

Libraries may want to embrace the structured logging style.

**TODO: fill this in more.**

### Exceptions to the rules

As this is only a general guideline, there are exceptions to this rule which must be decided on case-by-case depending on the library and situation at hand.

Here are a few examples of situations when logging a message on a relatively high level might still be tolerable for a library.

It's permittable for a library to log at `critical` level right before a _hard crash_, as a last resort of informing the log collection systems or end-user about additional information detailing the reason for the crash. This should be _in addition to_ the message from a `fatalError` and can lead to an improved diagnosis/debugging experience for end users.

Sometimes libraries may be able to detect a harmful misconfiguration of the library. For example, selecting deprecated protocol versions. In such situations it may be useful to inform users in production by issuing a `warning`. However you should ensure that the warning is not logged repeatedly! For example, it is not acceptable for an HTTP client to log a warning on every single http request using some misconfiguration of the client. It _may_ be acceptable however for the client to log such warning e.g. _once_ at configuration time, if the library has a good way to do this.
Some libraries may implement a "log this warning only once" or "log this warning only at startup" to achieve similar semantics.
