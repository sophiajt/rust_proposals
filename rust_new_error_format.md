- Feature Name: rust_new_error_format
- Start Date: 2016-06-07
- RFC PR: (leave this empty)
- Rust Issue: (leave this empty)

# Summary
This RFC proposes an update to error reporting in rustc.  Its focus is the format of Rust error messages an improvement to the --explain capability.  The end goal is for errors to be more readable, more friendly to new users, while still helping Rust coders fix bugs as quickly as possible.  We expect to follow this RFC with a supplimental RFC that recommends a writing guide for error messages and explain text focused on readability and education.

This RFC details work in close collaboration with Niko Matsakis and Yehuda Katz, with input from Aaron Turon and Alex Crichton.  Special thanks to those who gave us feedback on previous iterations of the proposal.

# Motivation
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

Secondary labels help to understand the error and use blue text and --- underline.  These labels explain the **why** of the compiler message.  You can see one such example in the above message where the secondary labels explain that there is already another borrow going on.  Another, simpler example is the case of 

Together together, primary and secondary labels help to create a 'flow' to the message.  Flow in the message helps it to be understood quickly by letting the user glance at the colored labels and quickly form an educated guess as to how to correctly update their code.

Note: We'll talk more about additional style guidance for wording to help create flow in the subsequent style RFC. 

# Drawbacks
[drawbacks]: #drawbacks

Why should we *not* do this?

# Alternatives
[alternatives]: #alternatives

What other designs have been considered? What is the impact of not doing this?

# Unresolved questions
[unresolved]: #unresolved-questions

What parts of the design are still TBD?
