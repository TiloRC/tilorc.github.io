---
layout: post
title: What are SAT and SMT solvers?
date: 2023-07-05 16:04 -0700
math: true
---

I’ll be walking through some examples of problems SAT and SMT solvers solve. Then I’ll be using python wrappers for PicoSAT and Z3 to solve the example problems. If you’re familiar with python, you should be able to follow along and run my code without trouble.

# What’s a SAT solver?

Let $b_1$, $b_2$, $b_3$ be boolean variables. Then is the following boolean formula satisfiable?

Note: $\mid$ is 'or',
$\&$ is 'and',
and $\neg$ is 'not'.




$$(b_1  \mid b_3)
\quad \& \quad
(b_2 \mid b_3)
\quad \& \quad
\neg b_1
\quad \& \quad
 \neg b_2$$

A boolean formula is satisfiable if there exists an assignment of its variables that causes the whole formula to be True. You’ll notice that if $b_1=False$, $b_2=False$, and $b_3=True$, then the above formula is True which means it is satisfiable.

Now what about this formula?

$$(b_1  \mid b_3)
\quad \& \quad
(b_2 \mid b_3)
\quad \& \quad
\neg b_1
\quad \& \quad
 \neg b_2
 \quad \& \quad
 \neg b_3
 $$

From  $\neg b1 \& \neg b2 \& \neg b3$, we can surmise that if $b_1$, $b_2$, or $b_3$ is True, then the whole formula is False. Thus all three must be False. However, if all three are False then
$(b_1  \mid b_3)
\; \& \;
(b_2 \mid b_3)$
is False which would make the whole formula False. This means that this new formula is not satisfiable. 

SAT solvers are algorithms that solve the boolean satisfiability problem. That is, given a boolean formula like the ones we’ve seen, a SAT solver checks if the formula is satisfiable. 

Not only do SAT solvers have a lot of practical applications such as software verification, they are also very interesting from a theoretical computer science perspective. As SAT was the first problem to be proven NP-complete, proofs that other problems are NP-complete depend on the proof that SAT is NP-complete. (If you can [reduce](https://en.wikipedia.org/wiki/Reduction_(complexity)) SAT to some other problem, you can prove that that other problem is also NP-complete). 

Now for some examples using pycosat. Let’s try using pycosat to solve the formulas from our previous examples. First we’ll need to convert them into a format that pycosat understands.

$$(b_1  \mid b_3)
\quad \& \quad
(b_2 \mid b_3)
\quad \& \quad
\neg b_1
\quad \& \quad
 \neg b_2$$

becomes 

$$[[1, 3], [2, 3], [-1], [-2]]$$

From here, using pycosat is simple:

```python
import pycosat
pycosat.solve([[1, 3], [2, 3], [-1], [-2]]) # will return [-1, -2, 3]
```

Just as we found, the formula is satisfiable.
We can interpret the output $ [-1, -2, 3]$ as $b_1=False$, $b_2=False$, $b_3=True$.

Pycosat finds our second formula to be unsatisfiable, just like we did:

```python
pycosat.solve([[1, 3], [2, 3], [-1], [-2], [-3]]) # will return 'UNSAT'
```

To try out my examples yourself or test your own examples, you can use the following command in your terminal to install pycosat:

```bash
pip install pycosat
```


# What’s an SMT solver?

SMT solvers are generalizations of SAT solvers. SAT solvers check if a boolean formula is satisfiable, whereas an SMT solver determines whether any mathematical formula is satisfiable—the formula could involve integers, real numbers, some type of data structure, or something else. 


Every SMT solver uses some theory or multiple theories.
First I’ll give an example of a problem using the theory of linear real arithmetic, and then I’ll give an example using the theory of inductive datatypes like you might see in a functional programming language such as Haskell.

Let $x, y, z$ be real numbers. Is the following satisfiable?

$$(w \ge x \mid w \ge y) \quad \& \quad (x \ge z \mid y \ge z) \quad \& \quad (z \ge w+1)$$

Yes it is. The following assignment of values satisfies the formula:

$$w=0, \; x=0, \;y=1, \;z=1$$

The following example probably won't make much sense if you don't have experience with a functional programing language. Even then, it might be a little odd. Is the following formula satisfiable?

$$\text{cons}(x, \text{nil}) = \text{cons}(y,z) \quad \& \quad (x=\text{red} | ~x=y) \quad \& \quad y=\text{green} $$




We know that $\text{cons}(x,\text{nil})=\text{cons}(y,z)$ must be true, which means that $x=y$ must be true. (If two lists are equal, then the heads of both lists must also be equal). Thus we can substitute $y$ with $x$ everywhere to get:

$$(x=\text{red} \mid \neg(x=x)) \quad \& \quad x=\text{green}$$

We know $x=\text{green}$ must be true, so we can substitute $x$ with $\text{green}$ everywhere to get:

$$\text{green}=\text{red} \mid \neg(\text{green}=\text{green})$$

Clearly neither $\text{green}=\text{red}$ nor $\neg(\text{green}=\text{green})$ is possible, thus the original formula is unsatisfiable.





Now for an example of using Z3 on the first formula we talked about.

```python
from z3 import Real, Solver, Or

w = Real('w')
x = Real('x')
y = Real('y')
z = Real('z')

solver = Solver()
```

Now we’ll represent the formula:
$$(w \ge x \mid w \ge y) \quad \& \quad (x \ge z \mid y \ge z) \quad \& \quad (z \ge w+1)$$

```python
solver.add(Or(w >= x, w >= y)) # (w >= x | w >= y)
solver.add(Or(x >= z, y >= z)) # (x >= z | y >= z)
solver.add(z >= w + 1) # (z >= w+1)

solver.check() # will return sat
```

We can get the the assignment of variables it used to determine the formula was satisfiable like this:
```python
solver.model() # will return [y = 1, x = 0, w = 0, z = 1]
```

To try out my examples yourself or test your own examples, you can use the following command in your terminal to install z3 in python:

```shell
pip install z3-solver
```

If you’re interested in using an SMT solver to test the inductive datatypes problem, I think Z3 might be able to handle that. However, you’ll probably have to use SMT-LIB, a standard format for SMT solvers if it’s possible. You’ll have to research that yourself if you’re interested though.


Here’s an example of how to use SMT-LIB in the python wrapper:
```python
string = """
(declare-const w Real)
(declare-const x Real)
(declare-const y Real)
(declare-const z Real)

(assert (or (>= w x) (>= w y)))
(assert (or (>= x z) (>= y z)))
(assert (>= z (+ w 1)))

(check-sat)
(get-model)
"""
s = Solver()
s.from_string(string)
s.check()
```
