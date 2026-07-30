# WorkLoad Design 



# 一、负载设计的总体原则

生成的负载需要符合真实场景，使得实验测试出的性能上下界能尽可能贴近现实应用中的性能上下界。因此，在生成负载时，尽量仅设置对性能影响较大且符合真实场景的条件，例如在构造有利场景时优先保证高正向点查询比例、高近期key查询比例和总内存受限等条件。



------

# 二、统一的逻辑时间单位

由于不同系统吞吐量不同，相同时间内产生的更新量和 compaction 次数可能相差很大。因此，在衡量读取的key版本的新鲜程度时不应该直接使用物理时间，需要重新定义一个逻辑时间。

## 1. 上层逻辑容量

记**上层区域** `L0...LK` 的配置容量为：


$$ C_U=\sum_{i=0}^{K}\text{target size}(L_i) $$

所有实验系统使用相同的 LSM 几何参数`L0...Lk`，因此拥有相同的名义$$ (C_U) $$。

## 2. Recency Epoch

定义一个 recency epoch：

> 累计写入 (C_U) 字节的**逻辑用户数据**，记为 1个逻辑时间周期，即1E。

这里计算用户写入字节，而不是实际设备写入字节，避免不同 write amplification 影响负载定义。

## 3. Key age

对 key (k)：

$$ A_k= \frac{\text{查询key时累计写入字节数}-\text{该 key 最后更新时累计写入字节数}} {C_U} $$

$$A_k $$代表从key最后一次更新到现在查询key之间经历了多少个时间周期：

- $$ (A_k\leq 1E) $$：近期更新 key；

- $$ (1E<A_k\leq 2E) $$：中等新鲜；

- $$ (A_k>2E) $$：历史 key。

  

## 4. Snapshot age

对于 snapshot (s)，定义$$A_s$$为查询的快照与当前快照之间经历的时间周期：

$$ [
A_s=
\frac{\text{最新快照创建时累计写入的字节数}-\text{查询的snapshot 创建时的累计写入字节数}}
{C_U}
]$$

建议定义：

| Snapshot 类型 | 年龄                 |
| ------------- | -------------------- |
| Latest        | $$ (A_s=0) $$        |
| Short         | $$(0<A_s\leq0.5E)$$  |
| Medium        | $$(0.5E<A_s\leq2E)$$ |
| Old           | $$(2E<A_s\leq4E)$$   |

这种定义能直接对应“目标版本是否可能仍在上层”，又不依赖具体机器速度。

------

# 三、需要控制和记录的负载特征

## 1. 主控制指标

| 指标                | 符号                    | 含义                                            |
| ------------------- | ----------------------- | ----------------------------------------------- |
| 更新操作比例        | $$(p_w)$$               | UPDATE 占全部操作的比例                         |
| 正向点查询比例      | $$(p_{pos})$$           | 读请求中查询已存在 key 的点查询比例             |
| 近期 key 查询比例   | $$(p_{recent})$$        | 正向查询中访问 (A_k\leq1E) key 的比例           |
| Latest 比例         | $$(p_{latest})$$        | 正向查询中读取当前版本的比例                    |
| Short snapshot 比例 | $$(p_{short})$$         | 正向查询中读取 (A_s\leq0.5E) snapshot 的比例    |
| 读写热点相关性      | $$(c_{rw})$$            | 读取热点与更新热点的重合程度                    |
| 总读路径内存        | $$(M_R)$$               | Filter、Locator、Index Cache 和 Data Cache 总和 |
| 访问倾斜度          | $$(\theta_r,\theta_w)$$ | 读、写 Zipf 参数                                |

## 2. 运行后实际测量的关键指标

最重要的不是生成器设定的“近期比例”，而是实际 fast-path 可用率：

$$ [
h=
P(\text{目标可见版本能被上层 locator 直接定位})
]$$

必须同时记录：

- upper fast-path hit rate；
- definite upper miss rate；
- fallback rate；
- 目标版本实际所在 level；
- 目标 key 在上层的版本数量；
- 每次查询检查的不可见版本数。

最终性能曲线应**优先以 (h) 为横轴**，而不是只以 `recent_ratio` 为横轴。

------

# 四、公共数据集和系统配置

上下界负载只改变关键负载特征，其余条件保持一致。

## 推荐默认配置

| 配置项            | 建议默认值                    |
| ----------------- | ----------------------------- |
| Key 大小          | 16 B                          |
| Value 大小        | 1 KiB，固定长度               |
| 数据库规模        | 至少为读路径总内存的 50 倍    |
| 访问分布          | Zipf，$$(\theta=0.9)$$        |
| 读路径总内存      | 逻辑数据库大小的 2%           |
| 主线程数          | 固定，例如 16 或 32           |
| Upper zone        | 固定为 L0～LK                 |
| Active update set | 约为上层可容纳 key 数量的 50% |
| Snapshot 间隔     | 每写入 (0.05E) 创建一次       |
| Snapshot 保留范围 | 最近 4E                       |
| 测量时间          | 至少 5E，且不少于 15 分钟     |
| 重复次数          | 至少 3 次                     |

如果上层大约能容纳 ($$K_U$$) 个键值版本，则 active update set 的键值数量$$H_w$$初始可设为：

$$ [
H_w=0.5K_U
] $$

这样在一个 epoch 内，活跃 key 平均会被更新多次，有机会产生 2～4 个近期版本，而不必人为指定每个 key 必须有多少版本。

为了避免热点集合永久不变，可以每经过 (0.5E)，随机替换 active update set 中 5%～10% 的 key。这个变化速度足够简单，同时比固定热点更接近实际 workload shift。

------

# 五、现实性能上界负载：Recent-MVCC

这个负载要有利于你们的方法，但不能做到“100% 上层命中、100% latest read”。

## 负载特征

| 维度                | 参数             |
| ------------------- | ---------------- |
| READ / UPDATE       | 95% / 5%         |
| 正向点查询          | 读请求的 95%     |
| 负向点查询          | 读请求的 3%      |
| 短范围查询          | 读请求的 2%      |
| 查询近期 key        | 正向点查询的 80% |
| 查询历史 key        | 正向点查询的 20% |
| Latest read         | 正向点查询的 70% |
| Short snapshot      | 正向点查询的 25% |
| Medium/old snapshot | 正向点查询的 5%  |
| 读写热点相关性      | 高               |
| 读写 Zipf 参数      | 0.9              |
| 总读路径内存        | 数据库大小的 2%  |

这里仍有：

- 20% 查询不访问近期 key；
- 5% 查询使用较旧 snapshot；
- 5% 非正向点查询；
- 持续的 update 和 compaction。

因此它不是纯机制上界，而是一个现实中可能出现的：

> 高频更新数据同时也是主要读取对象，绝大多数事务读取最新或较新 snapshot 的在线服务负载。

## 生成策略

读取近期 key 时：

1. 从最近 1E 内被更新过的 key ring 中抽样；
2. 使用 Zipf 分布决定近期 key 内部的热点；
3. 70% 使用 latest；
4. 25% 从最近 (0.05E\sim0.5E) 的 snapshot 中按截断几何分布抽样；
5. 如果 key 在该 snapshot 下尚不存在，则重新抽样。

读取非近期 key 时，从 (A_k>1E) 的 key 中抽样，不直接查看其物理层级。

## 预期实际状态

第一轮实验可将以下条件作为负载有效性检查，而不是硬编码条件：

- (h) 大约达到 70%～90%；
- 至少一半的近期查询 key 在上层存在两个及以上版本；
- 仍有 10%～30% 查询需要 fallback；
- Block Cache 不足以缓存全部数据和索引。

如果实际 (h) 接近 100%，说明场景可能过于理想；如果不足 50%，则需要检查 recency window 是否超过了上层版本的实际驻留周期。

------

# 六、现实性能下界负载：Read-Hot / Update-Cold

最合理的不利场景不是大量负向查询或 range scan，而是：

> 查询仍以正向点查询和 latest read 为主，但被频繁读取的 key 很少更新，因而已经沉入底层。

这正好体现你们方法的固有限制，同时仍符合缓存、目录、配置、内容对象等“读多写少”数据的现实特征。

## 负载特征

| 维度                | 参数             |
| ------------------- | ---------------- |
| READ / UPDATE       | 95% / 5%         |
| 正向点查询          | 读请求的 95%     |
| 负向点查询          | 读请求的 3%      |
| 短范围查询          | 读请求的 2%      |
| 查询近期 key        | 正向点查询的 20% |
| 查询历史 key        | 正向点查询的 80% |
| Latest read         | 正向点查询的 60% |
| Short snapshot      | 正向点查询的 20% |
| Medium/old snapshot | 正向点查询的 20% |
| 读写热点相关性      | 低               |
| 读写 Zipf 参数      | 0.9              |
| 总读路径内存        | 与上界负载相同   |

## 生成策略

不要严格构造完全不相交的读写热点，否则可能显得过于人工。可以采用：

- 更新热点：对 key 使用一个 Zipf rank permutation；
- 读取热点：使用另一个独立随机 permutation；
- 两者使用相同的 (\theta=0.9)。

这样读热点和写热点会有少量自然重合，但整体相关性很低。

对于 80% 历史 key 查询：

- 从$$ (A_k>1E)$$ 的 key 中选择；
- 其中大多数仍然查询 latest 版本；
- 不要求使用 old snapshot。

这很重要：方法失效的原因主要是**最新版本已沉入底层**，而不只是因为人为构造了大量历史 snapshot。

## 预期实际状态

- (h) 大约为 10%～30%；
- 大部分查询先访问 locator，随后 fallback；
- GRF 仍能进行全局过滤和定位；
- 你们的方法可能持平或出现小幅负优化；
- 负优化主要来自 locator probe 和上层结构维护。

这个场景可以较真实地衡量方法的性能下界。

------

# 七、中间负载与连续过渡

不能只测试上下界两个点。最重要的实验是找到从“不如 GRF”到“超过 GRF”的盈亏平衡位置。

建议构造一个参数$$ (\beta) $$：

$$[
W(\beta)=
\beta W_{\text{upper}}
+
(1-\beta)W_{\text{lower}}
] $$

取值：

$$[
\beta\in{0,0.25,0.5,0.75,1}
]$$

对应参数近似为：

| $$(\beta)$$ | 近期 key 查询比例 | Latest / Short / Old |
| ----------- | ----------------- | -------------------- |
| 0           | 20%               | 60% / 20% / 20%      |
| 0.25        | 35%               | 63% / 21% / 16%      |
| 0.5         | 50%               | 65% / 23% / 12%      |
| 0.75        | 65%               | 68% / 24% / 8%       |
| 1           | 80%               | 70% / 25% / 5%       |

对每组负载报告实际 (h)，最终绘制：

$$[
\text{Throughput or latency improvement over GRF}
\quad\text{vs.}\quad h
]$$

这样能够回答：

- fast-path hit rate 达到多少时开始超过 GRF；
- 多高命中率才能取得 10%、20% 或更高收益；
- fallback 比例升高后负优化有多严重；
- 实际适用范围是否足够宽。

------

# 八、具体负载生成算法

## 1. 生成器维护的数据结构

```text
last_update_seq[key]       key 最近更新时间
version_seq_list[key]      key 的版本时间戳列表
recent_buckets[epoch]      每个 recency bucket 中被更新的 key
update_hot_set             当前更新热点集合
read_hot_set               当前读取热点集合
snapshot_ring              最近 4E 内创建的 snapshot
committed_watermark        当前已提交逻辑版本
```

`version_seq_list` 只由 benchmark generator 使用，用于确保查询在指定 snapshot 下是正向的，不需要进入被测试系统。

## 2. 操作生成流程

```text
按操作比例决定 READ 或 UPDATE

UPDATE:
    从 update_hot_set 按 Zipf 分布选择 key
    写入新版本
    更新 last_update_seq
    把 key 放入当前 recent bucket

READ:
    决定 positive GET、negative GET 或 short scan

    如果为 positive GET:
        按 p_recent 决定查询近期 key 还是历史 key
        从对应 key pool 中按 Zipf 分布选 key

        按 p_latest / p_short / p_old 选择 snapshot
        检查该 key 在 snapshot 下是否存在可见版本
        不存在则重新选择

    如果为 negative GET:
        从保留的不存在 key namespace 中选择

    如果为 short scan:
        从现有 key 中选择起点
        固定扫描长度，例如 10 条记录
```

## 3. Snapshot 生成

每写入 (0.05E) 的逻辑数据，创建一个 snapshot，并记录其逻辑 commit watermark。

保留最近 4E 的 snapshots：

```text
latest:
    使用当前视图

short:
    从 0.05E～0.5E 的 snapshot 中抽样

medium:
    从 0.5E～2E 的 snapshot 中抽样

old:
    从 2E～4E 的 snapshot 中抽样
```

所有系统使用相同的逻辑 snapshot 标签，但各自在相同写入 watermark 创建本地 snapshot handle。

------

# 九、实验执行阶段

## 阶段 1：基础装载

1. 插入全部 (N) 个唯一 key；
2. 每个 key 初始只有一个版本；
3. 等待基础 compaction 完成；
4. 将基础数据主要沉入较低层；
5. 保存一份逻辑相同的数据库初始镜像。

这一步构造稳定的历史数据基座。

## 阶段 2：版本预热

按照目标负载的 update 分布持续执行至少 3E：

- 产生近期多版本；
- 建立上层 LSM 形态；
- 使 locator / GRF 进入正常维护状态；
- 创建 snapshot ring。

此阶段不记录正式性能。

## 阶段 3：缓存和结构预热

再运行 1E～2E 的完整混合负载，直到以下指标基本稳定：

- Block Cache hit rate；
- 上层文件数量；
- upper-zone 数据量；
- compaction pending bytes；
- fast-path hit rate。

## 阶段 4：正式测量

正式运行至少：

- 5E；
- 且不少于 15 分钟；
- 每个方案重复 3～5 次。

GRF 论文为了展示 worst case，将 YCSB 运行后的 LSM 放在最后一层 compaction 即将触发的位置；这种设置适合机制压力测试，但不建议作为现实边界的主实验。现实边界应采用持续写入下的稳态 LSM，随后再单独增加“compaction worst case”实验。([清华大学交叉信息研究院](https://people.iiis.tsinghua.edu.cn/~huanchen/publications/grf-sigmod24.pdf))

------

# 十、写入负载应采用两种运行模式

## 模式 A：固定绝对写入率

用于公平比较读取延迟和 locator 维护成本。

假设两种系统都使用：

```
90% read + 10% write
```

如果你的系统吞吐是 100 万 ops/s，而 GRF 是 80 万 ops/s，那么实际写入率分别是：

```
你的系统：10 万 writes/s
GRF：8 万 writes/s
```

此时两者承受的：

- flush 频率；
- compaction 频率；
- 版本过滤器更新频率；
- 后台 I/O 压力；

都不相同。这可能掩盖或放大方法的维护成本。

先测得所有方案中update 的ops的下界f，然后在测试时用一组独立的写线程按速率f向系统发送update 请求，确保不同系统执行update的速率是相同的。

RocksDB 官方 `readwhilewriting` benchmark 本身也采用单 writer、多 reader 并限制 writer 速率，因此这种受控写入方式具有现成实现基础。([GitHub](https://github.com/facebook/rocksdb/wiki/performance-benchmarks))

## 模式 B：固定操作比例

用于端到端系统吞吐：

- 95% read / 5% update；
- 90% read / 10% update；
- 80% read / 20% update。

模式 A 解释机制，模式 B 展示应用层总体效果。不能只使用固定比例，因为更快的系统每秒会执行更多 update，从而承担不同的 flush 和 compaction 压力。

------

# 十一、总内存控制方式

写路径内存保持完全一致：

- memtable 大小；
- write buffer 数量；
- compaction buffer；
- compaction thread 数量。

可竞争的读路径内存统一限制为：

$$[
M_R=
M_{\text{version locator}}
+
M_{\text{GRF/Bloom}}
+
M_{\text{index/filter cache}}
+
M_{\text{data block cache}}
]$$

推荐三个档位：

| 内存档位 | (M_R/D) |
| -------- | ------- |
| 紧张     | 0.5%    |
| 默认     | 2%      |
| 充足     | 5%      |

上下界主实验先统一使用 2%，内存档位只作为后续敏感性分析，避免同时改变过多因素。

GRF 的 YCSB 实验也通过固定 1GB Block Cache 来突出不同过滤结构在有限缓存下的差异，并报告在偏斜查询下约 50%的缓存命中率。([清华大学交叉信息研究院](https://people.iiis.tsinghua.edu.cn/~huanchen/publications/grf-sigmod24.pdf))

还需要统一：

- 是否启用 Direct I/O；
- OS Page Cache 策略；
- index/filter 是否允许 pin；
- 所有方案的总内存统计口径。

