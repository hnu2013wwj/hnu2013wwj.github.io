---
layout: page
title: A Critique of ANSI SQL Isolation Levels
tags: [Database]
excerpt_separator: <!--more-->
typora-root-url: ../
---

## A Critique of ANSI SQL Isolation Levels

在ANSI SQL-92 [MS, ANSI]（之后简称SQL-92）根据Phenomena（这个类似专有名词，不翻译了，中文意思是‘现象’）定义了SQL的隔离级别：Dirty Reads, Non-Repeatable Reads, and Phantoms。《A Critique of ANSI SQL Isolation Levels》[1]这篇paper阐述了有些Phenomena是无法用SQL-92中定义的一些隔离级别正确表征的，包括了各个基本上锁的实现。该论文讨论了SQL-92中的Phenomena中的定义模糊的地方。除此之外，还介绍了更好表征隔离级别的Phenomena，比如*Snapshot Isolation*。

### 介绍

不同的隔离级别定义了不同的在并发、吞吐量之间的取舍。较高的级别更容易正确的处理数据，而吞吐量比较低的隔离级别更低，较低的隔离级别更容易获得更高的吞吐量，却可能导致读取无效的数据和状态。SQL-92定义了4个隔离级别：(1)  READ UNCOMMITTED,  (2)  READ COMMITTED,   (3)  REPEATABLE READ, (4)  SERIALIZABLE. 

  它使用了**serializability**的定义，加上3种禁止的操作子序列来定义这些隔离级别，这些子序列被称为**Phenomena**，有以下3种：Dirty Read, Non-repeatable Read, and Phantom。规范只是说明phenomena是可能导致异常（anomalous)的动作序列。ANSI的隔离级别与lock schedulers的行为相关。一些lock scheduler允许改变lock的范围和持续的时间（一般是为优化了性能），这就导致了不符合严格的两阶段锁定。由[GLPT]引入了以下3种方式定义的Degrees of Consistency（一致性程度）：

    1. 锁(locking)；
    2. 数据流图(data flow graph)；
    3. 异常(anomalies)。通过**Phenomena**(或者叫 anomalies) 来定义隔离级别而不是基于锁。

### 隔离级别的定义

  在进入之前先来说明一种表示方法：
```
由读取(read)、写入(write),提交(commit)和中止(abort)组成的历史记录可以用简写符号表示：
  w1 [x]: 表示事务1怼数据项x的写入，数据项被修改；
  r2 [x]：表示事务2对x的读取。
  事务1满足谓词P的读取和写入一组记录分别由r1 [P]和w1 [P]表示。
  同理，事务1的提交和中止分别被记为c和a相同的方法表示。
```

### ANSI

 ANSI中使用了3中Phenomena来定义隔离级别(以Phenomena首字母P + 数字）:
* P0(Dirty Read): 可以读其它事务没有提交的数据；
* P1(Non-repeatable or Fuzzy Read): 当前事务重复读取相同的数据行前后的数据不同；
* P2(Phantom):当前事务相同条件重复读取数据行前后的数量不同(数量增加)。


用之前说过的表示方法表示一个操作序列:

```
2.1: w1[x] . . . r2[x] . . . (a1 and c2 in either order) 
```
P1可以被表述为不允许出现2.1这种情况。这个序列表明第2个事务(T2)可能读到了第1个事务(T1)中止写入的失误，就可能违法了读已提交。这个的语义是不明确的，不强调T1要中止，只是指出，如果这种情况发生可能会发生不一致。

```
2.2: w1[x]...r2[x]...((c1 or a1) and (c2 or a2) in any order) 
```
上面的这个2.2表示则是不允许有T1修改数据项x，且T2在T1提交或中止之前读取到这些数据，不强调T1中止或T2提交。2.2比2.1更宽松，2.2禁止了4种情况，而2.1只禁止了2种。将2.2 2.1分别表示为P1，A1:
```
P1: w1[x]...r2[x]...((c1 or a1) and (c2 or a2) in any order) 
A1: w1[x]...r2[x]...(a1 and c2 in any order) 
```


类似的，P2和P3也可以有类似的宽松和严格的表示:
```
P2: r1[x]...w2[x]...((c1 or a1) and (c2 or a2) in any order) 
A2: r1[x]...w2[x]...c2...r1[x]...c1
 P3: r1[P]...w2[y in P]...((c1 or a1) and (c2 or a2) any order) 
A3: r1[P]...w2[y in P]...c2...r1[P]...c1 
```
注意P3只是禁止范围内的写入。

### 分析ANSI SQL隔离级别

将P0定义为脏写，用上面的表示方法表示如下:
```
P0: w1[x]...w2[x]...((c1 or a1) and (c2 or a2) in any order)
```

考虑一个历史H1，在x和y之间转移$40：
```
H1：r1[x=50]w1[x=10]r2[x=10]r2[y=50]c2 r1[y=50]w1[y=90]c1 
```

H1是不可串形化的，这是一个经典的不一致分析。H1种T2读取到了总额为60的不一致的状态，而H1没有违法A1，A2，A3。但是对于P1，H1是显然违反了的[((c1 or a1) and (c2 or a2) in any order)不符合了]，也就是说，这里宽松的解释是准确的解释。类似的，有一下的操作序列：

```
H2: r1[x=50]r2[x=50]w2[x=10]r2[y=50]w2[y=90]c2r1[y=90]c1
```
H2是不可串行的，也是另外一个经典的不一致分析，T2读取到的总余额为140，H2没有违反P1，满足read commited。而且没有任何数据项被读取两次，也没有谓词范围内的数据被更改。 H2的问题是当T1读取y时，x的值已被修改，如果T2再次读取x，就会获得不一样的值，但是T2没有读取，A2也就不适用。用P2替代A2解决了这个问题，当w2[x=10]之后这个序列就不符合P2了。 最后，有这样一个操作序列H3:

 ```
H3: r1[P] w2[insert y to P] r2[z] w2[z] c2 r1[z] c1
 ```
这里T2执行条件搜索知道符合条件的记录，然后T2插入了新的数据，然后更新数量z，T1在读出z时数据就产生了不一致。谓词范围没有被访问两次，所以它是被A3所允许，同样，P3则没有这个问题，它不允许这样的操作。根据以上的几个例子，发现严格的A1、A2和A3有意想不到的缺点，所以认为ASNI旨在定义P1、P2和P3。

### 其它隔离级别

#### Cursor Stability(游标稳定)

Cursor Stability旨在解决丢失更新的问题。丢失更新是当T1读取数据项，然后T2(可能基于之前读到的数据)更新数据项，然后T1(可能基于之前读到的数据)更新数据项并提交时，就发生丢失的更新异常，可以表示为:
```
P4: r1[x]...w2[x]...w1[x]...c1
```
对于下面H4的操作序列，即使H4最终T2提交了操作，T2的更新数据也会丢失，
```
H4: r1[x=100] r2[x=100] w2[x=120] c2 w1[x=130] c1 
```
P4肯定不会是P0级别的，它不会产出脏写脏读，仔细对比P2盒P4，可以分析P2是包含了P4的：
```
P4: r1[x]...w2[x]...w1[x]...c1
P2: r1[x]...w2[x]...((c1 or a1) and (c2 or a2) any order) 
```
所以P4处于P1和P2之间。Cursor Stability隔离级别拓展了RC下对SQL游标的锁行为，游标上读取游标(rc)的操作要求在游标当前的数据项下保持长读锁(关于长读锁具体可以参考论文)，知道游标移动or关闭。使用引入P4C:

```
P4C: rc1[x]...w2[x]...w1[x]...c1 (Lost Update) 
```

#### Snapshot Isolation(快照隔离级别)

  在快照隔离下执行的事务始终从事务**开始时起**的已提交的数据的快照中读取数据。事务开始的时间戳称为Start-Timestamp。当事务运行在快照隔离中时，只要可以维护其开始时间戳对应的快照数据，在就不会阻塞读。事务的写入也将反映在这个快照中，如果事务第二次访问数据，则能再次读到，而这个事务开始时间戳之后的其他事务的更新对于本次事务是不可见的。
  SI隔离级别虽然不在ANSI的隔离级别中，却在实际应用非常广泛。SI是一种MVCC的技术，对于现在常见的数据库，基本都使用了MVCC。对于SI，当T1准备提交时，它将获得一个Commit-Timestamp，该值大于之前的所有时间戳。当T2提交了Commit-Timestamp在T1事务的间隔[Start-Timestamp，Commit-Timestamp]中时，只有在T1, T2数据不重叠情况下，T1事务才成功提交，否则，T1将中止，这叫做First-committer-wins。防止丢失，当T1提交时，其更改对于Start-Timestamp大于T1的Commit-Timestamp的所有事务都可见。
用之前的表示方法不能很好的表示SI的操作，因为一个数据在同一时间可能存在多个版本(多值历史和单值历史)，这里做如下的表示：

```
  H1.SI: r1[x0=50] w1[x1=10] r2[x0=50] r2[y0=50] c2 
  		   r1[y0=50] w1[y1=90] c1 
```

  所有的快照隔离的历史都可以映射到单值历史，同时保留数据流依赖性，例如，将上面的H1.SI映射到单值历史就是:
```
H1.SI.SV: r1[x=50] r1[y=50] r2[x=50] r2[y=50] c2 
  			  w1[x=10] w1[y=90] c1 
```
  Snapshot Isolation 是不可串行化的，因为读写不在一个时刻。
  有下面一个H5的操作序列:
```
 H5: r1[x=50] r1[y=50] r2[x=50] r2[y=50] 
      w1[y=-40] w2[x=-40] c1 c2 
```
这里先来看看约束违反：假设有这样一个约束: 事务在x y写入的操作保持有 x + y > 0的性质，而在SI中，T1和T2时隔离的，所以这个约束在H5中。约束违反是以恶搞通用的重要的并发异常类型。事务必须保留约束以保持一致性：如果数据库在事务启动时保持一致，则事务提交时也必须一致。

**A5 Data Item Constraint Violation(数据约束违反)** :

假设C()是数据库中一个关于 x y的约束，这里提出两种由于违反约束导致的异常:

    1. A5A,读偏。假设T1读取x，然后T2将x y更新并提交。如果现在T1再读取y，它就可能会看到不一致的状态，表示为：
```
A5A: r1[x]...w2[x]...w2[y]...c2...r1[y]...(c1 or a1)  (Read Skew) 
```

    2. A5B,写偏。假设T1读取符合约束的x y，然后T2读取x y并写入y提交，然后T1写入x，x y之间的约束就有可能被违反。
```
A5B: r1[x]...r2[y]...w1[y]...w2[x]...(c1 and c2 occur) (Write Skew) 
```

  在满足了P2的情况下，A5A和A5B都不会出现，因为A5A和A5B都存在T2写入了一个没有被T1读取的数据，造成了不可重复读。使用A5A A5B隔离级别低于可重复读。
SI的隔离是高于读以提交的，首先first-committer-wins排除了脏写入，时间戳机制下不会有脏读，因此快照隔离至少是读已提交级别。此外，加上A5A可能在满足读已提交，但却不满足快照隔离与时间戳机制，因此读已提交<快照隔离。
  SI也不会有A3，T2更新相同谓词下的数据时，T1能始终看到相同的数据，对于可重复读，则有可能遇到。在SI中，A5B时可能的，但是在可重复读中是不可能的。这样就有以下的现象: SI允许A5B禁止A3，可重复读(RR)则禁止A5B允许A3。但是，要注意的是，SI并不排除P3。考虑这样一个约束，表示由谓词确定的一组作业任务小时数不大于8。T1读取此谓词，发现总和只有7小时，就添加1小时持续时间的新任务，同时T2做同样的事情。由于T1 T2正在插入不同的数据项，First-Committer-Wins不排除此情况，这个可能发生在快照隔离中。但在P3不会出现这样的现象。

### Summary and Conclusions 

  总之，原始ANSI SQL隔离级别的定义存在许多的问题。下面是一个总结性的表格:
```
+-----------+--------+---------+---------+---------+----------+---------+---------+---------+
|Isolation  |  P0    |  P1     |  P4c    |  P4     |    P2    |   P3    |   A5A   |    A5B  |
|level      | 脏写    |  脏读   |游标丢失更新|丢失更新  | 不可重复读| 幻读     | 读偏斜   | 写偏斜   |
+-------------------------------------------------------------------------------------------+
|  读未提交  |    N   |   P     |    P    |   P     |     P    |   P     |    P    |    P    |
|           |        |         |         |         |          |         |         |         |
+-------------------------------------------------------------------------------------------+
| 读已经提交  |   N    |    N    |   P     |  P      |    P     |   P     |   P    |    P    |
|           |        |         |         |         |          |         |         |         |
+-------------------------------------------------------------------------------------------+
| 游标稳定   |  N     |    N    |   N     |   S     |    S     |    P    |    P    |    S    |
|           |        |         |         |         |          |         |         |         |
+-------------------------------------------------------------------------------------------+
| 可重复读   |   N    |    N    |   N     |   N     |    N     |    P    |   N     |    N    |
|           |        |         |         |         |          |         |         |         |
+-------------------------------------------------------------------------------------------+
|  快照隔离  |   N    |   N     |   N     |   N     |    N     |    S    |    N    |   P     |
|           |        |         |         |         |          |         |         |         |
+-------------------------------------------------------------------------------------------+
| ANSI序列化 |   N    |   N     |   N     |   N     |   N      |   N     |    N    |   N     |
|           |        |         |         |         |          |         |         |         |
+-------------------------------------------------------------------------------------------+
P: Possible
N: Not Possible
S: Sometimes Possible 
```

## 参考

1. A Critique of ANSI SQL Isolation Levels, SIGMOD'95.
2. https://en.wikipedia.org/wiki/Isolation_%28database_systems%29, 维基百科.


​			
​		
​	
