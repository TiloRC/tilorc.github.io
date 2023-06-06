---
layout: post
title: 'GSoC Week 1: Improving SymPy''s Assumptions With Z3'
date: 2023-06-05 22:30 -0700
---

Hello!

Welcome to the first weekly update on my Google Summer of Code project with SymPy. SymPy is an open-source Python library for symbolic computation: think WolframAlpha but a Python library. In this post, I'll provide some background on my project, which focuses on improving SymPy's assumption system.

The assumption system is a crucial component of SymPy, responsible for managing information about variables such as whether they're real numbers, negative, or even. Because of the assumption system, SymPy is able to deduce a lot of important information. For example, SymPy knows that an integer times 2 plus 1 is odd:

```python
from sympy import Symbol
x = Symbol("x",integer="True")
(x*2+1).is_odd # returns True
```

SymPy is in the process of switching to a new assumption system—creatively named the “New Assumptions.” One of the reasons for this change is that the old system’s syntax has no way of expressing relational assumptions (e.g. there’s no way to say ```x>y``` or even ```x=y```). 

While objects representing relational assumptions already exist in the New Assumptions, the current system is limited in its ability to work with them. It doesn’t even know that ```x>0``` is the same thing as ```Q.positive(x)```. This is where I come in. I’ll be implementing this functionality as part of Google Summer of Code.

Keeping track of this information and making deductions based off of it is not as simple as you might expect. Currently, the New Assumptions represent propositions about variables as Boolean variables and Boolean formulas are used to encode information between propositions. Making deductions involves a [SAT solver](https://en.wikipedia.org/wiki/SAT_solver), an algorithm used to determine whether a Boolean formula has any solutions.

However, relational propositions can not be represented as Boolean variables. (At least not in any reasonable way). Thus, we need a [SMT solver](https://en.wikipedia.org/wiki/Satisfiability_modulo_theories), a generalization of a SAT solver that can check if a formula involving integers and real numbers has any solutions (among other things). 

Building an SMT solver from scratch is not an easy task, so instead I will use Z3, a state of the art SMT solver developed by Microsoft. Z3 will serve as an optional dependency for SymPy, leaving room for future attempts at developing a solver from scratch if time permits.
