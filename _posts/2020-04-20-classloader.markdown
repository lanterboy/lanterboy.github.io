---
layout: post
title: 关于类加载时机的一道面试题
tag: [Java, 面试题, 类加载机制]
---
前几天面试被问到一个类加载相关的问题，答得有点乱，在此梳理一下

> 我们有 `M` 和 `N` 两个jar包，其中 `M` 里有 `A` 和 `B` 两个类， `N` 里有一个 `C` 类。已知 `B` 使用到了 `C` 类，而 `A` 并没有使用到 `C` 类。那么当我们的项目只引用了 `M` 包，却没有引用 `N` 包的代码时，编译可以通过吗？

在编译阶段我们只对代码进行检查和编译处理，其依赖会以符号引用的形式存储，因此编译时我们是不会检测jar包中的相互依赖的，jar包间的依赖检查在运行阶段类加载时才会发生。

对于问题中的场景，如果我们只是使用 `A` 类，则不会有问题。因为 `A` 类并没有依赖 `N` 包， `A` 类的整个生命周期都不会产生依赖不到的问题。

而如果我们使用了 `B` 类，则要看具体的使用情况。是否能正确运行取决于 `B` 类使用过程中是否涉及到了 `C` 类的加载。假设我们在调用时，只用到了 `B` 类的某些方法，在整个调用过程中都不涉及对 `C` 类的加载，则可以正常调用，反之则会抛出 `java.lang.NoClassDefFoundError` 错误。

> 什么情况下会加载 `C` 类，什么时候初始化？

上述场景中的 `java.lang.NoClassDefFoundError` 错误发生在 Linking 阶段，即加载到初始化之间。根据《深入理解Java虚拟机》第二版的说法，Java虚拟机规范中没有对加载的时机作强制约束，因此虚拟机的具体实现可以自由把握加载的时机。虽然加载阶段是不确定的，但是其一定在初始化之前（查资料时发现，网上很多博客把类加载和类初始化的时机混淆了，认为初始化类的行为就是加载）。

![class-init.jpg](https://wx1.sbimg.cn/2020/04/20/class-init.jpg)

虚拟机在遇到 `new`、`getstatic`、`putstatic` 或 `invokestatic` 这四条字节码指令时，如果没有进行过初始化，则会触发该类的初始化，这四条指令对应的常见场景

- 使用 `new` 关键字实例化对象的时候；
- 读取或设置一个类的静态字段（被 `final` 修饰，已在编译器把结果放入常量池的静态字段除外）的时候；
- 调用一个类的静态方法的时候。

因此，如果我们在应用程序中对 `C` 类进行了上述操作，则一定会加载 `C` 类，从而出现  `java.lang.NoClassDefFoundError` 错误。反之则不会出现错误。

> 这里会有一点疑问，既然我们只能确定类初始化的时机，那么有没有可能，我们并未执行上述操作，虚拟机却提前加载了 `C` 类呢，这种情况下会报错吗？

虚拟机确实存在提前加载还未使用的类的操作，称之为预加载，但是并不会直接抛出错误，而是等到使用该类时再抛出。通过 `-XX:+TraceClassLoading` 参数，我们可以打印出类加载信息，可以任写一个空的 main 方法执行，会发现 rt 包下的很多类都被加载了，但是并没有使用，这就是预加载。

回到我们的问题上来，可以通过如下代码验证我们的推测

```java
public static void main(String[] args) throws ClassNotFoundException {
    A a = new A();
    a.print();
    A.class.getClassLoader().loadClass("jar.test1.B"); // 这行只是加载了，并不会报错
    System.out.println("加载B类完成");
    B b = new B(); // 初始化时报错
}
```

`B` 类的代码如下

```java
public class B {

    private static final C c = new C();

    public B() {
        System.out.println("初始化B类完成");
    }

    public void print() {
        System.out.println(this.getClass().toString());
    }
}
```

我们使用 `A` 类的类加载器预加载了 `B` 类，这时还没有使用它，因此并不会立即报错。通过上述的 `-XX:+TraceClassLoading` 参数执行该段代码，输出如下

![class-loader-test.png](https://wx1.sbimg.cn/2020/04/20/class-loader-test.png)

可以看到 `B` 类被加载了，直到真正使用时才抛出错误。

注意到该场景下的 `NoClassDefFoundError` 是虚拟机在找不到类定义时的错误，容易与`ClassNotFoundException` 异常混淆。前者是错误，后者是异常。这里自然就会谈到 `Error` 和 `Exception` 的区别，`java.lang.Error` 和 `java.lang.Exception` 都继承自父类 `java.lang.Throwable` ，网上很多相关的介绍，这里就不赘述了。总结一点就是，显式地通过代码去加载类时抛出的是异常，这种情况是可以捕获并做相应的补救措施的，而 `NoClassDefFoundError` 则是虚拟机环境问题，通过程序代码是无法补救的。