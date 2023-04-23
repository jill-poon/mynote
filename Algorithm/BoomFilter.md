# BoomFilter

## 布隆过滤器的误报概率  

假设数组长度为 ***m***, 并使用 ***k*** 个 hash 函数，***n*** 是要插入到过滤器中的元素个数，那么误报率的计算如下：  

$$
P=(1-[1-\frac{1}{m}]^{kn})^k
$$

## Bitmap 的大小

如果过滤器中的**元素数量已知**，期望的误报率为 ***P***，那么二进制位数组大小计算公式如下：  

$$m=-\frac{n\ln P}{(\ln2)^2}$$

## 最优哈希函数数量

如果 ***m* 是数组长度**，***n* 是插入的元素个数**，***k* 是 hash 函数的个数**，k 计算公式如下：

$$
k = \frac{m}{n}\ln{2}
$$