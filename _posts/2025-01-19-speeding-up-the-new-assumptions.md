**THIS IS A DRAFT OF AN UPCOMING BLOG POST. IT'S ALMOST FINISHED BUT STILL INCOMPLETE.**

*If you’re not familiar with what SymPy’s assumptions are, read about the old assumptions and new assumptions [here](https://docs.sympy.org/latest/guides/assumptions.html) and [here](https://docs.sympy.org/latest/modules/assumptions/index.html) respectively.*

Despite the many advantages of the new assumptions, the old assumptions have yet to be replaced. The reason? The new assumptions are significantly slower, even after various efforts to optimize their performance. For example, the new assumptions take almost 3000 times longer to complete the following query:

```python
>>> import timeit
>>> from sympy.abc import x
>>> from sympy import ask, Q
>>> print(timeit.timeit(lambda: x.is_integer, number=10000))
0.0026039159999982076
>>> print(timeit.timeit(lambda: ask(Q.integer(x)), number=10000))
7.075687000000016
```

I believe the slowness is a consequence of fundamental design flaws. At the heart of the problem is `satask`, the core function of the new assumptions which solves queries by converting them into boolean satisfiability problems. Exacerbating the problem are various heuristic functions, which are called before `satask` in an attempt to efficiently handle simple queries. However, these heuristics increase code complexity and can sometimes significantly degrade performance by making recursive calls that trigger multiple invocations of `satask`.
  
In this blog post, I will outline how `satask` can be redesigned for improved performance. First, I’ll explain its current functionality and why I believe its design has fundamental flaws.
## A simple example

In broad strokes, `satask` answers queries by encoding them as [boolean satisfiability problems](https://en.wikipedia.org/wiki/Boolean_satisfiability_problem) and checking their satisfiability using a SAT solver. To gain a deeper understanding of how this works, a simple example can be helpful.

Suppose you have some variable `x`  that is known to be either prime or positive and you want to determine if it could be negative. To determine this, you could call `ask(Q.negative(x), Q.prime(x) | Q.positive)`. As this is a simple query, `ask` will be able to determine that it is false without relying on `satask`. However, for the sake of this explanation, let’s assume `satask` is called to resolve the query.

`satask` will construct two boolean formulas: one that’s satisfiable if the query can be true, and another that’s satisfiable if the query can be false. For example:
```plaintext
#  Satisfiable if the query can be true.
F1 = Q.negative(x) & (Q.prime(x) | Q.positive(x)) & rules

# Satisfiable if the query can be false.
F2 = ~Q.negative(x) & (Q.prime(x) | Q.positive(x)) & rules
```

| F1 Satisfiable | F2 Satisfiable | Result of `satask`                   |     |
| -------------- | -------------- | ------------------------------------ | --- |
| Yes            | Yes            | None (both are satisfiable)          |     |
| Yes            | No             | True (proposition is true)           |     |
| No             | Yes            | False (proposition is false)         |     |
| No             | No             | Exception (inconsistent assumptions) |     |

Here, "rules" represents a large formula encoding many of SymPy’s predicate rules. [One]([https://github.com/sympy/sympy/blob/97e74c1a97d0cf08ef63be24921f5c9e620d3e68/sympy/assumptions/facts.py#L87-L88](https://github.com/sympy/sympy/blob/97e74c1a97d0cf08ef63be24921f5c9e620d3e68/sympy/assumptions/facts.py#L87-L88)) of the rules encoded by this formula states that expressions cannot be both positive and negative. [Another]([https://github.com/sympy/sympy/blob/97e74c1a97d0cf08ef63be24921f5c9e620d3e68/sympy/assumptions/facts.py#L105](https://github.com/sympy/sympy/blob/97e74c1a97d0cf08ef63be24921f5c9e620d3e68/sympy/assumptions/facts.py#L105)) states that prime expressions must be positive. As a consequence of these two rules, F1 is unsatisfiable and F2 is satisfiable. Thus `satask` determines the query `Q.negative(x)` is false.
## What’s so bad about `satask`?

The primary issue with `satask` lies in the size of the rules formula it generates. These formulas cannot simply be fixed for every query; their size grows with the number of expressions and the complexity of each expression involved in a query. 

| Query                              | Number of clauses in rules formula |     |
| ---------------------------------- | ---------------------------------- | --- |
| `satask(Q.real(x))`                | 47                                 |     |
| `satask(Q.real(x) & Q.real(y))`    | 94                                 |     |
| `satask(Q.positive(x+y))`          | 183                                |     |
Simple queries should not require solving such large SAT problems. Only the rules relevant to a particular query should be involved, and it shouldn’t be necessary to repeat the same rules for each variable. Furthermore, it should also be possible to solve simple queries in constant time with respect to the number of rules by using a dictionary (as the [`_ask_single_fact`](https://github.com/sympy/sympy/blob/97e74c1a97d0cf08ef63be24921f5c9e620d3e68/sympy/assumptions/ask.py#L518) function accomplishes). 
## My solution 

Rather than creating large formulas that encode SymPy’s rules, a helper that understands these rules can be added to the SAT solver. The helper will add formulas representing some of the rules as needed during the SAT solving process. 

We’ll still construct two boolean formula as part of `satask`, but these won’t come with any rules. For example:
```plaintext
F1 = Q.negative(x) & (Q.prime(x) | Q.positive(x))
F2 = ~Q.negative(x) & (Q.prime(x) | Q.positive(x))
```

Without the rules, the SAT solver has no idea that `Q.negative(x)` and `Q.positive(x)` can’t be true at the same time, so it may find some satisfying assignment where they’re both true. For example, it could find the following satisfying assignment when checking F1:
```plaintext
Q.negative(x): True

Q.prime(x): False

Q.positive(x): True 
```

To check if the assignment follows SymPy’s rules, the SAT solver passes it to the helper. The helper is not only able to determine that the given assignment is contradictory, but it’s also able to provide an explanation, a subset of the assignment that is also contradictory. In this case, the explanation might be `Q.negative(x)` and `Q.positive(x)` as these two predicates are contradictory. The helper gives this explanation back to the SAT solver in the form of a conflict clause which can be added to the original formula:
```plaintext
F1 = Q.negative(x) & (Q.prime(x) | Q.positive(x)) 
F1 = F1 & (~Q.negative(x) | ~Q.positive(x)) # add the conflict clause
```
  
This way the SAT solver will know not to try any assignment where both `Q.negative(x)` and `Q.positive(x)` are true. This process repeats until either the SAT solver comes up with an assignment that the helper accepts or the conflict clauses the helper adds cause the formula to become unsatisfiable. 

This helper is what’s called an SMT theory solver. When I did my GSoC project, I implemented a theory solver for linear real arithmetic. That’s what allows the new assumptions to now make deductions like the following:

```python
>>> from sympy import symbols, ask
>>> x, y, z = symbols(‘x y z’, real=True)
>>> ask( x > z, (x > y) & (y > z))
True
```

But how do we actually implement this theory solver for the new assumptions?  
  
I have some original ideas involving a forward chaining algorithm much like the one used in the old assumptions. Actually, the old assumptions could currently be used as a theory solver without much effort — just not a very good one. And with more effort, the old assumptions could be modified to work as an effective theory solver.

### A bad but easy to implement theory solver

Let’s see how we can use the old assumptions to generate an explanation for the same assignment from before:

Q.negative(x) = True, Q.prime(x)=False, Q.positive(x)=True

If a SymPy symbol is created with contradictory assumptions, the InconsistentAssumptions exception is raised. Knowing this, we can check whether this assignment is consistent as follows:

```

from sympy.core.facts import InconsistentAssumptions

from sympy import Symbol

args = {"negative":True, "prime":False, "positive":True}

try:

Symbol(‘x’, **args)

except InconsistentAssumptions:

print(‘unsat’)

else:

print(‘sat’)

```

From running the above we’d know that our assignment is not satisfiable and our theory solver can produce the following explanation which we can add to F1:

~Q.negative(x) | Q.prime(x) | ~Q.positive(x)

Wait but the explanation is just the negation of the entire assignment. Didn’t I say that explanations were inconsistent subsets of the assignment? Well the assignment is a subset of itself. But this explanation is worse than the one I gave before; this one only eliminates one possibility for the SAT solver whereas the other explanation eliminated the two possibilities. Eliminating two instead of just one possibility doesn’t seem like a big deal, but for more complicated queries, being able to produce a minimal assignment can eliminate very large numbers of possibilities whereas simply using the entire assignment as an exmaplanation will always only eliminate one possibility. 

  

Despite this inefficiency, a theory solver made using the old assumption system like this might actually beat out the new assumptions just because of how much faster the old assumptions are and the simplicity of SymPy’s predicate rules.[Footnote]

  

### A theory solver that’s good enough

The best theory solvers, such as the one from the paper I implemented for GSoC, create minimal explanations. An explanation is minimal if only one of its subsets, itself, is an explanation; you can’t get rid of any literal from the explanation without it becoming consistent. Minimal explanations are desirable because the smaller an explanation is, the more possibilities it eliminates. 

Unfortunately, the theory solver I came up with is not guaranteed to always give minimal explanations — or at least my initial algorithm isn’t. Despite this, I think the algorithm I have in mind should usually give minimal explanations and when it doesn’t give minimal explanations, the explanations should still be small. 

My idea is to use a use a [rule-based system](https://en.wikipedia.org/wiki/Rule-based_system), such as the rete algorithm with some modifications. Rule-based systems are made up of a rule base, a collection of if–then rules, a database of facts, and a rules engine. The rules engine repeatedly applies the rules to the facts to infer additional facts to add to the database. In our application, the facts are predicates such as Q.positive and the rules are implications like Q.positive => ~Q.negative. 

Modifications to traditional rule-based systems algorithms are needed because we need to keep track of “why” each added fact is true to be able to generate explanations for inconsistent sets of facts. Whenever we add a new literal, we’ll keep track of its “source”, a subset of the iniitial literals that implies the new literal. Here is an outline of the algorithm:

  

1. Add each initial literal to the knowledge base as a source and queue it. For each added literal, check the knowledge base for its negation. If found, a contradiction exists, and the literal and its negation can be returned as an explanation.
    
2. Apply rules iteratively to derive new literals:
    

- For each queued literal, find all the literals it implies.
    
- For each implied literal:
    

- Check if its negation exists in the knowledge base. If it does, a contradiction exists, and the source literals of the implied literal and its negation are returned as the explanation.
    
- If the implied literal is not already in the knowledge base, add it to the knowledge base and the queue. Its source is derived from the source literals of the antecedent literal.
    

3. If the queue is empty, the initial literals are consistent.
    

  

I implemented a proof of concept version of the algorithm. You can find it in [this](https://colab.research.google.com/drive/1AMYJy6FotgPM1Q6yp2F4ccwZcoE8NyGZ?usp=sharing) colab notebook. As implemented, it can’t handle rules such as “a & b -> c” which have an “and” in their LHS.

  

When actually implementing the theory solver it should be implemented as a class — perhaps named something like `SympyTheorySolver`. The [theory solver I implemented](https://github.com/sympy/sympy/blob/master/sympy/logic/algorithms/lra_theory.py) for linear real arithmetic for GSoC can be used as a reference. It should implement two key methods: `assert_lit` and `check`. The `assert_lit` function should add a literal to the current state of the theory solver. It should also be able to detect if the asserted literal would conflict with any of the other literals. See [`_ask_single_fact`](https://github.com/sympy/sympy/blob/3c33a8283cd02e2e012b57435bd6212ed915a0bc/sympy/assumptions/ask.py#L518) for how this can be done efficiently and always catch conflicts with minimal explanations of length 2. The `check` should rigerously check the consistency of all asserted literals using the algorithm I outlined above and should always catch conflicts if they exist. 

  
  

---

  

Prove completeness

  

Assume TOC queue is empty but there remain true literals that have not been found. This mean that these true literals are not implied by any of the known literals or facts. Then how could they be true?

  

Only have rule a -> b (which is equivalent to ~b -> ~a but we don’t have)

  

So if we add ~b. And we’re not able to infer ~a even if ~a must be true. 

  
  
  

  a & b -> c,   (and thus ~c -> ~a | ~b)

  

(~a -> d)

(~b -> e)

d | e -> h (and thus d -> h, e -> h)

  
  

So ~c

thus ~a | ~b

thus h

  
  

  a & b -> c 

~c -> ~a | ~b 

~a -> d 

~b -> e

d -> h

e -> h

  

~c ~h

  

…    

  

Suppose TOC a contradiction exists but was not found. This mean there exists some atom, x, such that both x and ~x are entailed, and it’s not possible to derive them through repeated applications of modus pones. 

  
  

a -> b, ~b

  

so ~a

  

~a -> c

  
  

a -> b, ~b, ~a -> c, ~c

  
  
  

a -> b, ~b, ~a -> c, ~c

  

https://users.aalto.fi/~tjunttil/2020-DP-AUT/notes-sat/resolution.html

  
  

---

  
  
  

a -> b | c

  
  

a -> bc

b -> bc

c -> bc

  

Problem: Equivalent(Q.real(x), Q.rational(x) | Q.irrational(x))

- Q.rational
    
- Implies(Q.real(x),  Q.rational(x) | Q.irrational(x))
    

  
  
  

Consider Q.real iff Q.positive | Q.zero | Q.negative

  

It’s possible Q.real & ~Q.positive or Q.real & ~Q.zero or Q.real & ~ Q.negative could entail some predicate that Q.real alone and ~Q.positive/zero/negative alone do not entail

  
  

Consider Q.real iff Q.positive | Q.zero | Q.negative

  

Q.positive | Q.zero | Q.negative -> Q.real ALL GOOD

  

Q.real -> Q.positive | Q.zero | Q.negative ????

  
  

(Q.positive | Q.zero | Q.negative) is not horn!

~Q.positive & ~Q.zero -> Q.negative 

~Q.negative & ~Q.zero -> Q.positive 

~Q.positive & ~Q.negative -> Q.zero

  

Q.real & ~Q.positive & ~Q.zero -> Q.positive  

  

Q.real & ~Q.positive & ~Q.zero -> Q.positive  

  
  
  

---

  
  

##  Conclusion 

  

The scope of this blog post has been limited to replacing the need to convert the rules defined in `assumptions/facts.py` into large boolean formulas for every variable. But the approach should be able to genearlize to replace the need for the boolean formula generated to capture rules between expressions and subexpressions defined in `assumptions/sathandlers.py`. Ideally, it should also be possible to get rid of the recursive calls to `ask` in the rules in `assumptions/handlers/`.

  

I also didn’t come up with an algorithm that was guaranteed to generate minimal conflict clauses. And I do actually have some thoughts about how minimal conflict clauses could be generated in an efficient way. 

  

One limitation of this approach I have yet to discuss is handling rules where the RHS has a disjunction of predicates — rules like “a -> b | c”. It’s not really possible for a rule-based system to do anything with these rules as far as I can tell. Of course, you can get the contrapositive rule, “~a & ~c -> ~a”, and use that. 

  

I suppose “b | c” could be a literal. And you could add rules “b -> b | c” and “c -> b | c” and their contrapositives. 

  

As a consequence, check_consistency([Q.real(x), ~Q.negative(x), ~Q.zero(x), ~Q.positive(x)]) 

  
  

Minimal conflict clauses

  
  

This is all very abstract. Lets look at an example.

  

