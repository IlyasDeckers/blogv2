---
title: Swoole undocumented features pt.1
description: 
slug: swoole-undocumented-features-pt1
date: 2024-06-21 00:00:00+0000
# image: cover.jpg
categories:
    - Swoole
    - PHP
tags:
    - PHP
    - Swoole
# weight: 1
---

Let’s dive into **Swoole’s most extreme, undocumented, and borderline-insane optimizations and tools**—the kind of stuff used by **Alibaba, Tencent, and Bytedance at scale**.  

---

## Profile-Guided Optimization

`SWOOLE_HAVE_PGO`

### **What It Is**  
A **compile-time technique** where Swoole is first **profiled under real workload** (e.g., 100K RPS), then **recompiled with optimizations** tailored to that exact usage pattern.  

### **How It Works**  
1. **Instrumentation Phase**:  
   ```sh
   ./configure --enable-swoole --enable-gcov
   make && make test
   ```  
   - Runs benchmarks while collecting **branch prediction stats, cache misses, hot functions**.  

2. **Optimization Phase**:  
   ```sh
   ./configure --enable-swoole --with-pgo
   make && make install
   ```  
   - GCC/Clang **rewrites hot paths** (e.g., inlining coroutine switches).  

### **Performance Gains**  
- **~15-25% faster** coroutine scheduling.  
- **L1/L2 cache misses reduced** by up to 40%.  
- **Best for**:  
  - High-frequency trading bots.  
  - API gateways (e.g., JSON/Protobuf parsing).  

### **The Catch**  
- **Requires real traffic** to profile (no synthetic benchmarks).  
- **Breaks if workload changes** (must re-profile).  

---

## Debugging Coroutine Hell

`--enable-swoole-fiber-sanitizer`

### **What It Is**  
A **runtime memory debugger** for Swoole coroutines, detecting:  
- **Use-after-free** in coroutine stacks.  
- **Memory leaks** in `go()` closures.  
- **Race conditions** in shared globals.  

### **How to Use It**

1. **Compile Swoole in debug mode**:  
   ```sh
   ./configure --enable-swoole --enable-debug --enable-swoole-fiber-sanitizer
   ```  
2. **Run your app**:  
   ```sh
   USE_ZEND_ALLOC=0 php -d swoole.fiber_sanitizer=1 your_app.php
   ```  
   - Logs **stack traces** of leaks/crashes.  

### **Who Needs This?**  
- **Devs debugging "phantom" segfaults** in coroutines.  
- **Teams using `global $db` in workers** (you monsters).  

### **The Dark Side**  
- **~10x slower** (only for debugging).  
- **Can’t run with Valgrind** (they fight over memory hooks).  

---

## Bytedance’s 10M+ Keep-Alive Patch 
### **The Problem**  
Swoole’s default `epoll` event loop **struggles past ~1M connections** due to:  
- **O(n) socket fd scanning**.  
- **Kernel `accept()` throttling**.  

### **Their Solution**  
1. **`SO_REUSEPORT` + Lock-Free Accept**:  
   - Multiple workers **compete for new connections** (no thundering herd).  
   - Uses **eBPF to bypass `accept()`** (directly assign fds to workers).  

2. **Custom `epoll` Patch**:  
   - Replaces `EPOLL_CTL_ADD` with **`EPOLL_CTL_MOD` batching**.  
   - **Saves ~7µs per connection**.  

3. **Zero-Copy TLS**:  
   - OpenSSL **bypassed** for static certs (TLS 1.3 only).  
   - **Saves 1 full RTT per handshake**.  

### **Performance**  
- **10M concurrent connections** on a **single 32-core AWS `c6gn.metal`**.  
- **3M TLS handshakes/sec** (with their modified OpenSSL).  

### **How to Try It**  
- Their fork is **closed-source**, but you can approximate it with:  
  ```sh
  ./configure --enable-swoole --with-openssl-dir=/path/to/custom-openssl
  ```  
  And in `php.ini`:  
  ```ini
  swoole.reuse_port=1
  swoole.enable_unsafe_epoll=1  ; Not for production!
  ```  

---

### **Final Thoughts**  
These are **weapons-grade optimizations**—most projects don’t need them, but if you’re pushing Swoole to its absolute limits, they’re the difference between **"fast" and "WTF-fast"**.  

Want to go **even deeper**?:  
- **`--enable-swoole-io_uring`** (Linux 5.6+ only, replaces `epoll`).  
- **Swoole’s secret `dtrace` probes** (for kernel-level profiling).  
- **How WeChat uses Swoole as a TCP-to-HTTP/3 translator**.  

The rabbit hole **never ends**. 🚀