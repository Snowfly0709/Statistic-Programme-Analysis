# Intermediate_representation中间语言表示

## Review of Compiler

Source code -> 词法分析 -> 语法分析 -> 语义分析 -> 中间代码生成 -> **（静态分析）** -> 生成目标代码 -> machine code

更多有关编译原理的内容请查阅其他书籍或跳转我的专栏：

### AST抽象语法树与三地址码比较

#### AST

1. high-level & close to grammar
2. language dependent
3. fast type checking
4. lack of control-flow information

#### IR(3-address)

1. low-level & close to machine code
2. language independent
3. simple, compact and uniform
4. contain control-flow information

## Soot and Jimple

Soot是github上一个开源的java静态分析项目，它的IR名字叫Jimple

e.g.1 for循环

```bash
package xjtu.edu.e1;
public class ForLoop3AC {
    public static void main(String[] args){
        int x = 0;
        for(int i = 0; i < 10; i++){
            x = x + 1;
        }
    }
}

#对应的Jimple：
public static void main(java.lang.String[]){
    java.lang.String[] r0;
    int i1;

    r0 := @parameter0: java.lang.String[];
    i1=0; #这里编译器进行了优化，x dead-zone

    label1:
        if i1 >= 10 goto label2; #有条件goto语句

        i1 = i1 + 1;

        goto label1; #无条件goto语句

    label2:
        return;
}
```

e.g.2 方法调用

```bash
package example;
public class MethodCall3AC{
    String combine(String string1, String string2){
        return string1 + " " + string2;
    }

    public static void main(String[] args){
        MethodCall3AC mc = new MethodCall3AC();
        System.out.println(mc.combine("xjtu","se"))
    }
}

#combine函数对应的Jimple：
java.lang.String combine(java.lang.String, java.lang.String){
    example.MethodCall3AC r0;
    java.lang.String r1,r2,$r7; #$在Jimple中代表临时变量
    java.lang.StringBuilder $r3,$r4,$r5,$r6;

    r0 := @this: example.MethodCall3AC;
    r1 := @parameter0: java.lang.String;
    r2 := @parameter1: java.lang.String;
    $r3 = java.lang.StringBuilder;

    specialinvoke $r3.<java.lang.StringBuilder: void <init>()>();
    #函数签名 className.<classType: returnType methodName(paraType)>(paraName)

    $r4 = virtualinvoke $r3.<java.lang.StringBuilder: java.lang.StringBuilder append(java.lang.String)>(r1);
    #这一步相当于使用了StringBuilder底下的append方法，在空的基础上衔接了实际值为r1的StringBuilder作为返回值，下同
    $r5 = virtualinvoke $r4.<java.lang.StringBuilder: java.lang.StringBuilder append(java.lang.String)>(" ");
    $r6 = virtualinvoke $r5.<java.lang.StringBuilder: java.lang.StringBuilder append(java.lang.String)>(r2);

    $r7 = virtualinvoke $r6.<java.lang.StringBuilder: java.lang.String toString()>();
    return $r7;
}

#main函数对应的Jimple：
public static void main(java.lang.String[]){
    java.lang.String[] r0;
    example.MethodCall3AC $r3; #由于combine函数和main函数一起进行Soot转换，r1和r2已经使用，r3作为临时变量被释放
    java.lang.System $r4;
    java.io.printStream $r5;

    r0 := @parameter0: java.lang.String[];
    $r3 = new example.MethodCall3AC;
    $r4 = java.lang.System;

    specialinvoke $r3.<example.MethodCall3AC: void <init>()>();

    virtualinvoke $r3.<example.MethodCall3AC: java.lang.String combine(java.lang.String, java.lang.String
    )>("xjtu","se");

    $r5 = finalinvoke $r4.<java.lang.System: java.io.printStream out()>();
    #$r4即为java.lang.System，out是System中的final方法

    virtualinvoke $r5.<java.io.printStream: java.lang.String println(java.io.printString)>($r3);
    return;
}
```

e.g.3 类声明与使用

```BASH
package example;
public class class3AC{
    public static final double pi = 3.14;
    public static void main(String[] args){
    
    }
}

Jimple:
public class example.class3AC extends java.lang.object{
    public static final double pi;

    public void <init>(){
        example.class3AC r0;

        r0 := @this: example.class3AC;

        specialinvoke r0.<java.lang.Object void <init>()>();

        return;
    }

    public static void main(java.lang.String[]){
        java.lang.String[] r0;

        r0 := @parameter: java.lang.String[];

        return;
    }

    public static void<clinit>(){
        <example.class3AC: double pi> = 3.14;

        return;
    }
}
```

### 关于jvm字节码中调用的四种类型

1. invokespecial: call constructor/superclass/private朝父级调用
2. invokevirtual: instance method call(virtual)朝子级调用
3. invokeinterface: checking interface implementation
4. invokestatic: call static methods

## Static Single Assignment(SSA)

each variables in assignment have a distinct name

```bash
3AC:
p = a + b
q = p - c
p = q * e
p = p * c

SSA:
p1 = a + b
q1 = p1 - c
p2 = q1 * e1
p3 = p2 * c
```

对于多条数据流存在phi-function，给不同数据流的同一个变量进行选择，同时赋给新同名变量，再进行其他变量的定义

SSA may introduce too much phi-function

## Control Flow Graph控制流图

### Basic Block

1. contains 3 addr code
2. can only be entered in the first line
3. only have one exit(do not have exit except last line)

```bash
BB1:
(1)a = input
(2)b = a + 2

BB2:
(3)c = a * b
(4)if c > 20 goto (7)

BB3:
(5)b = b + 1
(6)goto (3)

BB4:
(7)d = b / a
(8)p = b - d
(9)if d==p goto (11)

BB5:
(10) goto(3)

BB6:
(11) return
```

总的来说，BB的首行判断方法为：

（1）整段代码的第一行

（2）跳转的目标行

（3）跳转行的下一行

### Flow的建立

1. 所有BB默认跳转到下一BB，除了exit为纯goto
2. 带有goto指令的还需跳转到对应的BB

```text
     2 <------ 5
     2 <- 3
1 -> 2 -> 3
       -> 4 -> 5
            -> 6
```
