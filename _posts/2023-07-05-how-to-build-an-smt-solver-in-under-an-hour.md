---
layout: post
title: How to build an SMT solver in under an hour
date: 2023-07-05 22:44 -0700
math: true
---


Are you tired of crawling the web for good tutorials for writing your own SMT solver? Is everything you find is either a 300 slide long powerpoint presentation or a dense paper?

Well you’re in luck. In this tutorial I’ll show you how to easily build an SMT solver. Don’t expect it to be efficient, but it should give you an idea of how SMT solvers work.

If you’re not sure what an SMT solver is, check out my [previous blog post](https://tilorc.github.io/posts/what-are-sat-and-smt-solvers/).

First, I’ll go over what you’ll need and outline the algorithm. Then I’ll walk you through my implementation. 

You’ll need an off the shelf SAT solver and some theory solver, preferably one for Linear Real Arithmetic (LRA) as that’s what I will be using. I’ll be doing everything in Python, so I’ll be using pycosat and the `find_feasible` function that will be a part of SymPy (I’m currently finishing up a pull request related to this). If `find_feasible` is not yet part of SymPy when you’re reading this, you can get the function [here](https://gist.github.com/TiloRC/4b9d7a907fcad0df6b463a30f5a8f1cd).

Our algorithm will take one input: a boolean formula involving inequalities in CNF format. Strict inequalities (< or >) and != are not allowed because `find_feasible` is a pathetic theory solver (although maybe strict inequality handling has been added at the time you’re reading this).

Wait what’s CNF format? Oh no. I’ll have to explain some terminology. Actually a lot of terminology.

A boolean formula is in Conjunctive Normal Form (CNF) if it is made up of a conjunction of clauses. Clauses are a disjunction of literals. Literals are an atom or the negation of an atom. An atom or atomic formula is a boolean formula without any logical connectives (and, or, not). 

Wow. That was a lot of definitions. Here’s an example of a CNF formula:

$$(b_1  \mid b_3)
\quad \& \quad
(b_2 \mid b_3)
\quad \& \quad
\neg b_1
\quad \& \quad
 \neg b_2$$
 
 And an example of a formula not in CNF:
 
 $$(b_1  \mid (b_3 \; \& \; b_5))
\quad \& \quad
(b_2 \mid b_3)
\quad \& \quad
\neg b_1
\quad \& \quad
 \neg b_2$$
 
All boolean formulas can be converted to CNF, so it’s not a problem that we’re assuming that the input to algorithm is a formula in conjunctive normal form.

## 1. Algorithm outline

### Step 1. Initialize the state as our input formula

### Step 2. Run SAT solver on the input (Treating inequalities as boolean variables). 

If we get back `unsat` here, we’re done: we return `unsat`.
If not, then we’ll take the satisfying assignment given to us by the SAT solver and continue.

### Step 3. Use theory solver to check satisfying assignment from SAT solver.

If our theory solver finds the assignment to be satisfiable, we’re done: we return `sat` and the assignment that resulted in `sat`. 

Otherwise, we add a conflict clause to the current state and go back to step 2. 

A conflict clause, or more specifically in this case, a theory lemma, is a clause that lets the SAT solver know that some combination of inequalities can’t all be true at the same time. 

## 2. Example

We'll be using this example:

$$(w \ge x \mid w \ge y) \quad \& \quad (x \ge z \mid y \ge z) \quad \& \quad (z \ge w+1)$$

We’ll represent it as follows:

```python
from sympy.abc import w, x, y, z
cnf = [[(w >= x, False), (w >= y, False)],
      [(x >= z, False), (y >= z, False)],
      [(z >= w + 1, False)]]
```

Now this may seem a bit odd. Why am I putting each inequality into a tuple with False?

Each tuple like (w >= x, False) is a literal. Each inequality like w >= x is an atom. The False indicates (w >= x, False) is also an atom. (w >= x, True) would be the negation of (w >= x, False).

Each list within `cnf` such as [(w >= x, False), (w >= y, False)] is a clause. 

Now, we’ll want to feed `cnf` into pycosat. But first we’ll need to transform it into a format that picosat understands. I wrote the following function to do so. The details of how it works are not important.

```python
def encode(cnf):
    encoding = {}
    i = 1
    ret = []
    for clause in cnf:
        encoded_clause = []
        
        for literal in clause:
            atom, negated = literal
            negated = -1 if negated else 1
            if atom not in encoding:
                encoding[atom] = i
                i += 1
            encoded_clause.append(encoding[atom]*negated)
            
        ret.append(encoded_clause)
    return ret, {value:key for key, value in encoding.items()}
```

You can use `encode` like this:

```python
encoded_cnf, encoding = encode(cnf)
encoded_cnf # [[1, 2], [3, 4], [5]]
encoding # {1: w >= x, 2: w >= y, 3: x >= z, 4: y >= z, 5: z >= w + 1}
```

You can think of `[[1, 2], [3, 4], [5]]` as the following boolean formula:

$$(b_1 \mid b_2) \quad \& \quad (b_3 \mid b_4) \quad \& \quad  b_5$$


Now we can give `encoded_cnf` to pycosat:

```python
import pycosat
encoded_assignment = pycosat.solve(encoded_cnf)
encoded_assignment # [1, 2, 3, 4, 5]
```

`[1, 2, 3, 4, 5]` is the assignment that pycosat found for us. It corresponds to 

$$b_1=True, \; b_2=True, \; b_3=True, \;b_4=True, \; b_5=True$$

We’ll need to unencode `[1, 2, 3, 4, 5]` and give it to find_feasible. 

```python
from find_feasible import find_feasible
unencoded = [encoding[enc] for enc in encoded_assignment] 
feasible = find_feasible(unencoded, [w, x, y, z]) 
feasible is None # True
```

As this particular assignment is not feasible, we're not done yet. We need to generate a conflict clause and continue. Making a conflict clause isn’t too difficult. We can just add a clause that contains the negated version of every literal in the assignment we got from pycosat:

```python
encoded_conflict_clause = [-1, -2, -3, -4, -5]
encoded_cnf.append(encoded_conflict_clause)
```

After we add our conflict clause, encoded_cnf will look like this:

$$(b_1 \mid b_2) \quad \& \quad (b_3 \mid b_4) \quad \& \quad  b_5 \quad \& \quad
(\neg b_1 \mid  \neg b_2 \mid \neg b_3 \mid \neg b_4 \mid \neg b_5)$$

Now we give encoded_cnf to pycosat again:

```python
encoded_assignment = pycosat.solve(encoded_cnf)
encoded_assignment # [1, 2, 3, -4, 5]
```

You’ll notice that we get a different assignment this time because our added conflict clause makes [1, 2, 3, 4, 5] impossible.

We give the new encoded_assignment into find_feasible. We omit -4 or (y >= z, True) because find_feasible can’t handle not equals or strict inequalities.
```python
unencoded = [encoding[enc] for enc in encoded_assignment if enc > 0]
feasible = find_feasible(unencoded, [w, x, y, z])
feasible is None # True
```
You might think that omitting -4 and that’s because it is. There will be some edge cases where our SMT solver thinks something is satisfiable when it’s not. 


To prevent the assignment [1, 2, 3, -4, 5], we’ll add a new conflict clause, and do another run through the algorithm.

```python
encoded_conflict_clause = [-1, -2, -3, 4, -5]
encoded_cnf.append(encoded_conflict_clause)
encoded_assignment = pycosat.solve(encoded_cnf)
print(encoded_assignment) # [1, 2, -3, 4, 5]
unencoded = [encoding[enc] for enc in encoded_assignment if enc > 0]
feasible = find_feasible(unencoded, [w, x, y, z])
feasible is None # True
```
We do everything again:

```python
encoded_conflict_clause = [-1, -2, 3, -4, -5]
encoded_cnf.append(encoded_conflict_clause)
encoded_assignment = pycosat.solve(encoded_cnf)
print(encoded_assignment)
feasible =find_feasible([encoding[enc] for enc in encoded_assignment if enc > 0] , [w, x, y, z])
feasible is None # False
feasible # {w: 0, x: 0, y: 1, z: 1}
```
We’ve found a solution! If you like, you can verify that this does indeed satisfy the original formula.

$$(w \ge x \mid w \ge y) \quad \& \quad (x \ge z \mid y \ge z) \quad \& \quad (z \ge w+1)$$

plug in our values

$$(0 \ge 0 \mid 0 \ge 1) \quad \& \quad (0 \ge 1 \mid 1 \ge 1) \quad \& \quad (1 \ge 0+1)$$

simplify 

$$(True \mid False) \quad \& \quad (False \mid True) \quad \& \quad (True)$$

As you can see, the assignment we found does indeed satisfy the formula. We had to add 3 conflict clauses to get to a solution. When we’re finished, `encoded_cnf` is equal to this: 

[[1, 2],
 [3, 4],
 [5],
 [-1, -2, -3, -4, -5],
 [-1, -2, -3, 4, -5],
 [-1, -2, 3, -4, -5]]


## 3. Full algorithm

```python
def rudamentary_smt_solver(ineq_cnf):
    variables = set.union(*[lit[0].free_symbols for clause in cnf for lit in clause ])
    loops = 0
    while True:
        loops +=1
        state = ineq_cnf
        encoded_cnf, encoding = encode(cnf)
        encoded_assignment = pycosat.solve(encoded_cnf)
        if encoded_assignment == 'UNSAT':
            print(f"{loops} loops were performed")
            return "UNSAT"
        not_negated_assigment = [encoding[i] for i in encoded_assignment if i > 0]
        variables = [w, x, y, z]

        feasible = find_feasible(not_negated_assigment, variables)
        if feasible:
            print(f"{loops} loops were performed")
            return "SAT", feasible
        else:
            conflict_clause = [(atom, True) for atom in not_negated_assigment]
            state.append(conflict_clause)
```

We can run our function on the example like this:

```python
cnf = [[(w >= x, False), (w >= y, False)],
      [(x >= z, False), (y >= z, False)],
      [(z >= w + 1, False)]]
sol = rudamentary_smt_solver(cnf)
sol # ('SAT', {w: 0, x: 0, y: 1, z: 1})
```

## 4. Take aways 

If there's one thing to take away, it's that this is a terrible SMT solver.

### 1. Handling strict inequalities is important

As I mentioned in the example, there are some edge cases where it thinks that a formula is satisfiable when it's not. For example, consider this formula that is unsatisfiable:

$$ \neg (x \ge 0) \quad \& \quad \neg (x \le -1) \quad \& \quad x \ge 1$$

```
cnf = [[(x >= 0, True)],
       [(x <= -1, True)],
       [(x >= 1, False)]]
sol = rudamentary_smt_solver(cnf)
sol # ('SAT', {w: 0, x: 1, y: 0, z: 0})
```

It may not seem like it, but the last clause $ x \ge 1$ is important. Suppose x can be a complex variable. Then $\neg (x \ge 0) \; \& \; \neg (x \le -1) $ alone is actually satisfiable. Because x is complex, if $\neg (x \ge 0)$ doesn’t imply $x > 0$. However, because we know $x \ge 1$ this also means that x must be real (it doesn't make sense to compare imaginary numbers with $\\ge$ since imaginary numbers are not ordered).

Now you could easily fix this problem by using a theory solver better than find_feasible. You could use z3 for example. However, it does feel a bit weird to use an SMT solver like z3 to build your own much worse SMT solver. But, as SymPy has an SMT-LIB printer, this wouldn’t be too bad to implement. You can see the bottom of my previous blog post for an example of how to use z3 and SMT-LIB in python. 

### 2. Small conflict clauses are better

However, there’s another much more difficult to fix problem with our SMT solver: It’s really inefficient. Perhaps the biggest issue is how we’re generating conflict clauses. They’re too big. Actually good SMT solvers have theory solvers that give conflict clauses of minimal size. 

In the function there’s a counter of how many times pycosat is called. This number gets printed out whenever the function is fun. This should help us understand why the size of conflict clauses matters so much.

Consider these examples. 

```python
cnf = [[(x <= -1, False)],
       [(y<= -1, False)],
       [(x >= 0, False), (y>=0, False)]]
rudamentary_smt_solver(cnf)
# 4 loops were performed
# 'UNSAT'
```

```
cnf = [[(x <= -1, False)],
       [(y<= -1, False)],
       [(z<= -1, False)],
       [(x >= 0, False), (y>=0, False), (z >= 0, False)]]
rudamentary_smt_solver(cnf)
# 8 loops were performed
# 'UNSAT'
```

```
cnf = [[(w <= -1, False)],
       [(x <= -1, False)],
       [(y<= -1, False)],
       [(z<= -1, False)],
       [(w >= 0, False), (x >= 0, False), (y>=0, False), (z >= 0, False)]]
rudamentary_smt_solver(cnf)
# 16 loops were performed
# 'UNSAT'
```

As you can see the time complexity for this type of example is O(2^n). Yikes. 

If we get a message saying 16 loops were performed, that means that the function needed to add 15 conflict clauses. We can do a lot better than that. If we’re adding conflict clauses of minimal size, we only need to add 4:

```
cnf = [[(w <= -1, False)],
       [(x <= -1, False)],
       [(y<= -1, False)],
       [(z<= -1, False)],
       [(w >= 0, False), (x >= 0, False), (y>=0, False), (z >= 0, False)],
       [(w <= -1, True), (w >= 0, True)],
       [(x <= -1, True), (x >= 0, True)],
       [(y <= -1, True), (y >= 0, True)],
       [(z <= -1, True), (z >= 0, True)]]
rudamentary_smt_solver(cnf)
# 1 loops were performed
# 'UNSAT'
```

### 3. The best SMT solvers use backtracking

Actually good SMT solvers don’t have to run their SAT solvers from scratch each time they add a new conflict clause. Same goes for their theory solvers. Instead they use a backtracking approach. If you’re up for it, you can read [this](https://link.springer.com/chapter/10.1007/11817963_11) paper to learn how efficient SMT solvers over linear real arithmetic work. [This](https://resources.mpi-inf.mpg.de/departments/rg1/conferences/vtsa17/slides/reynolds-vtsa-part1.pdf) 275 slide long powerpoint presentation is also a good resource.






 
 
