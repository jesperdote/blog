+++
title = "The unreasonable staying power of Hello, World"
date = 2026-08-08
+++

Every programming tutorial ever written opens the same way. Before loops, before
variables, before anyone explains what a compiler actually does, you type a program that
prints two words to the screen. It's such a fixed ritual that it's worth asking where it
came from, and what people did to sanity-check a new language or machine before it
existed.

## Where it comes from

The phrase is credited to Brian Kernighan, from an internal Bell Labs tutorial on the B
programming language in the early 1970s - B being the direct ancestor of C. The example
resurfaced a few years later in Kernighan and Ritchie's *The C Programming Language*
(1978), the book that taught most of a generation to program in C. Its opening example is
about as small as a working program can get:

```c
main()
{
    printf("hello, world\n");
}
```

That book sold in the hundreds of thousands and got photocopied and passed around
university CS departments for decades. Every reader's first working program was this one,
and when those readers went on to write tutorials of their own - for Pascal, for Perl, for
Python, for whatever came next - they reached for the same example out of habit. The
convention didn't spread because anyone decided it should; it spread because it was the
first thing an enormous number of programmers ever typed.

## Why this particular program, though

It's not just inertia. "Hello, world" earned the slot because it's close to the minimum
program that proves anything useful:

- It exercises the entire toolchain end to end - source file, compiler or interpreter,
  linker if there is one, runtime, and a working path for output - with nothing else in
  the way.
- It needs no input, no files, no network, no state. If it fails, the failure is in the
  environment, not in logic you wrote.
- The output is something a human can eyeball instantly. No expected numeric result to
  compute by hand, no test fixture - either the words show up or they don't.
- It's trivially portable to any language with a notion of "print a string," which is why
  it became the natural Rosetta Stone for comparing languages side by side.

That combination - minimal, unambiguous, and language-agnostic - is hard to beat as a
first example, which is probably why nothing has displaced it in fifty years.

## What people did before it

Before "hello, world" was the default, the first program you ran in a new environment was
usually whatever the local reference manual happened to open with, and manuals disagreed.
A lot of early language tutorials reached for arithmetic instead of text - summing two
numbers or printing a multiplication table - partly because early textbooks were written
by people teaching numerical computing, not string handling, and partly because formatted
text output wasn't something every device could be trusted to do consistently. A line
printer, a teletype, and a CRT terminal didn't all handle strings the same way, but they
could all be trusted to print a number.

On hardware with no text output at all, the equivalent gesture was even more physical.
Machines like the Altair 8800 were programmed by flipping front-panel switches, and the
"did this work" moment was a blink pattern on a row of LEDs, not a sentence. That's the
same test in spirit - the smallest possible program that proves the machine is alive and
doing what you told it - just constrained by hardware that had no concept of a character
set yet.

## Why it never got replaced

Once "hello, world" became the default first example in the most influential C book ever
written, every later language had an incentive to match it rather than invent its own
ritual - it let people compare a new language to the ones they already knew by looking at
the same five lines. That's ultimately why it outlasted the arithmetic examples and the
blinking lights that came before it: it wasn't a better test of the machine, it was a
better test for the human reading the tutorial.
