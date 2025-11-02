# 数组集合：ArraySet 与存储结构

位置：src/lib/ArraySet.ts

设计
- 小集合（<=8）用数组，越界后切换为原生 Set（混合结构）。
- 常量时间的 add/delete/has，且保留顺序遍历友好；
- 在信号父/子关系与 EffectScheduler 内被大量使用。

好处
- 避免为大量小集合引入 Set 的内存常量开销；
- 保持高频 has/insert/delete 的性能稳定。

