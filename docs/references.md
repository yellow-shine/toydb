# References（参考资料）

This is the main research material I used while building toyDB. It is a subset of my
[reading list](https://github.com/erikgrinaker/readings).

以下是我在构建 toyDB 时使用的主要研究资料，是我[阅读清单](https://github.com/erikgrinaker/readings)的一个子集。

## Introduction（导论）

Andy Pavlo's CMU lectures are an absolutely fantastic introduction to database internals:

Andy Pavlo 在 CMU 的课程是对数据库内部原理极其出色的入门材料：

- 🎥 [CMU 15-445 Intro to Database Systems](https://www.youtube.com/playlist?list=PLSE8ODhjZXjbohkNBWQs_otTrBTrjyohi)（CMU 15-445 数据库系统导论）(A Pavlo 2019)
- 🎥 [CMU 15-721 Advanced Database Systems](https://www.youtube.com/playlist?list=PLSE8ODhjZXjasmrEd2_Yi1deeE360zv5O)（CMU 15-721 高级数据库系统）(A Pavlo 2020)

Martin Kleppman has written an excellent overview of database technologies and concepts, while Alex
Petrov goes in depth on implementation of storage engines and distributed systems algorithms:

Martin Kleppmann 对数据库技术与概念做了出色的综述，而 Alex Petrov 则深入讲解了存储引擎与分布式系统算法的实现：

- 📖 [Designing Data-Intensive Applications](https://dataintensive.net/)（设计数据密集型应用）(M Kleppmann 2017)
- 📖 [Database Internals](https://www.databass.dev)（数据库内部原理）(A Petrov 2019)

## Raft

The Raft consensus algorithm is described in a very readable paper by Diego Ongaro, and in a talk
given by his advisor John Ousterhout:

Raft 共识算法在 Diego Ongaro 一篇非常易读的论文中提出，他的导师 John Ousterhout 也就此做过一场演讲：

- 📄 [In Search of an Understandable Consensus Algorithm](https://raft.github.io/raft.pdf)（寻找一种易于理解的共识算法）(D Ongaro, J Ousterhout 2014)
- 🎥 [Designing for Understandability: The Raft Consensus Algorithm](https://www.youtube.com/watch?v=vYp4LYbnnW8)（为可理解性而设计：Raft 共识算法）(J Ousterhout 2016)

However, Raft has several subtle pitfalls, and Jon Gjengset's student guide was very helpful in
drawing attention to these:

不过，Raft 存在一些微妙陷阱，Jon Gjengset 的学生指南在指出这些问题上非常有帮助：

- 🔗 [Students' Guide to Raft](https://thesquareplanet.com/blog/students-guide-to-raft/)（Raft 学生指南）(J Gjengset 2016)

## Parsing（解析）

Thorsten Ball has written a very enjoyable hands-on introduction to parsers where he implements
first an interpreter and then a compiler for the made-up Monkey programming language (in Go):

Thorsten Ball 写了一本非常有趣的解析器动手入门书：他用 Go 为自创的 Monkey 编程语言先实现了一个解释器，随后又实现了一个编译器：

- 📖 [Writing An Interpreter In Go](https://interpreterbook.com)（用 Go 写解释器）(T Ball 2016)
- 📖 [Writing A Compiler In Go](https://compilerbook.com)（用 Go 写编译器）(T Ball 2018)

The toyDB expression parser is inspired by a blog post by Eli Bendersky describing the precedence
climbing algorithm, which is the algorithm I found the most elegant:

toyDB 的表达式解析器受 Eli Bendersky 一篇介绍优先级爬升（precedence climbing）算法的博客文章启发，这是我认为最优雅的算法：

- 💬 [Parsing Expressions by Precedence Climbing](https://eli.thegreenplace.net/2012/08/02/parsing-expressions-by-precedence-climbing)（用优先级爬升法解析表达式）(E Bendersky 2012)

## Transactions（事务）

Jepsen (i.e. Kyle Kingsbury) has an excellent overview of consistency and isolation models, which
is very helpful in making sense of the jungle of overlapping and ill-defined terms:

Jepsen（即 Kyle Kingsbury）对一致性与隔离模型做了出色的综述，对于理清这片术语重叠、定义含糊的丛林非常有帮助：

- 🔗 [Consistency Models](https://jepsen.io/consistency)（一致性模型）(Jepsen 2016)

For more background on this, in particular on how snapshot isolation provided by the MVCC
transaction engine used in toyDB does not fit into the traditional SQL isolation levels, the
following classic papers were useful:

更多相关背景，特别是关于 toyDB 所用 MVCC 事务引擎提供的快照隔离为何无法归入传统 SQL 隔离级别，以下经典论文很有参考价值：

- 📄 [A Critique of ANSI SQL Isolation Levels](https://www.microsoft.com/en-us/research/wp-content/uploads/2016/02/tr-95-51.pdf)（对 ANSI SQL 隔离级别的批判）(H Berenson et al 1995)
- 📄 [Generalized Isolation Level Definitions](http://pmg.csail.mit.edu/papers/icde00.pdf)（广义隔离级别定义）(A Adya, B Liskov, P ONeil 2000)

As for actually implementing MVCC, I found blog posts to be the most helpful:

至于 MVCC 的具体实现，我发现博客文章最有帮助：

- 💬 [Implementing Your Own Transactions with MVCC](https://levelup.gitconnected.com/implementing-your-own-transactions-with-mvcc-bba11cab8e70)（用 MVCC 实现自己的事务）(E Chance 2015)
- 💬 [How Postgres Makes Transactions Atomic](https://brandur.org/postgres-atomicity)（Postgres 如何保证事务的原子性）(B Leach 2017)
