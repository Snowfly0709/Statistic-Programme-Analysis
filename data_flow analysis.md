# Data Flow Analysis

## What is data flow analysis

Analyze how **data**(application-specific) **flows**(safe-approximate) on:

1. Basic blocks(Transfer)
2. Edges(Control flow)
3. CFG

### may analysis

output may be true(over-approximate)

### must analysis

output must be true(under-approximate)

both over-approximate and under-approximate are under consideration of safe-approximate.

## Preliminaries of Data Flow analysis

### Constraints for BB

#### Input and output state

Each execution of an IR statement transform **input state** to **output state**.

There exits a point connected to the input and output **before/after** the program.

this point is called observation point

```bash
a -> b

thus

out[a] = in[b]
```

```bash
a -> b
  -> c

thus

out[a] = in[b] = in[c]
```

#### Forward Analysis 前向分析

$OUT[s] = f_s(IN[s])$

#### Backward Analysis 反向分析

$IN[s] = f_s(OUT[s])$

有点类似神经网络中的反向传播

### Constraints for CF

#### Control Flow within BB

Block B:s1 s2 s3

$IN[s+1] = f_s(OUT[s])$

#### Control Flow between BBs

$IN[B] = IN[s_1] = \cap OUT[P_{predecessors \ of \ B}]$

$OUT[B] = OUT[s_n] = f_B(IN[B])$

反向分析同理

## Reaching Definitions Analysis

前提：不包含Method call方法调用和Aliases别名(Intra-procedure CFG & variables have no aliases)

A definition D at program point P reaches point Q if there is a **path** between P and Q & **D is not killed** during the path

a definition of variables means **assigning the value**

### Detect undefined variables

introduce **dummy(Undefined) definition** for each variable at the **entry of CFG**,and if dummy definition of V reaches a **point P when V is used**, then V may be used **before definition**(may-analysis).

### Definition representing method

can be represented by bit vector

e.g. D1,D2,...,D100 can be represented by

```text
100...000
010...000
001...000
...

each contains 100 bits
```

### Line of Reaching Definitions

$D: v = x \ op \ y$

this statement generates a **definition D** of variable v and **kill all other definition of v** in the program.

Transfer Function: $OUT[B] = gen_B\cup (IN[B]-kill_B)$

Control Flow:$IN[B] = \cup OUT[P_{predecessors\ of\ B}]$

### Algorithms

$OUT[entry] = \emptyset;$

**for each basic block/entry:**

$OUT[B] = \emptyset; \ \#may-analysis为undefined，must-analysis为unknown(top)$

**while (changes to any Block occurs):**

**for each basic block/entry:**

$IN[B] = \cup OUT[P_{predecessors\ of\ B}]$

$OUT[B] = gen_B\cup (IN[B]-kill_B)$

#### Factor

OUT[B] never shrink. Last itertion will remains in the OUT[B].

#### Cons

对于iterative算法，最后会达到最大/最小stop point，取决于使用的是may-analysis或是must-analysis

## Live Variables Analysis

Live variables analysis tell whether the **value of variable v at program point P** could be **used along some path** in CFG **starting at P**. If so, v is **live at P**; if not, v is dead at P.

v should **NOT** be redefined before usage!

when all registers are full in the program, we should replace allocation of dead value with live value.

由该分析的性质决定，可知使用backward-analysis比较好。

### Line of LVA

due to its backward analysis:

Control flow:$OUT[B] = \cup IN[U_{successors\ of\ B}]$

Tranfer function:$IN[B] = use_B \cup (OUT[B]-kill_B)$

**IMPORTANT**:transfer function requires line in line backwards analysis.使用transfer function应该在Block内自下而上逐行分析

### Algorithms of LVA

$IN[exit] = \emptyset;$

**for(each basic block/exit):**

$IN[B] = \emptyset;$

**while changes to any IN occurs:**

**for(each basic block/exit):**

$OUT[B] = \cup IN[U_{successors\ of\ B}]$

$IN[B] = use_B \cup (OUT[B]-kill_B)$

## Available Expressions Analysis

an expression E *x op y* is **available** at program point P if (1)**all path from entry** to P **must** pass through evlauation of E (2)after last evaluation of E there should be **no redefinition**

expression can also be represented by bit vectors:E1,E2,...,E100

### Rules

Add to OUT[B] the expression generated

Delete from IN[B] the expression involving (variables from expression)

Tranfer function:$OUT[B] = gen_B \cup (IN[B]-kill_B)$

```bash
This is a must-analysis.

a = x * y; -> x = m;b = x * y; -> c = x * y;
           --------------------->

There is no redefinition of x after b = x * y. The redefinition is before usage of x in b. So this is AVAILABLE.
```

Control flow:$IN[B] = \cap OUT[P_{predecessors\ of\ B}]$

### Algorithms of AEA

$OUT[Entry] = \emptyset$

**for each BB/entry:**

$OUT[B]=\cup (fullset)$

**while change to any OUT occur:**

**for each BB/entry:**

$IN[B] = \cap OUT[P_{predecessors\ of\ B}]$

$OUT[B] = gen_B \cup (IN[B]-kill_B)$

## Data Flow Analysis:Foundations N Principles

### View Iteration Algorithm in another way

Given a CFG with **k nodes**, the algorithm update OUT[k] in each iteration(the **n** round).

V - the domain of the values in data flow

($OUT[n_1],OUT[n_2],OUT[n_3],...,OUT[n_k]$) => element_set($V_1\times V_2\times V_3\times ... \times V_k$) => $V^k$

**each iteration can be rewritten as:** $F:V^k \rightarrow V^k$ until the value of $V^k$ don't change.

This kind of don't change is called the **Fixed Point**.

### Questions

Does it always have a fixed point?

OUT NEVER shrinks.

If so, is there only one fixed point?

F(X) = X => y = xcontains ifinite fixed point.

If not, what is the best fixed point?

greatest/least fixed point

When will we reach fixed point?

### Partial Order偏序

We define poset as a pair(P,$\leq$),where $\leq$ is a binary relationship that defines partial ordering over P.偏序集上的大小关系

(1)Reflexivity自反性：$x \leq x$

(2)Antisymmetry反对称性: $ x \leq y \cap y \leq x \rightarrow x = y$

(3)Tansitivity传递性

### Upper and Lower Bounds

Given a poset(P,$\leq$) and its subset S. u is at the top of P, it is the **upper bound** of S. Similarly we have the lower bound l.

least upper bound(join): $\cup S(x\cup y) \leq u $

greatest lower bound(meet): $l \leq \cap S(x \cap y)$

### Lattice

Given a poset (P,$\leq$), it lub and glb exist, then (P,$\leq$) is called a **lattice**.

在半序集中任取两个位置的数均由上确界和下确界则为格。

if only lub exists, called **join semi-lattice**.

if only glb exists, called **meet semi-lattice**.

**Complete lattice**: for each subset of poset, have lub and glb.

finite lattice => complete lattice

complete lattice ≠> finite lattice

#### Product Lattice

Given lattice L1($P_1,\leq_1$)...Ln($P_n,\leq_n$), each has lub and glb,then we have a product lattice defined :

$L^n = (P,\leq)$

**Proprties:**

P = $P_1\times P_2\times ... \times P_n$

$(x_1,x_2,...,x_n)\leq (y_1,y_2,...,y_n) \Leftrightarrow (x_1\leq y_1)\cap ... \cap(x_n\leq y_n)$

$(x_1,x_2,...,x_n)\cup (y_1,y_2,...,y_n) = (x_1\cup y_1,...,x_n\cup y_n)$

$(x_1,x_2,...,x_n)\cap (y_1,y_2,...,y_n) = (x_1\cap y_1,...,x_n\cap y_n)$

### Data Flow Analysis Framework

(D,L,F)

D: direction of Data Flow

L: lattice including domain of the value and lub/glb.

F: transfer functions

### Monotonicity

function f : L ->L is monotonic if $x\leq y \rightarrow f(x) \leq f(y)$

### Fixed Point Theorem

Given a complete lattice, if:

(1) f(L) is monotonic

(2) L is finite(complete can not lead to finite!)

then the least fixed point of f can be found by iterating $f(\perp)$,$f(f(\perp))$,...,$f^k(\perp)$

#### Proof of fixed point

By the definition of $\perp$ and f:L -> L

We have: $\perp \leq f(\perp)$

as f is monotonic,then: $f(\perp) \leq f(f(\perp))$

...

### The height of lattice

the length of the longest path from top to bottom = the max iteration needed to reach the fixed point for **one variable**

worst case in iteration: h(height)*k(nums of variables)

### May/Must Analysis - Safe & Truth

**以Reaching-Definition&May-analysis为例：**

对于bottom $\bot$(undefined)而言，其意味着没有dummy definitions reach point P. 从而说明所有的definitions在analysis中均有定义，没查出错误，是unsafe不安全的（因为reaching definiton是为了查错）。

对于top $\top$(unknown)而言，其意味着所有的variables都可能为dummy definitions，这样的结果是safe安全的，因为reaching definition查出了错误。但是所有variable初始时本身都处于这样的状态，所以这个状态是没有意义的。

对于truth，其介于safe与unsafe中间，在may-analysis中体现为greatest lower bound

从unsafe到safe，越来越useless，所以我们要停在min fixed point

**以Available Expression&Must-analysis为例：**

对于top $\top$(unknown)而言，其意味着所有的expression都available，没查出哪个是不available的，这个结果是unsafe的。

对于bottom $\bot$ 而言，其意味着所有的expression都不available，这是一个safe符合标准的结果，但是对于最终确定可复用的expression而言并没有用。

对于truth，其介于safe与unsafe中间，在may-analysis中体现为least upper bound

从unsafe到safe，越来越useless，所以我们要停在greatest fixed point.

### Precision:Meet-Over-All-Path(MOP)

Path: entry -> a series of statement(s1,s2,...,si)

Path transfer function $F_p$ is a composition of transfer functions for all statements.

$MOP[s_i] = \cap/\cup\ F_p(OUT[s_i])$

Cons of MOP: Some path may be not executable -> not fully precise

basic iterative algorithm: $F(x\cup y)$

MOP: $F(x)\cup F(y)$

利用MOP求出来的lub更准,proof:

$\because x \leq x \cup y \ and \ y \leq x \cup y$

$\because F\rightarrow monotonic$

$\therefore F(x) \leq F(x \cup y) \ and \ F(y) \leq F(x \cup y)$

$\therefore F(x \cup y)$ is the upper bound of $F(x)$ and $F(y)$

$\because F(x) \cup F(y) $ is the lub of $F(x)$ and $F(y)$

$\therefore F(x) \cup F(y) \leq F(x \cup y)$

$\therefore MOP \leq basic\ iterative\ method$

### Constant Propagation

Given a variable x at point P, determine whether x is guarantee to have value in P.(forward must-analysis)

the OUT of each node in CFG is a pair(x,v) which x represent variable name and v is its value.

**Lattice** domain:

Start from $\top$(pair is undefined), 到所有的variable都不constant（safe but useless）

**Lattice** meet operator($\cup$):

NC $\cap$ variable = NC

Undefined $\cap$ variable = variable

C $\cap$ variable(sameConstant) = C

C $\cap$ variable(notsameConstant) = NC

#### Transfer Function

$OUT[s] = gen \cup (IN[s] - pair(x,\_))$

x = c => gen = (x,c)

x = y => gen = (x,val(y))

x = y op z => gen = (x,f(y op z))

e.g.

### Worklist Algorithm

$OUT[entry] = \emptyset;$

**for each basic block/entry:**

$OUT[B] = \emptyset;

Worklist <- all basic blocks

**while (Worklist not empty):**

pick basic block B from the Worklist

temp_OUT = OUT[B]

$IN[B] = \cup OUT[P_{predecessors\ of\ B}]$

$OUT[B] = gen_B\cup (IN[B]-kill_B)$

if(temp_OUT $\neq$ OUT[B])

Add all successor of B to Worklist
