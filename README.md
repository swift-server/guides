# Swift on Server Deployment Guide


## General Advice

- Run production code in release mode by compiling with `swift build -c release`.
- Until Swift 5.2 lands, always pass a `-Xswiftc -g` to get debugging symbols, otherwise your stacktraces will lack most symbol names.

## Deploying a Debuggable Configuration (Production on Linux)

- If you have `--privileged`/`--security-opt seccomp=unconfined` containers or are running in VMs or even bare metal, you can run your binary with

        lldb --batch -o run -k "image list" -k "register read" -k "bt all" -k "exit 134" ./my-program

    instead of `./my-program` to get something something akin to a 'crash report' on crash.
- If you don't have `--privileged` (or `--security-opt seccomp=unconfined`) containers (meaning you won't be able to use `lldb`) or you don't want to use lldb, consider using a library like [`swift-backtrace`](https://github.com/swift-server/swift-backtrace) to get stack traces on crash.
- For best performance in Swift 5.2 or later, pass `-Xswiftc -cross-module-optimization` (this won't work in Swift versions below 5.2)

## Debugging / Development

- `perf` output, strack traces, etc often appear "mangled" (lots of stuff looking like `3NIO14CircularBufferV5IndexV`), you can use `swift demangle` to turn that into human readable symbol names. In you have a file, use `cat file-containing-mangled-stuff | swift demangle` and it will look much more readable.
- Make use of the sanitizers, so before running code in production, do the following:

    * run your test suite with TSan (thread sanitizer): `swift test --sanitize=thread`
    * run your test suite with ASan (address sanitizer): `swift test --sanitize=address` and `swift test --sanitize=address -c release -Xswiftc -enable-testing`

- Generally, whilst testing, you may want to build using `swift build --sanitize=thread`. The binary will run slower and is not suitable for production but you'll see many many threading issues before you deploy your software. Often threading issues are really hard to debug and reproduce and also cause random problems, TSan helps catching them early.
