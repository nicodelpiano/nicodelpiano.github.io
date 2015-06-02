---
layout: post
title: "First shots on Exhaustivity Checking"
description: "This is a review of what I've done on my first week working on GSOC"
category: ""
tags: [PureScript, Haskell, Exhaustivity, Checking]
---
{% include JB/setup %}
### GSOC: Week 1
Well, past week was my first week working on the GSOC project. Although I've started a bit late, I think we obtained interesting results.

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

The first attempt I did was to represent clauses as lists of binders, and full definitions as lists of clauses. Though I solved the problem for the `Binder` datatype and that representation, it wasn't a clever solution.

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

This is a general pattern matching definition:

```
f x11 ... x1n
...
f xn1 ... xnn
```

The invariant that we have to assure is that the uncovered cases that we generate should have n-arity, in order to be valid cases. For that reason, we define a `Vec` GADT:

```
data Nat = Z | S Nat

data Vec n a where
  Nil :: Vec Z a
  Cons :: a -> Vec n a -> Vec (S n) a
```

then, if we define an `Match` instance for `Vec`

```
instance (Match c) => Match (Vec n c)
```

we have proven that the generated list contains uncovered cases that match the length of the arguments given! This is awesome!
(Thanks Phil!)

#### Match instance for Vec and some QuickChecking

This is the instance I defined:

```
instance (Match c) => Match (Vec n c) where
  missed Nil _ = []
  missed (Cons x xs) (Cons y ys) | null miss = map (Cons x) (missed xs ys)
                                 | otherwise = map (`Cons` xs) miss ++ map (Cons y) (missed xs ys)
    where
    miss = missed x y
```

Basically, the idea is to take each element of the second argument and `miss` them with each corresponding element of the first argument, obtaining the list of uncovered elements of type `c`.

Once we have that, two cases show up:

1. If there aren't uncovered elements: the element is full matched, so it's only needed to analyse the patterns on the right. 

2. If there are uncovered elements: for each uncovered element `e`, we add the case `(Cons e ucs)`, being `ucs` the tail of the uncovered cases from the first argument, and appending these with the recursive call. But, the trick here is to add on the head of each case obtained from the recursive call, the element on the head of the second argument.

As an example, consider this definition

```
crazy Zero (Succ NullBinder) (Succ $ Succ Zero) = Zero
```

We start with nothing uncovered, that is, `Cons NullBinder (Cons NullBinder (Cons NullBinder Nil))`. Let's analyse what happens in each case separately:

1. `miss NullBinder Zero = [Succ NullBinder]`

2. `miss NullBinder (Succ NullBinder) = [Zero]`

3. `miss NullBinder (Succ $ Succ Zero) = [Zero, Succ Zero, Succ $ Succ $ Succ NullBinder]`

So, first uncovered case for the first position in the pattern is `Succ NullBinder`. As this is not matched yet, we have to add the uncovered case `Cons (Succ NullBinder) (Cons NullBinder (Cons NullBinder Nil))`. The middle unmatched element on the clause is `Zero`, and check what patterns are missing for this case: `Cons Zero (Cons Zero (Cons NullBinder Nil))` and `Cons (Succ NullBinder) (Cons Zero (Cons NullBinder Nil))`. The last case has a bigger uncovered set, so, again, for each uncovered case we generate: `Cons Zero (Cons (Succ NullBinder) (Cons Zero Nil))`, `Cons Zero (Cons (Succ NullBinder) (Cons (Succ Zero) Nil))` and `Cons Zero (Cons (Succ NullBinder) (Cons (Succ $ Succ $ Succ NullBinder) Nil))`

All in all, the algorithm I follow could be viewed as:

```
At step i:

If u(xi) is non-empty: (u(xi) is `missed` between `xi` an its corresponding element on the uncovered case)

Vector x: our clause (x1 ... xn) (this is the second argument on the function missed)
Vector y: an uncovered case (y1 ... yn)

x1 ...x(i-1) xi x(i+1) ... xn
 \___________/
 bind (fixed)

For each element in u(y(i+1) ... yn) = [(y(i+1)_1 ... y(i+1)_m), ..., (yn_1, ..., yn_m)]
we create all possible permutations:

x1 ... x(i-1) xi y(i+1)_1 ... yn_1
x1 ... x(i-1) xi y(i+1)_2 ... yn_2
 ...
x1 ... x(i-1) xi y(i+1)_m ... yn_m

And also, for u(xi), we add t

m1_1 ... uxi_1 y(i+1) ... yn
...
m1_k ... uxi_k y(i+1) ... yn

If u(xi) is empty:

m1_1 ... xi y(i+1) ... yn
...
m1_k ... xi y(i+1) ... yn

```

If something wasn't clear enough, don't hesitate to leave me a comment! Anyway, the code is at [Code] (https://github.com/nicodelpiano/stlcnat-exhaustivity-checker)

Suggestions, ideas and critics are very welcome!

Thanks!

#### Plans for the upcoming week

What I am going to do this week:

  * Support other kind of patterns

  * Make this a suitable Cabal Project

  * Analyse how to extend for `(,)`, `Either`, and so on.

  * Write more test cases and properties for Vec (choose which is the correct implementation)
