# Building a Zig event loop: Understanding Async/Await
Hey, and welcome to a (hopefully) multi-part blog series where I build an event loop for Zig. An event loop in this case refers to a runtime or system which executes non-blocking tasks while often providing tools to communicate between those tasks and the outside world. You may be faimilar with some other "event loops" out there such as Node.js [`libuv`], Rust [`tokio`], or Go's [runtime](https://golang.org/doc/faq#runtime).

[`libuv`]: https://github.com/libuv/libuv
[`tokio`]: https://github.com/tokio-rs/tokio

When it comes to Zig, async/await is one part of the language seems to be the most misunderstood. Currently, I blame this on a few things: Lack of documentation in the language reference and lack-of/incorrectly-implied relations to existing concepts in the wild. Given we'll be writing effectively a non-blocking task scheduler, Zig's async/await serves a good way to simplify all the "non-blocking" and "concurrent"-ness of it to make it easier to read/write.

As a side-note, the series assumes you know how to read and write Zig code. Another assumption is that you're familiar with at least some form of Concurrency. If any of these are not true, I suggest at least playing with them to some extent in order to get familiar with these topics before hand, possibly in other langs/ecosystems; I might draw parallels to other implementations with the hopes of making it easier to understand. This is also my first actual blog post, so take of that what you will. 

### Outline:

Concurrency:
- concurrency vs parallelism
- coroutines: Promises, Goroutines, Futures, Fibers, Green Threads
- stackful/stackless: memory, thunking
- readiness/completion

Async/await API:
- @Frame, async, await, suspend block, resume
- colorless functions
- @frame()
- join/select

Implementation:
- async, pinning, and result locations
- suspend/resume desugared state machine
- await, nosuspend, and hidden control flow
- notes on the future

Async in Action:
- Generator
- Scheduler
- Spawning
- Timers

Conclusion:
- efficient concurrency
- improve scheduler design (and timers)

## Concurrency

In order to understand Zig async/await, you must first grasp the interleaving states that is the universe... Honestly, this might actually be what it feels like to someone new to the idea that your code can stop and start from different points, yet alone run at the same time. I remember someone comparing this mental shift with figuring out recursion: where you have to start executing code and storing the state it produces in your head in order to reason about it effectively. Before diving that deep however, there are a few definitions that should be set in place:

* **Concurrency**: The ability for separate units of execution (coroutines) to switch to/from one another. Or what I like to call: pausable functions.

* **Parallelism**: The ability for separate units of execution to run simultaniously. 

Another analogy I like to use is juggling. Concurrency is juggling more than one object using one hand. Parallelism is having multiple hands which can each juggle multiple objects. Coroutines are then the objects actually being juggled.

<sub>TODO: zig-fast juggling doodle</sub>

Because of how vague "pausable functions" are, people have come up with different ways of implementing coroutines. Of all the different coroutine properties, there are three that I'd like to focus on: Capturing, resuming, and multi-tasking.

### Multi-tasking

Coroutine multi-tasking is all about how coroutines and paused/unpaused. The two primiary ways I know of for going about this is *preemptive* multi-tasking and *cooperative* multi-tasking. When coroutines are preemptively scheduled, it generally means having an outside system control how they're executed. This is common in desktop operating systems where there's a [`Kernel`] which lets [`Processes`] (i.e. coroutines) run for a while but can (and will) force them to stop running so another process can run. Cooperative multi-tasking on the other hand means that the processes themselves dictate when another process can run instead of having a "kernel" forcefully decide for them. 

The latter is efficient as it avoids the overhead of running what is effectively a [`VirtualMachine`]. The former is good for when you can't trust all processes in the system to *cooperative* with each other and distribute run time fairly, so you have to force it. The abilty to boot a process from running in preemptive multi-tasking allows bounded latency for all processes while letting it run until its ready to switch in cooperative multi-tasking allows for maximum throughput.

[`Kernel`]:
[`Processes`]:
[`VirtualMachine`]:

### Capturing

Coroutine capturing refers to what strategy is used to store the state a coroutine needs to unpause when being paused. Think about pausing a function in the middle of doing something. Whatever it was in the middle of, it needs to pick it back up when its continued. How that "continue data" is stored is a decent distinction between multiple coroutine implementations

- Callbacks,
- Stackful,
- Stackless,

### Resuming

Finnaly, coroutine resuming here refers to how a coroutine itself is driven to completion.

- Readiness: Futures, Generators
- Completion: Callbacks, @Frames