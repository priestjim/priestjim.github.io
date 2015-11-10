---
layout: post
title: Optimizing your Linux server for memory-based NoSQL databases (Part 1)
excerpt: "Optimize your Linux server's kernel for memory-based NoSQL databases."
tags: [kernel, linux, memcache, memory, nosql, redis]
modified: 2012-07-04
date: 2012-07-04
comments: true
---

**Hey!** This article was originally written for BugSense’s blog and was published on 2012-07-03. You can find the original post [here](http://blog.bugsense.com/post/26442766999/optimizing-your-linux-server-for-memory-based-nosql).
{: .notice}

So you have been listening to the hype of NoSQL databases for some time now and how they can make you web applications run much faster and be more adaptive and welcoming to horizontal scaling and you’d like to try it too and see how it plays out for you. What you should know though, is that apart from setting up and configuring your selected flavor of NoSQL, be it something less complex, like **Memcache** or **Redis**, to more enterprise schemes like **Cassandra** or **HBase**, you will need eventually to optimize the server(s) hosting it to make the most out of your investment.

**Word of the wise**: Please read [this](http://static.usenix.org/publications/login/2011-10/openpdfs/Burd.pdf) before even thinking about dumping your RDMBS in favor of a NoSQL backend. A NoSQL database is not a replacement for traditional relational databases, and it never will be.

Optimizing a Linux server is a humongous topic that touches multiple layers of the application stack and is not an exact science. In this article series we’ll mostly care about optimizing for the more lightweight bunch of NoSQL databases – the bunch that does not rely on a VM (such as Cassandra, HBase or Couch) but runs on native code (such as Memcache, Redis and Mongo) and we will begin by rebuilding the most essential component of any Linux server: **the kernel**.

**Disclaimer**: Proceed on your own risk.  I assume you already know how to download, extract and build a vanilla kernel – if you don’t, the internet is swamped with such articles and GIF. Failing to properly build and install the kernel may render your server unbootable, erase your 9gag posts and force your country into asking financial aid by the IMF.

So, on a vanilla 3.x kernel (3.4.4 as of the day this article was written) the following options are your best friends for the aforementioned workloads:

##### CONFIG_SLUB

Chris Lameter’s kernel object caching system. Much more efficient in managing kernel memory allocations than the old SLAB, offers per-CPU slab queues and enhanced diagnostics via the slabinfo tool. It’s selected by default in recent kernels.

##### CONFIG_JUMP_LABEL

An in-kernel branching optimization that alters branching on the fly for specific cases. Makes the kernel faster. ’nuff said :-)

##### CONFIG_NUMA and friends

Useful for recent manycore servers, enables NUMA awareness in the kernel, improves cache coherency and memory locality on supported hardware.

##### CONFIG_SPARSEMEM_VMEMMAP

A sparse memory optimization option for pfn_to_page and page_to_pfn functions.

##### CONFIG_TRANSPARENT_HUGEPAGE, CONFIG_COMPACTION, CONFIG_MIGRATION and CONFIG_TRANSPARENT_HUGEPAGE_ALWAYS

Whew, that’s a lot of config options. These are probably some of the most important kernel options you can set for this kind of workload. What they essentially enable in-kernel is the ability to allocate larger memory pages than the 4KB default, speeding up memory allocation for memory-hungry processes. In addition to that, they also allow memory page compaction and migration to satisfy these huge page requests, further reducing memory fragmentation.

##### CONFIG_KSM

An important mechanism that actually deduplicates memory pages flagged with `MADV_MERGEABLE`, providing extreme memory savings.

##### CONFIG_ZRAM

Provides a memory-based block device. Data written in that block device will be compressed and stored only in-memory. Useful for temporary storage space (such us mounting under `/tmp`). As this feature is in the staging area, please build it as a module and proceed with caution.

##### CONFIG_ZCACHE, CONFIG_ZSMALLOC and CONFIG_CLEANCACHE

A memory page compression framework that transparently compresses clean and swap pages in-memory providing effortless performance improvements for memory-based workloads. CleanCache uses the ZCache framework as a transcendent memory provider to swap-in clean pages in tmem, further reducing I/O in high-memory environments. As this feature is in the staging area, please proceed with caution.
It goes of course without saying that your setup should always be running on x86_64 at least. Additionally, a number of nice-to-have options that does not necessarily pertain to our workloads are:

##### CONFIG_TASK_IO_ACCOUNTING

Extremely important, allows you to monitor the server’s disk activity per process via awesome tools like `iotop`.

##### CONFIG_PERF_EVENTS

Kernel performance counter hooks to use with tools like perf. Critical for in-depth performance monitoring.

##### CONFIG_PROFILING

Performance profiling hooks used by tools such as OProfile. Equally important to `PERF_EVENTS`.

##### HAVE_BPF_JIT

Just in time rule compiler for pcap-based userland tools that use the Berkeley Packet Filter (such as tcpdump & friends). Should speed up complex rules considerably! Enable it via `/proc/sys/net/core/bpf_jit_enable`.

Select these options with your favorite config method, save, run make or run it through your favorite .deb/.rpm packager to pack it for massive deployment and prepare for glory!

I have to note here that there are many other promising technologies available that may aid in building more efficient infrastructure schemes for memory-based workloads (such as RAMSter that offers swap clustering) but they are still in need of further testing to even consider building a production kernel with them. I invite you to try however, and if you do, please let us know how it went!

Of course, simply recompiling the kernel is not sufficient enough to even say that we have finished optimizing, but it’s a start! In the next article we’ll deal with system & scheduler tuning and how it can help us make the most of our setup, be it virtualized or physical!

Stay tuned!