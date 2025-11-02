# localStorageAtom 与持久化小工具

位置：src/lib/localStorageAtom.ts

用途
- 提供 `localStorageAtom<T>(key, initial, { serialize, deserialize })`，将 Atom 值持久化到 localStorage。
- 首次读取尝试从 localStorage 反序列化，失败时回退初始值；写入时序列化保存。

注意
- 序列化/反序列化函数需健壮处理版本与空值；
- 在隐私模式/无 localStorage 环境下需容错。

