转载自`https://pdai.tech`

# Spring Bean是什么
Spring里面的bean就类似是定义的一个组件，而这个组件的作用就是实现某个功能的，这里所定义的bean就相当于给了你一个更为简便的方法来调用这个组件去实现你要完成的功能。

# IoC是什么

`Inversion of Control`即控制反转，不是什么技术，而是一种设计思想。在Java开发中，`Ioc`意味着将你设计好的对象交给容器控制，而不是传统的在你的对象内部直接控制。

1. 谁控制谁，控制什么
	 - 传统Java SE程序设计，直接在对象内部通过new创建对象，是程序主动去创建依赖对象
	 - 而IoC是有专门一个容器来创建这些对象，即由Ioc容器来控制对象的创建
	 - 谁控制谁？当然是IoC 容器控制了对象
	 - 控制什么？那就是主要控制了外部资源获取

2. 为何是反转，哪些方面反转了
	 - 传统应用程序是由我们自己在对象中主动控制去直接获取依赖对象，也就是正转；
	 - 而反转则是由容器来帮忙创建及注入依赖对象；
	 - 为何是反转？因为由容器帮我们查找及注入依赖对象，对象只是被动的接受依赖对象
	 - 哪些方面反转了？依赖对象的获取被反转了。

# IoC能做什么
IoC 不是一种技术，只是一种思想，一个重要的面向对象编程的法则，它能指导我们如何设计出松耦合、更优良的程序。

传统应用程序都是由我们在类内部主动创建依赖对象，从而导致类与类之间高耦合，难于测试；有了IoC容器后，把创建和查找依赖对象的控制权交给了容器，由容器进行注入组合对象，所以对象与对象之间是松散耦合，这样也方便测试，利于功能复用，更重要的是使得程序的整个体系结构变得非常灵活。

其实IoC对编程带来的最大改变不是从代码上，而是从思想上，发生了主从换位的变化。应用程序原本是老大，要获取什么资源都是主动出击，但是在IoC/DI思想中，应用程序就变成被动的了，被动的等待IoC容器来创建并注入它所需要的资源了。

IoC很好的体现了面向对象设计法则之一：好莱坞法则：“别找我们，我们找你”；即由IoC容器帮对象找相应的依赖对象并注入，而不是由对象主动去找。

# IoC和DI是什么关系

控制反转是通过依赖注入实现的，其实它们是同一个概念的不同角度描述。通俗来说就是
- IoC是设计思想，DI是实现方式

依赖注入：组件之间依赖关系由容器在运行期决定，形象的说，即由容器动态的将某个依赖关系注入到组件之中。依赖注入的目的并非为软件系统带来更多功能，而是为了提升组件重用的频率，并为系统搭建一个灵活、可扩展的平台。通过依赖注入机制，我们只需要通过简单的配置，而无需任何代码就可指定目标需要的资源，完成自身的业务逻辑，而不需要关心具体的资源来自何处，由谁实现。

1. 谁依赖于谁？当然是应用程序依赖于IoC容器；
2. 为什么需要依赖？应用程序需要IoC容器来提供对象需要的外部资源；
3. 谁注入谁？是IoC容器注入应用程序某个对象，应用程序依赖的对象；
4. 注入了什么？就是注入某个对象所需要的外部资源（包括对象、资源、常量数据）。

IoC和DI是同一个概念的不同角度描述，由于控制反转概念比较含糊，所以2004年大师级人物Martin Fowler又给出了一个新的名字：依赖注入，相对IoC 而言，依赖注入明确描述了被注入对象依赖IoC容器配置依赖对象。通俗来说就是**IoC是设计思想，DI是实现方式**。