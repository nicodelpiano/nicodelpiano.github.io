---
layout: post
title: "GSOC: Redundancy Checker for PureScript"
description: "PureScript now has redundancy checking!"
category: ""
tags: [PureScript, Haskell, Redundancy, Checking, Overlapping, Google, GSOC]
---
{% include JB/setup %}
#### Redundancy Checker for PureScript

**PureScript now has overlapping patterns as a new warning type!**

After finishing with exhaustiveness checking, we decided to tackle redundancy checking, that is, getting rid of overlapping patterns in pattern matching definitions (or at least, warn about them).

In order to get that, we worked on top of what we had done for exhaustivity. So, making a few changes, we managed to obtain an efficient redundancy checker that works very well!

### Example

We'd like to warn about these:

```
data Nat = Zero | Succ Nat

f Zero Zero = Zero
f _ (Succ _) = Zero
f Zero (Succ Zero) = Zero
```

here, we've got the third case as a redundant case (or overlapping case, call it as you like), since it's already covered by the second. Hence, if I call `f` with `Zero` `Succ Zero` as the arguments, it will match the second clause of the definition, and I'll never match the third clause. So, it can (and should) be removed.

### The basic algorithm

The idea is, given the uncovered cases from the `(i-1)`-th clause and the current arguments, to check whether or not these new arguments cover something new, i.e., they partition or removes at least one uncovered case. Therefore, for each clause we have two possibilities: redundant or non-redundant (covers something or not).

Let's go step by step, taking the previous definition of `f`:

First clause:

```
f Zero Zero = Zero
```

`Uncovered = {[_, _]}` (from the `0-th` clause)

Does `Zero Zero` cover something? Yeah, as it partitioned `{[_, _]}` to `{[Succ _, _], [_, Succ _]}`.

So, the analysis of the first clause throw us the following information:

- `Uncovered = {[Succ _, _], [_, Succ _]}`
- `Covers = True`

Second clause:

```
f _ (Succ _) = Zero
```

`Uncovered = {[Succ _, _], [_, Succ _]}` (from the `1-th` clause)

Does `_ (Succ _)` cover something? Yes, since it partitions `Uncovered` to `{[Succ _, Zero]}`.

Then, we end up with:

- `Uncovered = {[Succ _, Zero]}`
- `Covers = True`

Third clause:

```
f Zero (Succ Zero) = Zero
```

Starting with `Uncovered = {[Succ _, Zero]}`, the third clause doesn't partition it since it doesn't match any case of `Uncovered`. It means that the case `Zero (Succ Zero)` is already covered by a previous clause. So it's redundant!

### The approach

We extended the exhaustivity checker to carry information about redundancy this way:

```
missingCasesSingle :: Binder -> Binder -> ([Binder], Maybe Bool)
```

The additional ```Maybe Bool``` has the following meaning:

- `Nothing`: used for those binders who don't have exhaustivity checking yet.
- `Just False`: means that the case is redundant (it covers nothing).
- `Just True`: means that the case covers something and it's not redundant.

We opt this representation as it allowed us to work with the `Applicative` machinery for free.

### Warnings

Finally, the program given above will throw this warning:

```
  Warning in module Main:
  Warning in value declaration f:
  Warning at /dev/stdin line 4, column 1 - line 5, column 1:
      Redundant cases have been detected.
      The definition has the following redundant cases:

      Main.Zero (Main.Succ Main.Zero)
```

and that's what we expected to see :).

### Conclusion

All in all, we have implemented a nice exhaustivity-redundancy checker for PureScript. It throws neat warnings and has an acceptable time complexity. There are a few possible extensions that might be useful to have in a future, like integer range analysis, literals and array support, a more deeper analysis on guards, etc. I'm pretty sure this will be a helpful tool for PureScript users!

I'd like to thank Phil Freeman, for his availability and guidance during the course of the project.

If something was not clear enough, don't hesitate to leave me a comment! Suggestions, ideas and critics are very welcome.

Thanks for reading!
 
