---
title: "Weekly Update - 2024-07-14"
date: 2024-07-15T02:23:33+08:00
draft: false
---

> My Arch Linux's fcitx5 is broken again, so this will be written in English.

It's been a long time since my last update. There are many things to share, and I'll update them in the following weeks.

## Technique

I recently had the opportunity to work with a NUMA machine (or cluster) and was astonished by the size of its available DRAM (>= 500 GiB).

The original task assigned to me seemed straightforward and not too difficult to implement. However, I encountered significant challenges with C++.

~~And then, I rewrote the whole original program in Rust.~~

Initially, the outcome was promising. With an improved codec schema, we achieved a 200% throughput boost in our development environment. The next step was to test its performance in our production environment.

It was my first time working with a NUMA machine. NUMA can be beneficial if your program can effectively utilize its locality. I assumed it would be simple, but after days of benchmarking, I discovered that network throughput was 1.2 times faster than writing to shared memory. (How could that be?)

After experimenting with various parallel patterns, I managed to achieve only a 10% performance boost at best. Considering the pre-processing and CPU usage, this was a disappointing result. My first task ended in a trial of failure.

## Amateur Radio

Using the Baofeng UV5R (a classic), I successfully made contact with `BH4HED` on 438.5MHz, WFM mode, loud and clear. (Date and time forgotten...)

Shortwave equipment is quite expensive. My stereotypes about analog devices have been reinforced. ;(

Hope to become a well-trained CW operator someday.

## Life

- Learned how some mental medications work. It was both fun and useful.
- Learning to cook.
- Learned to play the intro of _Wonderful Tonight_.
- Learned to play the first solo of _Wish You Were Here_.
