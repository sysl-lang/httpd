# httpd

An HTTP/1.1 server written in [sysl](https://github.com/sysl-lang/sysl): sockets, timers and tasks
from [kairos](https://github.com/sysl-lang/kairos)'s libuv driver, and parsing from
[llhttp](https://github.com/sysl-lang/llhttp). One task accepts; each connection is one task that
reads, parses, answers and decides when the connection ends.

It does keep-alive and pipelining. llhttp is fed whatever each read returned, so a request split
across reads is assembled across them, and several pipelined requests in one read are answered in
order, in one write.

## Running it

```
sysl build .
./httpd 8080 static
```

`httpd [port] [root]`: the port defaults to 8080 and the root to `static`. SIGINT or SIGTERM shuts it
down gracefully.

## Routes

| route | answer |
|---|---|
| `GET /` | `Hello, world!` |
| `GET /static/<file>` | the file from the root directory, read on libuv's thread pool; 404 if it is missing, or if the name holds a `/` or starts with `.` |
| `POST /echo` | the request body |
| `GET /sleep?ms=N` | `slept`, after an `await lp.sleep(N ms)`; N is 0 to 60000 |

A wrong method gets 405 and any other path gets 404.

## What it refuses, and how it ends

| case | answer |
|---|---|
| a request not complete within the request timeout (10 s), counted from the end of the previous answer | 408, then close |
| a body over the limit (1 MiB), declared or chunked | 413, then close |
| a head over 8 KiB | 431, then close |
| a malformed request | 400, with llhttp's reason as the body, after the good requests before it |
| a connection over the cap (10,000) | 503, then close |
| SIGINT / SIGTERM | the listener and every idle connection close at once; a connection that is answering finishes its answer, says `Connection: close`, and then closes. The loop ends when the last one has |

A client that disappears mid-answer only ends its own task: the failed write returns, and the
task's `defer` takes the connection out of the server's table and closes it.

## Testing

```
sysl test .
```

Twenty tests. Each one starts the server in process on a port the kernel chooses, and drives it with
clients that are kairos tasks on the same loop. The clients write raw bytes and compare the exact
bytes that come back. The suite covers every route, chunked bodies, keep-alive against
`Connection: close`, HTTP/1.0, pipelined requests split at every byte boundary, a request sent a byte
at a time, and every case in the table above. One more test is marked ignore. It needs a sysl fix,
because the task `lp.timeout` gives up on is released too late.

## Against C

The baseline is a minimal C server built on the same libuv and the same llhttp, compiled with
`clang -O2`. It serves `GET /` and `/sleep` only. The sysl server is built at `optimization = "2"`.
The load generator is `oha` (`-z 8s`), with keep-alive and everything on one machine: an Apple M5
Pro (18 cores, 64 GB) on macOS 26.6.2, measured 2026-10-13 with sysl 0.1.0-alpha.11, kairos 0.1.1,
llhttp 0.2.0 and libuv 1.x from Homebrew.

| route | connections | sysl req/s | C req/s | sysl / C | sysl p50 | C p50 | sysl p99 | C p99 |
|---|---:|---:|---:|---:|---:|---:|---:|---:|
| `GET /` | 1 | 31,243 | 31,619 | 0.99 | 0.029 ms | 0.029 ms | 0.078 ms | 0.073 ms |
| `GET /` | 64 | 154,085 | 180,105 | 0.86 | 0.40 ms | 0.35 ms | 0.62 ms | 0.57 ms |
| `GET /` | 512 | 149,976 | 199,305 | 0.75 | 3.46 ms | 2.51 ms | 4.37 ms | 4.87 ms |
| `/sleep?ms=10` | 1 | 90 | 90 | 1.00 | 11.12 ms | 11.12 ms | 11.22 ms | 11.21 ms |
| `/sleep?ms=10` | 64 | 5,501 | 5,741 | 0.96 | 11.64 ms | 11.41 ms | 11.96 ms | 11.89 ms |
| `/sleep?ms=10` | 512 | 35,735 | 47,516 | 0.75 | 14.32 ms | 10.73 ms | 16.21 ms | 12.03 ms |

At 512 connections a profile of `GET /` is mostly `write` and `read` system calls. What sysl adds
on top is mostly `malloc` and `free`: a fresh slice for every read (zeroed and then copied), and the
strings that build each answer. Over two 60-second runs of `GET /` at 512 connections (about 17
million requests), resident memory stayed flat during each run: 61.9 MB, then 69.5 MB. Most of that
is libuv's 64 KiB read buffer for each open connection.

**Connections leak, but requests do not.** Every connection leaves 17 counted allocations behind
(`SYSL_COUNT_ALLOCS=1`, 100 and 1,000 connections). Fourteen of them are llhttp's `Requests`: the
reader holds its parser, and the parser's callback captures the reader, so the two form a cycle that
nothing ever frees.

## License

ISC
