# 团队介绍

- [OneSizeFitsQuorum](https://github.com/OneSizeFitsQuorum)：清华大学软件学院研三
- [Alima777](https://github.com/Alima777)：清华大学软件学院研三
- [SpriCoder](https://github.com/SpriCoder)：清华大学软件学院研一
- [SzyWilliam](https://github.com/SzyWilliam)：清华大学软件学院研一

# 项目介绍

让 TiKV 无畏写热点（Fearless Write Hotspot）！！！

# 动机

## 热点问题

热点问题一直是计算机领域的一大难题。根据二八定律，实际的用户场景往往会出现热点。在分布式系统中，部分热点节点会承载大量用户的读写请求，而单个机器的负载往往是有限的；在单节点上，部分热点线程会承载大量的计算任务，而单核的性能往往也是有限的。整体来看，热点问题会对性能产生很大影响。

在分布式系统中，在节点间尽量均匀负载来解决热点问题已成为共识。在单节点上，理论上也可以拆分成小任务来达到多核并行的效果。

但是在部分热点场景下，最小的可并行单元由于若干因素无法再被进一步拆分。如果该单元成为了热点，那么不论将它分到哪个节点或者哪个线程，其都会成为热点，即使有闲置的系统资源也无法起到帮助。在这种场景下，不论是增加机器的核数，还是更换更好的硬盘，亦或者增加更多的客户端并发，可能对整体的性能提升都鲜有帮助。

那么，我们应该去解决热点问题呢？

## TiDB 的现有解决方案

TiDB 作为一款分布式数据库，内建了负载均衡机制。在 TiDB 侧，可以通过配置 HAProxy 的方式来将业务负载均匀的路由到不同的 TiDB 节点上。在 TiKV 侧，海量 region 能够使得 TiKV 在大多数场景下随节点个数线性扩展。这些机制都能够很好地利用整体系统资源。

然而这些机制不是万能的，比如 region 是一个最小的可并行单元，如果极个别 region 成为了单点瓶颈，此时就算增加系统资源也无济于事。因此，如何拆分 region 使得业务负载能够被均匀到所有 region 上是目前 TiDB 解决热点问题的整体思路。

对于写热点，TiDB 目前已经做了[如下工作](https://docs.pingcap.com/zh/tidb/dev/troubleshoot-hot-spot-issues)来更好的在多节点和多核上扩展写性能：

- 对于主键非整数或没有主键的表或者是联合主键，TiDB 会使用一个隐式的自增 RowID，大量 INSERT 时会把数据集中写入单个 Region，造成写入热点。通过设置 `SHARD_ROW_ID_BITS`，可以把 RowID 打散写入多个不同的 Region，缓解写入热点问题。
- 使用 `AUTO_RANDOM` 处理自增主键热点表，适用于代替自增主键，解决自增主键带来的写入热点。使用该功能后，将由 TiDB 生成随机分布且空间耗尽前不重复的主键，达到离散写入、打散写入热点的目的。

## 不适用的场景

令人遗憾的是，这样的写热点解决方案具有如下的若干约束，并不普适：

- 对于含有 `CLUSTERED` 主键的表，TiDB 会使用表的主键作为 RowID，因为 `SHARD_ROW_ID_BITS` 会改变 RowID 生成规则，所以此时无法使用 `SHARD_ROW_ID_BITS` 选项来打散写入热点。
- 使用 `AUTO_RANDOM` 来处理自增主键热点表并不具备普适性。用户的其他业务系统对表的自增主键可能有更复杂的依赖语义，此外存量数据的迁移成本不能被忽视。因而部分自增主键热点表的场景不能使用 `AUTO_RANDOM` 来打散写入热点。
- 唯一索引自增写入场景造成的写入热点目前还无法优雅解决，比如 timestamp 列的唯一索引。

由于这些约束，部分场景下当个别 region 成为热点时，现有的解决方案无法彻底解决热点问题。比如业务系统的批量任务（例如清算以及结算等业务）是典型的高并发批量插入场景。该场景下常会用到 bulk insert 或者 load data 功能，这些功能也最容易产生热点。

基于以上问题，本项目将采用 bottom-up 的设计思路，从更好地利用 CPU、磁盘等资源的角度出发，考虑如何自底向上解决 TiKV 的写热点问题。

![](https://user-images.githubusercontent.com/32640567/196174803-dbc3a03c-5336-444c-9b57-8edf904a035f.png)

# 调研与思路

## TiKV 在热点场景的问题

TiKV 目前使用了经典的 [SEDA](https://en.wikipedia.org/wiki/Staged_event-driven_architecture) 线程模型，RaftStore 是写入路径上有关共识算法最关键的模块，具体可以参考[该源码解析](https://zhuanlan.zhihu.com/p/571600297)。RaftStore 包含两个 thread pool：store pool 用于处理 raft message、append log 等，raft log 会写入 [raft-engine](https://github.com/tikv/raft-engine)；apply pool 用于处理 committed log，数据会写入 [rocksdb](https://github.com/tikv/rocksdb)。

在当前的设计下，每个 TiKV 进程的 RaftStore 模块都通过个位数的 thread 来管理成千上万 region 的状态变更。从 region 角度来看，每个 region 同时最多只能被一个线程处理，单个 region 内的状态变更是不能做到多核并行的。在所有 region 的负载都比较均匀时，这样的设计是一种很优的策略。然而如果 海量 region 中存在热点（这也是大部分互联网实际场景的真实情况），这样的策略就不一定很优了。

考虑只有一个热 region 的特殊场景，尽管所有的写入都路由到了该 region，然而该 region 的状态变更并不具备在多核上的扩展性，即使有很多空闲的 CPU 资源，也无法解决该 region 的热点问题，只有等到该 region 被触发一次 split 之后才有可能有所缓解。然而不论 TiKV 这边如何 split region，理论上用户都可能将高负载持续的打到一个 region 上（比如以上提到的递增主键场景），split 只能根据过往的流量来做抉择，但却很难实时预测到未来的流量。

当然大多数场景下写热点并不会只仅仅存在于一个 region 上，而是存在于少数 region 上，但即使如此计算资源不均衡的现象依然会持续存在。

那么我们有没有可能在更底层去解决热点 region 在现代硬件上的扩展性呢？答案是有的。

## Multi-Raft 与现代硬件

先来聊聊 Raft 的问题，TiKV 使用了 Raft 算法来保证副本之间的一致性。Raft 算法是 RSM 模型的一种标准实现，而 RSM 是一种解决共识问题的经典模型。RSM 将共识问题抽象成了如何在多个副本间维护一个相同的全序日志数组，那么只要所有副本都按照日志数组的顺序去状态机做操作，一致性便能够自然的在多个副本间保证。

RSM 模型的一大特点就是简单可用，然而其在现代硬件的不断发展下也逐渐暴露出了它自身的问题。对于计算，单核的时钟频率增长已经陷入了瓶颈，现代 CPU 大都是通过增加核数来提升整体性能，因此应用侧的并行对于发掘现代 CPU 的极致性能会越来越有帮助。对于存储，NMVE SSD 等高速硬盘需要高的 I/O depth 来打满 IOPS，或者大的 I/O size 加上不那么高的 I/O depth 来打满 bandwidth，但大的 I/O size 不适合 OLTP 系统，因为攒大 batch 通常意味着高的延迟。因此 OLTP 系统需要着重考虑如何提升 I/O depth 来发掘现代高速硬盘的极致性能。

![](https://user-images.githubusercontent.com/32640567/196174898-ea7a72b7-b0c5-47a9-85d9-24742568aab2.png)

也许由于 RSM 模型本身的语义就是串行的，不论是 Raft 算法的若干开源实现库（etcd, sofa-jraft, ratis 等），还是基于 Raft 算法的若干分布式 KV（TiKV 等），对于单 raft 组而言，其日志持久化的 IO depth 都是 1，其 apply 也都是单线程串行去做的，他们均不能很好的利用现代硬件。部分同学可能会认为如果是一个 Multi-Raft 的架构，就能够利用好现代硬件，然而事实并非如此。

就 TiKV 目前而言，对于计算，当前一个 region 在 ApplyPool 中同时最多只能被一个线程驱动执行，在写热点场景，这些热点 region 并不具备在多核上的扩展性。对于存储，所有 region 的 raft log 在 StorePool 中会通过 group commit 的机制来写入 [raft-engine](https://github.com/tikv/raft-engine)，IO depth 仅仅为 1，并没有利用好现代磁盘。去年 hackathon 的[ ](https://github.com/TPC-TiKV/rfc)[TPC-TiKV](https://github.com/TPC-TiKV/rfc) 项目做了 raft-engine parallel log 的优化，IO depth 可以打的很高，对于单 region 不同批日志的持久化也可以并行，这已经提升了不少性能。然而受限于 raft 顺序 commit 的语义，其依然不能将已经持久化但前面存在空洞的日志 commit 并尽早的提交给 ApplyPool 处理，在某些场景下这里依然存在进一步优化的可能性。

由此可见，即使是一个 Multi-Raft 的系统，要想在热点场景下利用好现代硬件也绝非易事。

## 理想的写热点解决方案

要想给出写热点的理想解决方案，需要先回到一个老生常谈的经典问题，Raft 与 Multi-Paxos 有什么区别？

相比 Multi-Paxos，Raft 不能乱序 ack，不能乱序 commit，不能乱序 apply，因而有同学认为 Raft 不如 Multi-Paxos。

dragonboat 的作者对此观点进行了[反驳](https://www.zhihu.com/question/278984902/answer/404439990)，其主要有两个观点：

- 乱序并行 apply 不如拆分出更多的 raft 组来并行。
- 对于一个通用的共识库，不能乱序 apply 是受限于 RSM 模型本身的限制，并不是 Raft 本身的问题。对于特定场景，乱序 apply 可以起到效果，但并不是一个通用性的优化。

对于第一个观点，从 TiKV 的角度来看，目前默认的 region 大小为 128M，尽管理论上 raft 组数拆的越多，单 raft 组内的串行化 apply 对性能的影响就越小。然而，raft 组过多带来的其他问题也会接踵而来，比如在 TiKV 实际测试中观察到过多的 region 和过大的 LSM Tree 都会导致性能的回退，因而未来 TiDB 的 [Dynamic Region](https://github.com/tikv/rfcs/pull/82) 工作计划一方面调大 region 大小到 512MB 至 10GB 从而减少 region 个数，另一方面则是拆分 RocksDB 实例。由此可见，影响 region 大小的因素并不只有 raft 串行化的问题。在未来，一方面 TiKV 的 region 会比现在更大，因此单 raft 组内的串行化问题会更加明显从而可能成为瓶颈；另一方面拆分 RocksDB 实例后 split 也不会再像现在这么轻量，因而实时动态的 split 负载均衡策略相比现在也会趋于保守。总体来看，在 TiKV 内实现乱序 apply 在未来是一个非常有可能的性能优化方向。

对于第二个观点，的确从通用的共识库的角度出发，乱序 apply 并无太大意义。但从 TiKV 中内嵌的共识算法来看，由于共识层之上的事务层已经定了一次序，因而共识层的重复定序在有些 case 下是没有意义，此时的乱序并行 apply 更可能提升热点场景的性能。

事实上，如果不支持乱序 apply，那共识算法的乱序 ack 和乱序 commit 可能没太大意义，因为整个共识组的瓶颈受限于最慢的模块。如果只能顺序 apply，那即使乱序 commit 了一批日志，如果这些日志之前存在空洞，那么这批日志也只能在内存中等待而不能被 apply。然而如果支持了乱序 apply，那结合乱序 ack 和乱序 commit 就更可能提高共识组的吞吐上限。比如一旦支持乱序 commit，那可以使用多个 IO depth 来持久化不同批的日志，这样每次 IO 的大小减少了，也可能能够减少每次 IO 的平均时间。此外支持乱序 commit 后也可以将前面存在空洞但确保与空洞日志无依赖关系的一批乱序 commit 日志提前 apply 处理，进而抬高 apply 的瓶颈天花板。

**总体来看，Parallel Apply 能够解决 region 写热点在多核上的扩展性问题，Multi-Paxos 能够解决 region 写热点在现代硬盘上的扩展性问题。因此，Multi-Paxos + Parallel Apply 理论上能够解决写入热点在现代硬件上的扩展性问题。预计能够在写热点场景更充分的利用硬件资源，从而提升性能。**

此外，如果能够解开单 region 在 commit 和 apply 时的逻辑阻塞链，则有可能结合 TPC 模型合并多个线程池，将 schedule task 的能力从内核态转移到用户态，从而对资源有更精细的控制。

![](https://user-images.githubusercontent.com/32640567/196175011-48439044-2202-43d3-8a90-7753a9dc1252.png)

# 设计

出于工作量考虑，本次 Hackathon 不会去做 Multi-Paxos（太复杂），只会做相对简单但实际也很复杂的 Parallel Apply demo，并进而验证在 apply 为瓶颈的热点场景下，性能收益会是多少。

本小节将首先介绍在 TiKV 当前架构上要实现 Parallel Apply 的一些约束，然后介绍具体的设计细节，最后分析可能的性能收益。

## 约束

### Follower 不能像 Leader 一样并行 Apply

在 Leader 侧由于 [Latches](https://cn.pingcap.com/blog/tikv-source-code-reading-12) （为了兼容 Percolator 模型读后写语义而产生的结构）的存在，可以保证所有 raft 组的 applyIndex 到 lastIndex 中的 Normal 日志 key 不存在重叠，所以不需要像 [Parallel Raft](https://www.vldb.org/pvldb/vol11/p1849-cao.pdf) 一样引入 LBB 的依赖检测机制，可以在 Leader 侧不加约束的并行即可。

在 Follower 侧，在当前类似于 etcd 的迭代获取 ready 的 codebase 下，一次获取到的 ready 中可能包含冲突的日志，此时不能随意并行，否则可能会破坏一致性。最简单但绝对正确的方式便是在 Follower 侧继续串行执行，更进一步也可以引入一些类似于拓扑排序的依赖检测并行 apply 机制，具体可参考 sofa-jraft [issue](https://github.com/sofastack/sofa-jraft/issues/614)。出于时间原因，本次 Hackathon 将在 Follower 侧串行执行。

### 需要对 Admin/Conf 等日志特殊处理，当做乱序中断标志

对于 Admin/Conf 日志例如 Split/Merge/ConfChange，其一般都需要基于当前 Raft 组的一个完整视图去进行实际操作。

在串行 apply 的语义下，在 admin 日志之前的日志不会被乱序到 admin 之后去执行，反之亦然。

在并行 apply 的语义下，该约束依然需要被满足，因而 Admin 日志类似于乱序中断标志，在其执行前需要保证其前面的所有日志均已执行完，其后的任何日志都还没有被执行。

### Normal 日志的 ApplyIndex 不需要实时持久化到磁盘中

实际证明较为复杂，此处不再细述。直观来理解的话，对于 Normal 日志，最终都会被转换为对底层 kv 引擎的 put 操作，语义上是幂等的，重启时按序重复重放不会影响正确性。此外在 TiKV 重启期间任何 region 在没有 apply 当前 term 的至少一条日志时不会服务读请求，所以中间的恢复状态不会对外暴露。对于 Admin 日志，applyIndex 需要实时持久化到磁盘中，因为其语义不一定是幂等的。

## 设计

基于以上约束，可以明确要想实现并行 apply 需要：

- 仅在 Leader 侧并行 apply，在 Follower 侧初步采用串行 apply 方案
- Admin 日志之前的日志不能被乱序执行到 Admin 日志之后，反之依然
- 在内存中维护 ApplyIndex，定期维护到磁盘上的 ApplyState 中
- 在 Leader 切换和重启时依然满足正确性

为了尽量复用现有的代码减少复杂度，本项目计划在 ApplyBatchSystem 以外新引入一个并行 apply 线程池，拥有和 ApplyBatchSystem 相同的线程数，其中每个线程拥有全量的 ApplyFSM。

为了避免线程之间的同步阻塞，所有的共享状态都使用了原子变量而非锁的结构。

整体思路便是在 StorePool 中对 region 的日志进行特判路由到不同的线程池中执行：

- 对于普通日志，如果满足对应可并行的条件则可以轮询路由到不同的并行 apply 线程上并行化处理。
- 对于 admin/conf 日志，则继续路由到 ApplyBatchSystem 上复用之前处理 admin/conf 日志的逻辑。

由此便可以在 Leader 侧将热点 region 的不同批普通日志在符合条件时路由到并行 apply 线程池中并行执行。

以下介绍一些具体的设计细节，会提到很多 TiKV 中源码级别的概念，供了解 TiKV 源码的同学参考。

### 如何判断一组 commitEntries 是否拥有 admin/conf 日志？

遍历一批 commitEntries，如果任何一个 entry 的 type 是 EntryConfChange/EntryConfChange2，或者 context 解析后包含 ProposalContext 中任何一个字段，则认为该批日志需要串行处理并路由到 ApplyBatchSystem，否则与其他条件一起判断是否需要轮询路由到并行 apply 线程池。

### 如何确保 admin 日志执行之前 index 比其小的普通日志均已经执行完？

在并行 apply 线程池和 ApplyBatchSystem 中的每个 ApplyFSM 上记录与对应 RaftFSM 中共享的 Arc<AtomicU64> 变量记录该 region 尚处于并行 apply 的 task 数目。

每次 RaftPoller 路由给并行线程池一个任务时对该字段递增 1，并行线程池中的每个线程维护一个 regionid -> count 的本地 map，记录属于每个 region 的新增 apply 次数，在 commit 向引擎写入一批日志后将该 map 中每个 region 对应的 count 同步到共享的 Arc<AtomicU64> 中去，即减少对应的数目。

对于包含 admin 日志的一批 commitEntries，其必定会被路由到 ApplyBatchSystem 中去，其中每个 ApplyFSM 也持有该 Arc<AtomicU64>，在 handle_normal 函数中，首先判断该字段是否为 0，如果为 0 则代表之前并行 apply 的日志均已执行完，此时可以从 channel 获取日志复用之前的流程串行处理，否则返回 StopAt 并让当前 ApplyPoller release 该 FSM，等到起满足执行条件后再被具体执行，防止 ApplyPoller 陷入忙等。至于如何得知该 FSM 已经满足执行条件呢？可以在并行 apply 线程池中的每个线程进一步维护 applyRouter，当且仅当 Arc<AtomicU64> 被 CAS 递减后为 0 时即可通过 applyRouter 向 ApplyBatchSystem 发送一条特殊的 Noop message，该 message 仅仅用来触发该 ApplyFSM 以让其能再被 ApplyPoller 调度到，进而可以进行下一步处理。

### 如何保证 admin 日志执行之后 index 比其大的普通日志可以继续并行执行？

在 PeerFsmDelegate 中记录一个 int 变量，代表当前在执行的 admin/conf 日志数量。在 storePool 处理 admin/conf 日志时将其 ++，在对应的 ApplyRes 回来之后再 --。实际上在代码中可以使用 cmd_epoch_checker.proposed_admin_cmd 的大小即可。

当且仅当其为 0 时代表当前没有正在被 ApplyBatchSystem 处理的特殊日志，因而可以在 leader 节点上与其他条件一起判断是否可以并行处理。

### 如何确保 leader 切换和重启时并行/串行逻辑不发生错误？

对于非 Leader 角色，强制路由到 ApplyBatchSystem 去处理。

对于 Leader 角色，当且仅当一批 committedEntries 的 term 均为当前任期的 term 时才可以与其他的判断条件一起去决定是否可以并行处理。这样可以防止新 leader 当选后其之前原本要串行执行的日志突然并行起来最终导致一致性出现问题。

对于分区后的老 leader，其 latches 约束依然存在，所以在 apply 当前 term 的日志时依然可以不加约束的并行化。如果分区恢复后其被 step down，尽量对应的 latches 还可能存在，但此时再路由日志时依然会保守的串行处理，从而避免一些可能出现问题的复杂 case。

对于 Leader 上 term 的约束还可以使得重启时所有的数据都通过 ApplyBatchSystem 来并行回放，从而避免潜在的一致性问题出现。

### 如何进行 applyIndex 的更新？

在内存中，对于每一批 commitEntries，除了 lastIndex 外还记录 firstIndex，不论是并行 apply 线程池还是 ApplyBatchSystem 在 apply 一个循环后都将带着 firstIndex 的 ApplyRes 返回给 StorePool。StorePool 中的每个 PeerFSM 记录一个类似于 BTreeMap 的结构，如果 ApplyRes 中的 firstIndex 大于 PeerFSM applyIndex + 1，则将其记录到 BTreeMap 中不再更新任何其他状态，否则将 PeerFSM 中的 applyIndex 递增至该 ApplyRes 的 lastIndex，接着对该 BTreeMap 的 key 从小到大依次判断是否和最新的 applyIndex 连续，如果连续则继续递增，否则终止即可。

在磁盘上，初步计划利用在路由到 ApplyBatchSystem 中串行 apply 的请求上维护磁盘上的 ApplyState，之后可以考虑如果大多数请求都并行化了，ApplyState 更新太少导致重启速度显著降低，则可以定期将 StorePool 中的 applyIndex 发送给某个并行 apply 线程由其去做一次 applyState 的持久化。

## 收益分析

对于 Parallel Apply 优化，其能带来性能提升的本质原因是增加了热点 region apply 的并行度，从而减少了一批 log 的 apply wait duration，在 CPU 资源不是瓶颈的环境下，最终能够在相同并发下减少延迟从而提升吞吐。

由于 apply duration 既包含排队等待的 apply wait duration 也包含实际攒批写 rocksdb 的 apply log duration。前者是排队的延迟时间，后者是实际的执行时间。考虑到这是一个排队系统，在热点 region 里 avg apply wait duration 一般与 avg apply log duration 和整体的 avg apply duration 均呈正相关关系，因为在负载足够大的情况下，apply log duration 越久 apply wait duration 一般也会越大。

在评估 Parallel Apply 是否可能有收益时，首先应该关注当前正在写入的 region 是否存在热点，如果不存在热点且负载都很均匀，那理论上将其打乱并行执行也不会有明显的收益。如果存在热点 region 且 CPU 资源尚有剩余，此时可以观察 apply duration，根据<u>阿姆达尔定律</u>，apply duration 在整条事务的 duration 中占比越高，Parallel Apply 的预期收益就会越大。

![](https://user-images.githubusercontent.com/32640567/196175161-12ad633c-d10a-403f-ba06-2816a94ee544.png)

# FAQ

## 有了该工作之后是否就不需要 split region 了？

否，他们的作用并不冲突，可以互补而不是互斥。主要有以下两个原因：

- Multi-Paxos + Parallel Apply 可以解决热点 region 在单节点上的扩展性问题，但不同节点上的负载均衡依然需要 region split 成多个来横向扩展。他们的互补可以更好地利用多个节点上的现代硬件。
- region 的大小受很多因素影响，比如 snapshot 大小，底层 KV 引擎是否分离等等。使得 region 在现代硬件上可扩展并不代表之后每个进程只有一个 region 就可以了。

## 为什么没做 Multi-Paxos 来彻底解决 commit 的热点问题？

想做，但时间不允许。实现一个 Multi-Paxos 的 demo 都十分复杂了，想要真正的落地需要更长的时间。希望本项目能够引发一些讨论和思考，获得一些实在的工业界/学术界反馈，这样便可以为未来的 State of the Art 引擎演进提供思路。
