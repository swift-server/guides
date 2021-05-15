# Build system

The recommended way of managing the builds for server applications is to use the [Swift Package Manager](https://swift.org/package-manager/), it provides a cross-platform foundation for building Swift code and works nicely for having one code base that can be edited as well as run on many Swift platforms

## Building

Running `swift build` from the terminal will trigger the build (or triggering the build action in Xcode).

Swift is architecture specific, so running the build command on macOS will create a macOS binary. Building on macOS is useful for development and for taking advantage of the great tooling that comes with Xcode. However, most server applications are designed to run on Linux.

To build on Linux and create a Linux binary, use Docker. For example:

`$ docker run -v "$PWD:/code" -w /code swift:latest swift build`

Note, if you want to run the Swift compiler for Intel CPUs on an Apple Silicon (M1) Mac, please add `--platform linux/amd64 -e QEMU_CPU=max` to the commandline. For example:

`$ docker run -v "$PWD:/code" -w /code --platform linux/amd64 -e QEMU_CPU=max swift:latest swift build`

The above commands will run the build using the latest Swift 5.3 Docker image, utilizing bind mounts to the sources on your Mac. Apple publishes Docker images to Docker Hub.

By default, SwiftPM will build a debug version of the application. Note that debug versions are not suitable for running in production as they are significantly slower. To build a release version of your app, run `swift build -c release`.

Binary artifacts that could be deployed are be found under .build/x86_64-unknown-linux, or .build/x86_64-apple-macosx for macOS binaries. SwiftPM can show you the full binary path using `swift build --show-bin-path -c release`.

### Building for production

- Build production code in release mode by compiling with `swift build -c release`. Running code compiled in debug mode will hurt performance signficantly. 

- For best performance in Swift 5.2 or later, pass `-Xswiftc -cross-module-optimization` (this won't work in Swift versions before 5.2)

- Integrate [`swift-backtrace`](https://github.com/swift-server/swift-backtrace) into your application to make sure backtraces are printed on crash. Backtraces do not work out-of-the-box on Linux, and this library helps to fill the gap. Eventually this will become a language feature and not require a discrete library.
