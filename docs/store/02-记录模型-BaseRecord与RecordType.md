# 记录模型：BaseRecord 与 RecordType

BaseRecord（src/lib/BaseRecord.ts）
- 统一的最小接口：`{ id: RecordId<R>, typeName: string }`
- 泛型 `RecordId<UnknownRecord>` 绑定 id 与具体记录类型（TS 级安全）

RecordType（src/lib/RecordType.ts）
- 职责：
  - 记录工厂：`createRecordType<T>('book', { validator, scope }).withDefaultProperties(() => ({ ... }))`
  - 默认属性与可选 `ephemeral`（非持久字段）
  - id 生成与作用域（scope: document/session/presence）
- 作用：与 StoreSchema 一起，约束 Store 中哪些 typeName 合法，以及各自的验证/迁移策略。

记录作用域（scope）
- document：持久 + 同步；缺省用于文档数据
- session：仅本实例；可持久但不跨实例（偏好/本地状态）
- presence：跨实例同步但不持久（在线光标等）

最佳实践
- 保持记录不可变；所有更新走 `store.update(id, updater)` 返回新对象
- 设计字段时考虑 diff 成本（体积大对象尽量拆分/引用）
- 为每类记录提供严格 validator，避免 create/update 路径不一致

