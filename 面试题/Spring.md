## Spring IoC

### IoC和DI

全称：
- `IoC`：Inversion of Control
- `DI`：Dependency Injection

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

### 回调注入

实现回调接口进行注入

| 接口名                         | 作用                                 |
| ------------------------------ | ------------------------------------ |
| BeanFactoryAware               | 回调注入 BeanFactory                 |
| ApplicationContextAware        | 回调注入 ApplicationContext          |
| EnvironmentAware               | 回调注入 Environment                 |
| ApplicationEventPublisherAware | 回调注入事件发布器                   |
| ResourceLoaderAware            | 回调注入资源加载器（xml驱动可用）    |
| BeanClassLoaderAware           | 回调注入加载当前 Bean 的 ClassLoader |
| BeanNameAware                  | 回调注入当前 Bean 的名称             |
|                                |                                      |

### 延迟注入

代码实例如下：
```java
@Component 
public class Dog { 

	private Person person;
	 
	@Autowired 
	public void setPerson(ObjectProvider<Person> person) { 
		// 有Bean才取出，注入 
		this.person = person.getIfAvailable(); 
	}
	
}
```

### FactoryBean

当一个Bean的创建比较复杂的时候，就需要工厂Bean来创建了：
```java
public interface FactoryBean<T> {
    // 返回创建的对象
    @Nullable
    T getObject() throws Exception;

    // 返回创建的对象的类型（即泛型类型）
    @Nullable
    Class<?> getObjectType();

    // 创建的对象是单实例Bean还是原型Bean，默认单实例
    default boolean isSingleton() {
        return true;
    }
}
```


#### 加载时机
`FactoryBean` 本身的加载是伴随 IOC 容器的初始化时机一起的
`Bean`的加载是当容器获取Bean的时候进行加载的

### Spring事件

#### 观察者模式
**观察者模式**，也被称为**发布订阅模式**，也有的人叫它**监听器模式**，行为型模式之一。观察者模式关注的点是某一个对象被修改 / 做出某些反应 / 发布一个信息等，会自动通知依赖它的对象。

观察者模式的三大核心是：**观察者、被观察主题、订阅者**。

观察者需要绑定要通知的订阅者，并且要观察指定的主题。

#### Spring事件的观察者模式
事件驱动核心概念划分为 4 个：**事件源、事件、广播器、监听器**。
- 事件源：发布事件的对象
- 事件：事件源发布的信息 / 作出的动作
- 广播器：事件真正广播给监听器的对象
- 监听器：监听事件的对象

广播器也就是`ApplicationContext`
- 实现 `ApplicationEventPublisher` 接口，具备事件广播器的**发布事件**的能力
- `ApplicationEventMulticaster` 组合了所有的监听器，具备事件广播器的**广播事件**的能力

监听器接口：
```java
@FunctionalInterface
public interface ApplicationListener<E extends ApplicationEvent> 
	extends EventListener {
	
	void onApplicationEvent(E event);
}
```

不仅可以实现接口，也可以用注解监听
```java
@Component
public class ContextClosedApplicationListener {
    
    @EventListener
    public void onContextClosedEvent(ContextClosedEvent event) {
        System.out.println("监听到ContextClosedEvent事件！");
    }
}
```

### 生命周期

Spring Bean 的**初始化**流程如下：
1. Spring 容器根据配置中的 Bean Definition **实例化** Bean 对象
2. Spring 使用依赖注入**填充**所有属性
3. 检查 Aware 相关接口并**回调注入**相关依赖
4. BeanPostProcessor 前置处理
5. 如果实现 `InitializingBean` 接口，则会调用 `#afterPropertiesSet()` 方法
6. 如果为指定了 `init` 方法，那么将调用该方法
7. BeanPostProcessor 后置处理

Spring Bean 的**销毁**流程如下：
1. 如果实现 `DisposableBean` 接口，则执行 `destory()` 方法
2. 如果配置自定义的 `detory-method` 方法，则执行

### 循环依赖

首先，对于构造器注入的循环依赖，是没有办法解决的，所以只能抛出异常。

Spring为了解决循环依赖，设置了三级缓存（实际两层就够）
- 一级缓存(完成品缓存)：存放的是单例 bean 的映射，这里的Bean都是已经创建过的
- 二级缓存(半成品缓存)：存放的是未初始化完的 bean
- 三级缓存(半成品缓存)：存放的是`ObjectFactory`，可以理解为创建未初始化完的 bean 的 factory

Spring 解决 singleton bean 的核心就在于提前曝光 bean
- 这里指的是将还没完成依赖注入的bean先放到半成品缓存中

下面是获取Bean的流程：
1. 先从一级缓存中取，如果不为空就直接返回
2. 如果没有获取到，而且指定的Bean正在创建，则从二级缓存中获取
3. 如果没有获取到，则从三级缓存中获取Bean工厂，通过工厂方法获取Bean，并放入二级缓存

举个例子：
1. 通过构建函数创建A对象（A对象是半成品，还没注入属性和调用init方法）
2. A对象需要注入B对象，发现缓存里还没有B对象，将`半成品对象A`放入`半成品缓存`
3. 通过构建函数创建B对象（B对象是半成品，还没注入属性和调用init方法）
4. B对象需要注入A对象，从`半成品缓存`里取到`半成品对象A`
5. B对象继续注入其他属性和初始化，之后将`完成品B对象`放入`完成品缓存`
6. A对象继续注入属性，从`完成品缓存`中取到`完成品B对象`并注入
7. A对象继续注入其他属性和初始化，之后将`完成品A对象`放入`完成品缓存`

为什么要有三级缓存呢，只用两级缓存也可以实现呀？
1. 三级缓存给对象包了一层`ObjectFactory`，用来实现AOP等一些代理
2. 如果没有循环依赖的话，Spring是在初始化的最后一步才创建AOP代理的，有循环依赖只能提前了

如果对象A和对象B循环依赖，且都有代理的话，那创建的顺序就是
1. `A`半成品加入第三级缓存
2. `A`填充属性注入`B` -> 创建B对象 -> B半成品加入第三级缓存
3. `B`填充属性注入`A` -> 创建A代理对象，从第三级缓存移除，加入第二级缓存（A为半成品代理对象）
4. 创建B代理对象（此时B是完成品） -> 从第三级缓存移除B对象，B代理对象加入第一级缓存
5. A半成品注入B代理对象
6. 从第二级缓存移除A代理对象，A代理对象加入第一级缓存

## Spring AOP

### 一些概念

AOP：Aspect-Oriented Programming 面向切面编程

下面是一些概念：
- `Aspect`： 切面，类似于 Java 中的类声明
- `Join Point`：连接点，一个在程序执行期间的某一个操作，比如一个方法的执行
- `Advice`：通知，切面中对某个连接点采取的动作，一般把通知作为为拦截器，
- `Pointcut`：切入点，表示一组连接点，Spring默认使用AspectJ切入点表达式语言
- `Introduction`： 介绍，可以为原有的对象增加新的属性和方法
- `Target Object`：目标对象，由一个或者多个切面代理的对象
- `AOP proxy`：代理对象，由AOP框架创建的对象，Spring框架中有两种：JDK动态代理和CGLIB代理
- `Weaving`：织入，是指把切面应用到目标对象来创建新的代理对象的过程

切面的通知类型：
- `around`：环绕通知
- `before`：前置通知
- `after`：后置通知
- `exception`：异常通知
- `return`：返回通知

### 实现方式

Spring的AOP实现原理其实很简单，就是通过动态代理实现的。如果我们为Spring的某个bean配置了切面，那么Spring在创建这个bean的时候，实际上创建的是这个bean的一个代理对象，我们后续对bean中方法的调用，实际上调用的是代理类重写的代理方法。而Spring的AOP使用了两种动态代理，分别是JDK的动态代理，以及CGLib的动态代理。

#### JDK 动态代理

Spring默认使用JDK的动态代理实现AOP，类如果实现了接口，Spring就会使用这种方式实现动态代理。

JDK实现动态代理需要两个组件：
- `InvocationHandler`接口
- `Proxy`类

```java
public class ProxyFactory {
    public static HttpApi getProxy(HttpApi target) {
	    // 这里生成的代理类实现了原来那个类的所有接口，并对接口的方法进行了代理
	    // 我们通过代理对象调用这些方法时，底层将通过反射，调用我们实现的invoke方法
        return (HttpApi) Proxy.newProxyInstance(
		        target.getClass().getClassLoader(),
                target.getClass().getInterfaces(),
                new LogHandler(target));
    }

    private static class LogHandler implements InvocationHandler {
        private HttpApi target;

        LogHandler(HttpApi target) {
            this.target = target;
        }
        // method底层的方法无参数时，args为空或者长度为0
        // 这个方法其实就是我们提供的代理方法
        @Override
        public Object invoke(Object proxy, 
						    Method method, 
					        @Nullable Object[] args) throws Throwable {
            // 扩展的功能
            Log.info("http-statistic", (String) args[0]);
            // 访问基础对象
            return method.invoke(target, args);
        }
    }
}

```

大概原理是
- 代理类继承于`java.lang.reflect.Proxy`，并实现了要代理的接口
- `InvocationHandler`作为一个字段由代理类保存
- 代理类的所有方法调用都要路由到`InvocationHandler`的`invoke`方法
- 同时传入`target`对应的`method`，供代理调用原本的方法

正因为如此，JDK代理只能代理接口，是不能代理类的，因为已经继承了`proxy`。

#### CGLib动态代理

大概的原理是利用ASM开源包，将真实对象类的class文件加载进来，通过修改字节码生成其子类，覆盖父类相应的方法。

先看看怎么用的
```java
// 被代理对象
public class TargetClass {

    public void targetInfo(){
        System.out.println("打印目标类信息");
     }
}

// 代理拦截处理器
public class MyInterceptor implements MethodInterceptor {

	/**
     * 目标对象（也被称为被代理对象）
     */
    private Object target;


	public MyInterceptor (Object target) {
        this.target = target;
    }

    @Override
    public Object intercept(Object o, 
						    Method method, 
						    Object[] objects, 
						    MethodProxy methodProxy) throws Throwable {
						    
        System.out.println("------插入前置通知代码-------------");
        methodProxy.invoke(target, objects);
        System.out.println("------插入后置处理代码-------------");
        return null;
    }

	public <T> T getProxy() {
        Enhancer enhancer = new Enhancer();
        //设置被代理类
        enhancer.setSuperclass(target.getClass());
        // 设置回调
        enhancer.setCallback(this);
        // create方法正式创建代理类
        return (T) enhancer.create();
    }
}

// 使用代码
public static void main(String[] args) {

	// 被代理对象
	TargetClass target = new TargetClass();

	// 根据目标对象生成代理对象
	MyInterceptor proxy =  new MyInterceptor(target)

	// 创建子类 即代理
	TargetClass targetClassProxy = (TargetClass) proxy.getProxy();
	
	targetClassProxy.targetInfo();
}

```


##### FastClass 机制

FastClass 的原理简单来说就是：为代理类和被代理类各生成一个 Class，这个 Class 会为代理类或被代理类的方法分配一个 index，根据这个 index，FastClass 就可以直接定位要调用的方法直接进行调用，这样**省去了反射调用**，所以调用效率比 JDK 动态代理通过反射调用高。
