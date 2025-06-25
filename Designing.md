# Designing Enhancemnents for WESL

This section discusses some general design guidelines for creating new shader language enhancements.

## Goals

We're aiming to create a modestly improved, practical variant of WGSL.
WESL should feel like WGSL with a few things added.

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

**Part time shader programmers**. Our programmers will have to relearn any intricacies we introduce over and over again, so keep it simple.

**Medium size teams**. Our programmers work on projects ranging in size from tiny programs
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
If it *looks just like* a Rust or TypeScript feature, programmers will expect it
*work like* the source language.

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

Some WESL enhancements may serve as prototypes for future
WGSL features. We'll work with the W3C team
to be supportive of future WGSL evolution.
