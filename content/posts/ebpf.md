---
title: "TIL: eBPF is awesome"
date: 2020-10-31T19:05:30+01:00
draft: true
---

I was doing some research at work for tracing and observability for microservices, when I came across [Pixielabs](https://pixielabs.ai/). This tool advertises that you can instantly troubleshoot applications without any instrumentation or special code inside the apps, which sounded ✨magical✨ to me. So naturally I wanted to know a little more about what enables this technology to work, and after scrolling through the site, under the "No Instrumentation" section was this acronym **eBPF**.

So what was eBPF exactly? After further digging on the internet, this technology really caught my attention, so I wanted to write some notes on this and I hope this post will spark some further interest in you as well. 

## What is eBPF?

The BPF [design paper](https://www.tcpdump.org/papers/bpf-usenix93.pdf) describes this technology as a tool for capturing and filtering network packets that matched specific rules, discarding any packets that were unwanted. That was the original use case and that's the reason why it's called the "Berkeley Packet Filter" (now eBPF can be used for many different things not just packet filtering but the name stuck for some reason). 

This architecture of network monitoring was apparently vastly superior from the current existing packet capture technologies in terms of performance, mainly because the filtering was done in the kernel instead of copying all of the packets in user space and doing
the calculations there. But with the advancement of processors it didn't hold up so well with it's initial design. That's why the **extended BPF** (eBPF) design was introduced, to take advantage of the modern processing power.

Initially eBPF was to be also used as a network packet filtering solution ofcourse, but the ability to run user-space programs inside the kernel proved to be very powerful. The new design extended what the BPF virtual machine could do, which was to allow it to run on all kinds of events, not just packets, and also do certain actions instead of just the ability to filter things.

You can pick any function from the kernel and execute the program every time that function runs. Running a program that "attaches" to network sockets, tracepoints and perf events can be extremely useful. Developers can debug the kernel without having to re-compile it. There are a myriad of use cases such as:

* Application performance & troubleshooting
* Networking & security
* Runtime security
* Application tracing
* Observability

## How does it work?

To my understanding, eBPF code is run inside a safe virtual machine in the kernel. Writing eBPF programs is done with the help of some frameworks (like [bcc](https://github.com/iovisor/bcc)) and the program is then loaded via the [`bpf`](https://man7.org/linux/man-pages/man2/bpf.2.html) system call.

There are normally security and stability risks when running a user space program inside the kernel, so a number of checks are performed
to ensure that the code we run is safe and that it terminates. The first check ensures that any unbounded loops which could freeze up the kernel are prohibited. 

## References

* https://www.tcpdump.org/papers/bpf-usenix93.pdf -- BPF paper
* https://lwn.net/Articles/740157/ -- A thorough introduction to eBPF
* https://youtu.be/1GSgyvn4N7E -- eBPF summit 2020
