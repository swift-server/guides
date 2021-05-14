# Testing 

Running `swift test` from the terminal will trigger the unit tests (or triggering the test action in Xcode).

SwiftPM is integrated with [XCTest, Apple’s unit test framework](https://developer.apple.com/documentation/xctest). Test results will be displayed in Xcode or printed out to the terminal.

Like building on Linux, testing on Linux requires the use of Docker. For example:

`$ docker run -v "$PWD:/code" -w /code swift:latest swift test`

The above command will run the tests using the latest Swift Docker image, utilizing bind mounts to the sources on your file system.

Swift supports architecture-specific code. By default, Foundation imports architecture-specific libraries like Darwin or Glibc. While developing on macOS, you may end up using APIs that are not available on Linux. Since you are most likely to deploy a cloud service on Linux, it is critical to test on Linux.

An historically important detail about testing for Linux is the `Tests/LinuxMain.swift` file. 

- For Swift version 5.4 or later, this file is not longer require and you can ignore the following section.

- For Swift versions betreen 5.1 and 5.4 this file is required, this file is no longer required, and testing with `--enable-test-discovery` seamlessly discovers the tests. Until this is the default, it can be useful  to keep `Tests/LinuxMain.swift` around but with with the following code to remind you to use the flag:

   `fatalError("Please use `swift test --enable-test-discovery` to run the tests instead")`

- For versions prior to Swift 5.1, This file provides SwiftPM an index of all the tests it needs to run on Linux and it is critical to keep this file up-to-date as you add more unit tests. To regenerate this file, run `swift test --generate-linuxmain` after adding tests. It is also a good idea to include this command as part of your continuous integration setup.

### Testing for production

- For Swift versions between Swift 5.1 and 5.4, always test with `--enable-test-discovery` to avoid forgetting tests on Linux.

- Make use of the sanitizers. Before running code in production, do the following:
    * Run your test suite with TSan (thread sanitizer): `swift test --sanitize=thread`
    * Run your test suite with ASan (address sanitizer): `swift test --sanitize=address` and `swift test --sanitize=address -c release -Xswiftc -enable-testing`

- Generally, whilst testing, you may want to build using `swift build --sanitize=thread`. The binary will run slower and is not suitable for production, but you'll see many many threading issues before you deploy your software. Often threading issues are really hard to debug and reproduce and also cause random problems. TSan helps catch them early.
