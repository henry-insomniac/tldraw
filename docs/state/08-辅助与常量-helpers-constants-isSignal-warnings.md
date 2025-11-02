# 辅助与常量：helpers/constants/isSignal/warnings

helpers.ts
- `equals(a,b)`：严格等/`Object.is`/原型 `equals` 链；
- `haveParentsChanged(signal)`：比较 `parentEpochs` 与 `parents[].lastChangedEpoch`；
- `singleton(key, factory)`：全局唯一类/函数实例持有。

constants.ts
- `GLOBAL_START_EPOCH` 初始 epoch；常量开关等。

isSignal.ts
- 类型守卫：判断对象是否为信号（Atom/Computed）。

warnings.ts
- 开发期警告（例如 computed getter 被误用时提示使用装饰器）。

