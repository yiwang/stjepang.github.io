---
layout: post
title:  "Why I'm building a new async runtime"
---

Rust has a relatively new async/await system, and one of its peculiarities is that it doesn't have a runtime baked in like many other programming languages do. To write an async program, you have to pick a library like [async-std](https://docs.rs/async-std) or [tokio](https://docs.rs/tokio), or implement a runtime on your own.

The apparent competition between async-std and tokio to become the *one true runtime* has created the dreaded ecosystem split, where some async libraries depend on one runtime and some on the other. Some libraries even depend on both runtimes, letting you pick one through Cargo feature flags.

I'm now building the third async runtime and publishing it very soon. While async-std and tokio are conceptually very similar, the new runtime is outside their boxes and approaches problems from a completely new angle.

But don't worry, this runtime is not intended to fragment the library ecosystem further - what's interesting about it is that it doesn't really need an ecosystem! I'm hoping it will have the opposite effect and bring libraries closely together rather than apart.

This new project reflects the values behind my previous work, so in order to explain why I've decided to take on this challenge and what it's all about, allow me to introduce myself and tell a story of how I got into Rust...

## Sorting

Between 2016 and 2017, as I was transitioning from a decade of working with C++ to Rust, it's no wonder I approached Rust with a performance-first mindset.

My first PR was optimizing the [sort algorithm](https://github.com/rust-lang/rust/pull/38192), which was then followed up by introducing a state-of-the-art implementation of [unstable sort](https://github.com/rust-lang/rust/pull/40601), and finally [parallel sorts](https://github.com/rayon-rs/rayon/pull/379) in Rayon. That was achieved by porting [timsort](https://en.wikipedia.org/wiki/Timsort) and [pattern-defeating quicksort](https://github.com/orlp/pdqsort) to Rust, the newest development coming from research at the time.

At that moment, Rust’s standard library became more efficient than C++'s standard libraries at sorting.

## Crossbeam

Feeling content with how fast Rust is at sorting, I moved on to [crossbeam](https://docs.rs/crossbeam), a library offering a set of synchronization primitives supplanting the standard library.

[crossbeam-channel](https://docs.rs/crossbeam-channel) is probably my best known work in that area. Today, we take crossbeam-channel for granted, but it's interesting to note how a library of its kind was not even thought to be possible before its inception!

Its particular combination of (1) straightforward API, (2) efficient lock-free internals, and (3) a powerful selection mechanism doesn’t really exist anywhere else, at least not as far as I know.

Channels in other programming languages tend to give in somewhere and are picking two out of those three qualities. But Rust, at its core, is always striving to *bend the curve*, as we like to say it.

## Tokio

Having solved the problem of channels, I moved on and in 2018 joined the [tokio](https://github.com/tokio-rs/tokio) team to improve the performance of its scheduler and implement robust async files.

However, even I, as a tokio developer, had difficulties understanding its complex internals and was struggling with implementing the optimizations I wanted. Over time, my vision of async Rust started diverging from tokio's vision further and further.

There was only one way forward for me: start from scratch, begin new research into async Rust, and try to push it into a different direction.

## async-task

I began by building [async-task](https://docs.rs/async-task), a small and efficient library for spawning futures.

async-task condenses all the ugliness of Rust’s task system and presents a simple and sweet interface. Its underlying implementation is pretty gnarly - it took months of hard work, required an extensive test suite, and optimization to absurdity. In fact, given its features, it’s as fast as it can possibly get!

The complexity in async-task’s implementation is what I like to call *boring complexity* because it's unavoidable, doesn’t leak through abstractions, and is utterly uninteresting. Once it’s done, we never have to think about it again.

async-task too achieved something that was previously unthinkable - it's the first time ever we got single-allocation tasks, which is how it outperformed tokio's executor at the time. And it did much more than that.

To me, async-task marks a new era in async Rust because it allows anyone to [build a really fast executor](https://stjepang.github.io/2020/01/31/build-your-own-executor.html) in just a few lines of safe code.

While building executors used to be a job reserved for a few experts developing Tokio, now anyone could match the performance of its executor with almost no effort!

## async-std

Now that task spawning was perfected, I could shift my focus onto more interesting problems. async-task became the foundation of [async-std](https://docs.rs/async-std), a new runtime that filled in tokio's gaps.

async-std was the first stable runtime that supported async/await, had fully complete documentation with examples, introduced single-allocation tasks, and had a simple implementation that invited a wider set of contributors.

The key reason why async-std managed to overtake tokio in the race to deliver the first stable async/await-ready runtime is because async-task made building new runtimes so easy!

What’s remarkable is that async-std’s whole runtime spanned only 400 lines of safe code, in addition to [futures-timer](https://docs.rs/futures-timer) and [mio](https://docs.rs/mio), which made me very happy because now I could finally play with all the optimizations I originally wanted to. More importantly, anyone could!

The differences between async-std and tokio kept shrinking over time and today they’re conceptually very similar with minor differences in opinions on how runtimes get created or are configured.

## Build your own X

Still, I was not completely content because it didn’t feel like async runtimes were a 'solved' problem yet. Async Rust is still immature and rather difficult.

My previous work had a satisfying pattern: identify a problem, figure out how to defy its rules of tradeoffs, implement the solution, and move on. But not here. I was really stuck.

I worked on async-std overtime, feeling constant exhaustion and stress. No matter how much effort was put into it, I was only making marginal progress.

My next *big idea* was to attempt to solve the problem of [blocking inside async code](https://stjepang.github.io/2019/12/04/blocking-inside-async-code.html), which ultimately [didn’t pan out](https://github.com/async-rs/async-std/pull/631). It was too controversial and still didn’t solve the problem it intended to completely.

I was left with a feeling of dismay. If I can’t solve this, perhaps someone else can?

In hopes of somebody else picking up where I left off, I began blogging about the craft of building runtimes, attempting to dump all the knowledge I have accumulated over the years and share it with the world.

Hence two blog posts: [Build your own block_on](https://stjepang.github.io/2020/01/25/build-your-own-block-on.html) function and [Build your own executor](https://stjepang.github.io/2020/01/31/build-your-own-executor.html).

## smol

I spent a lot of time thinking about how to teach readers of my blog about async runtimes effectively. Half-way through the *Build your own X* blog post series, an idea came to me and I became obsessed with it.

First I reduced the executor to a few lines of safe code, then work-stealing, then mio, then I/O reactor, then networking APIs, then the blocking thread pool... suddenly, piece by piece, the whole tower of complexity began falling apart and what was left seemed just too good to be true. I was *shocked*.

Excited by where this was going, I began work on a new runtime dubbed *smol*. Right now, it’s feature complete, contains a bunch of examples, and I’m only polishing it up and writing documentation before revealing it to the world.

Imagine a runtime that is just as powerful as async-std and tokio, just as fast, does not depend on [mio](https://docs.rs/mio), fits into a single file, has a much simpler API, and contains only safe code.

It brings a novel approach to structured concurrency, it integrates with the library ecosystem, it can run [hyper](https://docs.rs/hyper), [async-h1](https://docs.rs/async-h1), [async-native-tls](https://docs.rs/async-native-tls), and [async-tungstenite](https://docs.rs/async-tungstenite). It can spawn `!Send` futures, supports the full networking stack, handles files, processes, stdio, and custom I/O objects. It works on all platforms you’d expect it to.

Now I’ll be able to put an end to the *Build your own X* blog post series. After all, it’s hard to explain async runtimes better than this runtime explains itself!

It feels like this is finally it - it’s the big leap I was longing for the whole time! As a writer of async runtimes, I’m excited because this runtime may finally make my job obsolete and allow me to move onto whatever comes next.
