---
title: "详解 Bomb Lab (6)"
date: 2021-07-14T22:53:58+08:00
draft: false
tags: ["折腾", "CSAPP", "Reversing", "Bomb Lab"]
---

## 前言

这几天真的是因为工作摸鱼没有开始写 Lab-6, 完全不是太难了搞不明白。

## 工具

- [cutter](https://cutter.re)
- [GoodNotes](https://www.goodnotes.com/zh-cn/)
  
## Phase 6 —— 可惜你帅不过坂本大佬

### 第一个循环

这个 Phase 反复横跳，不过跟着它顺序执行还是能心中有数的。

从 phase_6 + 0x20 处开始到 phase + 0x5d 是第一个令人糟心的大循环，所以还是要拆开来看。

```NASM
0x00401114      movq %r13, %rbp ; move quadword ; rbp=0xffffffffffffff90
0x00401117      movl (%r13), %eax ; rax=0xffffffff
0x0040111b      subl $1, %eax ; eax=0xfffffffe ; of=0x0 ; sf=0x1 ; zf=0x0 ; pf=0x0 ; cf=0x0 ; af=0x0
0x0040111e      cmpl $5, %eax ; 5 ; zf=0x0 ; cf=0x0 ; pf=0x1 ; sf=0x1 ; of=0x0 ; af=0x0
0x00401121      jbe 0x401128 ; jump short if below or equal/not above (cf=1 or zf=1) ; unlikely
0x00401123      callq explode_bomb ; sym.explode_bomb ; rsp=0xffffffffffffff80 ; rip=0x40143a -> 0x8ec8348 sym.explode_bomb
; CODE XREF from sym.phase_6 @ 0x401121
```
容易得 rsp 所指向的 long 类型的数据应当有 $ 1 \leq x \leq 6 $，至于为何应当大于等于 1 是因为 `jbe` 的比较是无符号的。

```NASM
0x00401128      addl $1, %r12d ; r12d=0x1 ; of=0x0 ; sf=0x0 ; zf=0x0 ; cf=0x0 ; pf=0x0 ; af=0x0
0x0040112c      cmpl $6, %r12d ; 6 ; zf=0x0 ; cf=0x1 ; pf=0x0 ; sf=0x1 ; of=0x0 ; af=0x1
0x00401130      je 0x401153 ; jump short if equal (zf=1) ; unlikely
```

显然这一段是 if-break 形式的语句，猜测可以将 r12d 视为循环中常用的 `i` 变量。

接下来我们来到了循环中的一个小循环。

```NASM
0x00401132      movl %r12d, %ebx ; rbx=0x1
; CODE XREF from sym.phase_6 @ 0x40114b
0x00401135      movslq %ebx, %rax ; rax=0x1
0x00401138      movl (%rsp, %rax, 4), %eax ; rax=0xffffffff
0x0040113b      cmpl %eax, (%rbp) ; zf=0x1 ; cf=0x0 ; pf=0x1 ; sf=0x0 ; of=0x0 ; af=0x0
0x0040113e      jne 0x401145 ; jump short if not equal/not zero (zf=0) ; unlikely
0x00401140      callq explode_bomb ; sym.explode_bomb ; rsp=0xffffffffffffff78 ; rip=0x40143a -> 0x8ec8348 sym.explode_bomb
; CODE XREF from sym.phase_6 @ 0x40113e
0x00401145      addl $1, %ebx ; ebx=0x2 ; of=0x0 ; sf=0x0 ; zf=0x0 ; cf=0x0 ; pf=0x0 ; af=0x0
0x00401148      cmpl $5, %ebx ; 5 ; zf=0x0 ; cf=0x1 ; pf=0x0 ; sf=0x1 ; of=0x0 ; af=0x1
0x0040114b      jle 0x401135 ; jump short if less or equal/not greater (zf=1 or sf!=of) ; rip=0x401135 -> 0x8bc36348 ; likely
```

在这里首先给 ebx 赋值 r12d，然后关注 ebx 的变化，发现 ebx 最大也仅仅到 5 就跳出这个小循环了，那么这个 ebx 猜测作用是双层循环里头的 `j`；这么想就很简单了，我们知道栈上取出来的6个数在内存上是连续的，可以记成 `int stack[6]`，在这里就是进行了：

```NASM
mov i j
cmpl stack[j] (%rbp)
jne 0x401145
;...
```

那么 rbp 又是个甚么东西？我们看循环的开头，可以发现 rbp 指向 r13 指向的内容，那就是 stack[0] 了。当然无须担心出现 `cmpl stack[0] stack[0]`，根据 0x00401128 我们的 `j` 是一定大于 `i` - 1 的，而 `i` - 1 又是大于等于 0 的。那么一个个和 stack[0] 比有甚么用呢？

```NASM
0x0040114d      addq $4, %r13 ; r13=0xffffffffffffff94 ; of=0x0 ; sf=0x1 ; zf=0x0 ; cf=0x0 ; pf=0x0 ; af=0x0
0x00401151      jmp 0x401114 ; jump ; rip=0x401114 -> 0x41ed894c ; 回到循环开头
```

我们发现这里 r13 会向下一个 stack 中的数移动，也就是说从 stack[0] 移动到了 stack[1]。所以说，所有的数都会互相进行比较一次。

稍加融会贯通，不说整个写出来，起码也知道了输入的数有两个条件：

- 处于 1~6 之间的正整数；
- 互相之间不能相同。

~~于是我们可以爆破出来~~

## 第二个循环

所有数在之前已经 -1 的情况下对 7 求出余数。

## 载入链表

反复横跳起来了，但是跟着他就不会出问题。

```NASM
0x0040116f      movl $0, %esi ; rsi=0x0
0x00401174      jmp 0x401197 ; jump ; rip=0x401197 -> 
; ... 
; ...
; CODE XREF from sym.phase_6 @ 0x401174
0x00401197      movl (%rsp, %rsi), %ecx ; rcx=0xffffffff
0x0040119a      cmpl $1, %ecx ; 1 ; zf=0x0 ; cf=0x0 ; pf=0x0 ; sf=0x1 ; of=0x0 ; af=0x0
0x0040119d      jle 0x401183 ; jump short if less or equal/not greater (zf=1 or sf!=of) ; rip=0x401183 -> 0x6032d0ba ; likely
0x0040119f      movl $1, %eax ; rax=0x1
0x004011a4      movl $node1, %edx ; 0x6032d0 ; rdx=0x6032d0 -> 0x14c obj.node1
0x004011a9      jmp 0x401176 ; jump ; rip=0x401176 -> 0x8528b48
```

跟随这个 jmp，来到 0x401197，一来就拿 stack[0] 和 1 相比。

然后在纸上写写画画，加上自己的亿点点优化就可以写出如下代码：

```C
int i = 0;
long stack2[6];
do{
    int edx = $node1;
    if (stack[i] > 1) {
        int tmp = 1;
        do {
            edx = *(edx+8);
            tmp++;
        } while(tmp != stack[i])
    }
    stack2[i] = rdx;
    i++;
}while(i<6)
```

至于这个 `$node` 到底是甚么，用 `cutter` 双击去看只能看到一行行代码，但是事实上这些是规整的数据。要想知道这些数据的值，只能看 `objdump` 或者 `GDB` 直接查看输出。

可以用 gdb 查看得到：

```
gdb> x/x 0x6032d0+0x8
0x6032e0
gdb> x/x 0x6032e0+0x8
0x6032f0

# 是一个链表
# 查6个数就够了
0x6032d0 -> 0x6032e0 -> 0x6032f0 -> 0x603300 -> 0x603310 -> 0x603320
```

优化一下就得到了具体逻辑：

```C
int i = 0; long stack2[6];
long LOOKUP[6] = {0x6032d0, 0x6032e0, 0x6032f0, 0x603300, 0x603310, 0x603320};
do{
    index = stack[i]; // stack array get from read_six_numbers
    stack2[i] = LOOKUP[index - 1];
    i++;
} while(i != 6)
```

## 链表重建

> cutter 会将 %rsp + 0xYY 显示为 var_YYh
> 
```NASM
0x004011ab      movq var_20h, %rbx ; move quadword ; rbx=0xffffffffffffffff
0x004011b0      leaq var_28h, %rax ; rax=0x28
0x004011b5      leaq var_50h, %rsi ; rsi=0x50
0x004011ba      movq %rbx, %rcx ; move quadword ; rcx=0xffffffffffffffff
```

设上一个阶段的新数组为 long stack2[6], 可以知道有：

- stack2 = %rsp + 0x20
- rbx = stack2 = stack2[0]
- rax = stack2 + 0x8 = &stack2[1]
- rsi = stack2 + 0x30 = &stack2[6] (数组上界)
- rcx = rbx

```NASM
; CODE XREF from sym.phase_6 @ 0x4011d0
0x004011bd      movq (%rax), %rdx ; move quadword ; rdx=0xffffffffffffffff
0x004011c0      movq %rdx, 8(%rcx) ; move quadword
0x004011c4      addq $8, %rax ; rax=0x30 ; of=0x0 ; sf=0x0 ; zf=0x0 ; cf=0x0 ; pf=0x1 ; af=0x1
0x004011c8      cmpq %rsi, %rax ; zf=0x0 ; cf=0x1 ; pf=0x0 ; sf=0x1 ; of=0x0 ; af=0x0
0x004011cb      je 0x4011d2 ; jump short if equal (zf=1) ; unlikely
0x004011cd      movq %rdx, %rcx ; move quadword ; rcx=0xffffffffffffffff
0x004011d0      jmp 0x4011bd ; jump ; rip=0x4011bd -> 0x48108b48
```

经过亿番润色得到：

```cpp
int i = 1;
while(true) {
    *(stack2[i-1] + 8) = stack2[i];
    i++;
    if (i == 6) {
        break;
    }
}
```

值得一提的是这里改写的是上面提到的 LOOKUP 数组，可以认为是对 LOOKUP 数组进行了一次重排。

可以理解为，**将原来的链表按照 stack2 的存储顺序，进行一次重整！**

## 检验时刻

接下来的代码很容易就看出来逻辑是：

```C
(int)*(stack2[5]+0x8)=0;

for (int i = 0; i < 5 ; i--) {
    if ((int)*(stack2[i] + 0x8) < (int)*(stack2[i+1] + 0x8)) {
        bomb();
    }
}
```

上网查了资料才知道数组内容应当用 int 的方式去访问，以 int 方式查询内容得到：

```
gdb> x/d <addr>
0x6032d0 -> 332
0x6032e0 -> 168
0x6032f0 -> 924
0x603300 -> 691
0x603310 -> 477
0x603320 -> 443
```

因此我们知道最后的 stack2 内容一定是：

```
0x6032f0 0x603300 0x603310 0x603320 0x6032d0 0x6032e0
3   4   5   6   1   2
```

所以 stack 的内容是：

```
3   4   5   6   1   2
```

考虑到中途的对7求补，输入的六个数字应该是：

```
4   3   2   1   6   5
```

> Congratulations! You've defused the bomb!

## 后记

在二进制里头发现了一个 func7，这是怎么回事呢？等以后再看看罢！