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
Case [] []
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



### Exhaustivity for data constructors

### Exhaustivity for records (objects)

### Exhaustivity for other binders

### Binders we left aside

### Conclusion
