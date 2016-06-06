# Improving Rust's Errors

## Summary

This RFC proposes an update to error reporting in rustc.  Its focus is the Rust error format, wording used in errors, and an improvement to the --explain capability.  The end goal is for errors to be more readable, more friendly to new users, while still helping Rust coders fix bugs as quickly as possible.

This RFC details work in close collaboration with Niko Matsakis and Yehuda Katz, with input from Aaron Turon and Alex Crichton.  Special thanks to those who gave us feedback on previous iterations of the proposal.

## Motivation

Rust offers a unique value proposition in the landscape of languages in part by codifying concepts like ownership and borrowing. Because these concepts are unique to Rust, it's critical that the learning curve be as smooth as possible. And one of the most important tools for lowering the learning curve is providing excellent errors that serve to make the concepts less intimidating, and to help 'tell the story' about what those concepts mean in the context of the programmer's code.  

![Image of current error format](http://www.jonathanturner.org/images/old_errors_new2.png)
*Example of a borrow check error in the current compiler*

This RFC details a redesign of errors to focus more on the source the programmer wrote.  In doing so, these new messages help eliminate clutter, remove difficult language, and focus on more effectively "telling the story" about how an error occurred.

![Image of new error flow](http://www.jonathanturner.org/images/new_errors_new2.png)
*Example of the same borrow check error in the proposed format*

## Detailed Design

The RFC is separated into four parts.  The first three detail changes to how errors work today: the format of the error, the flow of the information given to the user, and language used in the error.  The fourth part covers updates to --explain that incorporate the user's actual source code, a style made popular by Elm.
Format

The proposal is a lighter error format focused on the code the user wrote.  Messages that help understand why an error occurred appear as labels on the source.  You can see an example below:

![Image of new error flow](http://www.jonathanturner.org/images/new_errors_new2.png)

The goals of this new format are to:

* Create something that's visually easy to parse
* Remove noise/unnecessary information
* Present information in a way that works well for new developers, post-onboarding, and experienced developers without special configuration
* Draw inspiration from Elm as well as Dybuk and other systems that have already improved on the kind of errors that Rust has.

In order to accomplish this, the proposed design needs to satisfy a number of constraints to make the result maximally flexible across various terminals:

* Multiple errors beside each other should be clearly separate and not muddled together.
* Each error message should draw the eye to where the error occurs with sufficient context to understand why the error occurs.
* Each error should have a "header" section that is visually distinct from the code section.
* Code should visually stand out from text and other error messages.  This allows the developer to immediately recognize their code.
* Error messages should be just as readable when not using colors (eg for users of black-and-white terminals, color-impaired readers, weird color schemes that we can't predict, or just people that turn colors off)
* Be careful using “ascii art” and avoid unicode. Instead look for ways to show the information concisely that will work across the broadest number of terminals.  We expect IDEs to possibly allow for a more graphical error in the future.
* Where possible, use labels on the source itself rather than sentence "notes" at the end.  
* Keep filename:line easy to spot for people who use editors that let them click on errors

## Parts of the design

### Header

![Image of new error format heading](http://www.jonathanturner.org/images/rust_error_1_new.png)

The header now spans two lines.  It gives you access to knowing a) if it's a warning or error, b) the text of the warning/error, and c) the location of this warning/error.  You can see we also use the [--explain E0499] as a way to let the developer know they can get more information about this kind of issue.  While we use some bright colors here, we expect the use of colors and bold text in the 'Source area' (shown below) to draw the eye first.

### Line number column

![Image of new error format line number column](http://www.jonathanturner.org/images/rust_error_2_new.png)

The line number column lets you know where the error is occurring in the file.  Because we only show lines that are of interest for the given error/warning, we elide lines if they are not annotated as part of the message (we currently use the heuristic to elide after one un-annotated line).  

Inspired by Dybuk and Elm, the line numbers are separated with a 'wall', a separator formed from |>, to clearly distinguish what is a line number from what is source at a glance.

### Source area

![Image of new error format source area](http://www.jonathanturner.org/images/rust_error_3_new.png)

The source area shows the related source code for the error/warning.  The source is laid out in the order it appears in the source file, giving the user a way to map the message against the source they wrote.

Key parts of the code are labeled with messages to help the user understand the message.  The primary label is the label associated with the main warning/error.  This is colored to match (yellow for warning, red for error) and uses the ^^^ underline.  Secondary labels help to understand the error and use blue text and --- underline.  Together these help to create a 'flow' to the message.

## Flow

The new error format helps the user understand what is wrong with a piece of code by showing key parts of the code and how they contribute to the error.  Let's look at our example again, with this in mind:

![Image of new error flow](http://www.jonathanturner.org/images/new_errors_new2.png)

You can see three different labels on different parts of the code:

* first mutable borrow occurs here
* second mutable borrow occurs here (the actual error)
* first borrow ends here

Taken together the user can read the "story" about the mutable borrow that already occurred, when that borrow will end, and that a second borrow happens between these two, which causes the error.  

Ideally, error messages should use labels to give the reader a way to understand the cause of the error by only reading the labels and related source.  In converting over a number of errors, we found these goals work well:

* Primary labels (those with ^^^) should be self-contained.  They should describe succinctly the issue without requiring reading the other labels
* Secondary labels (those with ---) can be used to describe helpful starts to the issue.  For example, if a const is redefined, the secondary label can point out the first time the const is defined.
* We found that using 'flow-sensitive' language like saying "first" and "second" helped readability.  That said, this also poses a challenge that we have to always ensure that the label text does not create additional confusion by being shown in a an order that feels unnatural to the user.
* Label text has limited space for readability.  Long labels can make the message difficult to read:

![Image of confusing cargo error](http://www.jonathanturner.org/images/cargo_error.png)

Instead, favor labels that are ~6 words or less.

Additionally, the above example shows the issue we've called the "rainbow problem" where too many colors and too much bold is used.  We are working on updates to the format which will hopefully address this issue.

In the next section, we'll discuss the words we use in the error text itself to help the user understand the error message as quickly as possible.

## Language

The words used in the error message are just as important as the format of the message.  Rust comes from a very technical background from industrial research.  Its pedigree means that there are a lot of formal programming language terms in its errors.  

As more people come to Rust, it's clear that the language we use in errors should focus on helping the programmer fix bugs and learn Rust.  To do so, we should avoid using technical terms when they aren't necessary.  I recognize this is a bit of a two-edged sword, as sometimes the technical term is both precise and terse.  Still, erring in the direction of making errors friendly to new users will help more people get over the hump of learning Rust.

Some technical terms that can be changed without losing the meaning of the error:

| Technical term | Friendly term |
| -------------- | ------------- | 
| shadows | uses the same name as |
| non-exhaustive pattern | missing a pattern |
| mutate | "update" or "write" |
| integral | "integer" or "number" |
| elided | "inferred" or "assumed" or "left out" |
| arity | number of arguments |

While other technical terms should remain because they are used in the language itself and are generally required to understand how to use Rust.  For example, mutable/immutable are the correct adjectives because 'mut' is a keyword in Rust.  The same is true for terms like associated type, derive, borrow, and lifetime.

##Updating --explain

Currently, --explain text focuses on the error code.  You can call the compiler with --explain <error code> and receive a verbose description of what causes errors of that number.

We propose changing --explain to no longer take an error code.  Instead, passing --explain to the compiler (or to cargo) will turn the compiler output into an expanded form.  Specifically, errors will become more verbose versions of themselves.

For example, today for E0507, the --explain text begins:

```
You tried to move out of a value which was borrowed. Erroneous code example:

use std::cell::RefCell;

struct TheDarkKnight;

impl TheDarkKnight {
    fn nothing_is_true(self) {}
}

fn main() {
    let x = RefCell::new(TheDarkKnight);

    x.borrow().nothing_is_true(); // error: cannot move out of borrowed content
}

Here, the `nothing_is_true` method takes the ownership of `self`. However,
`self` cannot be moved because `.borrow()` only provides an `&TheDarkKnight`,
which is a borrow of the content owned by the `RefCell`. To fix this error,
you have three choices:

* Try to avoid moving the variable.
* Somehow reclaim the ownership.
* Implement the `Copy` trait on the type.
```

This example shows off some of the great work that we've done to help explain the errors to users.  We propose extending this, and using the user's actual code, rather than having to fabricate example code.  An alternative using the proposed change may look like:

![Image of Rust error in elm-style](http://www.jonathanturner.org/images/elm_like_rust.png)

In addition, we propose that the final error message:

```
error: aborting due to 2 previous errors
```

Be changed to notify users of this ability:

```
note: Compile again with --explain for more information on these errors
```

## Drawbacks

Changes in the error format can impact integration with other tools.  For example, IDEs that use a simple regex to detect the error would need to be updated to support the new format.  This takes time and community coordination.

While this new format has a lot of benefits, it's possible that some errors will feel "shoehorned" into it and, even after careful selection of secondary labels, may still not read as well as the original format.  

There is a fair amount of work involved to update the errors to this new format and to provide updated wording to both the errors and the --explain text.  

## Alternatives

Rather than using this format, another format that's becoming more popular (even being called out by famous programmers like John Carmack) is the Elm error format.

![Image of Elm error](http://www.jonathanturner.org/images/elm_error.jpg)
*Example of an Elm error*

While this could be the default error format, I, and those who helped put this RFC together, feel that a tighter error format with good labels is a better error for everyday use.  Instead of being the default, we propose the extension to --explain to be able to output content like the example Elm error above.

## Stabilization

Currently, these new rust error format is available on nightly using the ```export RUST_NEW_ERROR_FORMAT=true``` environment variable.  Ultimately, this should become the default.  In order to get there, we need to ensure that the new error format is indeed an improvement over the existing format in practice.  

How do we measure the readability of error messages?  This RFC details an educated guess as to what would improve the current state but shows no ways to measure success.

Likewise, While some of us have been dogfooding these errors, we don't know what long-term use feels like.  For example, after a time does the use of color feel excessive?  We can always update the errors as we go, but it'd be helpful to catch it early if possible.

## Unresolved questions

There are a few unresolved questions:
* Editors that rely on pattern-matching the compiler output will need to be updated.  It's an open question how best to transition to using the new errors.  There is on-going discussion of standardizing the JSON output, which could also be used.
* Can additional error notes be shown without the "rainbow problem" where too many colors and too much boldness cause errors to beocome less readable?
