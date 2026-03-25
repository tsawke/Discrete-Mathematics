# CS 215 Discrete Math (H) — Optional Project Report (Sample)
## Topic A: Modern SAT Solving — From DPLL to CDCL, with Random 3-SAT Phase Transition Experiments

**Author:** (Sample Report)  
**Course:** CS 215: Discrete Math (H)  
**Semester:** 2025 Fall  
**Date:** Jan. 2026  

---

## Abstract

The **Boolean Satisfiability Problem (SAT)** asks whether a Boolean formula has an assignment that makes it true. SAT was the first problem proven **NP-complete**, and it is now a cornerstone technology in verification, planning, cryptography, and constraint solving. While the classical decision procedure **DPLL** (Davis–Putnam–Logemann–Loveland) forms the theoretical backbone, modern practical SAT solvers rely on **CDCL** (Conflict-Driven Clause Learning), a framework that combines **unit propagation**, **conflict analysis**, **clause learning**, and **non-chronological backtracking** together with effective engineering heuristics such as **watched literals**, **VSIDS branching**, **restarts**, and **clause database management**.

This report is self-contained: it introduces SAT, CNF/DNF, and NP-completeness at a high level, then develops DPLL and CDCL in detail (including implication graphs and 1-UIP learning). Finally, it presents experiments on **random 3-SAT** to illustrate the empirically observed **satisfiability phase transition** around clause-to-variable ratio \(\alpha = m/n \approx 4.2\)–\(4.3\), and provides a small encoding case study (Sudoku / graph coloring) demonstrating SAT as a universal modeling tool.

---

## Contents

1. Introduction  
2. Preliminaries  
   2.1 Boolean formulas, assignments, satisfiability  
   2.2 Normal forms: CNF and DNF  
   2.3 CNF conversion and Tseitin encoding  
3. Complexity Background: Why SAT Matters  
   3.1 P, NP, NP-completeness (overview)  
   3.2 Cook–Levin theorem (high-level idea)  
4. Classical SAT Solving: DPLL  
   4.1 Unit propagation and pure literals  
   4.2 DPLL algorithm and correctness sketch  
5. Modern SAT Solving: CDCL  
   5.1 Implication graph  
   5.2 Conflict analysis and clause learning (1-UIP)  
   5.3 Non-chronological backtracking (backjumping)  
   5.4 Restarts, VSIDS, watched literals, and clause deletion  
6. Experiments  
   6.1 Random 3-SAT generator  
   6.2 Phase transition: satisfiable fraction vs \(\alpha = m/n\)  
   6.3 Runtime comparison: (toy) backtracking vs DPLL vs CDCL (library)  
7. Modeling with SAT: Example Encodings  
   7.1 Graph \(k\)-coloring  
   7.2 Sudoku (outline)  
8. Discussion and Future Directions  
9. References  
Appendix A. Pseudocode  
Appendix B. Minimal demo code (illustrative)

---

## 1. Introduction

SAT is the problem of deciding whether a Boolean formula \(\varphi\) has an assignment of its variables that makes \(\varphi\) evaluate to TRUE. For example, \(\varphi = a \land \lnot b\) is satisfiable by \(a=T, b=F\), while \(\psi = a \land \lnot a\) is unsatisfiable.

SAT sits at the intersection of discrete mathematics and computation:

- It is historically important as the first **NP-complete** problem (Cook–Levin).
- It is practically important: modern SAT solvers can handle industrial instances with millions of variables/clauses in verification and synthesis tasks.
- It is a powerful modeling target: many constraints can be compiled into SAT (CNF), then solved with highly optimized solvers.

This report focuses on:
- Core definitions and transformations into CNF,
- Algorithmic evolution from DPLL to CDCL,
- A small empirical study of random 3-SAT behavior (phase transition),
- A brief demonstration of SAT encodings.

---

## 2. Preliminaries

### 2.1 Boolean formulas, assignments, satisfiability

Let \(V = \{x_1, x_2, \dots, x_n\}\) be Boolean variables. A **literal** is either a variable \(x\) or its negation \(\lnot x\). A **clause** is a disjunction (OR) of literals, e.g. \((x \lor \lnot y \lor z)\). A formula is **satisfiable** if there exists an assignment \(A: V \to \{T,F\}\) such that \(\varphi[A] = T\).

In SAT solving, we often use **partial assignments**. A partial assignment may assign some variables while leaving others unassigned. Under a partial assignment, a clause can be:
- satisfied (at least one literal TRUE),
- falsified (all literals FALSE),
- unresolved (otherwise).

### 2.2 Normal forms: CNF and DNF

- **CNF (Conjunctive Normal Form)**: \(\varphi = C_1 \land C_2 \land \cdots \land C_m\), each \(C_i\) is a clause (a disjunction of literals).
- **DNF (Disjunctive Normal Form)**: \(\varphi = T_1 \lor T_2 \lor \cdots \lor T_k\), each term \(T_j\) is a conjunction of literals.

SAT solvers almost always take CNF input, commonly in the **DIMACS CNF** format.

### 2.3 CNF conversion and Tseitin encoding

Any Boolean formula can be converted into CNF, but naive conversion can cause exponential blow-up. A standard remedy is **Tseitin encoding**, which introduces fresh variables to represent subformulas and yields a CNF equi-satisfiable with the original formula, with only linear growth.

**Example (Tseitin idea):**  
Let \(\varphi = (a \rightarrow b) \land (b \rightarrow c)\). Rewrite \(\rightarrow\) using \((p \rightarrow q) \equiv (\lnot p \lor q)\):  
\[
\varphi \equiv (\lnot a \lor b) \land (\lnot b \lor c).
\]
This is already CNF.

For a more complex formula like \(\psi = (a \land b) \lor (c \land d)\), direct CNF expansion yields:
\[
\psi \equiv (a \lor c) \land (a \lor d) \land (b \lor c) \land (b \lor d),
\]
which grows quickly for nested structures. Tseitin introduces \(u \leftrightarrow (a \land b)\), \(v \leftrightarrow (c \land d)\), \(w \leftrightarrow (u \lor v)\), then asserts \(w\). Each equivalence can be encoded with a small number of clauses.

---

## 3. Complexity Background: Why SAT Matters

### 3.1 P, NP, NP-completeness (overview)

- A decision problem is in **NP** if YES instances have polynomial-size certificates verifiable in polynomial time.
- SAT is in NP: an assignment is a certificate.
- SAT is **NP-complete**: it is in NP and every NP problem reduces to it in polynomial time.

Therefore, SAT is a “universal” NP problem: solving SAT efficiently in the worst case would imply \(P=NP\). Even though worst-case remains hard, many real instances are solvable in practice due to structure + clever heuristics.

### 3.2 Cook–Levin theorem (high-level idea)

Cook–Levin shows that any computation of a nondeterministic polynomial-time Turing machine can be encoded as a SAT instance. The key idea is to represent:
- the machine’s time steps (up to polynomial),
- tape cells,
- states and head positions,
as Boolean variables, and enforce valid transitions via local constraints. The resulting CNF is satisfiable iff the machine accepts.

This theorem is foundational because it links computation and logic, and motivates SAT as a core tool in theoretical CS and discrete math applications.

---

## 4. Classical SAT Solving: DPLL

The **DPLL** algorithm is a backtracking-based decision procedure for CNF SAT. It improves brute-force search by applying logical implications early.

### 4.1 Unit propagation and pure literals

- **Unit clause**: a clause with exactly one unassigned literal and all others falsified. To avoid falsifying the clause, that last literal must be TRUE. This forced assignment is called **unit propagation**.
- **Pure literal**: a variable that appears only as \(x\) or only as \(\lnot x\) in all remaining clauses. Setting it to satisfy all its occurrences is safe (it cannot create a contradiction in remaining clauses).

Unit propagation is essential; modern solvers implement it extremely efficiently.

### 4.2 DPLL algorithm and correctness sketch

At a high level:

1. If all clauses satisfied → SAT.
2. If some clause falsified → UNSAT under current partial assignment.
3. Apply unit propagation repeatedly.
4. Optionally apply pure-literal elimination.
5. Choose an unassigned variable and branch (assign TRUE/FALSE), recurse.

**Correctness (sketch):**
- Unit propagation preserves satisfiability because it assigns only logically forced values.
- Pure literal assignment preserves satisfiability because it can only satisfy clauses, never falsify any clause where the opposite literal appears (since it does not).
- Branching explores both possibilities; if neither branch yields SAT, then no assignment exists.

---

## 5. Modern SAT Solving: CDCL

DPLL is the conceptual base, but modern solvers typically implement **CDCL**, which can be viewed as DPLL + learning + intelligent backtracking.

### 5.1 Implication graph

During solving, assignments are of two types:
- **Decision assignments**: chosen by branching heuristic at a decision level.
- **Implied assignments**: forced by unit propagation.

CDCL represents implications using an **implication graph**:
- Nodes: assigned literals.
- Directed edges: if clause \((\lnot a \lor \lnot b \lor c)\) becomes unit with \(a=T, b=T\), then \(c=T\) is implied; edges \(a \to c\), \(b \to c\).

A **conflict** occurs when a clause becomes falsified (all literals FALSE). The conflict is represented as a special node connected from the literals that caused it.

### 5.2 Conflict analysis and clause learning (1-UIP)

When a conflict occurs, CDCL analyzes the implication graph to derive a new clause (a **learned clause**) that prevents repeating the same conflicting combination.

A standard method is **1-UIP (First Unique Implication Point)** learning:
- Consider the current decision level \(d\).
- The UIP is a cut point on paths from the decision literal at level \(d\) to the conflict.
- The learned clause corresponds to the negation of a set of literals on the “reason” side of the cut, ensuring the same conflict cannot reappear without violating the learned clause.

Intuition: learning adds a new constraint that “explains” the conflict, shrinking the search space.

### 5.3 Non-chronological backtracking (backjumping)

Instead of backtracking one level at a time (chronological), CDCL jumps back directly to the highest decision level that still matters for the learned clause. This is called **backjumping** or **non-chronological backtracking**.

Consequence: large parts of the search tree are pruned at once, which is crucial for performance.

### 5.4 Restarts, VSIDS, watched literals, and clause deletion

Modern solvers also use:

- **Watched literals**: Each clause watches two literals; unit propagation can be updated efficiently when a watched literal becomes false. This reduces propagation overhead.
- **Branching heuristics (VSIDS)**: Variables involved in recent conflicts get higher scores and are chosen earlier, focusing search on “hard” parts.
- **Restarts**: Periodically restart search but keep learned clauses; helps escape unproductive regions.
- **Clause deletion**: Not all learned clauses are kept forever; solvers delete low-utility clauses to control memory/time.

These engineering techniques often determine whether a solver is competitive.

---

## 6. Experiments

This section describes experiments that can be reproduced with short scripts. The goal is to illustrate two phenomena:
1. Random 3-SAT exhibits a phase transition in satisfiability probability near a critical \(\alpha = m/n\).
2. CDCL solvers outperform naive backtracking/DPLL significantly on harder regimes.

### 6.1 Random 3-SAT generator

A random 3-SAT instance with \(n\) variables and \(m\) clauses is generated by:
- For each clause, pick 3 distinct variables uniformly from \(\{x_1,\dots,x_n\}\).
- Negate each picked variable independently with probability \(1/2\).
- Form the clause as OR of the three literals.
- The formula is the AND of all clauses.

We define the clause density \(\alpha = m/n\).

### 6.2 Phase transition: satisfiable fraction vs \(\alpha\)

**Experimental design:**
- Fix \(n\) (e.g., \(n=50, 80, 100\)).
- Sweep \(\alpha\) from 2.0 to 6.0 (step 0.2).
- For each \(\alpha\), generate \(T\) random instances (e.g., \(T=100\)).
- Measure the fraction satisfiable.

**Expected qualitative result:**
- For small \(\alpha\), most instances are satisfiable.
- For large \(\alpha\), most instances are unsatisfiable.
- A sharp transition occurs around \(\alpha \approx 4.2\)–\(4.3\) for 3-SAT (empirical; threshold sharpens as \(n\) grows).

**(Sample) Result table format:**

| \(n\) | \(\alpha\) | Trials | SAT fraction |
|------:|-----------:|-------:|-------------:|
| 80    | 3.6        | 100    | 0.97         |
| 80    | 4.0        | 100    | 0.76         |
| 80    | 4.2        | 100    | 0.52         |
| 80    | 4.4        | 100    | 0.27         |
| 80    | 4.8        | 100    | 0.05         |

A plot of SAT fraction vs \(\alpha\) is recommended (Figure 1).

**Figure 1 (placeholder):**  
_SAT fraction vs clause density \(\alpha\) for random 3-SAT at several \(n\)._

### 6.3 Runtime comparison

**Solvers compared:**
1. Naive backtracking (no unit propagation).
2. DPLL (with unit propagation).
3. CDCL solver via a library interface (e.g., MiniSat / Glucose / PySAT).

**Metrics:**
- Median runtime per instance at each \(\alpha\).
- Timeout rate (e.g., cutoff at 1s or 5s).

**Expected qualitative result:**
- In easy regimes (very small or large \(\alpha\)), many methods are fast.
- Near the transition, naive methods slow dramatically; CDCL remains far more robust.

---

## 7. Modeling with SAT: Example Encodings

SAT is powerful because many discrete constraints can be compiled into CNF.

### 7.1 Graph \(k\)-coloring

Given a graph \(G=(V,E)\) and an integer \(k\), determine if vertices can be colored with \(k\) colors so adjacent vertices differ.

**Variables:**  
For each vertex \(v \in V\) and color \(c \in \{1,\dots,k\}\), define Boolean variable \(x_{v,c}\) meaning “vertex \(v\) has color \(c\).”

**Constraints:**

1. Each vertex has at least one color:
\[
\bigwedge_{v \in V} (x_{v,1} \lor x_{v,2} \lor \cdots \lor x_{v,k}).
\]

2. Each vertex has at most one color:
\[
\bigwedge_{v \in V} \bigwedge_{1 \le i < j \le k} (\lnot x_{v,i} \lor \lnot x_{v,j}).
\]

3. Adjacent vertices differ:
\[
\bigwedge_{(u,v) \in E} \bigwedge_{c=1}^{k} (\lnot x_{u,c} \lor \lnot x_{v,c}).
\]

This is CNF directly. The resulting SAT instance is satisfiable iff \(G\) is \(k\)-colorable.

### 7.2 Sudoku (outline)

Sudoku can be encoded similarly:
- Variables \(x_{r,c,d}\): cell \((r,c)\) contains digit \(d\).
- Constraints: each cell exactly one digit; each row/column/subgrid contains each digit exactly once; given clues fixed.

Sudoku is a good demo because it produces a medium-sized CNF and yields intuitive solutions.

---

## 8. Discussion and Future Directions

**Limitations:**  
SAT is NP-complete, so worst-case exponential behavior is unavoidable unless \(P=NP\). However, practical instances often have structure exploited by CDCL.

**Extensions:**
- **SMT (SAT Modulo Theories):** integrates SAT with theories like linear arithmetic, bit-vectors, arrays, enabling reasoning about programs/hardware at higher abstraction.
- **MaxSAT:** optimize the number of satisfied clauses (useful for scheduling, soft constraints).
- **Parallel/portfolio solvers:** run multiple heuristics/solvers in parallel for robustness.

**Research directions (high-level):**
- Better learned clause management policies,
- Machine-learning guided branching/restarts,
- Specialized encodings and preprocessing.

---

## 9. References

1. S. Cook, “The Complexity of Theorem-Proving Procedures,” *Proceedings of STOC*, 1971.  
2. L. Levin, “Universal Search Problems,” 1973 (Russian; often cited as the other half of Cook–Levin).  
3. M. Davis and H. Putnam, “A Computing Procedure for Quantification Theory,” *JACM*, 1960.  
4. M. Davis, G. Logemann, and D. Loveland, “A Machine Program for Theorem-Proving,” *CACM*, 1962.  
5. J. P. Marques-Silva and K. A. Sakallah, “GRASP: A Search Algorithm for Propositional Satisfiability,” *IEEE TCAD*, 1999.  
6. N. Eén and N. Sörensson, “An Extensible SAT-solver,” *SAT*, 2003. (MiniSat)  
7. A. Biere, M. Heule, H. van Maaren, and T. Walsh (eds.), *Handbook of Satisfiability*, IOS Press, 2009.  
8. (Optional for experiments) PySAT toolkit documentation / solver manuals (MiniSat/Glucose).

---

# Appendix A. Pseudocode

## A.1 DPLL (teaching version)

Given CNF \(F\) and partial assignment \(A\):

1. Simplify \(F\) under \(A\): remove satisfied clauses; remove falsified literals from clauses.
2. If \(F\) is empty → SAT.
3. If some clause is empty → UNSAT.
4. While there exists a unit clause \((\ell)\):
   - Set \(\ell\) to TRUE in \(A\), simplify.
5. (Optional) If there exists a pure literal \(\ell\):
   - Set \(\ell\) to TRUE in \(A\), simplify.
6. Choose an unassigned variable \(x\).
7. Recurse with \(x=T\); if SAT return SAT; else recurse with \(x=F\).

## A.2 CDCL (high-level)

Maintain:
- current assignment with decision levels,
- implication graph via reasons for propagations,
- clause database including learned clauses.

Loop:
1. Propagate (unit propagation).
2. If conflict:
   - analyze conflict to learn clause (e.g., 1-UIP),
   - determine backjump level,
   - add learned clause,
   - backjump and continue (or conclude UNSAT if conflict at level 0).
3. Else if all variables assigned: SAT.
4. Else pick branching literal using heuristic (e.g., VSIDS), increase decision level.

---

# Appendix B. Minimal demo code (illustrative)

> Note: The following snippets are illustrative and intentionally minimal for a sample report. In a real submission, include full scripts, argument parsing, and plots.

## B.1 Random 3-SAT CNF generator (DIMACS-like in Python)

```python
import random

def random_3sat(n, m, seed=None):
    rng = random.Random(seed)
    clauses = []
    for _ in range(m):
        vars_ = rng.sample(range(1, n + 1), 3)  # 1..n
        clause = []
        for v in vars_:
            lit = v if rng.random() < 0.5 else -v
            clause.append(lit)
        clauses.append(clause)
    return clauses

def to_dimacs(clauses, n):
    lines = []
    lines.append(f"p cnf {n} {len(clauses)}")
    for c in clauses:
        lines.append(" ".join(map(str, c)) + " 0")
    return "\n".join(lines)
~~~

## B.2 Measuring SAT fraction (requires a solver)

If using a SAT solver library (e.g., PySAT), the loop is:

```python
def sat_fraction_over_alpha(n, alphas, trials, solve_fn):
    results = []
    for alpha in alphas:
        m = int(round(alpha * n))
        sat_count = 0
        for t in range(trials):
            cnf = random_3sat(n, m, seed=100000 * n + 1000 * m + t)
            if solve_fn(cnf, n):
                sat_count += 1
        results.append((alpha, sat_count / trials))
    return results
```

Where `solve_fn` wraps a CDCL solver.

------

## Notes to Students (Sample)

- This report emphasizes both **theory** (definitions, NP-completeness context) and **practice** (CDCL mechanisms + experiments).
- A strong submission typically includes:
  1. clear definitions,
  2. at least one non-trivial algorithm explained (DPLL/CDCL),
  3. at least one empirical or modeling component (random 3-SAT, Sudoku, graph coloring),
  4. properly cited references.

------

```
::contentReference[oaicite:0]{index=0}
```