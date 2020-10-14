# liburing4cpp

Modern C++ binding for [liburing](https://github.com/axboe/liburing) that uses C++20 Coroutines ( but still compiles for `clang` at C++17 mode with `-fcoroutines-ts` )

Originally named liburing-http-demo ( this project was originally started for demo )

## Requirements

Requires the latest kernel ( currently 5.8 ). Since [io_uring](https://git.kernel.dk/cgit/liburing/) is in active development, we will drop old kernel support when every new linux kernel version is released ( before the next LTS version is released, maybe ).

Tested: `Ubuntu 5.9.0-050900rc6daily20200923-generic #202009222208 SMP Wed Sep 23 02:24:13 UTC 2020 x86_64 x86_64 x86_64 GNU/Linux` with `clang version 10.0.0-4ubuntu1`

## First glance

```cpp
#include "io_service.hpp"

int main() {
    // You first need an io_service instance
    io_service service;

    // In order to `co_await`, you must be in a coroutine.
    // We use IIFE here for simplification
    auto work = [&] () -> task<> {
        // Use Linux syscalls just as what you did before (except a little changes)
        const auto str = "Hello world\n"sv;
        co_await service.write(STDOUT_FILENO, str.data(), str.size(), 0);
    }();

    // At last, you need a loop to dispatch finished IO events
    // It's usually called Event Loop (https://en.wikipedia.org/wiki/Event_loop)
    service.run(work);
}
```

## Benchmarks

* Ubuntu 20.04.1 LTS
* Linux Ubuntu 5.9.0-050900rc6daily20200923-generic #202009222208 SMP Wed Sep 23 02:24:13 UTC 2020 x86_64 x86_64 x86_64 GNU/Linux
* Intel(R) Xeon(R) CPU E5-2620 v4 @ 2.10GHz
* Compiler: clang version 10.0.0-4ubuntu1

### demo/bench

```
service.yield:        6527085832
plain IORING_OP_NOP:  5884262348
this_thread::yield:   4486533904
pause:                  41502717
```

### demo/echo_server

* demo/echo_server 12345 ( C++, uses coroutines )
* [./io_uring_echo_server 12345](https://github.com/CarterLi/io_uring-echo-server) ( C, raw )

with `rust_echo_bench`: https://github.com/haraldh/rust_echo_bench
unit: request/sec

Also see [benchmarks for different opcodes](https://github.com/CarterLi/io_uring-echo-server#benchmarks)

#### command: `cargo run --release`

LANG | USE_LINK | USE_SPLICE | USE_POLL |         operations |     1st |     2nd |     3rd |     mid |    rate
:-:  | :-:      | :-:        | :-:      |                 -: |      -: |      -: |      -: |      -: |      -:
C    | -        | -          | 0        |          RECV-SEND |  114461 |  116797 |  112112 |  114461 | 100.00%
C    | -        | -          | 1        |     POLL-RECV-SEND |  109037 |  114893 |  117629 |  114893 | 100.38%
C++  | 0        | 0          | 0        |          RECV-SEND |  117519 |  121139 |  120239 |  120239 | 105.05%
C++  | 0        | 1          | 0        |      SPLICE-SPLICE |   90577 |   91912 |   92301 |   91912 |  80.30%
C++  | 1        | 1          | 0        |      SPLICE-SPLICE |   93440 |   92619 |   94201 |   93440 |  81.63%
C++  | 0        | 0          | 1        |     POLL-RECV-SEND |  107454 |  111525 |  111210 |  111210 |  97.16%
C++  | 0        | 1          | 1        | POLL-SPLICE-SPLICE |   89469 |   90663 |   89315 |   89469 |  78.17%
C++  | 1        | 1          | 1        | POLL-SPLICE-SPLICE |   87628 |   89099 |   88708 |   89099 |  77.84%

## Project Structure

### task.hpp

An awaitable class for C++2a coroutine functions. Originally modified from [gor_task.h](https://github.com/Quuxplusone/coro#taskh-gor_taskh)

NOTE: `task` is not lazily executed, which is easy to use of course, but also can be easily misused. The simplest code to crash your memory is:

```c++
{
    char c;
    service.read(STDIN_FILENO, &c, sizeof (c), 0);
}
```

The task instance returned by `service.read` is destructed, but the kernel task itself is **NOT** canceled. The memory of variable `c` will be written sometime. In this case, out-of-scope stack memory access will happen.

### promise.hpp

An awaitable class. It's different from task that it can't used for return type, but can be created directly without calling an async function.

Its design is highly inspired by [Promise of JavaScript](https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Global_Objects/Promise)

### when.hpp

Provide helper functions that working with an array of tasks

### io_service.hpp

Main [liburing](https://github.com/axboe/liburing) binding. Also provides some helper functions for working with posix interfaces easier.

### demo

Some examples

#### file_server.cpp

A simple http file server that returns file's content requested by clients

#### link_cp.cpp

A cp command inspired by original [liburing link-cp demo](https://github.com/axboe/liburing/blob/master/examples/link-cp.c)

#### http_client.cpp

A simple http client that sends `GET` http request

#### threading.cpp

A simple `async_invoke` implementation

#### test.cpp

Various simple tests

#### bench.cpp

Benchmarks

#### echo_server.cpp

Echo server, features IOSQE_IO_LINK and IOSQE_FIXED_FILE

See also https://github.com/frevib/io_uring-echo-server#benchmarks for benchmarking

## Build

This library is header only. It provides some demos for testing

1. Build [`liburing`](https://github.com/axboe/liburing) and install, the latest version required
1. `sudo apt install clang libc++-dev libc++abi-dev`. Make sure clang version >= 9
1. `git clone --recurse-submodules https://github.com/CarterLi/liburing4cpp.git`
1. `cd liburing4cpp/demo && make`

Note: When benchmarking, you may want to build it with optimization: `make MODE=RELEASE`.

## License

MIT
