---
title: "c10k of tokio-rs network"
date: 2019-04-15T21:56:00+08:00
tags:
- C10k
- Rust
categories:
- Rust
draft: true
---

[server](https://github.com/laohanlinux/c10k-rs)

[benchmark-client](https://github.com/Xudong-Huang/may)


## Benchmark

### v.0.1

``` shell
-> % target/release/examples/echo_client -t 2 -c 100 -l 100 -a 127.0.0.1:3941
==================Benchmarking: 127.0.0.1:3941==================
100 clients, running 100 bytes, 10 sec.

Speed: 100230 request/sec,  100230 response/sec, 9788 kb/sec
Requests: 1002303
Responses: 1002302
```

### v.0.2

mutil io reator

``` shell
==================Benchmarking: 127.0.0.1:3941==================
100 clients, running 100 bytes, 10 sec.

Speed: 115648 request/sec,  115648 response/sec, 11293 kb/sec
Requests: 1156483
Responses: 1156482
```

continue to challenge `500k`
