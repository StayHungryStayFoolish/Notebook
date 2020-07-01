# Redis 应用

## BloomFilter（布隆过滤器）

- **简介**
  - [WIKI](https://en.wikipedia.org/wiki/Bloom_filter) ：A **Bloom filter** is a space-efficient [probabilistic](https://en.wikipedia.org/wiki/Probabilistic) [data structure](https://en.wikipedia.org/wiki/Data_structure), conceived by [Burton Howard Bloom](https://en.wikipedia.org/w/index.php?title=Burton_Howard_Bloom&action=edit&redlink=1) in 1970, that is used to test whether an [element](https://en.wikipedia.org/wiki/Element_(mathematics)) is a member of a [set](https://en.wikipedia.org/wiki/Set_(computer_science)). [False positive](https://en.wikipedia.org/wiki/Type_I_and_type_II_errors) matches are possible, but [false negatives](https://en.wikipedia.org/wiki/Type_I_and_type_II_errors) are not – in other words, a query returns either "possibly in set" or "definitely not in set." Elements can be added to the set, but not removed (though this can be addressed with the [counting Bloom filter](https://en.wikipedia.org/wiki/Bloom_filter#Counting_Bloom_filters) variant); the more items added, the larger the probability of false positives.
  - **特点**
    - 一种空间效率高的 `概率型` 数据结构，用来检查一个元素是否在一个集合中
    - 对于一个元素检测是否存在的调用，BloomFilter 会返回两个结果之一：False positive（可能存在）或者False negatives（一定不存在）
- **原理**
  - `Bloom Filter` 有两个要素：长度为 **n** 的 `bit array` 和 **m** 个独立的 `hash function`，当要写入元素 `x` 的时候，用所有的 `hash function`  对 `x`  进行 `hash 后 mod n ` 得到 **m** 个位置，在 `bit array`  对应位置的 `bit` 设为 **1**，就完成了一次写入，验证元素是否存在，则只需要查看 `bit array` 上对应的 **m** 上的 `bit` 是否全部为 **1** 即可。 

