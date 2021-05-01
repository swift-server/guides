
## Deployment to Servers or Public Cloud

Once an application is built for production, it needs to be packaged for deployment. There are several strategies for packaging Swift applications for deployment, see the [Packaging Guide](packaging.md) for more information.

A separate guide for [Ubuntu on DigitalOcean](digital-ocean.md) is also available.

### Deploying a Debuggable Configuration (Production on Linux)

- If you have `--privileged`/`--security-opt seccomp=unconfined` containers or are running in VMs or even bare metal, you can run your binary with

        lldb --batch -o "break set -n main --auto-continue 1 -C \"process handle SIGPIPE -s 0\"" -o run -k "image list" -k "register read" -k "bt all" -k "exit 134" ./my-program

    instead of `./my-program` to get something something akin to a 'crash report' on crash.

- If you don't have `--privileged` (or `--security-opt seccomp=unconfined`) containers (meaning you won't be able to use `lldb`) or you don't want to use lldb, consider using a library like [`swift-backtrace`](https://github.com/swift-server/swift-backtrace) to get stack traces on crash.

_TODO: add guides for popular public clouds like AWS, GCP, Azure, Heroku, etc._
