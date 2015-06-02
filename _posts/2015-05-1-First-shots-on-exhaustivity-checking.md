---
layout: post
title: "First shots on Exhaustivity Checking"
description: "This is a review of what I've done on my first week working on GSOC"
category: ""
tags: [PureScript, Haskell, Exhaustivity, Checking]
---
{% include JB/setup %}
### GSOC: Week 1
Well, the past week was my first week working on the GSOC project. Although I've started a bit late, I think we obtained interesting results.

This post is about what I've been doing and understanding when trying to apply the exhaustivity checking problem to a simple typed lambda calculus + Nats language.

So, let's begin.

#### Exhaustivity Checking

(This is *better* explained in my previous post)

When writing pattern matching definitions, it's important to cover the whole set of type's constructors. For instance, if we are working with lists:

```
data List a = Nil | Cons a
```

we should have equations that matches both empty case (Nil) and non-empty case (Cons). If we fail to cover all the cases we might have an error at run-time.

As an example, see that

```
head :: forall a. List a -> a
head (Cons x _) = x 
```

is not exhaustive, since the case `Nil` is missing.

#### How do we tackle this issue?

There are a few approachs for solving this problem. The one we are following is explained in this great paper [1] (research.microsoft.com/en-us/um/people/simonpj/papers/pattern-matching/gadtpm.pdf) (GHC is implementing this solution now, but I guess it's not in the official version) although we are trying to **generalise** the idea and provide a library that could be used to solve the problem for any datatype.

#### STDL + Nats

To begin with, we define a small language to work with Nats and wildcards: the latter represent all values.

```
-- | Name binding association
data Binder = NullBinder       -- Wildcard binder
            | Zero             -- Zero binder
            | Succ Binder      -- Nat binder
  deriving (Show, Eq)

```

The first attempt I did was to represent clauses as lists of binders, and full definitions as a list of clauses. Though I solved the problem for the `Binder` datatype and that representation, it wasn't a clever solution.

Therefore, Phil suggested in using this more clever representation:

```
class Match c where
  full :: c
  missed :: c -> c -> [c]

```

So, we can instantiate `Match` for the `Binder` datatype and define `missed` as we did previously.

And this is what we get:

```
instance Match Binder where
  missed _ NullBinder = []

  missed NullBinder b = go b
    where
    go Zero = [Succ NullBinder]
    go (Succ n) = Zero : map Succ (go n)
    go _ = []

  missed Zero Zero = []
  missed (Succ n) (Succ m) = map Succ (missed n m)

  missed b _ = [b]
```

What should `missed` do? Well, it takes two elements of type `c` and returns a list of all uncovered cases that were not matched between the elements. The first argument is the *case to match*, and the second is the *case we have*.
For example, in the case of `Binder`, we have three constructors. So if we try to match `NullBinder` (which represents all cases) with `Zero`, we have to split `NullBinder` into two cases: `Zero` and `Succ NullBinder`. Since we have `Zero`, the only case we don't match is `Succ NullBinder`. Then, `missed NullBinder Zero` returns `[Succ NullBinder]`.

You can see a more complex example:

```
*Types> missed NullBinder (Succ $ Succ Zero)
[Zero,Succ Zero,Succ (Succ (Succ NullBinder))]
```

#### Tidying up things a bit more




#### Plans for next week

What I am going to do this week:
  * Support other kind of patterns
  * Make this a suitable Cabal Project
  * Analyse how to extend for `(,)`, `Either`, and so on.
  * Write more test cases and properties for Vec (choose which is the correct implementation)
