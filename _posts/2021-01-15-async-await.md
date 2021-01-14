Welcome to a (hopefully) multi-part blog series where I build an event loop for Zig. There's a lot to say so *this will be a long one.*

# Building a Zig event loop: Async/Await
As an introduction, the series assumes you know how to read and write Zig code. Another assumption is that you're familiar with at least some lower level C concepts. If any of these are not true, I suggest at least playing with them to some extent in order to get familiar with these topics before hand, possibly in other langs/ecosystems; I might draw parallels to other implementations with the hopes of making it easier to understand. The hope is to provide some info for those already experienced so it may not be beginner friendly. This is also my first blog post, so make of that what you will. 

If you're not aware, an event loop in this case refers to a runtime which executes non-blocking tasks while often providing tools to communicate between those tasks and the outside world. You may be faimilar with some other "event loops" out there such as Node.js [libuv], Rust [tokio], or Go's [runtime](https://golang.org/doc/faq#runtime). In order to help in representing non-blocking tasks, Zig provides a mechanism it calls `async/await` to help by desugaring synchronous code into non-blocking, asynchronous code.

[libuv]: https://github.com/libuv/libuv
[tokio]: https://github.com/tokio-rs/tokio

When it comes to Zig, and seemingly Rust as well, async/await is one part of the language seems to be the most misunderstood. Currently, on the Zig side, I blame this on a few things: Lack of documentation in the language reference and lack-of/incorrectly-implied relations to existing concepts in the wild. Given we'll be writing effectively a non-blocking task scheduler, Zig's async/await serves a good way to simplify all the non-blocking-ness to make it easier to use.


## (Brainstorm) Outline:

Async Concurrency:
- concurrency vs parallelism
- coroutines: Promises, Goroutines, Futures, Fibers, Green Threads
- stackful/stackless: memory, thunking
- readiness/completion

Async API:
- @Frame, async, await,
- suspend block, resume, nosuspend
- colorless functions
- @frame()

Async Internals:
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

## Async Concurrency

In order to understand Zig async/await, you must first grasp the interleaving states that is the universe... Honestly, this might actually be what it feels like to someone new to the idea that your code can stop and start from different points, let alone run at the same time. I remember someone comparing this mental shift with figuring out recursion: where you have to start executing code and storing the state it produces in your head in order to reason about it effectively. Before diving that deep however, there are a few definitions that should be set in place:

* **Concurrency**: The ability for separate units of execution (coroutines) to switch to/from one another. Or what I like to call: pausable functions.

* **Parallelism**: The ability for separate units of execution to run simultaniously. 

Another analogy I like to use is juggling. Concurrency is juggling more than one object using one hand. Parallelism is having multiple hands which can each juggle multiple objects. Coroutines are then the objects actually being juggled.

<sub>TODO: zig-fast juggling doodle</sub>

Because of how vague coroutines/"pausable functions" are, people have come up with different ways of implementing coroutines. Of all the different coroutine properties, there are three that I'd like to focus on: multi-tasking, capturing, and resolving.

### Multi-tasking

Coroutine multi-tasking is all about how coroutines and paused/unpaused. The two primiary ways I know of for going about this is *preemptive* multi-tasking and *cooperative* multi-tasking. When coroutines are preemptively scheduled, it generally means having an outside system control how they're executed. This is common in desktop operating systems where there's a [Kernel] which lets [Processes] (i.e. coroutines) run for a while but can (and will) force them to stop running so another process can run. Cooperative multi-tasking on the other hand means that the processes themselves dictate when another process can run instead of having a "kernel" forcefully decide for them. 

The latter is efficient as it avoids the overhead of running what is effectively a [VirtualMachine]. The former is good for when you can't trust all processes in the system to *cooperate* with each other and distribute run time fairly, so you have to force it. The abilty to boot a process from running in preemptive multi-tasking allows bounded latency for all processes while letting it run until its ready to switch in cooperative multi-tasking allows for maximum throughput.

[Kernel]: https://www.geeksforgeeks.org/kernel-in-operating-system/
[Processes]: https://www.tutorialspoint.com/operating_system/os_processes.htm
[VirtualMachine]: https://en.wikipedia.org/wiki/Virtual_machine

### Capturing

Think about pausing a function in the middle of performing a task. Whatever it was in the middle of, it needs to pick it back up when its continued. How that "continue data" is stored is a what I like to call coroutine capturing. It turns out that there's enough distinctions on how coroutines store their data to give them classifications:

##### Stackful

These are the most common types of coroutines to implement when no garbage collector is available. If you're familiar with [stack frames & call stacks](https://en.wikipedia.org/wiki/Call_stack) then its the same idea. There's a stack of frames for each function. The frame stores the functions locals and intermediary data. When calling a new function, a new frame is pushed onto the stack for it to use. Switching between coroutines then becomes pushing any needed stack for unpausing onto to stack and then switching to the next coroutine's stack to continue execution. You may also know this as "Fibers", "Green Threads", "Erlang Processes" or "Goroutines".

```c
1. x = 10; // context
// stack: [Frame1:{ x = 10 }] <- top

2. y = function(); // function() is called
// stack: [Frame1:{ x = 10 }, Frame2{ ...function's stuff }] <- top

3. y = function(); // function() returns, value store in "y"
// stack: [Frame1:{ x = 10, y = ... }] <- top
```

The benefit of this approach is that its fairly trivial to implement given operating system threads do just that. This also handles recursion naturally (just push a new frame) until it doesn't when you run out of stack memory. Some implementations can detect this limit being reached and allocate a new stack for you under the hood. 

Because of its simple nature, it can be alluring to implement this when one reaches for coroutines. However, there are a few downsides you must be aware of (as with any method) before doing so. One is that if you don't know how much stack memory is needed, you're generally left to *guess*. This either means allocating more memory than necessary (e.g. 2kb per corouting) or not allocating enough and having to grow or panic. The other inheritly annoying property is capturing the context often uses more memory that necessary. This is due to nested frames storing not just the continuation data, but also intermediary results (x86 register spilling), previous stack frame links (x86 `push rbp`), and returning function pointers (x86 `ret`).

##### Stackless

There is another approach for storing state; One which is closer to how you would optimally represent a pausable routine. If you're in a language without garbage collection and you want to represent a computation that can be started, stopped and continued, you don't actually reach for execution stacks when you want efficiency. You instead reach for [finite state machines](https://www.embedded.com/software-design-of-state-machines/). This is the scheme that many fast parsers, data protocols, generators and asynchronous systems use in the wild. The idea is that each execution of the routine updates internal state before pausing. Then, when unpaused, it continues from its updated state. This then repeats until the routine finishes (or forever). 

What this looks like for coroutines is that its split up into multiple stages, each separated by a coroutine pause point. Each stage specifies what data it needs to do its work. The coroutine is then a union of all stages with a reference to the current stage. When the coroutine is paused (a stage ended), its internal state is updated to point to the next stage and the next stage's data is prepared and ready to use when the coroutine is unpaused. It may sound complicated, but think of a `switch` statement and how each `case` is its own "stage" with its own variables that it either uses locally or from outside the case statement.

```c++
switch (coroutine.current) {
case entry:
    x = 10;
    // yield
    coroutine.data[after_yield].x = x;
    coroutine.current = after_yield;
    break;

case after_yield:
    x = coroutine.data[after_yield].x
    // ...
} 
```

This turns out to be a nice memory optimization due to the coroutine only needing to store `max(all(stages))` worth of memory and using the OS thread's normal execution stack for everything else. The idiomatic way to transfer data between stages is to write all of the next stage's data before pausing. This can be optimized to write into the next stage's memory directly instead of doing a large copy between transitions. Lastly, because all of the stages are known before hand, the entire coroutine can be allocated in one go without any wasted memory or dynamically growing memory. These optimizations make this approach more common when it comes to representing asynchronous state.

There is of course a downside, but for most cases its rarely hit. It relates to answering the question: *"what happens when you want to do recursion?"*. Because all of the stages need to be known before hand, this is incompatible with runtime recursion and many implementations simply default to banning immediate recursion in their stackfless coroutines. To achieve it nonetheless, a working solution is to dynamically allocate the recursive coroutine and continue it from there. It brings in the worst case dynamic allocation from stackful coroutines, but only for when recursion is necessary.

##### Callbacks

*TODO*

### Resolving

Finally, resolving here refers to how a coroutine its final result is waited on and extracted.

- Readiness: Futures, Generators
- Completion: Callbacks, @Frames
- pro/con: extra polls, 0-copy result, poll-chains, cancellation.

## Async API

*TODO*

## Async Internals

*TODO*

## Async In Action

*TODO*

## Closing

*TODO*