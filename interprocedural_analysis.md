# Interprocedural analysis

## Motivation of Interprocedural analysis

for **intra**procedural analysis, we make conservative assumption for method call, leaving most of variable imprecise.

e.g.

```bash
void foo(){
    int n =ten();
    addOne(42);
}

int ten(){
    return 10;
}

int addOne(int x){
    int y = x + 1;
    return y;
}
```

for **Constant Propagation** in intraprocedural analysis, n,x,y are all NAC because we ignore the connection between each method.

BUT for Interprocedural, we use **call graph** to point the call edges between each method, and n =10, x=42, y = 43.

## Call Graph

call graph is a representation of calling relationships in the program. It is made up by a set of call edges from one method to another.

### Call Graph Construction for OOPLs

Class Hierarchy Analysis(CHA)

Rapid Type Analysis(RTA)

Variable Type Analysis(VTA)

Pointer Analysis

越往下越精确，越往上效率越高

#### Method Dispatch on Virtual Call

a virtual call is resolved based on:

1. type of the receiving project:c

2. method signature:m

We use *Dispatch(c,m)* to jsimulate the procedure of method dispatch, if c doesn't contain non-abstract method c, then it will find in the **superclass** of c.

```bash
class A{void foo(){...}}

class B extends A{}

class C extends B{
    void foo(){...}

    void dispatch(){
        A x = new B();
        x.foo();

        A y = new C();
        y.foo();
    }
}
```

对于x.foo() 实际为Dispatch(B,A.foo()),由于在B中没有找到foo方法，所以寻找B的父类A，实际调用的是A类中的foo方法。

对于y.foo() 实际为Dispatch(C,A.foo()),由于C中就存在了foo方法，所以实际调用为C中的foo方法。

### Class Hierarchy Analysis

We define func *Resolve(cs)* to resolve possible target methods of a call site *cs* by CHA.

```bash
Resolve(cs)
    T = {}
    m = method signature at CS
    if CS is a static call
        T = {m}
    if CS is a special call
        cm = class type in m
        T = {Dispatch(cm,m)}
    if CS is a virtual call
        c = declared receiver type at CS
        foreach c' that is subclass of c do add Dispatch(c',m) to T
return T
```

e.g.对于虚拟调用

```bash
class A{void foo(){...}}

class B extends A{}

class C extends B{
    void foo(){...}
}

class D extends B{
    C c = ...
    c.foo()

    A a = ...
    a.foo()

    B b = ...
    b.foo()
}
```

Resolve(c.foo()) = {C.foo()}

Resolve(a.foo()) = {A.foo(),C.foo(),D.foo()}

Resolve(b.foo()) = {A.foo(),C.foo(),D.foo()}

如果在声明时使用了B b = new B() 此时Resolve的dispatch结果仍然不变，同样包含c.foo() & d.foo(), 与我们前文中所写的本类中没有应该在父类中调用有所不符，这就是CHA方法带来的“假”调用。

### Call Graph Construction in CHA

Start from entry method, for each reachable method,resolve each CS using CHA, until no method is discovered

Algorithms:

```bash
BuildCallGraph(entry)
    WaitList = [Entry],CallGraph = {},ReachableMethod = {}
    while WL is not empty
        remove top m from WL
        if m not belongs to RM
            add m to RM
            for each CS in m
                T = Resolve(CS)
                for each target method tm in T
                    add CS -> tm to CG
                    add tm to WL
    return CG
```

## Interprocedural Control Flow Graph

Besides in and out in the CFGs, ICFG has **Call edges** and **Return edges**.

*Call edges*: from CS to the **entry node of the called method**,pass the argument value

*Return edges*: from **return statement** to the **return sites**(the statement after the call in previous method),pass the argument value

In ICFG,when node is a **call site**, transfer function **DOESN'T contain gen_value** on the left of equation. The edge between CS and RS is called call-to-return edge, to transfer local data(which is not used by called method).

## Problem of CHA

if there are false positive, it may lead to false Constant Propagation.

e.g.

```bash
void foo(){
    Number x = new ONE()
    int x = n.get()
}

interface Number{
    int get()
}

class ONE implements Number{
    public int get(){
        return 1;
    }
}

class TWO implements Number{
    public int get(){
        return 2;
    }
}

class ZERO implements Number{
    public int get(){
        return 0;
    }
}
```

当 *Resolve(n.get())* 时，由于是CHA的virtual call，所以会返回zero.get(),one.get(),two.get()三个return edge返回的值0,1,2，此时交运算则会变为NAC（参考前面常量传播规则），这显然是不符合实际的。

But for Pointer Analysis, it will based on point-to relation, it will only return one.get() and return the value 1.

## Pointer Analysis

What are pointers pointing?

e.g.

```bash
void foo()
{
    A a = new A()
    B x = new B()
    a.setB(x)
    B y = a.getB()
}

class A {
    B b;
    void setB(B b){
        this.b = b
    }
    B getB(){
        return this.b
    }
}
```

Points-to Relation

|Variable | Object|
|:----:|:----:|
|a | new A|
|x | new B|
|this | new A|
|b | new B|
|new A.b | new B|
|y | new B|

### Pointer Analysis and Alias Analysis

Pointer Analysis: what a pointer to point to?

Alias Analysis: can two pointer point same obj?

### Key Factor in Pointer Analysis

Heap Abstraction

Context Sensitivity

Flow Sensitive

Analysis Scope

### Heap Abstraction

when in loops and recursion, the heap object can be unbounded.

```bash
for(i=0;i<3;i++){
    A a = new A();
}
```

when in dynamic analysis, this class will be called three times: o2.iteration i = 0,o2.iteration i = 1,o2.iteration i = 2

to ensure termination in static analysis, we use heap abstraction model dynamically allocated, unbounded concrete objects as finite abstract objects.

#### Allocation-site abstraction

Model concrete objects by their **allocation sites**, and each abstract object per allocation site represents ALL allocated concrete objects in dynamic analysis.

when in allocation-site abstraction, the class call will only be abstracted as **o2** in before example.

### Context Sensitivity

Context Sensitive: use each method multiple times. Different use has different data flow.

Context insensitie: use each method one time and all data flow merge.

### Flow Sensitivity

flow sensitive: respect the execution order of the statements

flow insensitive: ignore the CFG order, treat the whole program as a set of unorderd statement.

### Analysis Scope

whole-program: compute point-to information in all pointers.

demand-driven: only compute information for pointers demanded.

### Pointer Analysis in our case

Allocation-Site/Context-Sensitive/Flow-intensitive/Whole-program

### Concerned statement

we only focus on pointer affect statement

#### Pointers in Java

1. Local variable:x

2. Static field:C.f

3. Instance field:x.f

4. Array element:array[i]

#### Pointer affecting statements

New/Assign(x = y)/Store(x.f = y)/Load(y = x.f)/Call(r = x.k(a,...))

#### Domains and notations

Variables: V

Fields: F

Objects: $o_i,o_j \in$ O

Instance field: $o_i.f , o_j.g \in$ $O \times F$

Pointers include Variables & Instance field

Points-to relations: pt-> $\rho(O)$ , $\rho(O)$ is the powerset of the O

### Rules

1. x = new T() =>  $\frac{}{pt(x).append(o_i)}\frac{\leftarrow premise}{\leftarrow conclusion}$ if no premise then unconditional

2. y = x =>  $\frac{o_i\in pt(x)}{pt(y).append(o_i)}$

3. x.f = y => $\frac{o_i\in pt(x),o_j\in pt(y)}{pt(o_i.f).append(o_j)}$

4. y = x.f => $\frac{o_i\in pt(x),o_j\in pt(x.f)}{pt(y).append(o_j)}$

## Pointer Analysis Foundation

### Pointer Flow Graph(PFG)

display how objects flow among the pointers in the program.

Nodes: $V\bigcup U \times F$

Edges: $Pointer \times Pointer$

### Pointer Analysis Algorithms

```bash
# Main Algorithm
Solve(S){
    WL = [],PFG=[]
    for each i : x = new T() in S{
        add <x,o_i> to WL
    }
    for each x = y in S{
        addEdge(y,x)
    }
    while(WL not empty){
        remove top<n,pts> from WL
        delta = pts - pt(n)
        Propagate(n,delta)

        if(n == variable x){
            for each o_i in delta{
                for each x.f = y in S{
                    addEdge(y,o_i.f)
                }
                for each y = x.f in S{
                    addEdge(o_i.f,y)
                }
            }
        }
    }
}

#Propagate
Propagate(n,pts){
    if(pts not empty){
        pt(n) = pt(n) & pts
        for each n -> (s in PFG){
            add <s,pts> to WL
        }
    }
}

# addEdge
addEdge(s,t){
    if(s -> t not in PFG){
        add s-> t to PFG
        if(pt(s) not empty){
            add <t, pt(s)> to WL 
        }
    }
}
```

Propagation aims to transfer delta to reduce loop count.

### Pointer Analysis with Method Calls

#### Difference between CHA and pointer-analysis in Call Graph Construction

1. CHA:imprecise, may introduce spurious call edges
2. pointer analysis:precise

#### Rules for method call

r = x.k(a1,...,an) => $\frac{o_i \in pt(x), m = Dispatch(o_i,k),o_u \in pt(a_j),o_v \in pt(m_{ret})}{o_i\in pt(m_{this}),o_u\in pt(m_{pj}),o_v \in pt(r)}$

1. $Dispatch(o_i,k),o_i\in pt(x)$ resolve the virtual dispatch of k on o_i

2. $m_{this}$ : this variable of m

3. $m_{pj}$ : the j-th parameter of m

4. $m_{ret}$ : the variable that holds return value

```bash
C x = new T()

r = x.foo(a1,a2);

class T {
    B foo (A p1, A p2){
        this;
        return ret;
    }
}
```

PFG Edges: a1 -> p1, a2 -> p2, ret -> r

### Algorithms(pointer analysis and Call Graph Construction combined)

S: set of reachable statement

S_M: set of statement in method M

RM: reachable method

CG: call graph edges

```bash
Solve(m_entry){
    WL=[],PFG=[],S=[],RM=[],CG=[]
    AddReachable(m_entry)
    while(WL not empty){
        remove top<n,pts> from WL
        delta = pts - pt(n)
        Propagate(n,delta)

        if(n == variable x){
            for each o_i in delta{
                for each x.f = y in S{
                    addEdge(y,o_i.f)
                }
                for each y = x.f in S{
                    addEdge(o_i.f,y)
                }
                ProcessCall(x,o_i)
            }
        }
    }
}

#Reachable
AddReachable(m){
    if(m !in RM){
        add m to RM
        S = S combine S_M
        for each i: x = new T() in S_M{
            add <x,o_i> to WL
        }
        for each x = y in S_M{
            AddEdge(y,x)
        }
    }
}

#MethodCall
ProcessCall(x,o_i){
    for each l: r = x.k(a1,...,an) in S{
        m = Dispatch(o_i,k)
        add <m_this,o_i> to WL
        if(l -> m !in CG){
            add l -> m to CG
            AddReachable(m)
            for each p_i of m{
                AddEdge(a_i,p_i)
            }
            AddEdge(m_ret,r)
        }
    }
}

#Propagate
Propagate(n,pts){
    if(pts not empty){
        pt(n) = pt(n) & pts
        for each n -> (s in PFG){
            add <s,pts> to WL
        }
    }
}

# addEdge
addEdge(s,t){
    if(s -> t not in PFG){
        add s-> t to PFG
        if(pt(s) not empty){
            add <t, pt(s)> to WL 
        }
    }
}
```
