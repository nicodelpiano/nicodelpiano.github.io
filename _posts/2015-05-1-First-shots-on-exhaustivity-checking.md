---
layout: post
title: "First shots on Exhaustivity Checking"
description: "This is a review of what I've done on my first week working on GSOC"
category: ""
tags: [PureScript, Haskell, Exhaustivity, Checking]
---
{% include JB/setup %}
### GSOC: Week 1
Well, the past week was my first week working on the GSOC project. Although I have started a bit late, I think we obtained interesting results.

This post is about what I've been doing and understanding when trying to apply the exhaustivity checking problem to a simple typed lambda calculus + Nats language.

So, let's begin.

#### Exhaustivity Checking

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

There are a few approachs for solving this problem. The one we are following is explained in the following paper [1] (GHC is implementing it now, but I guess it's not in the official version) although we are trying to **generalise** the idea to solve the problem for any datatype.

#### 
