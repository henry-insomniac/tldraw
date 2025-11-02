# 迁移系统详解（migrate.ts）

目标：将历史数据安全升级到当前版本；支持回滚、依赖、子类型迁移与 legacy 兼容。

关键概念（src/lib/migrate.ts）
- MigrationId：带命名空间的版本 id（如 `com.myapp.book/1`），`parseMigrationId` 校验解析。
- MigrationSequence：有序迁移序列，支持依赖与拓扑排序。
- createMigrationSequence({ sequenceId, sequence })：定义一条序列（up/down）。
- 子类型迁移：对记录内部嵌套/属性级别进行迁移。

Store 协作
- StoreSnapshot：{ store: SerializedStore, schema: SerializedSchema }
- StoreSchema.migrateStoreSnapshot：读取 snapshot.schema 版本，执行需要的序列 up；失败返回 error。

最佳实践
- 小步快跑：一次迁移做一件事，保持幂等与可回滚。
- 真实数据测试：在测试中引入旧版样本，验证 up/down 与数据完整性。
- 文档：为 breaking change 写明迁移要求与注意事项。

