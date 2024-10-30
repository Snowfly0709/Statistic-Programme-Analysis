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

#### Method Dispatch o Virtual Call

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

Resolve(a.foo()) = {a.foo(),c.foo(),d.foo()}

Resolve(b.foo()) = {a.foo(),c.foo(),d.foo()}

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

**Heap Abstraction**:

**Context Sensitivity**:

