# lambda
lambda表达式是使用尽可能少的语法编写的函数定义。尽管lambda表达式产生的是函数，而不是类，但是在JVM中一切都是类，所以幕后会有一些操作让lambda作为一个类但是看起来像是函数。底层编译器在类中生成一个静态函数，运行时调以内部类形式调用该静态函数。

例子：
```java
interface Description {
	String brief();
}

interface Body {
	String detailed(String head);
}

interface Multi {
	String twoArg(String head, Double d);
}

public class LambdaExpressions {

	static Body bod = h -> h + " No Parens!"; // [1]
	
	static Body bod2 = (h) -> h + " More details"; // [2]
	
	static Description desc = () -> "Short info"; // [3]
	
	static Multi mult = (h, n) -> h + n; // [4]
	
	static Description moreLines = () -> { // [5]
		System.out.println("moreLines()");
		return "from moreLines()";
	};
}
```

组成：
1. 参数。
2. 接着是 `->`，您可以选择将其视为生成
3. `->`之后的所有东西都是方法体。

## lambda递归
可以编写递归的 lambda 表达式，但需要注意：递归方法必须是实例变量或静态变量，否则会出现编译时错误。
```java
interface IntCall {
	int call(int arg);
}

public class RecursiveFactorial {
	static IntCall fact;
	public static void main(String[] args) {
		fact = n -> n == 0 ? 1 : n * fact.call(n - 1);
		for(int i = 0; i <= 10; i++)
			System.out.println(fact.call(i));
	}
}
```

## 方法引用

方法引用指的是没有以前版本的 Java 所需的额外包袱的方法。 方法引用是类名或对象名，后面跟`::`然后是方法的名称。
```java
import java.util.*;

// 单一接口方法
interface Callable {
	void call(String s);
}

class Describe {
	void show(String msg) {
		System.out.println(msg);
	}
}

public class MethodReferences {
	static void hello(String name) { 
		System.out.println("Hello, " + name);
	}
	static class Description {
		String about;
		Description(String desc) { about = desc; }
		void help(String msg) { // 静态内部类中的非静态方法
			System.out.println(about + " " + msg);
		}
	}
	static class Helper {
		static void assist(String msg) { // 静态内部类中的静态方法
			System.out.println(msg);
		}
	}
	public static void main(String[] args) {
		Describe d = new Describe();
		Callable c = d::show; // 方法签名一样
		c.call("call()"); 
		
		c = MethodReferences::hello; 
		c.call("Bob");
		
		c = new Description("valuable")::help;
		c.call("information");
		
		c = Helper::assist;
		c.call("Help!");
	}
}
/* Output:
call()
Hello, Bob
valuable information
Help!
*/
```

### 未绑定的方法引用

未绑定的方法引用是指没有关联对象的普通（非静态）方法。 要使用未绑定的引用，您必须提供以下对象：

```java
class X {
	String f() { return "X::f()"; }
}

interface MakeString {
	String make();
}

interface TransformX {
	// 我们需要一个x对象，所以我们的接口实际上需要一个额外的参数
	String transform(X x);
}

public class UnboundMethodReference {
	public static void main(String[] args) {
		// MakeString ms = X::f; 
		// 会产生编译器关于无效方法引用的错误
		// 即使 make()与f()具有相同的签名。 
		// 问题是实际上还有另一个隐藏的参数：this
		TransformX sp = X::f;
		X x = new X();
		System.out.println(sp.transform(x)); // [2]
		System.out.println(x.f()); // Same effect
	}
}
/* Output:
X::f()
X::f()
*/
```

### 构造函数引用

```java
class Dog {
	String name;
	int age = -1; // For "unknown"
	Dog() { name = "stray"; }
	Dog(String nm) { name = nm; }
	Dog(String nm, int yrs) { name = nm; age = yrs; }
}

interface MakeNoArgs {
	Dog make();
}

interface Make1Arg {
	Dog make(String nm);
}

interface Make2Args {
	Dog make(String nm, int age);
}

public class CtorReference {
	public static void main(String[] args) {
		MakeNoArgs mna = Dog::new; // [1]
		Make1Arg m1a = Dog::new;   // [2]
		Make2Args m2a = Dog::new;  // [3]
		
		Dog dn = mna.make();
		Dog d1 = m1a.make("Comet");
		Dog d2 = m2a.make("Ralph", 4);
	}
}
```

## 函数式接口

函数式接口`Functional Interface`就是一个有且仅有一个抽象方法，但是可以有多个非抽象方法的接口。函数式接口为lambda表达式和方法引用提供目标类型。每个功能接口都有一个抽象方法，称为该功能接口的功能方法，lambda表达式的参数和返回类型与之匹配或适配。

`Java8`为函数式接口引入了一个新注解`@FunctionalInterface`，主要用于编译级错误检查，加上该注解，当接口不符合函数式接口定义的时候，编译器会报错。此注解不是编译器将接口识别为功能接口的必要条件，而仅是帮助捕获设计意图并获得编译器帮助识别意外违反设计意图的帮助。

正确例子：
```java
@FunctionalInterface
public interface HelloWorldService {
    void sayHello(String msg);
}
```

函数式接口里是可以包含默认方法，因为默认方法不是抽象方法，其有一个默认实现，所以是符合函数式接口的定义的。

```java
@FunctionalInterface
public interface HelloWorldService {
    void sayHello(String msg);
    default void doSomeWork1() {
        // Method body
    }

    default void doSomeWork2() {
        // Method body
    }
}
```

函数式接口里是可以包含静态方法，因为静态方法不能是抽象方法，是一个已经实现了的方法，所以是符合函数式接口的定义的。
```java
@FunctionalInterface
public interface HelloWorldService {
    void sayHello(String msg);
    static void printHello() {
        System.out.println("Hello");
    }
}
```

函数式接口里是可以包含`Object`里的`public`方法，这些方法对于函数式接口来说，不被当成是抽象方法；因为任何一个函数式接口的实现，默认都继承了`Object`类，包含了来自`Object`里对这些抽象方法的实现；
```java
@FunctionalInterface
public interface HelloWorldService {

    void sayHello(String msg);

	@Override
    boolean equals(Object obj);

}
```

JDK1.8之前已有的函数式接口:
- `java.lang.Runnable`
- `java.util.concurrent.Callable`
- `java.security.PrivilegedAction`
- `java.util.Comparator`
- `java.io.FileFilter`
- `java.nio.file.PathMatcher`
- `java.lang.reflect.InvocationHandler`
- `java.beans.PropertyChangeListener`
- `java.awt.event.ActionListener`
- `javax.swing.event.ChangeListener`

JDK1.8新增加的函数接口:
- `java.util.function`

## 闭包
闭包的定义：内部函数可以访问函数外面的变量。

之前闭包只能使用`final`变量，从Java8开始，我们可以在匿名内部类中直接使用非`final`变量。不过，这样做是有前提的，就是这个局部变量不能被再被重新赋值！`effectively final`是Java8引入的新概念。一个非`final`的局部变量或方法参数，其值在初始化后就从未更改，那么该变量就是`effectively final`。