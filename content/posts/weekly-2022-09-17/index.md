---
title: "显示器与精神内耗 Weekly 2022 09 17"
date: 2022-09-17T21:21:02+08:00
draft: false
---

{{< image src="./%E7%BA%AA%E5%BF%B5.jpg" caption="为贵阳转运巴士遇难同胞致哀" alt="为贵阳转运巴士遇难同胞致哀" title="为贵阳转运巴士遇难同胞致哀" width="30%" >}}

---

这是第三篇的开源周报了。

## 技术

本周的工作依然集中在 OpenDAL 上。主要的成果有：

- 给 Redis 服务提出了方便正确的文件树组织形式
- 实现了 Redis 服务的完整非阻塞 API 支持
- 为 Redis listing 状态机实现了无 clone 的实现

在 Databend 方面，为 meta 实现了 etag 支持。

### Redis 后台服务的文件树组织

周二打开 IDE，想着自己之前的基于 SCAN 的列出目录内容的算法不禁骂道：“这设计的是个么子鬼？！”

最初设计的算法相当懒惰，如果要在服务中存储一个路径为 `/path/to/dir/file_a` 的文件的话，那么只会创建一个 `/path/to/dir/file_a` 的 key（当然实际上会分别创建带不同前缀的键值对分别用来存储具体内容和元数据）。

当想要列出一个文件夹下的子目录时，如 `/path/to/` ，此时我们期望输出且只输出 `/path/to/dir/`。即会调用 `SCAN /path/to/*`, 让 redis 返回所有匹配的键，然后由服务自己过滤出子目录。

好了，看到这里相信读者朋友们马上就会反应过来有多么💩了：

1. 如果一个文件夹下有 $N$ 个子文件夹，子文件夹中平均有 $M$ 个文件，那么每次列出的算法复杂度就会是 $O(NM)$
2. 需要在服务中手动写正则表达式精确匹配下级目录，如果平均目录长度是 $n$ 的话，复杂度难以估计，反正不是 $O(n)$
3. 为了过滤子目录文件夹，后台起码需要维护一个 BTreeSet，这时就需要至少 $O(N)$ 的空间复杂度
4. 遍历过多的 key 也浪费了 redis 存储的计算资源

当时显然是没有动脑子。于是重新使用 Redis 提供的 SET 为每个目录存储其所有子目录，类似：

```
+------------------------------------------+
|Object: /home/monika/                     |
|                                          |           SET
|child: Key: v0:k:/home/monika/           -+---------> 1) /home/monika/poem0.txt
|                                          |
|/* directory has no content  */           |
|                                          |
|metadata: Key: v0:m:/home/monika/         |
+------------------------------------------+

+------------------------------------------+
|Object: /home/monika/poem0.txt            |
|                                          |
| /*      file has no children        */   |
|                                          |
|content: Key: v0:c:/home/monika/poem0.txt-+--+
|                                          |  |
|metadata: Key: v0:m:/home/monika/poem0.txt|  |
|  |                                       |  |
+--+---------------------------------------+  |
   |                                          v
   +> STRING                                STRING
     +----------------------+              +--------------------+
     |\x00\x00\x00\x00\xe6\a|              |1JU5T3MON1K413097321|
     |\x00\x00\xf8\x00\a4)!V|              |&JU5$T!M0N1K4$%#@#$%|
     |\x81&\x00\x00\x00Q\x00|              |3231J)U_ST#MONIKA@#$|
     |         ...          |              |1557(m0N1ka3just4M  |
     +----------------------+              |      ...           |
                                           +--------------------+
```

如此便可以方便快捷地使用 `SSCAN` 遍历当前目录了。遍历一次文件夹只需要$O(N)$的时间复杂度，非常银杏。要想创建一个目录或者文件，只需一路向上创建所有的父文件夹。

现在可以思考如何删除一个文件夹了。为了防止出现游离节点最好是从文件树的底部开始删除。最自然的方法当然是从当前节点开始，使用后序遍历的方法删除所有的下级文件与文件夹。然而此时我了解到 Rust 居然没有尾递归优化？！于是依靠 VecDeque 造了一个队列和一个栈，手写了一个逆向的层序遍历实现删除。

> @xuanwo: 实现的好，不过直接返回一个 `DirNotEmpty` 错误就行了 🕶

实现起来非常顺利，嘿嘿，看来实现 list 便是手到擒来了。没想到真正的折磨这才刚刚开始。

### A pathetic lifetime

具体的协议是不可能自己写的，有库不用是笨蛋。[`redis-rs`](https://docs.rs/redis/latest/redis) 库是一个 Rust 实现的 redis 客户端库，内置一套类型体操，可以非常方便地将从 Redis 返回的数据转换成 Rust 的原生数据结构如 Vec、String、bool 以及各种数值类型。同时它实现了大部分的 Redis 命令，同时具有同步和异步 API 支持，非常方便。

`redis-rs` 当然提供了 `SSCAN` 命令的支持，并且它很贴心地将 redis 的 cursor 包装成了一个 AsyncIter，你大可只需要和其他的异步迭代器一样使用即可。

而在 `OpenDAL` 中，文件的列出是依靠 `DirStream` Trait 实现的。DirStream 本身是一个对 [`Stream`](https://docs.rs/futures/latest/futures/stream/trait.Stream.html) Trait 的包装，是一个关于文件元数据的异步迭代器。一般实现是依靠大伙喜闻乐见的异步状态机模型。下面是一个大差不差的例子：

```rust
enum State {
    // 单次迭代的起点，也往往是终态
    Idle,
    // 中间状态，基本上都在这里等待结果
    Intermediate(/* state parameters*/),
    // 目标状态，在这个状态下执行 poll_next 时会返回结果
    Yielding(/* state parameters */)
}

struct DirStream {
    // 状态
    state: State,
    // 有乜有完？
    done: bool,
    // 还有一些往往只是用来输出报错信息的玩意
    path: String,
}
impl Stream for DirStream {
    type Item = DirEntry;
    fn poll_next(mut self: Pin<&mut Self>, cx: &mut Context<'_>) -> Poll<Option<Self::Item>> {

        // 进入函数，搞一些初试化的工作...

        // 为实现状态转移，需要可变性
        match &mut self.state {
            State::Idle => {
                if self.done {
                    // 整个遍历结束了，到达终态
                    return Poll::Ready(None)
                }

                // 一些操作，进入中间态
                self.state = State::Intermediate(/* parameters */);
                return self.poll_next(cx);
            }
            State::Intermediate((inter0, inter1, /*other params*/)) => {
                let param_tuple = ready!(Pin::new(&mut inter).poll_next(cx));
                self.state = State::Yielding(param_tuple);
                return self.poll_next(cx);
            }
            State::Yielding(param_tuple) => {

                // 更改自己的状态到下次 yield 的初态

                return Poll::Ready(to_yield);
            }
        }
    }
}
```

本来以为在 `AsyncIter` 上再次进行 `Stream` 的实现应当不会特别困难，但是 `AsyncIter` 这玩意带了个生命周期，于是就导致了极致的痛苦。最初定义的 `DirStream` 如下：

```rust
struct DirStream {
    it: AsyncIter<'_, String>,
    // other fields
}
```

很遗憾，这个生命周期是省不掉的。如果为 `AsyncIter` 带上一个 `'a` 的生命周期，那么包含它的 `DirStream` 也得带一个 `'a`，因为它们基本上同生共死。

```rust
struct DirStream<'a> {
    it: AsyncIter<'a, String>,
}
```

`DirStream` 带上了 `'a`，那么实现它的函数也得带上 `'a`。这还不算完，由于 `poll_next` 操作需要一个可变引用，由此可能需要在中间态中拿出 `self.it` 的可变引用——但是已经在前边 `match &mut self` 的时候借到了一个可变引用了，这时候就不能再借一个可变引用了。

如果打算在 Stream 里为 `Pin<&mut Self>` 加生命周期，那么 rust 就会觉得这不是同一个函数，干脆甩手不干了；`Context<'_>` 的生命周期参数也是一样的。一句话，rust 不吃这一套。作为一个碰到生命周期就只会 `'a` 的新手，只能请求旁边的同学和@xuanwo，用他们无敌的生命众筹经验想想办法。

> 没关系，DirStream 会被交给用户，试试让它~~万寿无疆~~带一个 `'static` 的生命周期。

试着让它永远健康之后，我啪就站起来了，很快啊！Stream Trait 的实现半点毛病都没有了！结果一看其他 API 的实现报出来了一堆奇怪的错误，都是抱怨自己为啥活得没有它这么长的，这不公平！奇怪，别人活得长与你何关？于是我打开了 `AsyncIter` 的定义，没想到它有拳拳孝心，带着一个生它养它的网络链接的引用登堂入室。于是其他 API 容不下这尊大佛，全都炸了。

非常惭愧，我的异步 rust 知识一直非常不完全，尤其是涉及到自引用和生命周期这块。要是用 unsafe 吧，debug 起来又非常困难。搞了一个下午，快把脊椎练坏了。只能放弃。

当然，rust 编程是讲化劲的，既然 `AsyncIter` 本身就是一个 `Stream` 实现，那么用实现它的那套API写一个自己的 `Stream` 不就行了？于是转换了思维之后，直接依靠它提供的低层 API 手动调用 SSCAN 维护 cursor，写了两个小时，没有再遇到上述问题，顺利实现了上述功能。

### 一点 move 法

看一眼我实现的状态转移图，就找到了手上工作的一个优化点。

调用 SSCAN 会返回一个 cursor 值和一个字符串数组。当 `SSCAN` 返回时，就可以进入上面提到的 `Yielding` 状态。可以将 Yielding 状态表示为：

```rust
Yield(usize, Vec<String>)  // (cursor, directories)
```

每次 Poll，都会从 Vec 里取出一个元素，直到 `Vec<String>` 空，再进行下一次的 `SSCAN`。

```
                     yield
Yield(cursor, x:xs)  ----> x
    |  Poll
    v
Yield(cursor, xs)
    |  xs empty
    v
Idle
```

按道理来讲这里应该先将 `Vec<String>` pop 一下，然后将其放回。但是不幸的是前面已经说过 `self` 的可变引用已经被借出了，而 pop 则需要一个可变引用。所以只能先将其深拷贝一遍，然后将其拷贝 pop 一次，最后用拷贝代替原来的数组存到状态机里。鄙人对 rust 的优化不甚了解，想不出来这种情况下会不会被优化。

最终的修改实际上非常简单，`Vec` 容器换成可变切片`&mut [String]`后，每次进入 Yielding 把切片收缩一个长度，然后将最后一个元素作为返回值交出来。使用了模式匹配之后可以更简单。

```rust
// match &mut self => State::Yield(cursor: usize, children: &mut [String])
if let Some(child) = children.last() {
    // shrink
    self.state = State::Yield(cursor, &children[..children.len() - 1]);
    // yield one
    return Poll::Ready(Some(child))
} else {
    if cursor == 0 {
        self.done = true;
    }
    self.cursor = cursor;
    self.state = State::Idle;
    // go to Idle
    return self.poll_next(cx);
}
```

可以看出，实际上将对整个 Vector 的可变引用，转化为了对单一元素的 move 操作。从而避免了开销极大的拷贝操作。

### 总结

经过了一系列操作，成功将整个文件创建的复杂度降低到了最高$O(N)$，文件删除的复杂度降低到$O(1)$。文件列出的复杂度也从 $O(NMn)$ 改到 $O(N^2)$，最后改进到 $O(N)$。还是比较满意的。

## 生活

本周发生的事情比较少，但是是承前启后的一周。

### 精神内耗

本来以为在家里没有同龄人就足够压抑了。不过在老家的时候可能还会偶尔在下午忙里偷闲上界买杯饮料当做唯一的户外活动，到了学校里基本上就呆在寝室里，坐在椅子上感受着自己的思维腐坏。

其实早在去年考研的时候就有这种感觉，某天晚上实在感受到胸中积郁难以排除，就去学校里的宰客超市(上周刚被我的低性能代码霍霍了，不过下周再展开说罢)买了一罐啤酒，想不到在微醺的时候反而感受到了以前灵感迸发思维活跃的感觉。当即就意识到心情长期低沉不是啥好事。

到了本学期开学，可能是因为长期不和人进行线下交流，语言尤其是口语表达能力严重退化。现在基本上不指望能用嘴巴跟别人把问题讲清楚。

入学之后组寝室找了一个作息非常健康的室友，带动全寝室每天早上8点固定能起床，避免熬夜。由此也开始坚持每天早晚洗漱。不过最重要的事情是将自己的办公地点搬进了某社团实验室，感觉到自己的精神状态由此改变了。

在办公室里白嫖了[@hyiker](https://hyiker.com)的工位和 2K 显示器，另外配置了 1000Mbps 的有线网络接入。另有一大群高技术的成功人士唇枪舌剑。原来和人说上话的感觉还是挺好:)，比如可以吐槽自己上周又在仓库里拉了甚么 shit 没被 review 出来；协同他们考古，查看他们热热闹闹拆除机柜的盛况；比如你在外网乘风破浪，旁边的小伙在听机箱报警声判断是不是要擦亮金手指。这种生活化的感觉确实比较奇妙，以往都是被社交媒体直接拉进血淋淋的现实，现在反倒有一种走出掩体，面前炸开的隆隆炮声反而愈发收敛，像是燃尽的鞭炮，偶尔噗噜爆开，倒也没那么可怕了。

连续高效率地工作了一周，偶尔进入低落状态也能迅速恢复过来。诶，重新进入专注状态的感觉真好。

### Apex

本周达到了白银 I，还没升黄金，但是滋崩很快乐。
