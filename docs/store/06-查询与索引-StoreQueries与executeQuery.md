# 查询与索引：StoreQueries 与 executeQuery

StoreQueries（src/lib/StoreQueries.ts）
- 提供：
  - `records(typeName?, expr?)`：按类型与表达式过滤的 reactive 结果
  - `index(typeName, prop)`：建立 RSIndex（属性值 → 记录 id 集合）
  - `filterHistory(typeName?)`：按类型分流的变更流
  - 内部维护多个 `Computed` 与缓存，按 diff 增量更新

RSIndex（概念）
- 形态：`Computed< Map<PropertyValue, Set<RecordId>>, DiffMap<PropertyValue, CollectionDiff<RecordId>> >`
- 维护：当 `RecordsDiff` 到来时，只更新受影响记录对应的键值集合（add/remove），空集合自动清理

executeQuery（src/lib/executeQuery.ts）
- 将表达式（eq/neq/in/range/exists 等）编译为增量可执行单元；
- 结合索引命中与 set 运算（见 `IncrementalSetConstructor.ts` 与 `setUtils.ts`），在变更时只更新必要集合。

性能要点
- 索引是“主动维护”的：写入成本稍高，但读和订阅成本极低；
- 大型集合建议先建索引再用表达式组合；
- 对只访问一次的数据使用 `unsafeGetWithoutCapture`/`serialize` 等非 reactive 读，避免多余订阅。

