# Introduction

## Structure of Programming Language

### Theory

### Environment

### Application

1. **Program analysis(You are here!)**
2. Program verification
3. Program synthesis

## Why do we need program analysis?

1. Program Reliability
2. Program Security
3. Compiler Optimization
4. Program Understanding

## Static Analysis

analysis program P to reason about its behaviors and determines whether it satisfies some demands before running.

*Rice Theory: Any non-trival property of the behavior of programs in a r.e language is undecidable.*
*总的来说，一个完美的静态分析是不存在的*

Perfect Static Analysis can not be done:Sound&Complete

=>Sound ~= overapproximate

=>Complete ~= underapproximate

p.s:*Sound和Complete二选一：选sound则出现false positive误报，选complete则出现false negative漏报*

一般来说，我们更倾向soundness

### Necessity of Soundness

Soundness is critical in compiler optimization and program verification
