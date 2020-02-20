# Swift on Server Deployment Guide

## Introduction

This guide is designed to help teams and individuals running Swift Server applications on Linux. It focuses on how to compile, test, deploy and debug such application and provides tips in those areas.

The guide is a community effort, and all are invited to share their tips and know-how.

## Building

Triggering the build action in Xcode or running `swift build` from the terminal will trigger the build.

Swift is architecture specific, so running the build command on your Mac, will create a macOS binary. Building on macOS is useful for development and taking advantage of the great tooling that comes with Xcode, however, most server application are designed to run on Linux.

To build on Linux and create a Linux binary, use Docker. For example:

`$ docker run -v $PWD:/code -w /code swift:5.1 swift build`

The above command will run the build using the latest Swift 5.1 Docker image, utilizing bind mounts to the sources on your Mac. Apple publishes Docker images to Docker Hub.

By default, SwiftPM will build a debug version of the application. Note that debug versions are not suitable for running in production as they are significantly slower. To build a release version of your app, with debug symbols, run `swift build -c release -Xswiftc -g`

Binary artifacts, that could be deployed, are be found under .build/x86_64-unknown-linux, or .build/x86_64-apple-macosx for macOS binaries.

### Building for production

- Build production code in release mode by compiling with `swift build -c release`. Running code compiled in debug mode will hurt performance signficantly. 

- For best performance in Swift 5.2 or later, pass `-Xswiftc -cross-module-optimization` (this won't work in Swift versions before 5.2)

- For versions prior to Swift 5.2, always build with `-Xswiftc -g` to get debugging symbols. Otherwise your stacktraces will lack most symbol names.

- Integrate [`swift-backtrace`](https://github.com/swift-server/swift-backtrace) into your application to make sure backtraces are printed on crash. Backtraces do not work out-of-the-box on Linux, and this library helps to fill the gap. Eventually this will become a language feature and not require a discrete library.

## Testing 

Triggering the test action in Xcode or running swift test from the terminal will trigger the unit tests.

SwiftPM is integrated with XCTests, Apple’s unit test framework. Test results are available in Xcode, or printed out to the terminal.

Like building on Linux, testing on linux requires the use of Docker. For example:

`$ docker run -v $PWD:/code -w /code swift:5.1 swift test`

The above command will run the tests using the latest Swift 5.1 Docker image, utilizing bind mounts to the sources on your Mac.

Swift supports architecture-specific code. By default, Foundation imports architecture-specific libraries like Darwin or Glibc. While developing on your Mac, you may end up using APIs that are not available on Linux. Since you are most likely to deploy a cloud service on Linux, it is critical to test on Linux.

Another important detail about testing for Linux is the `Tests/LinuxMain.swift` file. For versions prior to Swift 5.1, This file provides SwiftPM an index of all the tests it needs to run on Linux. 

- For version Swift 5.1 and later, this file is no longer required and testing with `--enable-test-discovery` seamlessly discovers the tests. This will be the default behvior in future releases of Swift. Until this is the default, it can be useful 
to keep `Tests/LinuxMain.swift` around but with with the following code to remind you to use the flag:

   `fatalError("Please use `swift test --enable-test-discovery` to run the tests instead")`

- For versions prior to Swift 5.1, it is critical to keep this file up-to-date as you add more unit tests. To regenerate this file, run `swift test --generate-linuxmain` after adding tests. It is also a good idea to include this command as part of the continuous integration setup. For version Swift 5.1 or later, this 

### Testing for production

- For versions Swift 5.1 and later, always test with `--enable-test-discovery` to avoid forgeting tests on Linux.

- Generally, whilst testing, you may want to build using `swift build --sanitize=thread`. The binary will run slower and is not suitable for production but you'll see many many threading issues before you deploy your software. Often threading issues are really hard to debug and reproduce and also cause random problems, TSan helps catching them early.

- Make use of the sanitizers, so before running code in production, do the following:
    * run your test suite with TSan (thread sanitizer): `swift test --sanitize=thread`
    * run your test suite with ASan (address sanitizer): `swift test --sanitize=address` and `swift test --sanitize=address -c release -Xswiftc -enable-testing`

## Debugging / Development

- `perf` output, strack traces, etc often appear "mangled" (lots of stuff looking like `3NIO14CircularBufferV5IndexV`), you can use `swift demangle` to turn that into human readable symbol names. In you have a file, use `cat file-containing-mangled-stuff | swift demangle` and it will look much more readable.

## Deploying a Debuggable Configuration (Production on Linux)

- If you have `--privileged`/`--security-opt seccomp=unconfined` containers or are running in VMs or even bare metal, you can run your binary with

        lldb --batch -o run -k "image list" -k "register read" -k "bt all" -k "exit 134" ./my-program

    instead of `./my-program` to get something something akin to a 'crash report' on crash.

- If you don't have `--privileged` (or `--security-opt seccomp=unconfined`) containers (meaning you won't be able to use `lldb`) or you don't want to use lldb, consider using a library like [`swift-backtrace`](https://github.com/swift-server/swift-backtrace) to get stack traces on crash.

## Deployment to Public Cloud

TODO: add guides for popular public clouds like AWS, GCP, Azure, Heroku, et
