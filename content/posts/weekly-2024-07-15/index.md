---
title: "weekly 2024 07 14"
date: 2024-07-15T02:23:33+08:00
draft: false
---

> My arch linux's fcitx5 is broken again. So this will be written in English.

It's been a long time since the last update. There are too many to be told,
I'll update them in the following weeks.

## Technique

I had a glance at a NUMA machine (or cluster), strongly astonished by the size
of its available DRAM (>= 500 TiB).

The original task for me was pretty straight forward, and should not be too
hard to implement. However, I still had a great struggle with C++ the bloody
programming language.

~~And then, I rewrite it in rust.~~

The outcome was good at first, with improved codec schema, it did achieved a
200% throughput boost on our development environment.
Then I need to test if it really works in our production environment.

It was my first time seeing a NUMA machine. NUMA could be good if your program
could drain up its locality. I did thought it would be easy, so I made some days
of benchmarking, only to find out the network throughput is even 1.2 times
faster than writing to shared memory. (how could it be?)

After switching around a few parallel patterns, ultimately, only at most 10%
percent boost could be achieved. Well, if counting on the pre-processing, and
CPUs used, that was a total defeat. My first task ends up a trail of failure.

## Amateur Radio

Using equipment Baofeng UV5R, made a successful contact with `BH4HED`,
on 438.5MHz, WFM mode, loud and clear. (Date Time forgotten...)

## Life

- Learned how some mental medicines work. It was fun and useful.

- Try to make myself looks good, it takes patience.

- Learning cooking.

- Learnt how to play the intro of _wonderful tonight_.

- Learnt how to play the first solo of _wish you were here_.
