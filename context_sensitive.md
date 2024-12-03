# Context Sensitive

## Example

```bash
void main(){
    Number n1, n2, x, y;
    n1 = new One();
    n2 = new Two();
    x = id(n1);
    y = id(n2);
    int i = x.get();
}

Number id(Number n){
    return n;
}

interface Number{
    int get();
}

class One implements Number{
    public int get(){
        return 1;
    }
}

class Two implements Number{
    public int get(){
        return 2;
    }
}
```

e.g.1

```bash
n1 -> o1
n2 -> o2
n  -> o1,o2
x  -> o1,o2
y  -> o1,o2
x.get() -> NAC
y.get() -> NAC
```

对于常量传播来说，n1对应了o1，n2对应了o2，id类使n对应了o1,o2，再传到x,y中时他们分别都指向了o1和o2，在最终get的时候就会使得$1\cap 2$ = NAC

但是对于上下文敏感（动态分析）的时候，就会分开考虑，从而使x.get()返回正确的值1

e.g.2

```bash
n1 -> o1
n2 -> o2
id(n1):n -> o1
id(n2):n -> o2
x  -> o1
y  -> o1
x.get() -> 1
y.get() -> 2
```

## Context Sensitivity

Context Sensitivity models calling contexts by distinguishing different dataflows of different contexts to improve precision

### Call-Site sensitivity

The oldest and best-known context sensitivity strategy. It represent each context a chain of call site.

e.g.3

```bash
1.x = id(n1);
2.y = id(n2);
3.int i = x.get();


Number id(Number n){
    return n;
}
```

method id has two context: 1 and 2.

### Cloning-Based Context Sensitivity

In Cloning-Based Context Sensitivity, each method and variable are qualified by context. for example in the e.g.2

In practice, JAVA and other OOPLs are **heap-sensitive**, so context sensitivity should also apply to heap abstraction, and each object should be qualified by context

e.g.4

```bash
n1 = new One();
n2 = new Two();
x1 = newX(n1);
x2 = newX(n2);
n = x1.f();

X newX(Number n){
    X x = new X();
    x.f = p;
    return x;
}

class X{
    Number f;
}
```

对于variable和method有CS，而heap没有时

|variable|Object|Note|
|:----:|:----:|:----:|
|n1|o1||
|n2|o2||
|3:p|o1||
|3:x|o8||
|x1|o8||
|4:p|o2|可以看到p和x作了区分|
|4:x|o8||
|x2|o8||
|o8.f|o1,o2|这里会出现问题|
|n|o1,o2||

heap也有CS时

|variable|Object|Note|
|:----:|:----:|:----:|
|n1|o1||
|n2|o2||
|3:p|o1||
|3:x|3:o8|这里作了区分|
|x1|3:o8||
|4:p|o2||
|4:x|4:o8||
|x2|4:o8||
|3:o8.f|o1|如此就规避了o8的多重指向|
|n|o1||

**IMPORTANT**:cs heap must based on context sensitivity

## Domain and Notations

Context $C$: c`,c``

Context-sensitive method $C \times M$: c:m

Context-sensitive variable $C \times V$: c:v

Context-sensitive object $C \times O$: c:o

Field F: f,g

Context-sensitive instance field $C \times O \times F$: c:o.f

Context-sensitive pointer $(C \times V) \cup (C \times O \times F)$

## Rules

1. new: $\frac{}{c:o_i \in pt(c:x)}$

2. assign 'x = y': $\frac{c:o_i \in pt(c:y)}{c:o_i \in pt(c:x)}$

3. store 'x.f = y': $\frac{c':o_i \in pt(c:x),c'':o_j \in pt(c:y)}{c'':o_j \in pt(c':o_i.f)}$

4. load 'y = x.f': $\frac{c':o_i \in pt(c:x),c'':o_j \in pt(c':o_i.f)}{c'':o_j \in pt(c:y)}$

5. call

## Algorithms

```bash
Build Pointer Flow Graph with Contxt Sensitive <-- mutual dependent --> Propagate points-to information on PFG with Context Sensitive
```

Edges in Graph represent the objects pointed by pointer x may flow to pointer y

|Statement|PFG Edges|
|:----:|:----:|
|x = new T()|N/A|
|x = y|c:x <- c:y|
|x.f = y|c':oi.f <- c:y|
|y = x.f|c:y <- c':oi.f|
|r = x.k(a1,...,an)|c:a1 -> c^T:m_1,c:a2 -> c^T:m_2,c:r <- c^T:m_ret|

The only difference between CI and CS is the specification of context

```bash
Solve(m_entry){
    WL=[],PFG=[],S=[],RM=[],CG=[]
    AddReachable([]:m_entry)
    while(WL not empty){
        remove top<n,pts> from WL
        delta = pts - pt(n)
        Propagate(n,delta)

        if(n == variable c:x){
            for each c‘:o_i in delta{
                for each x.f = y in S{
                    addEdge(c:y,c’:o_i.f)
                }
                for each y = x.f in S{
                    addEdge(c‘:o_i.f,c:y)
                }
                ProcessCall(c:x,c’:o_i)
            }
        }
    }
}

#Reachable
AddReachable(c:m){
    if(c:m !in RM){
        add c:m to RM
        S = S combine S_M
        for each i: x = new T() in S_M{
            add <c:x,c:o_i> to WL
        }
        for each x = y in S_M{
            AddEdge(c:y,c:x)
        }
    }
}

#MethodCall
ProcessCall(c:x,c‘:o_i){
    for each l: r = x.k(a1,...,an) in S{
        m = Dispatch(o_i,k)
        ct = Select(c,l,c’:o_i,m)
        add <ct:m_this,c‘:o_i> to WL
        if(c:l -> ct:m !in CG){
            add c:l -> ct:m to CG
            AddReachable(ct:m)
            for each p_i of m{
                AddEdge(c:a_i,ct:p_i)
            }
            AddEdge(ct:m_ret,c:r)
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

## Context Sensitive Varient

### Call-site sensitivity

Select(caller context,call site,receive object with heap CS,callee site)

call-site **chain**

```text
Select(c,l,c‘:o,m)=[l1,l2,...,l]

where c = [l1,l2,...]
```

如果类中出现递归会是什么情况？

```bash
void main(){
    a.foo()
}

void foo(){
    foo()
}
```

对于foo来说，由于递归，其context会变为[2,6,6,6,...]无限重复下去，此时我们需要引入一个限制来避免这种情况

#### k-limiting context abstraction

set an upper bound k for length of context(method context and heap context use different bound)

e.g.

1-call-site:Select(c,l,c‘:o,m)=[l]

2-call-site:Select(c,l,c‘:o,m)=[l‘’,l] where c = [l’，l‘’]

### Object sensitivity

### Type sensitivity
