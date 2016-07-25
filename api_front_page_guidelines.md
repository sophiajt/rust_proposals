- Feature Name: api_front_page_guideline
- Start Date: 2016-06-07
- RFC PR: (leave this empty)
- Rust Issue: (leave this empty)

# Summary

This RFC is a companion to the [API doc guidelines proposal](https://github.com/rust-lang/rfcs/pull/1574/).  In this RFC, we focus on the format and style of the "front page" of an API.  The goal of this RFC is to outline a clear presentation style for crate documentation that is helpful to new users of a crate.

# Motivation

A user's first experience with a crate is the front page of the crate's documentation. Taken together with the API documentation, the front page is a key piece of understanding a crate's purpose, how to use it, and its limitations.  Making sure this front page is clear will help users of your crate get the most out of it.

# Detailed design

The front page of the crate should cover four main points.  In this RFC, we cover each in turn.

* [Introduction text to the crate](#introduction-text-to-the-crate)
* [First Example](#first-example)
* [Crate Capabilities](#crate-capabilities)
* [Source Examples](#source-examples)

## Introduction text to the crate

The first thing potential users of your crates will see is the introductory summary. This section is a good place to introduce the what and the why the crate. A good introduction is concise but also provides enough information for the reader to understand the purpose of the crate. In addition, a good introduction provides a learning path for the user as they read the rest of the documentation.

Here's an example of a good introduction from the [log crate](https://doc.rust-lang.org/log/log/index.html):
    
> A lightweight logging facade.
>
> A logging facade provides a single logging API that abstracts over the actual logging implementation. Libraries can use the logging API provided by this crate, and the consumer of those libraries can choose the logging framework that is most suitable for its use case.
>
> If no logging implementation is selected, the facade falls back to a "noop" implementation that ignores all log messages. The overhead in this case is very small - just an integer load, comparison and jump.
>
> A log request consists of a target, a level, and a body. A target is a string which defaults to the module path of the location of the log request, though that default may be overridden. Logger implementations typically use the target to filter requests based on some user configuration.

There are several things that make this a good introduction section:
    
* A concise explanation of the crate's purpose.
* The summary describes how the crate behaves in 3 short paragraphs.
* Even if you don't want to read the rest of the `README`, you have enough to get started.
* Readable by a layman, no jargon is introduced in the first few paragraphs.
    
What do we mean by "no jargon is introduced in the first few paragraphs"? Using the `rand` crate as an example - the crate's initial documentation probably should _not_ include a discussion of cryptographically secure random number generators since its primary purpose, for most Rust developers, will be to produce any random number. This information should instead be provided in a separate section that specifically discusses cryptographically secure random number generation.

In general - it's good to give a developer a place to easily learn what your crate is about. Once they have the general idea, you can dive into more details that are specific to your crate, including jargon that would be common with its use.

## First Example

The first example should provide a concise example to help a developer understand how they can begin working with your crate. Ideally, this example should only demonstrate your crate and core Rust functionality. Avoid large examples with lots of features; larger examples are better fit for the `examples` directory.

Let's look at a sample of a good first example of a crate. Notice that the author has focused on getting started, showing how to import and use the crate, and a few simple uses of common functionality.

```rust
#[macro_use]
extern crate log;

pub fn shave_the_yak(yak: &Yak) {
    info!(target: "yak_events", "Commencing yak shaving for {:?}", yak);

    loop {
        match find_a_razor() {
            Ok(razor) => {
                info!("Razor located: {}", razor);
                yak.shave(razor);
                break;
            }
            Err(err) => {
                warn!("Unable to locate a razor: {}, retrying", err);
            }
        }
    }
}
```

## Crate Capabilities

The core capabilities of a crate are the main reason people will use your crate.  In the next section, you can document what each of these core capabilities are and how to use them.  For example, in a crate about random numbers, you may have sections about: generating random numbers, fitting numbers to statistical distributions, use in cryptography, thread-local random numbers, and so on.

It's helpful to introduce and give a clear description for each capability separately to help your readers understand each concept individually before they begin to combine capabilities.

Here's a good example from the 'rand' crate:
    
> ## Thread-local RNG
>
> There is built-in support for a RNG associated with each thread stored in thread-local storage. This RNG can be accessed via thread_rng, or used implicitly via random. This RNG is normally randomly seeded from an operating-system source of randomness, e.g. /dev/urandom on Unix systems, and will automatically reseed itself from this source after generating 32 KiB of random data.
>
> ## Cryptographic security
>
> An application that requires an entropy source for cryptographic purposes must use OsRng, which reads randomness from the source that the operating system provides (e.g. /dev/urandom on Unixes or CryptGenRandom() on Windows). The other random number generators provided by this module are not suitable for such purposes.

## Capability Examples

Sample code for each crate capability should be as simple as possible and side effect free. In the rand crate, thread safe RNG is demonstrated 5 lines of code:

```
use rand::Rng;

let mut rng = rand::thread_rng();

if rng.gen() { // random bool
    println!("i32: {}, u32: {}", rng.gen::<i32>(), rng.gen::<u32>())
}
```

After this example, a user should immediately be able to use a thread safe random number generator in their own program.

More complex source examples should be included as a separate program in the `examples` directory.  Be wary of putting large examples in your doc where users will have to read past the code to get the beginning of your API documentation.

You should also avoid examples that require understanding of external crates unless it's absolutely necessary. As a hypothetical example - your logging crate should not require the user to have experience with [diesel](http://diesel.rs/). It's okay to leave these capabilities as an exercise left to the reader.

# Drawbacks

A possible drawback of this approach is that it risks over-specifying a format that is ill-served for a particular crate.  For example, a crate with a lot of moving parts may need to use more complex examples because smaller examples lose educational value or overall impact.

# Alternatives

Alternatively, the format could use fewer sections.  For example, we specify that each discrete piece of functionality should be documented separately.  However, by not specifying this to be the case, developers could write sections that make sense for their domain, which includes more cross-cutting examples.

# Unresolved questions

It is currently unclear how best to document a crate's limitations.
