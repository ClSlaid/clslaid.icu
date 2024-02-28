---
title: "详解 Bomb Lab (1-5)"
date: 2021-07-10T14:38:43+08:00
draft: false
tags: ["折腾", "CSAPP", "Reversing", "Bomb Lab"]
categories: ["CSAPP", "体系结构", "计算机导论"]
---
## 前言

最近在过 CSAPP，花了两天时间看了看《程序的机器级表示》，俺深知硬看 x86_64 那个用🦶🏻写出来（$金^3$学长语）的 Register 和 ASM 是学不会滴。如果学不会的话，成为成功人士的道路就中断了罢；对了，那就整个很难的 Lab 掩盖过去罢！

## 概览

拿到工程文件，解压一看，一 Note，一源文件，一二进制而已：

```bash
$ ls -l
.rwxr-xr-x  bomb
.rw-r--r--  bomb.c
.rw-r--r--  note.md
```

note 中莫得任何可用信息，bomb.c 交代了大致逻辑，无非是：

{{< mermaid >}}
sequenceDiagram
    participant main
    participant phase_x
    main ->> main: input = read_line()
    main ->> phase_x: phase_x(input)
    phase_x ->> phase_x: check input
    phase_x ->> main: exit normally
    main ->> main: bomb defused
    main ->> main: go to next pahse or exit
{{< /mermaid >}}

一眼看过去大概有 6 个 Phase，Phase 里面具体是个啥都在 .h 文件里面了，然而 .h 文件缺失了。于是只能上手 Reversing ？

## 工具

先动手跑一跑看看具体有甚么。

```bash
$ ./bomb
Welcome to my fiendish little bomb. You have 6 phases with
which to blow yourself up. Have a nice day!
# 毫无头绪，ctrl-c 退了退了～
^CSo you think you can stop the bomb with ctrl-c, do you?
Well...OK. :-)
```

可以使用 Object Dump 生成具体的 ASM 到文本文件

```bash
objdump -D ./bomb > bomb.asm
```

可以使用 ~~radare2~~ CGDB 进行拆弹。CGDB 的资料[在此](https://cgdb.github.io/docs/cgdb-split.html)——当然如果你的英语和我一样烂的话也可以看[中文版](https://github.com/leeyiw/cgdb-manual-in-chinese)

也可以使用 `radare2` 或者 `rizin` 拆弹，本文后半部分使用 `cutter` （`rizin` 的图形前端）拆弹。

## Phase 1——热身

> Welcome to my fiendish little bomb. You have 6 phases with which to blow yourself up. Have a nice day!

观察 asm 文件中的 phase_1 过程：

```NASM
0000000000400ee0 <phase_1>:
  400ee0:	48 83 ec 08          	sub    $0x8,%rsp
  400ee4:	be 00 24 40 00       	mov    $0x402400,%esi
  400ee9:	e8 4a 04 00 00       	call   401338 <strings_not_equal>
  400eee:	85 c0                	test   %eax,%eax
  400ef0:	74 05                	je     400ef7 <phase_1+0x17>
  400ef2:	e8 43 05 00 00       	call   40143a <explode_bomb>
  400ef7:	48 83 c4 08          	add    $0x8,%rsp
  400efb:	c3                   	ret    
```

我们可以发现一个奇怪的地址 `0x402400`，这个过程将该地址塞进了 `%esi` 中，然后调用了 `strings_not_equal`。

但是 `%rsi` 是第二个参数，那么第一个参数是什么呢？第一个参数是 `%rdi`，也就是我们向 `phase_1` 传入的参数，即变量 `input`，字符串的起始地址。

也就是说，phase_1 调用了 strings_not_equal(input, 0x402400)，比较的结果返回到 %eax，然后使用 TEST 指令。如果 TEST 的结果为 ZF==0，那么就会跳转到 0x400ef7，自然就完成拆弹了。

所以我们可以查看 0x402400 的具体内容，想必是个字符串罢。

```text
(gdb)x/s 0x402400
0x402400:        "Border relations with Canada have never been better."
```

拿到 ~~flag~~ 字符串，走人。

> Phase 1 defused. How about the next one?

## Phase 2——参数顺序与参数压栈

看看 ASM？Emmm?

```NASM
0000000000400efc <phase_2>:
  400efc:	55                   	push   %rbp
  400efd:	53                   	push   %rbx
  400efe:	48 83 ec 28          	sub    $0x28,%rsp
  400f02:	48 89 e6             	mov    %rsp,%rsi
  400f05:	e8 52 05 00 00       	call   40145c <read_six_numbers>
  400f0a:	83 3c 24 01          	cmpl   $0x1,(%rsp)
  400f0e:	74 20                	je     400f30 <phase_2+0x34>
```

注意 %rsp 复制到了 %rsi。

read_six_numbers? 让我看看！

```NASM
000000000040145c <read_six_numbers>:
  40145c:	48 83 ec 18          	sub    $0x18,%rsp
  401460:	48 89 f2             	mov    %rsi,%rdx
  401463:	48 8d 4e 04          	lea    0x4(%rsi),%rcx
  401467:	48 8d 46 14          	lea    0x14(%rsi),%rax
  40146b:	48 89 44 24 08       	mov    %rax,0x8(%rsp)
  401470:	48 8d 46 10          	lea    0x10(%rsi),%rax
  401474:	48 89 04 24          	mov    %rax,(%rsp)
  401478:	4c 8d 4e 0c          	lea    0xc(%rsi),%r9
  40147c:	4c 8d 46 08          	lea    0x8(%rsi),%r8
  401480:	be c3 25 40 00       	mov    $0x4025c3,%esi
  401485:	b8 00 00 00 00       	mov    $0x0,%eax
  40148a:	e8 61 f7 ff ff       	call   400bf0 <__isoc99_sscanf@plt>
  40148f:	83 f8 05             	cmp    $0x5,%eax
  401492:	7f 05                	jg     401499 <read_six_numbers+0x3d>
  401494:	e8 a1 ff ff ff       	call   40143a <explode_bomb>
  401499:	48 83 c4 18          	add    $0x18,%rsp
  40149d:	c3                   	ret    
```

这个过程首先给 rsp - 0x18，也就是申请了 16 + 8 = 24 bytes 的栈空间，联想到这个函数的名字是“读六个数”，显然是读 6 个 int 进来。下面 0x40148a 调用了 sscanf，应当是读取 6 个数字的函数。如果是和我一样用惯了 `iosteam` 的 C with Class 人可以上网查询 sscanf 的函数签名：

```C
int sscanf(const char *str, const char *format, ...)
```

这个函数从第一个参数 `*str` 读取内容，依靠 `*format` 作为匹配模版，将匹配得到的结果赋值给后面的参数，返回匹配到的值的个数。

可知第一个参数是 `%rdi`，也就是输入的字符串；自然是看 %rsi 了，在 0x401480 给 %rsi 赋值了 0x4025c3，一看内容不出所料的是“%d %d %d %d %d %d”，匹配 6 个数字。接下来将返回值和 5 相比，可知输入数字数目必须等于 6。否则炸弹会爆炸。

剩下的参数就是匹配到的数字了。这整个过程中，数字的去向如下：

| 输入顺序 | 参数位置 | 地址寄存器 | 栈上位置 |
| :---: | :--: | :---: | :---: |
| 1 | arg3 | rdx | %rsi |
| 2 | arg4 | rcx | %rsi + 0x4 |
| 3 | arg5 | r8 | %rsi + 0x8 |
| 4 | arg6 | r9 | %rsi + 0xC |
| 5 | arg7 | (%rsp) | %rsi + 0x10  |
| 6 | arg8 | (%rsp) + 0x8 | %rsi + 0x14 |

按顺序叫这些数 num1 ~ num6 好了。

然后继续看 Phase_2 的 asm:

```NASM
  400f0a:	83 3c 24 01          	cmpl   $0x1,(%rsp)
  400f0e:	74 20                	je     400f30 <phase_2+0x34>
  400f10:	e8 25 05 00 00       	call   40143a <explode_bomb>
  400f15:	eb 19                	jmp    400f30 <phase_2+0x34>
  400f17:	8b 43 fc             	mov    -0x4(%rbx),%eax
  400f1a:	01 c0                	add    %eax,%eax
  400f1c:	39 03                	cmp    %eax,(%rbx)
  400f1e:	74 05                	je     400f25 <phase_2+0x29>
  400f20:	e8 15 05 00 00       	call   40143a <explode_bomb>
  400f25:	48 83 c3 04          	add    $0x4,%rbx
  400f29:	48 39 eb             	cmp    %rbp,%rbx
  400f2c:	75 e9                	jne    400f17 <phase_2+0x1b>
  400f2e:	eb 0c                	jmp    400f3c <phase_2+0x40>
  400f30:	48 8d 5c 24 04       	lea    0x4(%rsp),%rbx
  400f35:	48 8d 6c 24 18       	lea    0x18(%rsp),%rbp
  400f3a:	eb db                	jmp    400f17 <phase_2+0x1b>
  400f3c:	48 83 c4 28          	add    $0x28,%rsp
  400f40:	5b                   	pop    %rbx
  400f41:	5d                   	pop    %rbp
  400f42:	c3                   	ret    
```

容易得到 num1 应当为 1。接下来：

1. 跳到 400f30 将 %rsp+0x4 (&num2) 赋值给 %rbx，%rsp + 0x18 赋值给 %rbp。
2. 跳到 400f17 将 (%rsp) (num1) 赋值给 %eax，从下面的比较可知要求 2*num1 == num2，可知 num2 == 2。
3. 跳到 400f25，%rbx += 4 变为 &num3，显然和 rbp 不相等，跳到 400f17；

    可以知道，这是一个 do-while 型的循环结构。每一个数字都应该是前一个数字的两倍。
4. 当 %rbx == %rbp - 4 时，跳出循环。

于是得到各个地址对应的值：

| 地址 | 数字 | 值 |
| :--: | :--: | :--: |
| %rsp | num1 | 1 |
| %rsp + 0x4 | num2 | 2 |
| %rsp + 0x8 | num3 | 4 |
| %rsp + 0xc |num4| 8 |
| %rsp + 0x10 |num5| 16 |
| %rsp + 0x14 |num6| 32 |

所以 flag 为：(高亮部分)

```text
1 2 4 8 16 32
```

> That's number 2.  Keep going!

## Phase_3——switch

> 在这里开始使用 [cutter](https://cutter.re/) 进行拆弹工作

```nasm
0000000000400f43 <phase_3>:
  400f43:	48 83 ec 18          	sub    $0x18,%rsp
  400f47:	48 8d 4c 24 0c       	lea    0xc(%rsp),%rcx
  400f4c:	48 8d 54 24 08       	lea    0x8(%rsp),%rdx
  400f51:	be cf 25 40 00       	mov    $0x4025cf,%esi
  400f56:	b8 00 00 00 00       	mov    $0x0,%eax
  400f5b:	e8 90 fc ff ff       	call   400bf0 <__isoc99_sscanf@plt>
  400f60:	83 f8 01             	cmp    $0x1,%eax
  400f63:	7f 05                	jg     400f6a <phase_3+0x27>
  400f65:	e8 d0 04 00 00       	call   40143a <explode_bomb>
```

利用 GDB 查看 $0x4025cf 显示 "%d %d"，显然本题读入两个数字到 %rsp+0x8 和 %rsp+0xc。令这两数分别为 num1 和 num2。

```NASM
  400f6a:	83 7c 24 08 07       	cmpl   $0x7,0x8(%rsp)
  400f6f:	77 3c                	ja     400fad <phase_3+0x6a>
  400f71:	8b 44 24 08          	mov    0x8(%rsp),%eax
  400f75:	ff 24 c5 70 24 40 00 	jmp    *0x402470(,%rax,8)
  400f7c:	b8 cf 00 00 00       	mov    $0xcf,%eax
  400f81:	eb 3b                	jmp    400fbe <phase_3+0x7b>
  400f83:	b8 c3 02 00 00       	mov    $0x2c3,%eax
  400f88:	eb 34                	jmp    400fbe <phase_3+0x7b>
  400f8a:	b8 00 01 00 00       	mov    $0x100,%eax
  400f8f:	eb 2d                	jmp    400fbe <phase_3+0x7b>
  400f91:	b8 85 01 00 00       	mov    $0x185,%eax
  400f96:	eb 26                	jmp    400fbe <phase_3+0x7b>
  400f98:	b8 ce 00 00 00       	mov    $0xce,%eax
  400f9d:	eb 1f                	jmp    400fbe <phase_3+0x7b>
  400f9f:	b8 aa 02 00 00       	mov    $0x2aa,%eax
  400fa4:	eb 18                	jmp    400fbe <phase_3+0x7b>
  400fa6:	b8 47 01 00 00       	mov    $0x147,%eax
  400fab:	eb 11                	jmp    400fbe <phase_3+0x7b>
  400fad:	e8 88 04 00 00       	call   40143a <explode_bomb>
  400fb2:	b8 00 00 00 00       	mov    $0x0,%eax
  400fb7:	eb 05                	jmp    400fbe <phase_3+0x7b>
  400fb9:	b8 37 01 00 00       	mov    $0x137,%eax
  400fbe:	3b 44 24 0c          	cmp    0xc(%rsp),%eax
  400fc2:	74 05                	je     400fc9 <phase_3+0x86>
  400fc4:	e8 71 04 00 00       	call   40143a <explode_bomb>
  400fc9:	48 83 c4 18          	add    $0x18,%rsp
  400fcd:	c3                   	ret    
```

首先让 num1 和 0x7 进行比较，如果大于 0x7 的话炸弹会引爆；如果不大于7，那么将会来到一个跳转表。

我们复习书上的图 3.3 可以知道 0x400f5 处的地址为 0x402470 + num1*8，利用 cutter 查看对应地方的值，然后人肉反汇编，很快啊！

```C
int temp = num1; // temp in %rax
switch(c){
    case 0: { // 0d4198268  0x400f7c
        temp = 0xcf;
        break;
    }
    case 1: { // 0d4198329 0x400fb9
        temp = 0x137;
        break;
    }
    case 2: { // 0d4198275 0x400f83
        temp = 0x2c3;
        break;
    }
    case 3: { // 0d4198282 0x400f8a
        temp = 0x100;
        break;
    }
    case 4: { // 0d4198289 0x400f91
        temp = 0x185;
        break;
    }
    case 5: { // 0d4198296 0x400f98
        temp = 0xce;
        break;
    }
    case 6: { // 0d4198303 0x400f9f
        temp = 0x2aa;
        break;
    }
    case 7: { // 0d4198310 0x400fa6
        temp = 0x147;
        break;
    }
}
```

出跳转表后，来到 0x400fbe，接下来比较 temp 与 num2，倘若 temp 与 num2 相等则拆除成功。结果集那就很显然了嘛：

$$\lbrace \langle 0, 207 \rangle,
\langle 1, 311 \rangle,
\langle 2, 707 \rangle,
\langle 3, 256 \rangle,
\langle 4, 289 \rangle,
\langle 5, 206 \rangle,
\langle 6, 682 \rangle,
\langle 7, 327 \rangle \rbrace $$

> Halfway there!

## Phase_4——递归

到了这里差不多都熟练起来了罢。

```NASM
000000000040100c <phase_4>:
  40100c:	48 83 ec 18          	sub    $0x18,%rsp
  401010:	48 8d 4c 24 0c       	lea    0xc(%rsp),%rcx
  401015:	48 8d 54 24 08       	lea    0x8(%rsp),%rdx
  40101a:	be cf 25 40 00       	mov    $0x4025cf,%esi
  40101f:	b8 00 00 00 00       	mov    $0x0,%eax
  401024:	e8 c7 fb ff ff       	call   400bf0 <__isoc99_sscanf@plt>
  401029:	83 f8 02             	cmp    $0x2,%eax
  40102c:	75 07                	jne    401035 <phase_4+0x29>
  40102e:	83 7c 24 08 0e       	cmpl   $0xe,0x8(%rsp)
  401033:	76 05                	jbe    40103a <phase_4+0x2e>
  401035:	e8 00 04 00 00       	call   40143a <explode_bomb>
  ```

稍有常识的人，一眼就能看出在这里输入了两个数 num1（%rsp+0x8） 和 num2（%rsp+0xc）；num1 <= 0xe。

  ```NASM
  40103a:	ba 0e 00 00 00       	mov    $0xe,%edx
  40103f:	be 00 00 00 00       	mov    $0x0,%esi
  401044:	8b 7c 24 08          	mov    0x8(%rsp),%edi
  401048:	e8 81 ff ff ff       	call   400fce <func4>
  40104d:	85 c0                	test   %eax,%eax
  40104f:	75 07                	jne    401058 <phase_4+0x4c>集合集合
  401051:	83 7c 24 0c 00       	cmpl   $0x0,0xc(%rsp)
  401056:	74 05                	je     40105d <phase_4+0x51>
  401058:	e8 dd 03 00 00       	call   40143a <explode_bomb>
  40105d:	48 83 c4 18          	add    $0x18,%rsp
  401061:	c3                   	ret    
```

在这里可以知道调用了 func4(num1, 0x0, 0xe)，要求返回值必须为 0；同时可以知道 num2 的值只能是 0。接下来查看 func4 的具体内容。

```nasm
0000000000400fce <func4>:
  400fce:	48 83 ec 08          	sub    $0x8,%rsp
  400fd2:	89 d0                	mov    %edx,%eax
  400fd4:	29 f0                	sub    %esi,%eax
  400fd6:	89 c1                	mov    %eax,%ecx
  400fd8:	c1 e9 1f             	shr    $0x1f,%ecx
  400fdb:	01 c8                	add    %ecx,%eax
  400fdd:	d1 f8                	sar    %eax
  400fdf:	8d 0c 30             	lea    (%rax,%rsi,1),%ecx
  400fe2:	39 f9                	cmp    %edi,%ecx
  400fe4:	7e 0c                	jle    400ff2 <func4+0x24>
  400fe6:	8d 51 ff             	lea    -0x1(%rcx),%edx
  400fe9:	e8 e0 ff ff ff       	call   400fce <func4>
  400fee:	01 c0                	add    %eax,%eax
  400ff0:	eb 15                	jmp    401007 <func4+0x39>
  400ff2:	b8 00 00 00 00       	mov    $0x0,%eax
  400ff7:	39 f9                	cmp    %edi,%ecx
  400ff9:	7d 0c                	jge    401007 <func4+0x39>
  400ffb:	8d 71 01             	lea    0x1(%rcx),%esi
  400ffe:	e8 cb ff ff ff       	call   400fce <func4>
  401003:	8d 44 00 01          	lea    0x1(%rax,%rax,1),%eax
  401007:	48 83 c4 08          	add    $0x8,%rsp
  40100b:	c3                   	ret    
```

一开始就拿 arg3 和 arg2 做差，在里面又搀了一个递归调用，联想到参数在 0x0 和 0xe 之间，难道是个二分查找？那么返回的应该是个 index，我猜答案是 (0, 0)。

可以手动照着汇编进行迫 真 反 编 译，然后~~结合猜测~~优化你的表达结果。

```C
int func4(int arg1, int arg2, int arg3) {
    int temp1 = (arg3 - arg2)/2;
    temp2 = temp1 + arg2; 
    // temp2 = average(arg2, arg3)
    // calculate like this could prevent overflowing.
    if (temp2 <= arg1) {
        temp1 = 0;
        if (temp2 >= arg1) {
            return temp1;
        }
        arg2 = temp2 + 1;
        return 2*func4(arg1, arg2, arg3) + 1;
    }else{
        arg3 = temp2 - 1;
        return 2*func4(arg1, arg2, arg3);
    }
    return func4(arg1, arg2, arg3)
}
```

照着汇编写一段C语言，注意 SAR 和 SHR 是等同的，SAR rd 等价于 SAR 1 rd。常用右移操作作为快速的除法操作。

算了算，结果真的是 <0, 0>。

> So you got that one.  Try this one.

## Phase_5 —— 字符串

```NASM
0000000000401062 <phase_5>:
  401062:	53                   	push   %rbx
  401063:	48 83 ec 20          	sub    $0x20,%rsp
  401067:	48 89 fb             	mov    %rdi,%rbx
  40106a:	64 48 8b 04 25 28 00 	mov    %fs:0x28,%rax
  401071:	00 00 
  401073:	48 89 44 24 18       	mov    %rax,0x18(%rsp)
  401078:	31 c0                	xor    %eax,%eax
  40107a:	e8 9c 02 00 00       	call   40131b <string_length>
  40107f:	83 f8 06             	cmp    $0x6,%eax
  401082:	74 4e                	je     4010d2 <phase_5+0x70>
  401084:	e8 b1 03 00 00       	call   40143a <explode_bomb>
```

过程一开始往 %rsp + 0x18 塞了一个 %fs 寄存器里面弄出来的值，`cutter` 提示这是个 `canary` 值，所以大可以忽略它；同时这里需要你输入六个字符，不多不少；注意这里将 input 的起始地址放入了 %rbx 中。

```NASM
  40108b:	0f b6 0c 03          	movzbl (%rbx,%rax,1),%ecx
  40108f:	88 0c 24             	mov    %cl,(%rsp)
  401092:	48 8b 14 24          	mov    (%rsp),%rdx
  401096:	83 e2 0f             	and    $0xf,%edx
  401099:	0f b6 92 b0 24 40 00 	movzbl 0x4024b0(%rdx),%edx
  4010a0:	88 54 04 10          	mov    %dl,0x10(%rsp,%rax,1)
  4010a4:	48 83 c0 01          	add    $0x1,%rax
  4010a8:	48 83 f8 06          	cmp    $0x6,%rax
  4010ac:	75 dd                	jne    40108b <phase_5+0x29>
```

之后清零 %eax 来到 0x40108b，进入了一个 do-while 循环（0x40108b~0x4010ac）。在这里看到一个可疑的地址 0x4024b0，可以用 cutter 查看：

```string
"maduiersnfotvbylSo you think you can stop the bomb with ctrl-c, do you?"
```

人肉反编译出来：

```C
char* stk_str = rsp + 0x10;
do{
    char ecx = input[rax];
    *rsp = ecx;
    edx = ecx%16;
    edx += 0x4024b0;
    stk_str[rax] = *(edx%16);
    rax++;
} while (rax != 0x6);
```

```NASM
  4010ae:	c6 44 24 16 00       	movb   $0x0,0x16(%rsp)
  4010b3:	be 5e 24 40 00       	mov    $0x40245e,%esi
  4010b8:	48 8d 7c 24 10       	lea    0x10(%rsp),%rdi
  4010bd:	e8 76 02 00 00       	call   401338 <strings_not_equal>
  4010c2:	85 c0                	test   %eax,%eax
  4010c4:	74 13                	je     4010d9 <phase_5+0x77>
  4010c6:	e8 6f 03 00 00       	call   40143a <explode_bomb>
  4010cb:	0f 1f 44 00 00       	nopl   0x0(%rax,%rax,1)
  4010d0:	eb 07                	jmp    4010d9 <phase_5+0x77>
```

可知，我们需要将字符串 stk_str 和 0x40245e 处的字符串相比较，可知 0x40245e 处的字符串为“flyers”。在字符串里的偏移量分别为 9, 15, 14, 5, 6, 7。查阅 ASCII 码表可以得到一个结果“IONEFG”，代入检查结果。

> Good work!  On to the next...

## Phase_6

这个 Phase 真的好长长长长长长长长长啊！

更到[这里](/csapp-bomb-lab-bravo)罢。