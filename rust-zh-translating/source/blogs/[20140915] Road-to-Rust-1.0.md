Road to Rust 1.0
================
Sep 15, 2014 • Niko Matsakis

Rust 1.0 is on its way! We have nailed down a concrete list of features and are hard at work on implementing them. We plan to ship the 1.0 beta around the end of the year. If all goes well, this will go on to become the 1.0 release after the beta period. After 1.0 is released, future 1.x releases will be backwards compatible, meaning that existing code will continue to compile unmodified (modulo compiler bugs, of course).

Of course, a Rust 1.0 release means something more than “your code will continue to compile”. Basically, it means that we think the design of Rust finally feels right. More specifically, it feels *minimal*. The language itself is now focused on a simple core concept, which we call ownership and borrowing (more on this later). Leveraging ownership and borrowing, we have been able to build up everything else that we have needed in libraries. This is very exciting, because any library we can write, you can write too. This really gives us confidence that Rust will not only achieve its original goals but also go beyond and be used for all kinds of things that we haven’t even envisioned.

# The road to Rust 1.0

Rust has gone through a long evolution. If you haven’t looked at Rust in a while, you may be surprised at what you see: over the last year, we’ve been radically simplifying the design. As a prominent example, Rust once featured several pointer types, indicated by various sigils: these are gone, and only the reference types (`&T`, `&mut T`) remain. We have also been able to consolidate and simplify a number of other language features, such as closures, that once sported a wide variety of options. (Some of these changes are still in progress.)

The key to all these changes has been a focus on the core concepts of *ownership and borrowing*. Initially, we introduced ownership as a means of transferring data safely and efficiently between tasks, but over time we have realized that the same mechanism allows us to move all sorts of things out of the language and into libraries. The resulting design is not only simpler to learn, but it is also much “closer to the metal” than we ever thought possible before. All Rust language constructs have a very direct mapping to machine operations, and Rust has no required runtime or external dependencies. When used in its own most minimal configuration, it is even possible to write an [operating](https://github.com/charliesome/rustboot) [systems](https://github.com/ryanra/RustOS) [kernel](https://github.com/jvns/puddle) in Rust.

Throughout these changes, though, Rust has remained true to its goal of providing the **safety** and **convenience** of modern programming languages, while still offering the **efficiency** and **low-level control** that C and C++ offer. Basically, if you want to get your hands dirty with the bare metal machine, but you don’t want to spend hours tracking down segfaults and data races, Rust is the language for you.

If you’re not already familiar with Rust, don’t worry. Over the next few months, we plan on issuing a regular series of blog posts exploring the language. The first few will focus on different aspects of ownership and how it can be used to achieve safe manual memory management, concurrency, and more. After that, we’ll turn to other aspects of the Rust language and ecosystem.

# What is left to do

We’ve made great progress, but there is still a lot to do before the release. Here is a list of the big-ticket changes we are currently working on:

- *Dynamically sized types*: This extension to the type system allows us to uniformly handle types where the size is not known at compile time, such as an array type. This enables us to support user-designed smart pointers that contain arrays or objects. Nicholas Cameron [recently landed](https://github.com/rust-lang/rust/commit/7932b719ec2b65acfa8c3e74aad29346d47ee992) a heroic commit implementing the bulk of the work.

- *Unboxed closures*: Our new [closure design](https://github.com/rust-lang/rfcs/blob/master/text/0114-closures.md) unifies closures and object types. Much of the spec has been implemented.

- *Associated types*: We are moving our trait system to use [associated types](https://github.com/rust-lang/rfcs/pull/195), which really help to cut down on the level of generic annotations required to write advanced generic libraries. Patrick Walton has done an initial implementation.

- *Where clauses*: We are adding a flexible new form of constraints called [where clauses](https://github.com/rust-lang/rfcs/pull/135). Patrick Walton already landed support for the basic syntax, and I have implemented the remaining functionality on a branch that should be landing soon.

- *Multidispatch traits*: We are extending traits so that they can [match on more than one type at a time](https://github.com/rust-lang/rfcs/pull/195), which opens up a lot of new opportunities for more ergonomic APIs. I have prototyped this work on a branch.

- *Destructors*: We are improving our destructor semantics to not require zeroing of memory, which should improve compilation and execution times. Felix Klock has implemented the requisite analysis and is in the process of landing it.

- *Green threading*: We are removing support for green threading from the standard library and moving it out into an external package. This allows for a closer match between the Rust model and the underlying operating system, which makes for more efficient programs. Aaron Turon has [written the RFC](https://github.com/rust-lang/rfcs/pull/230) and is getting started on that work now.

At the library level, we are currently engaged in a sweep over libstd to decide what portions are stable and which are not. You can [monitor the progress](http://doc.rust-lang.org/std/stability.html) here. (Note though that many of the ‘unstable’ items are simply things whose name will be changed slightly to conform to conventions or other minor tweaks.)

# Cargo and the library ecosystem

Earlier I wrote that Rust 1.0 is not so much an endpoint as it is a starting point. This is very true. The goal for Rust 1.0 is to be an flexible substrate for building efficient libraries – but libraries aren’t any good if nobody can find them or they are difficult to install.

Enter [Cargo, the Rust package manager](http://crates.io). Cargo has been undergoing rapid development lately and is already quite functional. By the time 1.0 is released, we plan to also have a central repository up and running, meaning that it will be simple to create and distribute Rust libraries (which we call “crates”). Oh, and of course Cargo and its associated server are both written in Rust.

# Release process

Rust releases have been following a train schedule for a long time and we don’t plan on changing that. Once we start having stable releases, however, we’ll also build up a bit more infrastructure. Our plan is to adopt the “channel” system used by many other projects such as [Firefox](https://www.mozilla.org/en-US/firefox/channel/), [Chrome](http://www.chromium.org/getting-involved/dev-channel), and [Ember.js](http://emberjs.com/builds/).

The idea is that there are three channels: Nightly, Beta, and Stable. The Nightly channel is what you use if you want the latest and greatest: it includes unstable features and libraries that may still change in backwards incompatible ways. Every six weeks, we cut a new branch and call it Beta. This branch excludes all the unstable bits, so you know that if you are using Beta or Stable, your code will continue to compile. At the same time, the existing Beta branch is promoted to the Stable release. We expect that production users will prefer to test on the Beta branch and ship with the Stable branch. Testing on Beta ensures that we get some advanced notice if we accidentally break anything you are relying on.

With regard to the 1.0 release specifically, the plan is to release the 1.0 beta and then follow this same process to transition to the official 1.0 release. However, if we find a serious flaw in the 1.0 beta, we may defer and run an additional beta period or two. After all, it’s better to wait a bit longer than wind up committed to something broken.

# Looking forward

In many ways, Rust 1.0 is not so much an endpoint as it is a starting point. Naturally, we plan on continuing to develop Rust: we have a lot of features we want to add, many of which are already in the pipeline. But the work that’s most exciting to me is not the work that will be done by the Rust team. Rather, I expect that having a stable base will allow the Rust community and ecosystem to grow much more rapidly than it already has. I can’t wait to see what comes out of it.
