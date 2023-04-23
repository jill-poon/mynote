# Facebook Gorilla 时序数据压缩原理

Gorilla源自Facebook对于内部基础设施的近乎”变态”的监控需求，我们先来看看这些数据：

- 20亿个不同的 Time Series
- 每分钟产生7亿个 Data Points，即每秒钟产生1200 万 Data Points
- 数据需要存储26个小时
- 高峰期的查询高达40000次每秒
- 查询时延需要小于1ms
- 每个 Time Series 每分钟可产生4个 Data Points
- 每年的数据增长率为200%

对于监控数据，本身还具有如下几个典型特点：

- **重写轻读**
- 以**聚合分析**为主，几乎不存在针对单 Data Point 的点查场景。因此，即使丢失个别的 Data Points，一般不会影响到整体的分析结果。
- 越老的数据价值越低，而应用通常只读取最近发生的数据。**Facebook的统计表明，85%的查询与最近26个小时的数据写入有关。**

## Timestamp压缩 (Delta-Of-Delta)

在大多数情形下，可以将一个 Time Series 中的连续的 Data Points 的 Timestamp 列表视作一个**等差数列**，这是 Delta-Of-Delta 编码算法的最适用场景。编码原理如下：

- 在Block Header中，存放起始Timestamp T(-1)..一个Block对应2个小时的时间窗口。假设第一个Data Point的Timestamp为T(0)，那么，实际存放时，我们只需要存T(0)与T(-1)的差值。
- 对于接下来的 Data Point 的 Timestamp T(N), 按如下算法计算 Delta Of Delta 值：

$$D= (t_{n} - t_{n-1}) - (t_{n-1} - t_{n-2})$$

**按照D的值所处的区间范围，分别有5种不同情形的处理：**

1. 如果D为0，那么，存储一个bit '0'
2. 如果D位于区间[-63, 64]，存储2个bits '10'，后面跟着用7个bits表示的D值
3. 如果D位于区间[-255, 256]，存储3个bits '110'，后面跟着9个bits表示的D值
4. 如果D位于区间[-2047, 2048]，存储4个bits '1110'，后面跟着12个bits表示的D值
5. 如果D位于其它区间则存储4个bits '1111'，后面跟着32个bits表示的D值

**关于这些 Range 的选取，是出于个别 Data Points 可能会缺失的考虑。例如：**

假设正常的 Interval 为每60秒产生一个 Data Point，如果缺失一个 Data Point，那么，相邻的两个 Data Points 之间的 Delta 值为：60, 60, 121 and 59，此时，Delta Of Delta 值将变为： 0, 61, -62。

这恰好落在区间[-63, 64]之间。

如果缺失4个 Data Point，那么，Delta Of Delta 值将落在区间[-255, 256]之间。

下图针对 Gorilla 中的440,000条真实的 Data Points 采样数据，对 Timestamp 数据应用了 Delta-Of-Delta 编码之后的效果：

**96.39%的Timestamps只需要占用一个Bit的空间**，这样看来，压缩的效果非常明显。
57.6+
3,840
250

## Point Value压缩（XOR）

Gorilla 中限制 Point Value 的类型为**双精度浮点数**，在未启用任何压缩编码的前提下，每个 Point Value 理应占用64个 Bits。

同样，Facebook 在认真调研了 ODS 中的数据特点以后也有了这样一个**关键发现**：**在大多数 Time Series 中，相邻的 Data Points 的 Value 变化比较轻微**。这一点比较好理解，假设某一个 Time Series 关联某个仪器温度指标的监控，那么，温度的变化应该是渐进式的而非跳跃式的。**XOR 编码**就是用来描述相邻两条 Point Value 的变化部分，下图直观描述了 Point Value “24.0”与”12.0″的变化部分：

**XOR编码**详细原理如下：

- 第一个 `Value` 存储时不做任何压缩。
- 后面产生的每一个`Value`与前一个`Value`计算`XOR`值：
  1. 如果 XOR 值为0，即两个 Value 相同，那么存为'0'，只占用一个 bit。
  2. 如果 XOR 为非0，首先计算 XOR 中位于**前端**的和**后端**的0的个数，即 Leading Zeros 与 Trailing Zeros。
  - 第一个bit值存为'1'。
  - 如果Leading Zeros与Trailing Zeros与前一个XOR值相同，则第2个bit值存为'0'，而后，紧跟着去掉Leading Zeros与Trailing Zeros以后的**有效XOR值**部分。
  - 如果 Leading Zeros 与 Trailing Zeros 与前一个 XOR 值不同，则第2个 bit 值存为'1'，而后，紧跟着5个 bits 用来描述 Leading Zeros 的值，再用6个 bits 来描述**有效 XOR 值**的长度，最后再存储**有效 XOR 值**部分（这种情形下，至少产生了13个 bits 的冗余信息）

如下是针对 Gorilla 中1,600,000个 Point Value 采样数据采用了 XOR 压缩编码后的效果：

从结果来看：
- 只占用1个 bit 的 Value 比例高达**59.06%**，这说明约一半以上的 Point Value 较之上一个 Value 并未发生变化。
- 30%比例的 Value 平均占用26.6 bits，即上面的情形2.1。
- 余下的12.64%的 Value 平均占用39.6 bits，即上面的情形2.2。

另外，XOR 压缩编码机制，对于 Integer 类型的 Point Value 效果尤为显著。

## 合理的 Block 大小

**压缩算法通常都是基于 Block 进行的，Block 越大，效果越明显**。对于时序数据的压缩，则需要选择一个合理的时间窗大小的数据进行压缩。Facebook 测试了不同的时间窗大小的压缩效果：

可以看出来，随着时间窗的逐步变大，压缩效果的确越显著。但超过2个小时(120 minutes)的时间窗大小以后，随着时间窗口的逐步变大，压缩效果的改善并不明显。**时间窗口为2小时的每个 Data Point 平均只占用1.37 bytes**。

## 参考

- [Gorilla: A Fast, Scalable, In-Memory Time Series Database](https://www.vldb.org/pvldb/vol8/p1816-teller.pdf)
- [Facebook的时序数据库技术](https://bbs.huaweicloud.com/blogs/103626)
