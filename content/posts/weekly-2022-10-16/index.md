---
title: "鍠滆繋銆€20銆€澶� Weekly 2022 10 16"
date: 2022-10-17T00:05:57+08:00
draft: false
---

## 技术

本周的输出数量较少，主要集中在 OpenDAL 与 Databend 之中：

### OpenDAL

1. 在 OpenDAL 中实现了阿里云对象存储支持，在实现 OSS 的过程中耗费了许多不必要的时间，感觉有必要去弄个 Copilot 玩玩。
2. 一些微不足道的对 CI 的提升。

### Databend

1. `COPY INTO` from OSS backend
2. 完全的在 OSS 上运行的支持

### Iceberg

学习了 [`Apache Iceberg`](https://iceberg.apache.org) 格式。在越来越火热的数据湖浪潮中，越来越多的数据库支持从外部数据文件如 CSV、Parquet、XLSX 等数据文件导入数据，提供给上层的应用进行数据分析。然而数据文件有其缺点，数据仓库从外部读取数据文件需要在读取时保存文件的元数据，再应用文件模式，这就对性能造成了不小的影响。
表格式是一种显示定义了数据文件对应的数据库表、元数据以及组成数据库表的文件。无须在查询时再分析数据文件的模式，从而可以提供更高的性能。

Hive 是一种常用的表格式，本质上是将一个文件夹中的所有文件数据组织起来，提供一种类似数据库表的视图提供给用户，从而可以在其上运行 SQL。但是 Hive 用起来非常不爽：试问，谁希望在自己的读取中还需要去管里头的分区是啥样呢？

Iceberg 提供了更高的抽象，提供了一个更高层次的逻辑视图，用户无需关注 Iceberg 的具体分区；同时，Iceberg 的 catalog 机制，分离了读写，塑造了一种类似 CoW 的视图，使得并发读写的实现更加简单：Hive 的读写虽然是原子性的，但是原子性仅限于单个分区上的读写，多个分区上的安全读写就需要耗不少功夫了。
Iceberg 相比 Hive 的性能也更高，因为它无需在每次读写时都进行一次关于底层 n 个分区的复杂度为 O(n) 的文件扫描。

目前 OLAP 领域如 Snowflake、Dremio 等产品都提供了 Iceberg 的支持。本周尝试为 Databend 提出了 Iceberg 支持的 RFC，目前还在 review 途中。

### 北邮人团队

在北邮人团队接了一个上传图片的活。我原本以为大干快上只是一种传说，但是一个主力用 C 语言但是学习 rust 只有 3 天的凶猛硬件人士在 copilot 的帮助下实现出数千行的 CRUD，不觉得这很酷吗？作为一个工科生我觉得真是太酷炫了，很符合我对未来生活的想象，科技并带着趣味。

只是代码里头充斥着：

```rust
let mut db_connection = match pool.acquire().await {
    Ok(conn) => conn,
    Err(e) => return Err(e)
};

let outcome = match db_connection.query(built_query).await {
    Ok(res) => res,
    Err(e) => return Err(e)
}

drop(db_connection)
```

算了，用 rustfmt 和 clippy 提了个 +1213 -1475 的 PR。首先完成代码的焕新工作。;(

## 生活

### 鍠滆繋銆 €20 銆 € 澶 �

在北邮人团队办公室组织观看了 20 銆 € 澶 � 的记者会，认真听取了 20 銆 € 澶 � 记者会所作的总结。我深刻认识到，20 銆 € 澶 � 是非常重要的，承前启后的。

### 台式电脑装机

组装完成了台式电脑：

1. AMD 5800x3D
2. DDR4 32G RAM x2
3. AMD 6800x ~Mining~ Graphic Card
4. 1T SSD x2

尝试将工作切换到 WSL 中，祝我好运...
