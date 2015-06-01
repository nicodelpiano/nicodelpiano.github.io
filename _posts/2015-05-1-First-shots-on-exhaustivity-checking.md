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
```haskell
f :: [a] -> [a]
```
we should have equations that matches both empty case ([]) and non-empty case (:). 
