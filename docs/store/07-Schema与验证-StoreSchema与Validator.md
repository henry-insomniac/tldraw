# Schema 与验证：StoreSchema 与 Validator

StoreSchema（src/lib/StoreSchema.ts）
- 内容：类型注册（typeName → RecordType）、作用域信息聚合、schema 序列化与迁移协调。
- 作用：提供 `validateRecord(store, record, phase, knownGood?)`、`migrateStoreSnapshot(snapshot)` 等能力。

验证（Validator）
- 接口：`StoreValidator<R> { validate(record), validateUsingKnownGoodVersion?(knownGood, record) }`
- create：在 RecordType 中配置 validator；Store 在 put/update 时统一调用。

迁移协同
- StoreSchema.serialize() 输出版本信息供 StoreSnapshot；
- `migrateStoreSnapshot(snapshot)` 调用 `migrate.ts` 的迁移序列，升级到当前版本。

建议
- 校验逻辑需与迁移后的结构一致；更新路径也要走校验，避免“创建可过、更新不行”的不一致。
- 针对大对象可实现 `validateUsingKnownGoodVersion`，利用旧值降低校验成本。

