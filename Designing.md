# Designing Enhancemnents

(TBD)

This section discusses some general design guidelines for creating new WGSL enhancements.

## Goals

We're aiming to create a modestly improved, practical variant of WGSL.
The result should feel like WGSL with a few things added.

## Priorities

We're aiming to prioritize WGSL enhancements that are:

1) important for community projects
2) feel natural to the WGSL programmer
3) not too difficult to integrate into community tools

We're guided by demonstrated needs in community projects.

We're willing to work harder on tools to make the experience feel more natural
to the programmer.

## Who are WESL Programmers?

**Vanilla WGSL programmers first**.
Our programmers will learn WGSL before learning our extensions.
We should be additive to WGSL, not change semantics.

**Part time shader programmers**. Our programmers will have to relearn any intracacies we introduce over and over again, so keep it simple.

**Medium size teams**. Our progammers work on projects ranging in size from tiny programs
up to teams of few dozen programmers.

**General programming knowledge**.
Our programmers will likely know
JavaScript/TypeScript or Rust,
though not both. They'll also have been trained in at least one
other popular language like Python, Java, or C++.
Overall, we can expect that our programmers will know general 
concepts available in most mainstream languages,
but we can't count on them to know the details from any particular language
other than WGSL.

### Syntax from Rust and TypeScript?

Avoid trying to imitate a TypeScript or Rust language feature syntactically
unless it can fully match the source language semantics.
If it _looks just like_ a Rust or TypeScript feature, programmers will expect it
_work like_ the source language.

But fully matching the source language will often be impossible
in WGSL, or undesirable for complexity reasons.
And a feature that looks the same but works differently will be hard
for programmers to keep straight in their minds.

Aim for simple instead of imitative.

## Stable Editions

We want community tools to support all stable editions, and we hope to support
stable enhancements for years.
Before we include an enhancement in a stable edition,
we'll want it to be
polished, proven in use, tested, and well documented.

## Experimental Enhancements

The tools we're developing (e.g. the linker/transpilers)
will be a good base to build out new enhancement ideas.
It should be easy to implement and try out new WESL enhancements on
local WESL projects.

Experimental enhancements aren't intended be supported across tools,
and don't need to be compatible with other experiments.
As experiments move toward stability
we'll move experimental enhancements into more tools behind a feature flag
so more projects can try them.

## Words and Reserved Words in WGSL

We'll typically want to avoid using keywords used in current WGSL, unless
the meaning is consistent and additive to the keyword's use in WGSL.

Any differences we introduce in WESL will
will be erased during transpilation to WGSL.
But we don't want to confuse our programmers by
unexpectedly changing the meaning of a keyword.

It's not as clear whether we should avoid using all the keywords
[reserved by WGSL](https://www.w3.org/TR/WGSL/#reserved-words)
for future use.
Some WESL enhancements may serve as prototypes for future
WGSL features, and it'll be easier for programmers if the syntax
stays the same.

In any case, we'll work with the W3C team
to be supportive of future WGSL evolution.
