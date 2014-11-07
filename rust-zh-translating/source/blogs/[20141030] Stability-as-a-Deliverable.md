Stability as a Deliverable
==========================
Oct 30, 2014 • Aaron Turon and Niko Matsakis

The upcoming Rust 1.0 release means [a lot](http://blog.rust-lang.org/2014/09/15/Rust-1.0.html), but most fundamentally it is a commitment to stability, alongside our long-running commitment to safety.

Starting with 1.0, we will move to a 6-week release cycle and a menu of release “channels”. The stable release channel will provide pain-free upgrades, and the nightly channel will give early adopters access to unfinished features as we work on them.

# Committing to stability

Since the early days of Rust, there have only been two things you could count on: safety, and change. And sometimes not the first one. In the process of developing Rust, we’ve encountered a lot of dead ends, and so it’s been essential to have the freedom to change the language as needed.

But Rust has matured, and core aspects of the language have been steady for a long time. The design feels right. And there is a huge amount of pent up interest in Rust, waiting for 1.0 to ship so that there is a stable foundation to build on.

It’s important to be clear about what we mean by stable. We don’t mean that Rust will stop evolving. We will release new versions of Rust on a regular, frequent basis, and we hope that people will upgrade just as regularly. But for that to happen, those upgrades need to be painless.

To put it simply, our responsibility is to ensure that you never dread upgrading Rust. If your code compiles on Rust stable 1.0, it should compile with Rust stable 1.x with a minimum of hassle.

# The plan

We will use a variation of the train model, first introduced in web browsers and now widely used to provide stability without stagnation:

- New work lands directly in the master branch.

- Each day, the last successful build from master becomes the new nightly release.

- Every six weeks, a beta branch is created from the current state of master, and the previous beta is promoted to be the new stable release.

In short, there are three release channels – nightly, beta, and stable – with regular, frequent promotions from one channel to the next.

New features and new APIs will be flagged as unstable via feature gates and [stability attributes](http://doc.rust-lang.org/reference.html#stability), respectively. Unstable features and standard library APIs will only be available on the nightly branch, and only if you explicitly “opt in” to the instability.

The beta and stable releases, on the other hand, will only include features and APIs deemed *stable*, which represents a commitment to avoid breaking code that uses those features or APIs.

# The FAQ

There are a lot of details involved in the above process, and we plan to publish RFCs laying out the fine points. The rest of this post will cover some of the most important details and potential worries about this plan.

## What features will be stable for 1.0?

We’ve done an analysis of the current Rust ecosystem to determine the most used crates and the feature gates they depend on, and used this data to guide our stabilization plan. The good news is that the vast majority of what’s currently being used will be stable by 1.0:

- There are several features that are nearly finished already: struct variants, default type parameters, tuple indexing, and slicing syntax.

- There are two key features that need significant more work, but are crucial for 1.0: unboxed closures and associated types.

- Finally, there are some widely-used features with flaws that cannot be addressed in the 1.0 timeframe: glob imports, macros, and syntax extensions. This is where we have to make some tough decisions.

After extensive discussion, we plan to release globs and macros as stable at 1.0. For globs, we believe we can address problems in a backwards-compatible way. For macros, we will likely provide an alternative way to define macros (with better [hygiene](http://en.wikipedia.org/wiki/Hygienic_macro)) at some later date, and will incrementally improve the “macro rules” feature until then. The 1.0 release will stabilize all current macro support, including import/export.

On the other hand, we *cannot* stabilize syntax extensions, which are plugins with complete access to compiler internals. Stabilizing it would effectively forever freeze the internals of the compiler; we need to design a more deliberate interface between extensions and the compiler. So syntax extensions will remain behind a feature gate for 1.0.

Many major uses of syntax extensions could be replaced with traditional code generation, and the Cargo tool will soon be growing specific support for this use case. We plan to work with library authors to help them migrate away from syntax extensions prior to 1.0. Because many syntax extensions don’t fit this model, we also see stabilizing syntax extensions as an immediate priority after the 1.0 release.

## What parts of the standard library will be stable for 1.0?

We have been steadily stabilizing the standard library, and have a plan for nearly *all* of the modules it provides. The expectation is that the vast majority of functionality in the standard library will be stable for 1.0. We have also been migrating more experimental APIs out of the standard library and into their own crates.

## What about stability attributes outside of the standard library?

Library authors can continue to use stability attributes as they do today to mark their own stability promises. These attributes are not tied into the Rust release channels by default. That is, when you’re compiling on Rust stable, you can only use stable APIs from the standard library, but you can opt into experimental APIs from other libraries. The Rust release channels are about making upgrading Rust *itself* (the compiler and standard library) painless.

Library authors should follow [semver](http://semver.org); we will soon publish an RFC defining how library stability attributes and semver interact.

## Why not allow opting in to instability in the stable release?

There are three problems with allowing unstable features on the stable release.

First, as the web has shown numerous times, merely *advertising* instability doesn’t work. Once features are in wide use it is very hard to change them – and once features are available at all, it is very hard to prevent them from being used. Mechanisms like “vendor prefixes” on the web that were meant to support experimentation instead led to de facto standardization.

Second, unstable features are by definition work in progress. But the beta/stable snapshots freeze the feature at scheduled points in time, while library authors will want to work with the latest version of the feature.

Finally, we simply *cannot* deliver stability for Rust unless we enforce it. Our promise is that, if you are using the stable release of Rust, you will never dread upgrading to the next release. If libraries could opt in to instability, then we could only keep this promise if all library authors guaranteed the same thing by supporting all three release channels simultaneously.

It’s not realistic or necessary for the entire ecosystem to flawlessly deal with these problems. Instead, we will enforce that stable means stable: the stable channel provides only stable features.

## Won’t this split the ecosystem? Will everyone use nightly at 1.0?

It doesn’t split the ecosystem: it creates a subset. Programmers working with the nightly release channel can freely use libraries that are designed for the stable channel. There will be pressure to stabilize important features and APIs, and so the incentives to stay in the unstable world will shrink over time.

We have carefully planned the 1.0 release so that the bulk of the existing ecosystem will fit into the “stable” category, and thus newcomers to Rust will immediately be able to use most libraries on the stable 1.0 release.

## What are the stability caveats?

We reserve the right to fix compiler bugs, patch safety holes, and change type inference in ways that may occasionally require new type annotations. We do not expect any of these changes to cause headaches when upgrading Rust.

The library API caveats will be laid out in a forthcoming RFC, but are similarly designed to minimize upgrade pain in practice.

## Will Rust and its ecosystem continue their rapid development?

Yes! Because new work can land on master at any time, the train model doesn’t slow down the pace of development or introduce artificial delays. Rust has always evolved at a rapid pace, with lots of help from amazing community members, and we expect this will only accelerate.
