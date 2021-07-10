---
title: "Bomb Lab"
date: 2021-07-10T14:38:43+08:00
draft: true
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

{{<mermaid>}}
sequenceDiagram
    participant main
    participant phase_x
    main ->> main: input = read_line()
    main ->> phase_x: phase_x(input)
    phase_x ->> phase_x: check input
    phase_x ->> main: exit normally
    main ->> main: go to next pahse or exit
{{</mermaid>}}