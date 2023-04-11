# Facebook Gorilla 时序数据压缩原理

Gorilla源自Facebook对于内部基础设施的近乎”变态”的监控需求，我们先来看看这些数据：

- 20亿个不同的Time Series

- 每分钟产生7亿个Data Points，即每秒钟产生1200万Data Points

- 数据需要存储26个小时

- 高峰期的查询高达40000次每秒

- 查询时延需要小于1ms

- 每个 Time Series每分钟可产生4个Data Points

- 每年的数据增长率为200%

对于监控数据，本身还具有如下几个典型特点：

- **重写轻读**

- 以**聚合分析**为主，几乎不存在针对单Data Point的点查场景。因此，即使丢失个别的Data Points，一般不会影响到整体的分析结果。

- 越老的数据价值越低，而应用通常只读取最近发生的数据。**Facebook的统计表明，85%的查询与最近26个小时的数据写入有关。**

## Timestamp压缩 (Delta-Of-Delta)

在大多数情形下，可以将一个Time Series中的连续的Data Points的Timestamp列表视作一个**等差数列**，这是Delta-Of-Delta编码算法的最适用场景。编码原理如下：

- 在Block Header中，存放起始Timestamp T(-1)..一个Block对应2个小时的时间窗口。假设第一个Data Point的Timestamp为T(0)，那么，实际存放时，我们只需要存T(0)与T(-1)的差值。

- 对于接下来的Data Point的Timestamp T(N), 按如下算法计算Delta Of Delta值：

    ```text
    D = (T(N) – T(N-1)) – (T(N-1) – T(N-2))
    ```

**按照D的值所处的区间范围，分别有5种不同情形的处理：**

1. 如果D为0，那么，存储一个bit '0'
2. 如果D位于区间[-63, 64]，存储2个bits '10'，后面跟着用7个bits表示的D值
3. 如果D位于区间[-255, 256]，存储3个bits '110'，后面跟着9个bits表示的D值
4. 如果D位于区间[-2047, 2048]，存储4个bits '1110'，后面跟着12个bits表示的D值
5. 如果D位于其它区间则存储4个bits '1111'，后面跟着32个bits表示的D值

**关于这些Range的选取，是出于个别Data Points可能会缺失的考虑。例如：**

假设正常的Interval为每60秒产生一个Data Point，如果缺失一个Data Point，那么，相邻的两个Data Points之间的Delta值为：60, 60, 121 and 59，此时，Delta Of Delta值将变为： 0, 61, -62。

这恰好落在区间[-63, 64]之间。

如果缺失4个Data Point，那么，Delta Of Delta值将落在区间[-255, 256]之间。

下图针对Gorilla中的440,000条真实的Data Points采样数据，对Timestamp数据应用了Delta-Of-Delta编码之后的效果：

**96.39%的Timestamps只需要占用一个Bit的空间**，这样看来，压缩的效果非常明显。
57.6+
3,840
250
## Point Value压缩（XOR）

Gorilla中限制Point Value的类型为**双精度浮点数**，在未启用任何压缩编码的前提下，每个Point Value理应占用64个Bits。

同样，Facebook在认真调研了ODS中的数据特点以后也有了这样一个**关键发现**：**在大多数Time Series中，相邻的Data Points的Value变化比较轻微**。这一点比较好理解，假设某一个Time Series关联某个仪器温度指标的监控，那么，温度的变化应该是渐进式的而非跳跃式的。**XOR编码**就是用来描述相邻两条Point Value的变化部分，下图直观描述了Point Value “24.0”与”12.0″的变化部分：

**XOR编码**详细原理如下：

- 第一个 `Value`存储时不做任何压缩。
- 后面产生的每一个`Value`与前一个`Value`计算`XOR`值：
  1. 如果XOR值为0，即两个Value相同，那么存为'0'，只占用一个bit。
  2. 如果XOR为非0，首先计算XOR中位于**前端**的和**后端**的0的个数，即Leading Zeros与Trailing Zeros。
  - 第一个bit值存为'1'。
  - 如果Leading Zeros与Trailing Zeros与前一个XOR值相同，则第2个bit值存为'0'，而后，紧跟着去掉Leading Zeros与Trailing Zeros以后的**有效XOR值**部分。
  - 如果Leading Zeros与Trailing Zeros与前一个XOR值不同，则第2个bit值存为'1'，而后，紧跟着5个bits用来描述Leading Zeros的值，再用6个bits来描述**有效XOR值**的长度，最后再存储**有效XOR值**部分（这种情形下，至少产生了13个bits的冗余信息）

如下是针对Gorilla中1,600,000个Point Value采样数据采用了XOR压缩编码后的效果：

从结果来看：

- 只占用1个bit的Value比例高达**59.06%**，这说明约一半以上的Point Value较之上一个Value并未发生变化。
- 30%比例的Value平均占用26.6 bits，即上面的情形2.1。
- 余下的12.64%的Value平均占用39.6 bits，即上面的情形2.2。

另外，XOR压缩编码机制，对于Integer类型的Point Value效果尤为显著。

## 合理的Block大小

**压缩算法通常都是基于Block进行的，Block越大，效果越明显**。对于时序数据的压缩，则需要选择一个合理的时间窗大小的数据进行压缩。Facebook测试了不同的时间窗大小的压缩效果：

可以看出来，随着时间窗的逐步变大，压缩效果的确越显著。但超过2个小时(120 minutes)的时间窗大小以后，随着时间窗口的逐步变大，压缩效果的改善并不明显。**时间窗口为2小时的每个Data Point平均只占用1.37 bytes**。

## 参考

- [Gorilla: A Fast, Scalable, In-Memory Time Series Database](https://www.vldb.org/pvldb/vol8/p1816-teller.pdf)
- [Facebook的时序数据库技术](https://bbs.huaweicloud.com/blogs/103626)
