## HyperLogLog
- HyperLogLog 是用来做基数统计的算法，HyperLogLog 的优点是，在输入元素的数量或者体积非常非常大时，计算基数所需的空间总是固定 的、并且是很小的。

- 在 Redis 里面，每个 HyperLogLog 键只需要花费 12 KB 内存，就可以计算接近 2^64 个不同元素的基 数。这和计算基数时，元素越多耗费内存就越多的集合形成鲜明对比。

- 但是，因为 HyperLogLog 只会根据输入元素来计算基数，而不会储存输入元素本身，所以 HyperLogLog 不能像集合那样，返回输入的各个元素。


>HyperLogLog是一种算法，并非redis独有,目的是做基数统计，故不是集合，不会保存元数据，只记录数量而不是数值。

>耗空间极小，支持输入非常体积的数据量

>核心是基数估算算法，主要表现为计算时内存的使用和数据合并的处理。最终数值存在一定误差。
>redis中每个hyperloglog key占用了12K的内存用于标记基数（官方文档）pfadd命令并不会一次性分配12k内存，而是随着基数的增加而逐渐增加内存分配；而pfmerge操作则会将sourcekey合并后存储在12k大小的key中，这由hyperloglog合并操作的原理（两个hyperloglog合并时需要单独比较每个桶的值）可以很容易理解。

>误差说明：基数估计的结果是一个带有 0.81% 标准错误（standard error）的近似值。是可接受的范围

>`Redis 对 HyperLogLog 的存储进行了优化`，在计数比较小时，它的存储空间采用稀疏矩阵存储，空间占用很小，仅仅在计数慢慢变大，稀疏矩阵占用空间渐渐超过了阈值时才会一次性转变成稠密矩阵，才会占用 12k 的空间

### 应用场景 

- 用于数据量大的场景，有局限性，只能统计基数数量，无法知道具体内容是什么
    - 统计访问量(IP数)
    - 统计在线用户数
    - 统计每天搜索不同词条的个数
    - 统计文章真实阅读数

### Redis HyperLogLog 命令
- PFADD key element [element ...]
    - 添加指定元素到 HyperLogLog 中。
- PFCOUNT key [key ...]
    - 返回给定 HyperLogLog 的基数估算值。
- PFMERGE destkey sourcekey [sourcekey ...]
    - 将多个 HyperLogLog 合并为一个 HyperLogLog

