# 数据库内核杂谈（九）：开源优化器 ORCA

- 顾仲贤

- 田晓旭

- 2020 年 2 月 03 日

- 本文字数：7355 字

  阅读完需：约 24 分钟

![数据库内核杂谈（九）：开源优化器ORCA](https://static001.infoq.cn/resource/image/00/06/0094811e65dd9a2cd26f1bfccf4d4406.jpg)

**本篇文章选自**[**数据库内核杂谈**](https://www.infoq.cn/theme/46)**系列文章。**



前两期文章介绍了数据库优化器的一些技术细节。这篇文章，我们通过介绍一款真实的，开源的，已经搭载在生产环境中的数据库优化器 ORCA，带大家从工程实践的角度来了解数据库优化器，也算是对优化器内容的一个小结。



今天介绍的内容主要来自于 2014 年 SIGMOD 的[ORCA paper](https://15721.courses.cs.cmu.edu/spring2016/papers/p337-soliman.pdf) (本人也是作者之一，沾沾自喜一波)。ORCA 是 Pivotal(原 Greenplum)公司构建的。



Github link 是:https://github.com/greenplum-db/gporca。



> *注：本期内容中英文结合比较多，因为有些词汇我实在是不知道如何翻译，请各位读者见谅。*



## ORCA 的诞生

ORCA 项目大致立项于 2011-2012 年(因为 12 年我正好去 Pivotal 实习，当时 ORCA 项目应该是刚开始没多久)。



那个时候，大数据风口才起来。Hadoop，HDFS 等词也开始频频出现。Cloudera 作为第一个以 Hadoop-based 的大数据公司逐渐崭露头角。我想， Pivotal 开始准备花精力来重新研发一款优化器，也是瞄准了这个风口；由于数据量爆炸式地增长，原有的优化器对于海量数据的处理已经捉襟见肘。但同时，客户对于运行复杂的分析语句的需求却越来越高：不仅希望可以支持更复杂，更庞大的数据处理，甚至希望时间上能更快。当时 Pivotal 旗下是有两款大数据产品：



**1）Greenplum Database(GPDB)**是一款基于开源 PostgreSQL 扩展的 MPP(massively parallel processing)，可支持大规模水平扩展的分布式数据库。 GPDB 采用的是 master-worker 模式，每个 worker process 运行在不同的机器上，拥有各自的存储和运算资源。客户端通过 master 把查询语句分发到各个机器上，以达到并行计算来处理海量数据。



![img](https://static001.infoq.cn/resource/image/25/18/254aeb7ff367a59df5ee4056fcda5818.png)



上图展示了 GPDB 的架构图。Master 节点管理所有的 worker 节点，worker 节点负责存储和处理数据，所有这些节点构成一个逻辑数据库。用户只和 Master 节点交互来发送 SQL 语句：当用户提交查询语句给 master 节点后，master 会根据数据分布进行优化，把最终的执行计划发给各个 worker 执行，执行的过程中各个 worker 之间也会有数据交互。最终结果会返回给 master 再返回给客户。



**2) HAWQ：**针对 Hadoop 存储的 SQL 执行引擎。HAWQ 通过数据接口可以直接读取 Hive 表里的数据(也支持原生存储格式)，然后用 SQL 执行引擎来计算得到查询结果。与 HiveQL 通过把 SQL 解析成一连串的 MapReduce job 的执行模式相比，速度要快好几个量级。HAWQ 虽然在开发执行引擎过程中借鉴了很多 GPDB 的东西，但毕竟是一款不同的数据库引擎，Pivotal 因此希望有一款兼容的优化器能够服务于它。



另一方面，虽然关于优化器的研究一直在进行，但是大部分的工作都是对于原始优化器的修修补补，低垂的果实也摘得差不多了。如果还要强行在原先的优化器上加入新的功能，就有点事倍功半了。



基于上述这些原因，Pivotal 决定开发一款新的，最先进的优化器 ORCA。



## ORCA 的架构

在 ORCA 构建的伊始，就制定了如下这些目标：



1）模块化：开发的第一天起，ORCA 就完全以一个独立的模块进行开发。所有的数据输入和输出都接口化。这也是为了能够让 ORCA 能够兼容不同的数据库产品。



2）高延展性：算子类，优化规则(transformation rule)，数据类型类都支持扩展，使得 ORCA 可以不断迭代。方便更高效地加入新的优化规则。



3）高并发优化：ORCA 内部实现了可以利用多核并发的调度器来进一步提高对复杂语句的优化效率。



4）  可验证和测试性：  在构建 ORCA 的同时，为了 ORCA 的可验证性和可测试性，同时构建了一批测试和验证工具，确保 ORCA 一直在正确的道路上迭代。



绝大部分的优化器都是紧耦合于数据库系统的。简单来说，就是代码耦合度高。比如共享数据结构，可以直接调用内部方法等。而 ORCA 为了能够适配不同的数据库引擎，一大特点就是把自己做成了一个完全独立运行于数据库系统之外的程序。做个类比，可以把 ORCA 想象成一个微服务，独立运行，只能通过暴露的 RESTAPI 进行交互。DXL(Data eXchange Language)就是 ORCA 暴露出的接口: DXL 定制了一套基于 XML 语言的数据交互接口。这些数据包括：用户输入的查询语句，优化器输出的执行计划，数据库的元数据及数据分布等。



![img](https://static001.infoq.cn/resource/image/13/c7/135f3093de234752008e7f2a8b7187c7.png)



上图给出了 ORCA 通过 DXL 与数据库系统交互的示例：数据库输入 DXL Query 和 DXL Metadata，然后 ORCA 返回 DXL plan。任何数据库系统只要实现 DXL 接口，理论上都可以使用 ORCA 进行查询优化。DXL 接口的另一个好处就在于，大幅度简化了优化器的测试，验证 bug 修复的难度。只需要通过 DXL 输入 mock(假数据)数据，就可以让 ORCA 进行优化并输出执行结果，而不需要真正搭建一个实体数据库来操作。举个例子，比如传统情况下要测试 TPC-DS 中的一个语句优化在 100TB 流量 20 个服务器下的表现，就需要搭建一个 20 个服务器的环境外加把 100TB 的数据导入进行测试。这样的测试成本无疑是很高的。但如果测试 ORCA，只需要通过 DXL 来 mock 一个测试环境，让 ORCA 给出相应的优化结果即可，甚至都不用启动数据库就能进行测试。在后续的工具章节会详细介绍。



看完了 ORCA 的交互机制，我们通过 ORCA 的架构图来详解它的架构机制。



![img](https://static001.infoq.cn/resource/image/c0/e4/c0b48e6239f67a050b2e19fd130eb2e4.png)



ORCA 的架构分成几大块：



### 1）Memo

用来存储执行计划的搜索空间的叫 Memo。Memo 就是一个非常高效的存储搜索空间的数据结构。它有一系列的集合(group)构成。每个 group 代表了执行计划的一个子表达式(对应于查询语句的一个子表达式)。不同的 group 又产生相互依赖的关系。根 group 就代表整个查询语句。举个例子，假设语句是 selct * from table1 join table2 on (table1.col1 = table2.col1)



那 memo 就由 3 个 group 构成。根 group 就是 join。 Group1 是 table scan of table1, group2 是 table scan of table2. 每个 group 除了表达抽象的语句表达式，在优化过程中，还会加入具体的物理算子。我们暂且不深入，到后面再细说。



### 2）Search&Job scheduler

ORCA 实现了一套算法来扫描 Memo 并计算得到预估代价最小的执行计划。搜索由 job scheduler 来调度和分配，调度会生成相应的有依赖关系或者可并行的搜索子工作。



这些工作主要分成三步，一是 exploration，探索和补全计划空间，就是根据优化规则不断生成语义相同的逻辑表达式。举个例子，***select \* from a, b where a.c1 = b.c2*** 可以生成两个语义相同的逻辑表达式： **a join b 和 b join a**。第二步是 implementation，就是实例化逻辑表达式变成物理算子。比如， **a join b** 可以变成 ***a hash_join b*** 或者 ***a merge_join b***。第三步是优化，把计划的必要条件都加上，比如某些算子需要 input 被排过序，数据需要被重新分配，等等。然后对不同的执行计划进行算分，来计算最终预估代价。



### 3）Transformations

Plan transformation 就是刚才优化中第一步 exploration 的详解，如何通过优化规则来补全计划空间。举个例子，下面就是一则优化规则 **InnerJoin(A,B) -> InnerJoin(B,A)**。这些 transformation 的条件通过触发将新的表达式，存放到 Memo 中的同一个 group 里。



### 4）Property enforcement

在优化过程中，有些算子的实现需要一些先决条件。比如，sortGroupBy 需要 input 是排序过的。这时候就需要 enforce order 这个 property。加入了这个 property，ORCA 在优化的过程中就会要求子节点能满足这个要求。比如要让子节点满足这个 sort order property，一个可能的方法是对其进行排序，或者，有些子节点的算子可以直接满足条件，比如 index scan。



### 5）Metadata Cache

数据库中表的元数据(column 类型)等变动不会太大，因此 Orca 把表的元数据缓存在内存用来减少传输成本，只有当元数据发生改变时(metadata version 改变时)，再请求获取最新的元数据。



### 6）GPOS

为了可以运行在不同操作系统上，ORCA 也实现了一套 OS 系统的 API 用来适配不同的操作系统包括内存管理，并发控制，异常处理和文件 IO 等等。



## 优化过程详解

这一章节，我们通过一个具体的示例来看 ORCA 是如何对 SQL 语句进行优化的。语句是：



![img](https://static001.infoq.cn/resource/image/55/c6/55dc485d5e1419aa9f21713efbc400c6.png)



考虑到分布式数据库，我们假定 T1 的数据分布是按照 T1.a 的 hash 后分配到不同的 node 上， 而 T2 的数据分布是根据 T2.a 进行分配。



![img](https://static001.infoq.cn/resource/image/e7/f7/e71bc010d06bfa4cda8cfb72bf8889f7.png)



上图给出了以 DXL 形式传给 ORCA 的 SQL 语句。首先，可以看出，XMLbased 的形式确实是比较繁琐的。DXL 中定义了输出 column：除了给出了 output name，也给出了 metadataID 信息。同时也给出了需要排序的 column。Metadata（表和操作符都被添加了相应的 metadataID), ORCA 可以通过这些 ID，快速从 Metadata Cache 中定位信息。同时 metadata 中有 version 信息，用来确认是否需要更新 metadata。



DXL 语句传送到 ORCA 后，以多组逻辑表达式的的形式存放到 Memo 中。



![img](https://static001.infoq.cn/resource/image/e4/2e/e4ed1f55e319fe5222ada51c5b775b2e.png)



上图展现了 Memo 中存入的逻辑表达式 group：分为 3 个 group：2 个 table scan，1 个 inner_join。Group 0 是 root group。因为它就是整个逻辑语法树的根节点。我们通过边来表示 group 之间的依赖关系。InnerJoin(1, 2)代表了 group1 和 group2 是它的子 group。得到初始化的 Memo 后，下面进入具体的优化阶段：



### 1）exploration

根据现有的优化规则(transformation rule)来生成语义相同的逻辑表达式。比如， 通过 join commutatitivty 规则，可以从 innerjoin(1,2)生成 innerjoin(2,1)。



### 2）cardinality estimation (statistics derivation)

Cardinality estimation 在之前的文章中介绍过，用来估算每一个 SQL 节点的输入和输出量。每一组逻辑表达式其实是一个 SQL 节点的一部分，举个例子，scan of table1 估计出有多少行数据被输出。ORCA 内部用 column histogram 来预估 cardinality 和可能的数据分布情况。具体的算法上一篇也介绍过一二，这边就不再赘述了。Cardinality estimation 是自底向上进行，也就是先从叶 group 开始，最后至根节点。 下图给出了示例。



![img](https://static001.infoq.cn/resource/image/59/80/59595ee9c42eb745ee4a76998301fe80.png)



首先，从根节点自上而下来请求 statistics, Group0 会向子 group 发送请求来得到 statistics。举例来说， ***InnerJoin(T1, T2) on (T1.a = T2.b)*** 会分别向子 group 请求 T1.a 和 T2.b 的 histogram。表的元数据会缓存的 MDCache 中，如果不存在，Orca 会发起 MD 调用来像数据库系统获取最新的 metadata。



### 3）Implementation 逻辑到物理的转换

第三部开始实施从逻辑表达式到物理算子的转换。举个例子，local table_scan 可以转换成物理的 sequentialScan，或者 BtreeIndexScan。InnerJoin(T1, T2) 可以转换成 IndexInnerjoin(T1, T2), 或者 MergeJoin(T1, T2) 等等。



### 4）优化

在优化这一步中，会首先进行 property enforcement，然后不同的物理执行计划被计算出预估代价 Cost。这就是[之前](https://www.infoq.cn/article/JCJyMrGDQHl8osMFQ7ZR)介绍过的 Cost Model Calibration。每个对应的物理算子会有一个 cost formula，配合上 cardinality estimation 计算出来的输入和输出，就可以算出相应的 cost。整个过程是首先发送一个优化请求给根 group。这个优化请求一般会有结果的分布或者结果的排序的条件，同时它要求获取当前 group 里 Cost 最小的执行计划。



对于优化请求，每一组物理算子会先计算自身的那一部分 cost。同时把请求发送给子算子，这和 statistics derivation 的模式是一样的。对于一个物理算子组来说，可能在整个优化过程中接受到重复的优化请求，ORCA 会把这些请求 cache 起来去重，用来确保对相同请求，只计算一次。



具体如何发送优化请求，如何计算 cost，如何分发子请求，请允许作者省略几千字。倒不是我不想细写。写了这一段，有兴趣读下来的读者也是绝少数，反而倒是消磨了广大读者继续读下去的意愿。考虑再三，为了不引起反感，我还是不瞎忙活了。 如果读者真的感兴趣，可以参考原论文，或者在文章下面留言，我们可以继续交流。



### 5）执行

优化完成以后，ORCA 会通过 DXL 把执行计划发回给数据库，数据库可以根据执行计划，分配给每个 worker。每个 worker 在执行计算的同时，会通过 distribution operator 把数据分发到其他 node 上继续执行。最终所有计算结果会汇聚到 master node 返回给用户。



## 并行优化

执行优化可能是最消耗 CPU 资源的过程。更高效地执行优化过程来产生高效的执行计划可以大幅度提升整个数据库的性能。考虑到现在服务器的多核配置，如何利用多核资源来加速优化过程是 ORCA 的另一个重要的特性。ORCA 的 job scheduler 配合 GPOS 就可以利用多核进行并行优化。在前面的章节提到过，整个优化过程其实是被分发到每个物理算子组中进行，每个组在执行优化的过程中，根据依赖关系，可以并行进行。现在 ORCA 的优化任务有下面几类：



1）给定一个逻辑表达式，根据变换规则生成所有语义相同的逻辑表达式



2）给定一个逻辑表达式，根据变换规则生成所有物理算子



3）对某个表达式组作用某个变换规则



4）优化某一个表达式或者一个物理算子组



因此，对与一个语句，ORCA 会产生几百甚至上千个优化子任务。这些任务之间是有依赖关系的。举例来说，一个 group 必须等到它的所有子节点被优化完成后才能进行优化。这里就需要 job scheduler 来协调子任务的优化过程。 job scheduler 会根据优化任务的依赖关系，来决定先优化哪些任务。



下图给出了一个优化子任务的依赖关系示例。



![img](https://static001.infoq.cn/resource/image/7d/28/7d18dbb9a95330eaf392038d08549828.png)



## 元数据获取

ORCA 一大特性就可以独立于数据库系统运行。元数据获取就是 ORCA 和数据库系统一个重要的交互。在优化中，ORCA 需要知道表的数据分布，对于 column 的 histogram，或者对于某个 column 是否有 index 等的信息。下图展示了 ORCA 是如何与不同的数据库获取元数据信息。



![img](https://static001.infoq.cn/resource/image/1e/39/1ea18e398963165add5f140fc879e139.png)



在优化过程中，所有的元数据信息会被 cache 在 ORCA 中。优化过程中，ORCA 通过 MDAccessor 来获取所有的元数据。MDProvider 除了 plug-in 到其他系统，也提供了文件形式导入 metadata。这就使得测试 ORCA 变得非常容易：我们可以单独运行 ORCA 程序，通过文件形式提供 metadata 和 SQL 语句来获取 ORCA 的执行计划来测试。



## 测试和验证

测试和验证一个优化器的难度不亚于实现一个优化器。在实现 ORCA 的初期，测试和验证需求就被放在了第一位。拜 DXL 接口和文件形式的 MD provider 所赐，我们可以很容易地添加回归测试用例来确保在迭代 feature 的过程中，不引入 bug。本文我们会介绍两个 ORCA 构建中比较特别的工具。



### AMPERe

AMPEre 是一款用来重现和调试 bug 的工具，类似于 core dump。当出现问题时，可能是优化器 crash 了，或者是生成的执行计划非常慢。AMPERe 可以把当前的整个状态复制下来，用作复盘和调试。 整个状态包括要优化的语句，优化器的当前配置，数据库的元数据，如果是崩溃了，还会有 stack trace 等信息。 这些信息以 DXL 的形式被保存到文件中。 工程师可以读取这些这些数据来调试。 更重要的是，我们可以通过把语句，元数据以 DXL 的形式导入到 patch 了 fix 的新版本 ORCA 中，用来测试原有的问题是否被修复，比如不再 crash，或者 ORCA 生成了一个新的执行计划，等等。



下图展示了这样一个过程。



![img](https://static001.infoq.cn/resource/image/f6/ab/f63168e7a3355ca8104164c0760385ab.png)



AMPERe 同时也是 ORCA testing framework 中重要的一部分，我们可以用 AMPERe 记录很多已知的客户遇到过的真实问题并把它们做成回归测试用例。每次有新版本更新的时候都可以用这些用例来做回归测试。



### 测试优化器的准确性

哈！终于讲到这了。我小小地骄傲一下，这是我当时在 Pivotal 实习的项目。用来测试 Optimizer 生成的执行是否优秀。Testing Accuracy of Query Optimizer， 所以工具的名字叫[TAQO](https://dl.acm.org/doi/abs/10.1145/2304510.2304525)。



TAQO 的想法很简单，对于某个 SQL 语句，如何判断一个优化器作出了正确的选择呢。



假定优化器生成了 3 个执行计划，分别是 P1， P2， P3。然后对应的 cost 是 C1， C2，和 C3，并假设 C1<C2<C3。我们可以如下操作：对每个执行计划，用这个执行计划执行一下语句，分别得到执行时间 T1，T2 和 T3。如果 T1<T2<T3，这不就说明 ORCA 给出的判断是正确的，因为它给出的执行计划的 cost 的排序和实际执行时间是相同的。



在测试过程中，对于一个测试语句，TAQO 让 ORCA 生成不同的执行计划，然后再执行这些计划得到相应的运行时间。最后计算运行时间和预估 Cost 的相关值。相关值越高，则说明越准确。



同样，我们可以用 TAQO 为 ORCA 做回归测试。比如，运行当前版本对应 TPC-DS 语句并计算 TAQO 值。有了新版本后，再运行一下 TAQO 来计算新值。确保所有语句的 TAQO 值没有下跌。如果有下跌，就可以第一时间发现是否这个版本引入了某些修改导致了 ORCA 在优化中犯了一些错误。



## (!)summary

后续还有测试结果的比较，这边就不赘述了。我们当时比较了 Cloudera 的 Impala，Hontonworks 的 Stinger，和当时 Facebook 刚推出的 Presto。结果当然是 ORCA 要更好一些，毕竟，好多竞品也才出现，很多功能还不够完善，很多 SQL 语法也还支持得不好。



至此，文章的主要内容就介绍到这。可见，除了我们前两篇介绍到的技术，实践中构建一个优化器是一个非常庞大和复杂的工程。



说些题外话，后来作者虽然离开了 Pivotal，和原来的这些同事关系都很好。QueryProcessing 组的成员陆陆续续后面都离开了，一半加入了初创公司 Datometry(我就在这一半中)，参与数据库虚拟化系统的开发。另一半加入了 AWS，参与 Redshift 的开发，大家还是依然活跃在数据库领域方面。现在作者并不直接参与数据库系统开发啦。但还是时刻保持关注，读读 paper，有机会也去参加参加会议，写点 blog，也算间接对数据库领域做些贡献吧。