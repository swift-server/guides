# Swift on Server Development Guide

## Introduction

This guide is designed to help teams and individuals running Swift Server applications on Linux and to provide orientation for those who want to start with such development. 
It focuses on how to build, test, deploy and debug such application and provides tips in those areas.

## Contents

- [Setup and code editing](docs/setup-and-ide-alternatives.md)
- [Building](docs/building.md)
- [Testing](docs/testing.md)
- [Debugging Memory leaks](docs/memory-leaks-and-usage.md)
- [Performance troubleshooting and analysis](docs/performance.md)
- [Debugging multithreading issues and memory checks](docs/llvm-sanitizers.md)
- [Deployment](docs/deployment.md)

### Server-side library development

Server-side libraries should, to the best of their ability, play well with established patterns in the SSWG library ecosystem.
They should also utilize the core observability libraries, such as: logging, metrics and distributed tracing, where applicable.

The below guidelines are aimed to help library developers with some of the typical questions and challenges they might face when designing server-side focused libraries with Swift:

- [SwiftLog: Log level guidelines](docs/libs/log-levels.md)

_The guide is a community effort, and all are invited to share their tips and know-how. Please provide a PR if you have ideas for improving this guide!_
