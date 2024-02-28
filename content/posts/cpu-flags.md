---
title: "简介四个考试中常见的程序状态字"
date: 2021-12-06T12:36:19+08:00
draft: false
tags: ["计算机组成原理", "考研", "程序状态字"]
categories: ["体系结构"]
---
> 本文作于作者考研期间，因为过于菜经常分不清**溢出标志**和**进位标志**之间的区别怒而提笔撰文
> 
> 结果不仅没有写完而且还忘了发了（

## 前言

程序状态字是 CPU 中实现条件指令如 `JZ`/`JE`、`CMOVBE` 的重要机制。在这里简单介绍一下考试中常见的四个程序状态字：
- `ZF` 零标志 (Zero Flag)
- `SF` 符号标志 (Sign Flag)
- `CF` 进位标志 (Carry Flag)
- `OF` 溢出标志 (Overflow Flag)

及其简单人肉计算方法。
## 零标志

**零标志**, 通常记为 `ZF`, 表示计算结果是否为 0. 如果计算结果是 0, `ZF` 会被设置为 1.

我们亦可使用 `ZF` 来判断两个数之间是否相等。如 `JZ`、`BEQ` 之类的指令即使用该标志。
```NASM
CMP %rdi, %rsi  ; SUB %rdi, %rsi
JZ <loop>   ; if %rdi - %rsi == 0, jump to <loop>
            ; jump when ZF == 1
```
## 符号标志

**符号标志**, 通常记为 `SF`, 顾名思义，该标志主要用于表示计算结果的正负性。如果计算结果为负，那么 `SF` 会被设为 1。

显然这个标志仅仅在带符号计算中有意义。在常见的使用[补码表示法](https://zh.wikipedia.org/wiki/%E4%BA%8C%E8%A3%9C%E6%95%B8)的系统中，可以使用计算结果最高位判断正负性。

```NASM
SUB %edi, %esi, %eax
JGE <loop>      ; jump to loop if %edi - %esi >= 0
                ; jump when SF == 0
                ; JGE is a signed instruction
```

## 进位标志

> In *unsigned* arithmetic, watch the carry flag to detect errors;
>
> In *signed* arithmetic, the carry flag tells you nothing interesting.

**进位标志**, 通常记为 `CF`, 往往指示在计算中是否发生了无符号借位和无符号进位。

甚么是无符号进位呢？无符号进位指的是在计算中最高位发生了进位；同理无符号借位即最高位从更高位发生了借位。

### 无符号进位

若两数相加过程中最高位向更高位置发生进位，将 `CF` 置 1.

```
0b1111 + 0b0001 --> 0b0000
CF --> 1
```

```
0b0111 + 0b0001 --> 0b1000
CF --> 0
```

### 无符号借位

若两数相减过程中向最高位的更高位发生借位，将 `CF` 置 1.

```
0b0001 - 0b0010 --> 0b1111
CF --> 1
```

```
0b1000 - 0b0001 --> 0b0111
CF --> 0
```

## 溢出标志

> In *unsigned* arithmetic, the overflow flag tells you nothing interesting;
>
> In *signed* arithmetic, watch the overflow flag to detect error.

**溢出标志**通常记为 `OF`，当有符号运算发生溢出时置 1。

### 加法溢出
若同号相加得到异号结果，则说明发生溢出。
值得一提的是负数和零应当也被看成相异号的。

```
0b1000 + 0b1000 --> 0b0000
OF --> 1
```

```
0b1011 + 0b0100 --> 0b1111
OF --> 0
```

### 减法溢出
对于减法，记住正数减负数不可能得到负数，负数减正数也不可能得到正数即可。即“`正负负，负正正`”即表明发生了溢出。
```
0b0011 - 0b1000 --> 0b1011
OF --> 1
```
```
0b1000 - 0b0001 --> 0b0111
OF --> 1
```
```
0b1000 - 0b1111 --> 0b1001
OF --> 0
```
## 参考
- [ICS 312; Processor Status and Flags Register; University of Hawaii](http://www2.hawaii.edu/~pager/312/notes/07Flags/)
- [Ian! D. Allen; The CARRY flag and OVERFLOW flag in binary arithmetic](http://teaching.idallen.com/dat2343/10f/notes/040_overflow.txt)