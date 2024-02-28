---
title: "Weekly 2022 09 04"
date: 2022-09-04T19:21:34+08:00
draft: false
---

受 [@xuanwo](https://github.com/xuanwo) 的推荐，现在俺也开始尝试撰写开源周报咧。以后就固定周日晚上更新吧。

# 技术

## OpenDAL

### Layer

本周阅读了 OpenDAL 的 layer 实现，原来这些 layer 是使用一个一个实现了对应 trait 的 struct 进行包装，从而实现的对应特性。

```rust
struct ToWrap {}
impl A for ToWrap {
    fn blood_pressure(&self) -> f64 {}
}
```

比如俺们现在手上已经有了一个实现了 A 接口的类 ToWrap，现在我们需要在 ToWrap 的基础上增加一些其他功能，比如在 `<ToWrap as A>::blood_pressure()` 大于 120 时打印一条信息到标准错误输出。

```rust
struct Warper {
    inner: Box<dyn ToWrap>
}

impl A for Warper {
    fn blood_pressure(&self) -> f64 {
        let b = self.inner.blood_pressure();
        if b > 120.0 {
            eprintln!("blood pressure too high!");
        }
        b
    }
}
```

非常简单。于是据此为 `OpenDAL` 的数据读写 API 如 `Reader` 和 `DirStreamer` 分别提供了跟踪(`tracing`)，日志(`logging`)以及统计(`metrics`)的支持。

### Redis

翻看 issue 目录发现了一个给 `OpenDAL` 加入 `Redis` 后端的 issue。与 `OpenCache` 有关。想到 OpenCache 已经很久没有动工了，实现一个 redis 后端支持是否会好一点？于是简单学习了 `Redis` 的使用方法，虽然去年秋招时也时有在技术文章里看到，但是并没有亲自去使用过。简单尝试后感觉 `Redis` 可能是从一个简单的 KV 存储逐渐成长起来的。

想了很久具体实现方法，但是怎么想都没有想清楚如何高效而正确地实现一个类似 `ls` 的，在本级目录中 listing 的功能。简单撰写了一篇 RFC，目前还在 review 中。具体实现可能还需要去参考参考 `JuiceFS` 之类的。

## 其它

很久没有看论文了，同事分享了一篇 VLDB 的论文，讲的是使用一种开放的编译器多层次中间表示（Multi-Level Intermediate Representation, MLIR）法，而非传统数据库的多层架构（compiler-planner-executor-backend）实现数据库，从而可以让大伙不用每一个 idea 都要从头实现一个数据库，降低数据库研究的门槛；方便与其他优化的集成，如人尽皆知的机器学习大法。目前还在看。

# 生活

1. 前前后后搞了三周，总算是搞定了银行开卡和学费支付。俺还是很怀疑，在开卡之前需要社区为你的开卡理由做背书真的可以防止他们所说的非法银行卡交易么？
2. Apex Legends 开始主玩机器人，排位上白银 II 了，但是最近水平下降明显，对枪基本没赢过。
3. 作息调整再次失败，每天还是9点半起床。
4. 网易云音乐上回归了崔健的歌曲，虽然没有[一块红布](https://youtu.be/l8UPST1ZKSw)，但是还是[快让我在雪地上撒点野](https://music.163.com/#/song?id=1962385459)！
5. 彻底丧失对学校这种集体的归属感。
