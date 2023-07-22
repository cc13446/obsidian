## Spring IoC

### IoC和DI
IoC:Inversion of Control
DI:Dependency Injection

Ioc指的是控制反转，对象不再控制自己的生命周期和他们之间的关系，而是交给类似Spring这样的框架来做，是一种思想。
DI指的是依赖注入，是IoC的一种实现方式，通过依赖注入，Spring可以实现IoC。

### 依赖注入的三种方式
依赖注入有三种方式：
- 基于 field 注入（属性注入）
- 基于 setter 注入
- 基于 constructor 注入（构造器注入）

为什么不推荐使用属性注入了呢？
1. 容易违背单一指责原则，注入过多属性使得类比较臃肿
2. 依赖注入机制和容器耦合，不能脱离依赖注入来创建对象

应该怎么用呢？
1. 强制依赖：使用构造期注入
2. 可变依赖：使用 setter 注入
