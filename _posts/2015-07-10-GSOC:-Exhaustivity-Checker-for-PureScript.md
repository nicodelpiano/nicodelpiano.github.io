---
layout: post
title: "GSOC: Exhaustivity Checker for PureScript"
description: "PureScript now has exhaustivity checking!"
category: ""
tags: [PureScript, Haskell, Exhaustivity, Checking, Google, GSOC]
---
{% include JB/setup %}
#### Exhaustivity Checker for PureScript

**PureScript now has exhaustivity checking as a new warning type!**

This blog post is a brief description of what we have done in the first part of my GSOC. This covers the implementation of the exhaustivity checker for a great language called PureScript (http://purescript.org).

### The issue

An important issue while writing compilers is to alert the programmer of possible failures he could get if he executes the code he is trying to compile. In other words, avoid potential program crashes and ensure safety.
Many functional languages support pattern matching definition of functions over data constructors, literals and other possible binders. PureScript, like Haskell and ML, gives support for pattern matching functions.
So, what is the problem with that? The problem comes when we write partial definitions, such as:

```
module Main where

data List a = Empty | Cons a (List a)

head :: forall a . List a -> a
head (Cons x _) = x
```

`head` just gives the first element of a *list*. It is a widely known function that brings us a question: what happens when we evaluate `head` with `Empty`? What do you think? Yes, this crashes our program.

Let us see how this error looks like in *Haskell*, *ML* and *PureScript*:

**Haskell**

```
ghci> head []
*** Exception: Prelude.head: empty list
```

**ML**

```
- List.hd [];
! Uncaught exception: 
! Empty
```

**PureScript**

```
Failed pattern match at test.purs line 5, column 1 - line 6, column 1: Empty
```

This behaviour is undesirable at run-time! It could lead to unexpected crashes.

So, we get with an important problem: *raise warnings when a function definition has missing patterns.*

A desirable compiler behaviour for the above definition is to warn the programmer that he is missing a case:

```
Warning:
  The definition of 'head' has the following uncovered cases:

  Empty
```

I think this is a simple but good introduction to the problem. You can read my previous post about exhaustivity checking if you like to read a deeper introduction.

### Desugaring PureScript patterns

The first step I have to deal with when figuring out how to apply the solution to PureScript code, is how does PS represent pattern matching functions.

What PureScript does is desugaring all top-level binders (in particular pattern matching definitions) into case expressions, with this structure:

```
You can find this code is in src/Language/PureScript/AST/Declarations.hs

data Expr =
  ...

-- |
-- A case expression. During the case expansion phase of desugaring, top-level binders will get
-- desugared into case expressions, hence the need for guards and multiple binders per branch here.
--
  | Case [Expr] [CaseAlternative]

  ...

-- |
-- An alternative in a case statement
--
data CaseAlternative = CaseAlternative
  { -- |
    -- A collection of binders with which to match the inputs
    --
    caseAlternativeBinders :: [Binder]
    -- |
    -- The result expression or a collect of guarded expressions
    --
  , caseAlternativeResult :: Either [(Guard, Expr)] Expr
  }


```

So, roughly speaking, `head`'s definition will be desugared as:

```
Case [TypedValue True (Var _0) (TypeApp (TypeConstructor Main.List) (TUnknown 5))] [CaseAlternative {caseAlternativeBinders = [(ConstructorBinder Main.Cons [VarBinder x,NullBinder])], caseAlternativeResult = Right (Var x)}]
```

Hence, we can limit our work to just this expressions: we know that all the information we want (up to now) is wrapped inside a `Case` expression.

### Understanding PureScript binders

Each argument is represented by a `Binder`. As an example, consider the following datatype:

```
data Nat = Succ Nat | Zero
```

data constructors for `Nat` will be represented as `Binders` like this:

- `Zero` as `ConstructorBinder () []`
- `Succ Zero` as `ConstructorBinder () [ConstructorBinder () []]`

Binders for constructors are represented with the following constructor:

```
data Binder =
  ...

  ConstructorBinder (Qualified ProperName) [Binder]
  
  ...
```

where `Qualified ProperName` stands for the name of the current constructor, and the list of binders is not empty if the constructor has arguments, like `Succ` for example.

You can read more about the `Binder` ADT in the file `src/Language/PureScript/AST/Binders.hs`.

### The algorithm we follow

In this section I would like to explain our approach in a very intuitive way:

```
f p11 p12 ... p1n = e1  -> Clause 1

...

f pm1 pm2 ... pmn = em  -> Clause m
```

Our algorithm analyses the clauses of a definition one by one from top to bottom, where in each step we have the cases already missing (uncovered), and we generate the new set of missing cases.

We start the approach with `uncovered = { _ }` (one `_` for each argument, `_` stands for a representation of all values), and feeding each clause with this set of uncovered cases we obtain the new set of missing cases. We also describe uncovered cases by a general representation of those values, for example, non-empty lists will be represented as `Cons _ _`.

Example:

let us consider the following datatype representing natural numbers, with a function that adds two naturals:

```
data Nat = Zero | Succ Nat

add :: Nat -> Nat -> Nat
add Zero m = m
add (Succ n) Zero = Succ n
```

Clearly, `add` is not exhaustive. How do the algorithm realize that?

As we have two arguments, we start with `uncovered = { [_ , _] }` (that is, a list of lists representation for now) and take the first clause of the definition: `add Zero m = m`. We are only interested in the arguments, so that clause will be represented as `[Zero, m]` (with binders would be something alike: `[ConstructorBinder "Zero" [], VarBinder "m"]`).

After that, we start partitioning `uncovered` as follows:

- `Zero` matches against the first `_` (since `uncovered` is a list) and it gives us the new case missing: `[Succ _]` (it is in a list because we might generate more than one missing case).

- `m` matches against the second `_` and, as `VarBinder`s represent all values as wildcards, we have no missing cases: `[]`.

What do we do with `[Succ _]` and `[]`? Taking the `uncovered` set, we complete the missing patterns as follows:

- As we have got `Succ _` over the first argument, we do not match anything "at the right" of that pattern, so we complete the first missing case with: `[Succ _, _]` (we take `_` from the second element of the single uncovered case).

- As we have got `[]` over the second argument, we do nothing by now.

So, after processing the first clause we get: `uncovered = {[Succ _ , _]}`.

Processing the second clause will be as follows:

- From the first argument of the second clause, `Succ n`, we get after matching it to `Zero` the empty set, since `Succ n` stands for all naturals greater than `0`, so the missing set is `[Zero]`. 

- We match the second argument, `Zero`, to `_` and we obtain the missing case: `[Succ _]`.

As we do not have empty missing sets, how do we complete the full set of uncovered cases? That is, we have to complete gaps: `[Zero, gap]` and `[gap, Succ _]`.

We have two choices to fill those gaps:

**From the current uncovered set**

This is what we are using, and it is a great solution if we want to let the programmer choose the uncovered structure he missed. This could repeat cases, but uncovered at least.

If we take this approach, we will have the missing cases: `{[Zero, _], [Succ _, Succ _]}`

**From the clause**

This tells the programmer all the missing cases he left uncovered, but respecting the order of the clauses. This is a better solution but requires a bit more of work. We hopefully develop this version after redundancy checking.

If we take this approach, we will have the missing cases: `{[Zero, Zero], [n, Succ _]}`

Hence, depending on what we use to fill those gaps, we will obtain different cases, though uncovered, which is the important issue here.

You can read the full code (in particular, this last step is written in `missingCasesMultiple`) in [Exhaustive.hs](https://github.com/purescript/purescript/blob/master/src/Language/PureScript/Linter/Exhaustive.hs).

### Exhaustivity for data constructors

For data constructors, we just take each name of each constructor in the datatype, and create the missing cases. Given one constructor from some data declaration, we can know using the environment which are the other constructors.

For instance, if we have:

```
data Tree a = Empty | Leaf a | Branch (Tree a) (Tree a)
```

having `Empty` in some clause, we can look up the environment and know that the other constructors are `Leaf` with arity one, and `Branch` with arity two. This help us to generate the missing cases, just deleting the constructor we are covering:

```
isEmpty :: forall a . Tree a -> Boolean
isEmpty Empty = True
isEmpty _ = False
```

the first clause will generate the uncovered set `{[Leaf _], [Branch _ _]}`.

### Exhaustivity for records (objects)

Having:

```
g {bar : Zero} = Zero
g {foo : Zero, bar : _} = Zero
```

we obtain:

```
Warning in value declaration g:
Warning at /dev/stdin line 3, column 1 - line 4, column 1:
  Pattern could not be determined to cover all cases.
  The definition has the following uncovered cases:

    { bar: Main.Succ _, foo: Main.Succ _ }
```

What we did here is to sort the names of the fields and check for exhaustivity, having in mind that could have records which lack some fields. So first we complete those missing fields: a missing field is a wildcard, so it is fully covered when it is not present (for that specific clause). Once we have a full record with all the information we need (all the fields present in the definition), we check for exhaustiveness.

### Exhaustivity for guards

Guards are the top problem of exhaustivity checking. Consider this function:

```
abs :: Number -> Number
abs x | x < 0 = -x
      | x >= 0 = x
```

How do we check if those guards are exhaustive or not? This depends on knowledge of the properties of `<` and `>=`: *"in general, the exhaustiveness for pattern matches involving guards is clearly undecidable; for example, it could depend on a deep theorem of arithmetic."* [Gadts meet their match](http://research.microsoft.com/en-us/um/people/simonpj/papers/pattern-matching/gadtpm.pdf).

For that reason, we say a definition with guards is exhaustive if and only if it has an `otherwise` or `true` guard case. 

### Exhaustivity for other binders

Booleans and literals are also checked for exhaustiveness, but we do something like we did for guards for the latter.

Literals are non-exhaustive by default:

```
f :: Number -> Number
f 0 = 0
```

It is non-exhaustive and the compiler will warn with `{ _ }` missing case. If we add the case:

```
f :: Number -> Maybe Number
f 0 = Just 0
f _ = Nothing
``` 

we are fully covered. The same goes for all literals except for booleans.

### Binders we left aside

Arrays are not checked for exhaustivity. The reason is that we might have to add another binder representing them as lists, which is a bit dirty in my opinion. So, we are in favour of leave them for now.

### Conclusion

To sum up, I guess we have a neat and useful exhaustivity checker. It covers most of the cases a programmer could get, so in my opinion a PureScript user can be sure that he will not have a run-time error caused by non-exhaustive patterns without an alert from the compiler, which it is not a small thing :).

This project was very interesting. I learned a lot of the PureScript's code and I will keep contributing to it!

**Next objective:** get rid of overlapping patterns with redundancy checking!

If something was not clear enough, do not hesitate to leave me a comment! Suggestions, ideas and critics are very welcome.

Thanks for reading!

