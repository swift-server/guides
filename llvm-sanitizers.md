# LLVM TSAN / ASAN

For multithreaded and low-level unsafe interfacing server code, the ability to use LLVM:s [ThreadSanitizer](https://clang.llvm.org/docs/ThreadSanitizer.html) and  
[AddressSanitizer](https://clang.llvm.org/docs/AddressSanitizer.html) can help troubleshoot invalid thread usage and invalid usage/access of memory.

There is a [blog post](https://swift.org/blog/tsan-support-on-linux/) outlining the usage of TSAN.

The short story is to use the swiftc command line options `-sanitize=address` and `-santize=thread` to each respective tool.
