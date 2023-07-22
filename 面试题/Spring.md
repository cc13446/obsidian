## Spring IoC

### IoC和DI

`IoC`：Inversion of Control
`DI`：Dependency Injection

Ioc指的是控制反转，对象不再控制自己的生命周期和他们之间的关系，而是交给类似Spring这样的框架来做，是一种思想，这种思想可以有多种实现方法，比如依赖查找、依赖注入。
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

### BeanFactory和ApplicationContext

BeanFactory实现了IoC的基本功能，提供了一个抽象的配置和对象的管理机制
- 读取bean配置文档
- 管理bean的加载、实例化
- 维护bean之间的依赖关系
- 负责bean的生命周期

ApplicationContext继承了BeanFactory，并提供了更完整的框架功能
- AOP支持
- 国际化支持
- 资源访问
- 事件传递

### 属性注入的原理
1. 通过反射拿到Bean需要注入的属性
2. 通过属性的类去查找相应的Bean
3. 如果不唯一，通过属性的name去查响应的Bean
4. 注入属性

可以将多个属性注入到一个List或者Map中

### 属性注入相关的注解
- `@Autowired`：优先根据类型注入，可以标注`required=false`
- `@Resource`：优先根据名字注入，没找到则报错
- `@Inject`：优先根据类型注入，没找到则报错

和`@Autowired`配合的注解：
- `@Qualifier`：标注属性对应Bean的名字
- `@Primary`：如果有多个相同类型的 Bean 同时注册到 IOC 容器中，优先注入此主角标注的



