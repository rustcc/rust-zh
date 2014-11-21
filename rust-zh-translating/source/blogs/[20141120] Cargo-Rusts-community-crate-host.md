Cargo: Rust's community crate host
==================================
Nov 20, 2014 • Alex Crichton

Today it is my pleasure to announce that [crates.io](https://crates.io) is online and ready for action. The site is a central location to discover/download Rust crates, and Cargo is ready to start publishing to it today. For the next few months, we are asking that intrepid early adopters [help us](http://doc.crates.io/crates-io.html) get the registry battle-tested.

Until Rust itself is stable early next year, registry dependencies will need to be updated often. Production users may want to continue using git dependencies until then.

# What is Cargo?
Cargo is a package manager [for Rust](http://www.rust-lang.org), [in Rust](https://github.com/rust-lang/cargo). Managing dependencies is a fundamentally difficult problem, but fortunately over the last decade there’s been a lot of progress in the design of package managers. Designed by Carl Lerche and Yehuda Katz, Cargo follows the tradition of successes like [Bundler](http://bundler.io) and [NPM](https://www.npmjs.org):

- Cargo leverages crates.io to foster a thriving community of crates that can easily interoperate with one another and last for years to come.

- Cargo releases developers from the worry of managing dependencies and ensures that all collaborators are building the same code.

- Cargo lets your dependencies say how they should be built, and manages the entire build process for you.

# A Community on Cargo

To get a feel for how Cargo achieves its goals, let’s take a look at some of its core mechanics.

## Declaring Dependencies

Cargo makes depending on third-party code as easy as depending on the standard library. When using Cargo, each crate will have an associated [manifest](http://doc.crates.io/manifest.html) to describe itself and its dependencies. Adding a new dependency is now as simple as adding one line to the manifest, and this ease has allowed Cargo in just a few short months to enable a large and growing network of Rust projects and libraries which were simply infeasible before.

Cargo alone, however, is not quite the entire solution. Discovering dependencies is still difficult, and ensuring that these dependencies are available for years to come is also not guaranteed.

## crates.io

To pair with Cargo, the central crates.io site serves as a single location for publishing and discovering libraries. This repository serves as permanent storage for releases of crates over time to ensure that projects can always build with the exact same versions years later. Up until now, users of Cargo have largely just downloaded dependencies directly from the source GitHub repository, but the primary source will now be shifting to crates.io.

Other programming language communities have been quite successful with this form of central repository. For example [rubygems.org](http://rubygems.org) is your one-stop-shop for [Bundler](http://bundler.io) dependencies and [npmjs.org](https://www.npmjs.org) has had over 600 million downloads in just this month alone! We intend for crates.io to serve a similar role for Rust as a critical piece of infrastructure for [Rust’s long-term stability story at 1.0](http://blog.rust-lang.org/2014/10/30/Stability.html).

# Versioning and Reproducible Builds

Over the past few years, the concept of [Semantic Versioning](http://semver.org) has gained traction as a way for library developers to easily and clearly communicate with users when they make breaking changes. The core idea of semantic versioning is simple: each new release is categorized as a minor or major release, and only major releases can introduce breakage. Cargo allows you to specify version ranges for your dependencies, with the default meaning of “compatible with”.

When specifying a version range, applications often end up requesting multiple versions of a single crate, and Cargo solves this by selecting the highest version of each major version (“stable code”) requested. This highly encourages using stable distributions while still allowing duplicates of unstable code (pre-1.0 and git for example).

Once the set of dependencies and their versions have been calculated, Cargo generates a [Cargo.lock](http://doc.crates.io/guide.html#cargo.toml-vs-cargo.lock) to encode this information. This “lock file” is then distributed to collaborators of applications to ensure that the crates being built remain the same from one build to the next, across times, machines, and environments.

# Building Code

Up to this point we’ve seen how Cargo facilitates discovery and reuse of community projects while managing what versions to use. Now Cargo just has to deal with the problem of actually compiling all this code!

With a deep understanding of the Rust code that it is building, Cargo is able to provide some nice standard features as well as some Rust-specific features:

- By default, Cargo builds as many crates in parallel as possible. This not only applies to upstream dependencies being built in parallel, but also items for the local crate such as test suites, binaries, and unit tests.

- Cargo supports unit testing out of the box both for crates themselves and in the form of integration tests. This even includes example programs to ensure they don’t bitrot.

- Cargo generates documentation for all crates in a dependency graph, and it can even run [Rust’s documentation tests](http://doc.rust-lang.org/rustdoc.html#testing-the-documentation) to ensure examples in documentation stay up to date.

- Cargo can run a [build script](http://doc.crates.io/build-script.html) before any crate is compiled to perform tasks such as code generation, compiling native dependencies, or detecting native dependencies on the local system.

- Cargo supports cross compilation out of the box. Cross compiling is done by simply specifying a `--target` options and Cargo will manage tasks such as compiling plugins and other build dependencies for the right platform.

# What else is in store?

The launch of crates.io is a key step in moving the Cargo ecosystem forward, but the story does not end here. Usage of crates.io is architected assuming a stable compiler, which should be [coming soon](http://blog.rust-lang.org/2014/09/15/Rust-1.0.html)! There are also a number of extensions to crates.io such as a hosted documentation service or a CI build infrastructure hook which could be built out using the crates.io APIs.

This is just the beginning for crates.io, and I’m excited to start finding all Rust crates from one location. I can’t wait to see what the registry looks like at 1.0, and I can only fathom what it will look like after 1.0!
