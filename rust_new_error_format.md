- Feature Name: rust_new_error_format
- Start Date: 2016-06-07
- RFC PR: (leave this empty)
- Rust Issue: (leave this empty)

# Summary
This RFC proposes an update to error reporting in rustc.  Its focus is to change the format of Rust error messages and --explain text to focus on the user's code.  The end goal is for errors and explain text to be more readable, more friendly to new users, while still helping Rust coders fix bugs as quickly as possible.  We expect to follow this RFC with a supplemental RFC that provides a writing style guide for error messages and explain text with a focus on readability and education.

This RFC details work in close collaboration with Niko Matsakis and Yehuda Katz, with input from Aaron Turon and Alex Crichton.  Special thanks to those who gave us feedback on previous iterations of the proposal.

# Motivation

## Default error format

Rust offers a unique value proposition in the landscape of languages in part by codifying concepts like ownership and borrowing. Because these concepts are unique to Rust, it's critical that the learning curve be as smooth as possible. And one of the most important tools for lowering the learning curve is providing excellent errors that serve to make the concepts less intimidating, and to help 'tell the story' about what those concepts mean in the context of the programmer's code.  

![Image of current error format](http://www.jonathanturner.org/images/old_errors_new2.png)

*Example of a borrow check error in the current compiler*

Though a lot of time has been spent on the current error messages, they have a couple flaws which make them difficult to use.  Specifically, the current error format:

* Repeats the file position on the left-hand side.  This offers no additional information, but instead makes the error harder to read.
* Print messages about lines often out of order.  This makes it difficult for the developer to glance at the error and recognize why the error is occuring
* Lacks a clear visual break between errors.  As more errors occur it becomes more difficult to tell them apart.
* Uses technical terminology that is difficult for new users who may be unfamiliar with compiler terminology or terminology specific to Rust.

This RFC details a redesign of errors to focus more on the source the programmer wrote.  This format addresses the above concerns by eliminating clutter, following a more natural order for help messages, and pointing the user to both the "what" of the error and the "why" using color-coded labels.  Below you can see the same error again, this time using the proposed format:

![Image of new error flow](http://www.jonathanturner.org/images/new_errors_new2.png)

*Example of the same borrow check error in the proposed format*

## Expanded error format (revised --explain)

Likewise, today --explain text is generic and does not incorporate the code the user wrote.  For example, today for E0507, the --explain text begins:

```
You tried to move out of a value which was borrowed. Erroneous code example:

use std::cell::RefCell;

struct TheDarkKnight;

impl TheDarkKnight {
    fn nothing_is_true(self) {}
}
```

Languages like Elm have shown how effective an educational tool error messages can be if the verbose explanations like our --explain text are mixed with the user's code.  As mentioned earlier, it's crucial for Rust to be as easy-to-use as possible, especially since it introduces a fair number of concepts that may be unfamiliar to the user.  Even experienced users may need to use --explain text from time to time when they encounter unfamiliar messages.

To that end, this RFC proposes that --explain no longer users an error code.  Instead, --explain becomes a flag in a cargo or rustc invocation that enables a richer, expanded error-reporting mode inspired by languages like Elm.  This more textual mode gives additional explanation to help understand compiler messages better.  

![Image of Rust error in elm-style](http://www.jonathanturner.org/images/elm_like_rust.png)

# Detailed design

The RFC is separated into two parts: the format of error messages and the format of --explain messages.

## Format of error messages

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

Key parts of the code are labeled with messages to help the user understand the message.  

The primary label is the label associated with the main warning/error.  It explains the **what** of the compiler message.  By reading it, the user can begin to understand what the root cause of the error or warning is.  This label is colored to match the level of the message (yellow for warning, red for error) and uses the ^^^ underline.  

Secondary labels help to understand the error and use blue text and --- underline.  These labels explain the **why** of the compiler message.  You can see one such example in the above message where the secondary labels explain that there is already another borrow going on.  In another example, we see another way that primary and secondary work together to tell the whole story for why the error occurred:

![Image of new error format source area](http://www.jonathanturner.org/images/primary_secondary.png)

Taken together, primary and secondary labels create a 'flow' to the message.  Flow in the message lets the user glance at the colored labels and quickly form an educated guess as to how to correctly update their code.

Note: We'll talk more about additional style guidance for wording to help create flow in the subsequent style RFC. 

## Updating --explain

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

## Tying it together

In addition, we propose that the final error message:

```
error: aborting due to 2 previous errors
```

Be changed to notify users of this ability:

```
note: Compile again with --explain for more information on these errors
```

Rather than printing the error number on each message.

## Drawbacks

Changes in the error format can impact integration with other tools.  For example, IDEs that use a simple regex to detect the error would need to be updated to support the new format.  This takes time and community coordination.

While the new error format has a lot of benefits, it's possible that some errors will feel "shoehorned" into it and, even after careful selection of secondary labels, may still not read as well as the original format.  

There is a fair amount of work involved to update the errors and explain text to the proposed format.  

## Alternatives

Rather than using the proposed error format format, we could only provide the verbose --explain style that is proposed in this RFC.  Famous programmers like [John Carmack](https://twitter.com/ID_AA_Carmack/status/735197548034412546) have praised the Elm error format.

![Image of Elm error](http://www.jonathanturner.org/images/elm_error.jpg)

In developing this RFC, we experimented with both styles.  The Elm error format is great as an educational tool, and we wanted to leverage its style in Rust.  For day-to-day work, though, we favor an error format that puts heavy emphasis on quickly guiding the user to what the error is and why it occurred, with an easy way to get the richer explanations when necessary.

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
