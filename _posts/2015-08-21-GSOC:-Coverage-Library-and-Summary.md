---
layout: post
title: "GSOC: Coverage library and summary"
description: "This post is about the final work I did in the last part of my GSoC,
and a brief conclusion of the whole project."
category: ""
tags: [PureScript, Haskell, Coverage, Library, Google, GSOC]
---
{% include JB/setup %}

### Coverage: an exhaustivity/redundancy checking library

To start with, I just wanted to develop a library (it was one of the
first thoughts of my project) that will perform, at least, a simple
but useful approach of exhaustivity checking over any Haskell-like
language. That is, a language with pattern matching definitions
including data constructors, booleans, records, guards, and probably
literals (this is a future goal for my
[library](http://github.com/nicodelpiano/coverage)).

The intended usage of the library is as follows:

- First step: translate the language binder representation to the one
I use to do the checking.

- Second step: collect the results of the checking, and translate them
back to the language.

Although these steps require a little effort of the user, I reckon it
won't be more than two functions that make the translation between
both representations.

The main data-type I use for the checking is:

```
data Binder lit
```

`lit` is the type of the object literal wrapped inside a binder. For
example, we might have a list of ints like this: `Tagged "Cons" (Lit
1) (Tagged "Empty" (Var Nothing))` which could represent the list
`[1]`.

Listed below are the constructors for the `Binder` data-type:

- `Var (Maybe Name)`: this embodies a variable

- `Lit lit`: just a literal object

- `Tagged Name (Binder lit)`: this stands for constructors

- `Product (Binders lit)`: tuples (`Binders` are lists of `Binder`)

- `Record [(Name, Binder lit)]`: records
```

As you may guess, this is a very simple representation, but general
enough to represent language declarations.

Next, I took the previous work I had done with the PureScript
compiler, and just port it to these new types, providing exhaustivity
and redundancy checking. It fitted perfectly!

### Case definitions

How do we present pattern-matching definitions? This is very straightforward:

```
type Alternative lit = (Binders lit, Maybe Guard)
```

a case alternative consists of a collection of binders which match a
collection of values, and an optional guard. Example: `([Zero, Zero],
Nothing)` stands for the clause `f Zero Zero = ...`. A list of
alternatives embodies a full definition.

You can see the full code at: [coverage
repo](https://github.com/nicodelpiano/coverage)

### An example of usage

Let's define a simple language for `Nat`s:

```
data NatBinder = AnyBinder        -- Wildcard binder
               | Zero             -- Zero binder
               | Succ NatBinder   -- Succ binder
```

we can use the checker, just providing an environment for our type and
a translation function:

```
natToBinder :: NatBinder -> Binder NatBinder
natToBinder NullBinder = Var Nothing
natToBinder Zero = Tagged "Zero" $ Var Nothing
natToBinder (Succ nb) = Tagged "Succ" $ natToBinder nb
```

```
env :: String -> Maybe [String]
env "Zero" = Just $
  ["Zero", "Succ"]
env "Succ" = Just $
  ["Zero", "Succ"]
env _ = error "The given name is not a valid constructor."
```

The environment provided helps the checker to find the right names for
our constructors; otherwise it won't be able to tell us which ones of
them are missing.

After that, we can use the `check` function like this:

```
check (makeEnv env) toBinder
```

where `toBinder = map (\(nbs, g) -> (map natToBinder nbs, g)) def`
just maps the translation to our pattern match definition of type
`[([NatBinder], Maybe Guard)]`.

You can see this full example in [coverage
examples](https://github.com/nicodelpiano/coverage/examples).

### GSoC summary

Alright, GSoC came to an end. The way we finally faced the project
diverged a bit from the proposal, but the results were the expected.
I'm very pleased to say that Google Summer of Code 2015 has been an
amazing experience for me. I learned a lot of software development,
code maintenance and testing working with
[PureScript](http://purescript.org) and [Haskell](http://haskell.org)
stuff! It was fantastic to get involved in the open source world in
this manner and I'll keep contributing to the PureScript's community
as well!

Finally, I'd like to thank to the people who guided me throughout this
process. I'm indebted to Phil Freeman, who gave me great advices on
how to face this project and three months of daily dedication
answering my questions and providing me feedback. To John De Goes and
Gary Burgess, who also helped me understanding the compiler's code and
reviewed my contributions. It was outstanding to be reviewed by such
skilled developers!

I want to wrap this up by saying that I enjoyed this program so much
I'd recommend it to any student who wants to contribute to the open
source community.

Thanks for reading!
