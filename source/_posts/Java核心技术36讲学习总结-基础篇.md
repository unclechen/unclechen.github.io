---
layout: post
title: Java核心技术学习总结-基础
date: '2018-11-12'
tags:
  - Java
  - 编程语言
  - 面试
  - 基础
categories: 
  - [面试]
  - [技术]
---

## 0.介绍

基础知识就是内功，修炼内功可到达更高层次。

- 课程介绍：[https://time.geekbang.org/column/intro/82](https://time.geekbang.org/column/intro/82)，这里面因为篇幅限制，不可能像写教科书一样面面俱到，更多都是在抛砖引玉，点到为止。涉及关键知识点需要自己再深入地去研究和学习，非常适合参加面试/面试别人的人，也比较适合夯实Java基础的人。

- 目录：

<img src="https://ws4.sinaimg.cn/large/006tNbRwly1fxb7ky9nuwj30ku2y9gyb.jpg" width="480">

<!--more-->

## 1.对JAVA的平台基本理解



Write once, run anywhere.

- jre、jdk

    - jre包含了Java的核心类库，运行环境，JVM。

    - jdk是jre的超集，包含了编译器，诊断工具等。

- 解释执行、编译执行：主流 Java 版本中，如 JDK 8 实际是解释和编译混合的模式（-Xmixed）。JVM会收集足够多的调用信息，进行高效的编译，这就是预热过程。Oracle JDK9的引入了实验性的AOT特性，直接把字节码编译成机器码（编译期间，静态的），避免预热开销。

    - 解释执行：JVM内嵌的解释器，将编译好的字节码解释成机器码执行。

    - 编译执行：Hotspot JVM，提供了JIT（Just In Time）编译器，将热点代码编译成机器码执行。这需要运行时才会知道哪些是热点代码。

> 解释执行，有点像涮火锅，边涮边吃；编译执行有点像点菜，做好了端上来直接吃。



## 2.Exception和Error有什么区别



- Throwable的继承关系
    - Error：不可恢复的异常，也不应该恢复程序。
    - Exception：可恢复的异常
        - Checked Exception：代码中必须进行显示地捕获，比如IOException，在编译器会进行检查的。
        - Unchecked Exception：比如NPE、ArrayIndexOutOfBoundsException这些RuntimeException。

<img src="https://ws4.sinaimg.cn/large/006tNbRwly1fx55v8lub0j31aq0o8gp9.jpg" width="640"/>

- `Try...Catch`的注意点：除了注意以外，还可以用到一些更加现代的特性，比如JDK1.7的`try-with-resources`和`multiple catch`.
    - 不要捕获`Exception`这样通用异常
    - 不要生吞异常：捕获之后，什么也不做。
    - `try`块不宜过大：额外性能开销，影响JVM对代码的优化。所以不要用`try...catch`来实现控制流，也不要把`try...catch`放到循环体内。
    - `finally`：不要在`finally`里面处理返回值。虽然try里面可以写return，但是finally块中的代码仍然会被执行，如果try中返回的是对象引用，而finally又修改了这个对象，那方法的返回值就会被修改成finally的结果。因为finally的处理前，会将try里面return的**对象**暂存起来，执行完finally块中的代码之后，再把返回值返回。



- 自定义`Exception`：
    - 实例化`Exception`：会对当前的栈进行快照，该操作比较重。但是也可以创建不进行栈快照的Exception（Native底层实现）。

> 当你去new一个Exception的时候，会调用父类Throwable的构造函数，Throwable的构造函数中会调用native的fillInStackTrace()，这个方法就会构造整个异常栈了。


- NoClassDefFoundError和ClassNotFoundException的区别？两者的区别首先是Error和Exception的区别。细一点说：
	- NoClassDefFoundError：当JVM或ClassLoader实例试图在类的定义中加载（作为通常方法调用的一部分，或者是使用`new`来创建新的对象）时，却找不到类的定义（要查找的类在编译的时候是存在的，运行的时候却找不到了，比如打包的时候误exclude掉了），抛出此异常。
	- ClassNotFoundException：动态加载某一个类的时候，例如`Class.forName`、`ClassLoader.loadClass`、`ClassLOader.findSystemClass`来通过一个类的名字，动态载入一个类时，如果没有在classPath里面找到它，则抛出这个异常。此外，当一个类已经某个类加载器加载到内存中了，此时另一个类加载器又尝试着动态地从同一个包中加载这个类。（违背了JVM的双亲加载机制）





## 3.final、finally、finalize有什么区别

单纯分清楚三者的区别，的确是一个非常基础的问题。finally比较简单，用于try-finally或者try-catch-finally，保证资源的释放（比如关闭JDBC连接、unlock锁、文件描述符的关闭等）。


- final：
    - 修饰不希望发生修改的类、方法、变量：类不可以继承、方法不可以重写、变量不可以修改。
    - final修饰变量时，可以产生一定程度的Immutable效果，但是不可以修改指的是基础类型。如果修饰的是对象，只能限制不能修改对象的引用不会被再次赋值，但是引用本身指向的对象行为，例如`final List<String> list = new ArrayList<String>();`，仍然可以add元素。Java本身没有原生的不可变immutable类，需要类自己支持这个功能。
        - `Collections.unmodifiableList(list);`可以实现，但与Guava相比，据说要差一些。因为原始对象还是可能被修改，如果原始对象修改了，生成的对象也会变。
        - Java 9的List.of创建的就是不可变List，调用add会抛`UnsupportedOperationException`异常。
        - Guava的`ImmutableList.copyOf(list);`也有类似能力。

```
final List<String> strList = new ArrayList<>();
strList.add("Hello");
strList.add("world");  
List<String> unmodifiableStrList = List.of("hello", "world");
unmodifiableStrList.add("again"); // 抛异常
```

- Java如何实现一个Immutable的类？（使用不可变对象，是一种防御性编程技巧，因为输入的对象可能被修改。或者自己的对象传给第三方库时，也可能被修改。此外也不用担心线程安全，不会产生竞态。）

    - 用final修饰类

    - 成员都用private final修饰，且不要setter方法

    - 构造对象时，采用深拷贝来初始化，而不是直接赋值。

    - 需要实现getter类的方法时，采用copy-on-write的原则，创建私有的copy。



- finalize：只需要了解它是Object类中的方法，会在对象被垃圾收集前调用。如果我们自己实现了非空的这个方法，那么可能导致对象回收变慢几十倍。所以它会拖慢垃圾回收，导致对象堆积，OOM。JAVA平台正在逐步使用java.lang.ref.Cleaner来替换所有的finalize实现，它用的是幻象引用。



## 4.强引用、软引用、弱引用、幻象引用有什么区别

这几个概念都是和内存管理、垃圾回收有关的，不同之处在于它们的生命周期长短。

- 强引用：存活时间最长的对象所关联的引用，例如`Object obj = new Object();`创建的对象引用。只有当这个对象没有引用了，或者显示地赋值为null，才有可能被垃圾收集器回收。

- 软引用：SoftReference，介于强与弱之间的对像引用，只有在JVM认为内存不足时，才会被回收。JVM会在抛出OOM之前，清理软引用指向的对象。软引用所引用的对象被回收之后，会被加入ReferenceQueue中。

- 弱引用：WeakReference，最弱的对象引用，JVM随时可能回收它（下一次GC扫描时，不管当时内存是否充足）。同样可以和ReferenceQueue一起使用，当弱引用所引用的对象被回收会后，引用会被加入ReferenceQueue中。

- 虚引用（幻象引用）：PhantomReference，无法用来引用对象的引用。它只是一种可以在对象被finalize之后，做一些事情的机制。它必须和ReferenceQueue一起使用，当它引用的对象被回收后，引用会被加入ReferenceQueue中，我们的程序可以通过判断ReferenceQueue中是否加入了虚引用来判断对象是否**将要**被回收，如果发现它了可以做一些事情。也可以利用幻象引用监控对象的创建和销毁。


上面4中引用可以使得对象进入不同的可达状态，也可以理解为对象的生命周期。

<img src="https://ws1.sinaimg.cn/large/006tNbRwly1fx5h6kikxuj30um13a7a1.jpg" width="500" />

- ReferenceQueue：引用队列，与软引用、弱引用、幻象引用联合使用。创建引用，关联到响应对象时，可以选择是否关联到引用队列。JVM会在合适的时机将引用enqueue到引用队列中。

```
// 前面说了幻象引用必须和引用队列联合使用

Object counter = new Object();
ReferenceQueue refQueue = new ReferenceQueue<>();
PhantomReference<Object> p = new PhantomReference<>(counter, refQueue);
counter = null;
System.gc();
try {
    // Remove 是一个阻塞方法，可以指定 timeout，或者选择一直阻塞
    Reference<Object> ref = refQueue.remove(1000L);
    if (ref != null) {
        // do something
    }
} catch (InterruptedException e) {
    // Handle it
}

```

- 关于引用，还有一篇不错的文章：[http://www.kdgregory.com/index.php?page=java.refobj](http://www.kdgregory.com/index.php?page=java.refobj)，其实几种引用，实际做业务开发时用的不多。以前一些有Java背景的开发者，常在Android中的图片缓存使用软引用和弱引用，现在谷歌官方也不再推荐使用了（原因是Android2.3之后Gabage Collector很积极地回收它们，从而使得它们的作用微乎及微）。关于3中非强引用的认识，可以看看这篇[Gist](https://gist.github.com/unclechen/a155ea44c45e1dd7c17fba2626ab187b)



## 5.String、StringBuffer、StringBuilder有什么区别



- String：Java中最常用的类之一，是一个Immutable类。被声明为final class，属性也都是final的。由于是Immutable的，所以裁剪、拼接都会产生新的对象。

- StringBuffer：为了解决String操作时产生太多新对象而出现的，可以使用append、add方法，将字符串添加到已有序列的末尾或者指定位置（char[]数组），它是一个线程安全的可修改字符序列，里面每个修改数据的方法都添加了synchronized修饰，因此性能也不是特别好。如果不需要线程安全，那么推荐使用StringBuilder。

- StringBuilder：JDK1.5新增，与StringBuffer功能完全相同（也都是继承了AbstractStringBuilder抽象类），去掉了线程安全，有效减小了开销，大部分情况下推荐使用这个。

- 使用细节：

    - StringBuffer、StringBuilder都是用到char[]数组存储着实际的字符串序列，并设置一个初始大小，每次添加前都会确认容量，如果不够的话，会尝试扩容(`int newCapacity = value.length * 2 + 2;`)，然后再把原来的内容拷贝到扩容后的char[](`value = Arrays.copyOf(value, newCapacity);`，这个方法底层是native实现)。可以看出扩容开销不小，如果我们是用的时候知道最终需要的容量，那么可以指定一个大小，避免这种开销，如果不指定的话，那么默认初始化的大小是16。

    - 是不是所有的字符串拼接都需要用StringBuilder（如果不需要考虑线程安全的话）呢？其实也不是，例如这段代码`String strByConcat = "aa" + "bb" + "cc" + "dd";`其实被JDK1.8的javac编译之后，会自动转换成StringBuilder的实现。只有在大量频繁拼接字符串的时候，需要使用SB，例如请求参数的拼接等场景。

    - **字符串常量池：**Java为了避免产生大量的String对象，引入了字符串常量池。

        - 当使用`String s = "a";`来创建一个字符串时，先去常量池查找是否有值相同的字符串对象（equals判断），如果有那么直接返回这个对象的引用，而不是创建新的对象；如果没有那么创建新的字符串对象，返回对象引用，并把新的对象放入常量池（这个技术可以大幅度节省内存空间）。

        - 当使用`String s1 = new String("a");`创建字符串时，不会去检查字符串常量池，而是直接在java heap创建一个String对象，栈上创建这个对象的引用，也不会把这个字符串放到常量池。

        - 使用`String#intern`方法（native实现的），可以把new出来的对象缓存到常量池中。不过这个方法不是一种很好的实践（需要人工调用，污染了代码），而且不同的JDK版本底层实现也不相同。而且在JDK1.6中，JVM对常量的缓存位置是在Perm Space（永久代很小，只有4M，一旦常量池中大量使用intern会直接产生java.lang.OutOfMemoryError: PermGen space错误）；在JDK1.7中，字符串常量池移动到了堆中，大小也从1009变成了可以调整`-XX:StringTableSize=99991`，且还有一个优化，**就是如果堆中已经存在了对象，不会创建对象，而会保存对象的引用**；而JDK1.8甚至直接取消了Perm区，建立一个元区域。[这篇博客](https://tech.meituan.com/in_depth_understanding_string_intern.html)介绍的非常详细，值得一读。不过实际业务开发中，的确很少用到intern方法，更多是SDK组件里面优化性能才会去用。

    - 字符串的hashCode是取决于内容，而不是地址；判断字符串内容是否相同要用`equals`，判断其他对象是否同用`==`；一定要区分，==，equals，hashCode。



## 6.动态代理是基于什么原理？



- 什么是动态代理：动态代理是一种运行时动态构建代理、动态处理代理方法调用的机制。利用动态代理，可以包装RPC调用，实现AOP。动态代理可以通过反射实现。

- 反射：反射是Java语言提供一种能力，赋予程序在运行时自省(introspect)的能力，通过反射我们可以直接操作类或者对象，比如获取某个对象的类定义，获取类定义的属性、方法，调用方法或者构造对象，甚至可以修改类定义。主要使用的是`java.lang.reflect`包里面的类来实现，比较常见的一种用法是用来绕过API设置的方法访问权限，`AccessibleObject.setAccessible(boolean flag)`，可以把一个private修饰的方法设为可以访问，从而调用它（Android里面各种hack用法）。此外，我们还可以在大量的ORM框架中看到这种用法，运行时自动生成getter/setter方法。不过Java 9之后出现一个Jigsaw模块化系统，出于强封装的考虑，对反射访问进行了限制，必须声明Open的反射调用者模块，才可以使用setAccessible。

- 动态代理可以解决什么问题？应用场景是什么？

    - 设计模式中有一个代理模式：代理模式、装饰器模式，类似的思想都是对调用目标进行包装，调用目标的代理来实现它的功能，实现调用者和被调用者之间的解耦。但是设计模式中的代理是一种静态代理，需要代理类和被代理类都是实现功能接口。这里我们讲的是动态代理，发生在运行时。

    - 应用场景：SpringAOP(Aspect of Programming)面向切面的编程是一种非常典型的场景，作为OOP的补充，因为OOP对于跨越不同对象或者类的分散、纠缠逻辑表现力不够。AOP可以处理不同模块之间公共阶段需要做的一些事情，比如日志、鉴权、全局异常处理、性能监控、事务处理等等，将这些繁琐、重复的工作统一抽取出来处理。

- 如何实现动态代理？不同的实现方式有什么区别（性能、可靠性、开发工作量）？

    - JDK Proxy：实现InvocationHandler，在invoke方法中反射调用被代理对象的方法。然后调用JDK的`Proxy.newProxyInstance`方法创建一个代理对象，这种方法需要被代理类和代理类都实现接口，但是因为是JDK本身的支持，比较可靠，新版也开始用到ASM。

    - cglib：采用创建被代理目标类的子类，实现动态代理，貌似Guice也是这样，不需要实现接口。这样就摆脱了实现接口的限制，开发量小，而且据说性能比JDK自带的高很多。（具体性能好在采用了ASM、Javaassist操作字节码，后面还会有专门学习这一块知识的。）

    - more：SpringAOP提供两种方式，可以显式地指定采用哪种方式。后面还会单独对SpringAOP和Guice做一个更加具体的学习。



## 7.int和Integer的区别？



- int：是Java中8个原始数据类型（Primitive Types: byte, short, int, long, float, double, boolean, char）的一种。

- Integer：是int原始类型对应的包装类，内部用一个int字段存储数据，提供了基本的操作方法，例如数学计算、int和String之间的转换等等。Java6还引入了Auto Boxing和UnBoxing功能，根据上下文自动转化，简化了编程。和String类型类似，Integer内部保存着一个final int value对应到实际的int值。

    - IntegerCache缓存：如果每次创建一个Integer对象都用new方法的话，可能会产生太多的对象。JDK1.5提供了Integer.valueOf方法来创建对象，它内部会缓存-128~127之间的值，如果缓存中存在，那么直接返回对象引用，不会创建新的对象，如果缓存中没有那么会**创建一个新的对象**。在其他的包装类例如Boolean（缓存了Boolean.TRUE和Boolean.FALSE）、Short（缓存了-128~127）、Byte（所有值都缓存）、Character（缓存`\u0000~\u007F`）等，也都有这个机制。不过其实JVM参数里面可以修改Integer缓存的大小：-XX:AutoBoxCacheMax=size，其他的不行。

    - Auto Boxing/UnBoxing：**发生在编译阶段**，所以是一种语法糖，生成的字节码里面会自动变成`Interger.valueOf`或者`Interger.intValue`。

    - 编程时的注意：避免不必要的拆箱装箱，例如循环中，因为会占用更多的内存空间。所以在性能极度敏感的场合，建议用原始类型替换包装类。

    - 原始类型的一些问题：

        - 线程安全：很显然原始类型的数据，要是用并发相关的控制，才可以保证线程安全。特别宽的类型，比如float、double，甚至不能保证更新操作的**原子性**，可能出现程序读取到只更新了一般数据位的值。JDK中有AtomicInteger、AtomicLong、AtomicIntegerArray、AtomicLongArray等类型是线程安全的。

        - 无法与泛型结合使用：因为Java里面的泛型是一种伪泛型，编译期会自动将类型转换为对应的特定类型，所以泛型里面的类型必须能转换成Object。





## 8.Vector、ArrayList、LinkedList有何区别？



- Vector：线程安全的动态数组。自动扩容到2倍。

- ArrayList：线程不安全的动态数组。自动扩容到1.5倍。

- LinkedList：线程不安全的双向链表。

- 三者对比：三者都是集合框架中的List，属于有序集合，功能近似。但Vector、ArrayList内部是以数组的形式存储元素列表的，适合随机访问。LinkedList适合节点插入、删除，随机访问性要慢。实际开发时，根据数据操作的场景去选择这几种结构，需要考虑的就是用数组还是链表，插入是在中间，还是在尾部，由于数组需要扩容开销大，链表的读取效率差。

    - 关于插入速度的一些结论：若只对单条数据插入或删除（还有不用扩容+copy数组时的尾部），ArrayList的速度反而优于LinkedList。但若是批量随机的插入删除数据（不在尾部），LinkedList的速度大大优于ArrayList。因为ArrayList每插入一条数据，要移动插入点及之后的所有数据。 综合来看，如果是随机插入，ArrayList的效率可能会更高。**所以，如果待插入、删除的元素是在数据结构的前半段尤其是非常靠前的位置的时候，LinkedList的效率将大大快过ArrayList，因为ArrayList将批量copy大量的元素；越往后，对于LinkedList来说，因为它是双向链表，所以在第2个元素后面插入一个数据和在倒数第2个元素后面插入一个元素在效率上基本没有差别，但是ArrayList由于要批量copy的元素越来越少，操作速度必然追上乃至超过LinkedList。**另外尽量不要使用普通for循环遍历LinkedList，效率极其的慢（LinkedList.get(index)可想而知有多慢，O(index)的时间复杂度了啊）。

        - LinkedList做插入、删除的时候，慢在寻址，快在只需要改变前后Entry的引用地址 

        - ArrayList做插入、删除的时候，慢在数组元素的批量copy，快在寻址。

- 扩展：Java中的集合类（Collection）是一个非常重要高频的考核点，需要掌握整体结构，容器有哪些，数据结构和里面涉及的算法，使用时的性能、并发等。

    - List：有序集合，通用逻辑抽象到了AbstractList

    - Set：不重复元素的集合，通用逻辑抽象到了AbstractSet

    - Queue、Deque(double-ended双端队列)：FIFO/LIFO队列，通用逻辑抽象到了AbstractQueue

    - 扩展到三种Set的区别：

        - TreeSet：内部用TreeMap的key存储，是一个红黑树结构，支持自然顺序访问，但添加、删除、contain相对低效(时间复杂度log(n))，内置排序功能。

        - HashSet：内部用HashMap存储，利用哈希算法，可以提供常数时间的添加、删除、contain操作，但不保证有序。

        - LinkedHashSet：内部用记录插入顺序的双向链表，提供按照插入顺序遍历的能力，也提供常数时间的添加、删除、contain操作，性能略低于HashSet。



下面是一个示意图，没有包含线程安全有关的类还有Map。关于集合类的知识，还应该再找一些资料好好学习。



<img src="https://ws1.sinaimg.cn/large/006tNbRwly1fx7njw4s8tj31c00psdlc.jpg" width="640"/>



## 9.HashTable、HashMap、TreeMap有什么区别？



这三者都是常见的Map实现，是以键值对的形式存储和操作数据的容器类。但是它们并不是集合类java.util.Collection，而是java.util.Map。



<img src="https://ws4.sinaimg.cn/large/006tNbRwly1fxa5ujjs0cj315a0pewi6.jpg" width="640" />



- HashTable：早期实现的线程安全的哈希表，键和值都不可以为null。

- HashMap：非线程安全的哈希表，行为上基本与HashTable一致。主要区别在于HashMap不同步，支持null键和值。通常可以在常数时间性能下实现get、put操作，被应用地更加广泛。

- TreeMap：基于红黑树提供的一种有序的Map，因为是红黑树，get、put、remove都是O(log(n))的复杂度，排序比较接口可以自定义Comparator。

    - LinkedHashMap（也是一种保证“某种”顺序的Map，但是实现区别很大）：从名字就看的出来，LinkedHashMap是通过链表来实现，保证的是和插入顺序相同的访问顺序（TreeSet是可以自己指定Comparator比较key来保证顺序）。

- HashMap的考点：最常用的HashMap也是最常见的考察点，需要理解它的设计实现细节。

    - equals和hashCode有关知识：equals相同，hashCode一定相同；**hashCode相同equals却不一定相同（hash冲突正是在下）**。重写hashCode时一定要同时重写equals，要搞清楚什么时候需要去重写equals。equals函数里面一定要是Object类型作为参数，且本身不要过于智能，只要判断一些值相等即可。

        - `==`和`equals`：在Object类中`equals`的实现就是`return (this == obj);`，而`==`是判断两个对象是否是同一个对象，比较的是地址。而重写`equals`通常是追求对象在逻辑上相等，内容相同，比如String类。

        - 重写equals的原则：自反性(x.equals(x)=true)、对称性(x.equals(y)=y.equals(x))、传递性(x.equals(y)=true&y.equals(x)=true时，x.equals(z)=true)、一致性(多次调用结果相同)、非空性(任何非空引用x，x.equals(null)=false)。重写该方法时，一般只兼容同类型的变量，否则很容易违反对称性。有时的面试官甚至会考察String里面的equals实现。

        - 重写hashCode的原则：为不相等的对象产生不相等的散列码，同样的，相等的对象必须拥有相等的散列码。很多时候，都是用idea自动生成override的代码，可以仔细去看看它的实现。

    - HashMap内部实现细节：

        - 结构：数组（Node<K, V>[] table）加拉链（hash冲突的时候，拉出一个链表）的复合结构，拉链长度过长时产生树化。

        - 初始化：lazy_load，构造函数中不会初始化数组，而是在put(putVal方法处理的)第一个元素的时候才初始化数组。

        - 分桶：不是简单地使用key的hashCode，而是内部实现了一个忽略了容量以上的高位的计算`(h = key.hashCode()) ^ (h >>>16)`。

        - 哈希冲突：put操作时，当key的hashCode相同时，会判断equals是否为true，如果为true那么putVal就会替换之前的value，如果为false，那么会采用拉链法存储这个value。

    - 理解HashMap的容量(capacity)、负载因子(load factor)、门限值以及它们对性能的影响：

        - 门限值：元素个数的门限，Threshold = 容量 x 负载因子，以倍数调整（newThr=oldThr<<1），调整时需要ArrayCopy。

        - 性能：容量*负载因子，决定了可用桶的数量，空桶太多浪费空间，使用太满则影响操作性能（假如只有一个桶，那相当于完全退化成链表，也就不存在常数时间的存取性能了）。

        - capacity和loadFactor怎么设置：实际应用中，我们应该尽可能满足`capacity * load_factor > 元素数量`，去预先设置一个初始大小。HashMap的构造函数(HashMap(int initialCapacity, float loadFactor))里面我们设置的是capacity，而`capacity=entryNumber(预估的元素数量)/loadFactor`，且它是2的幂，capacity就是数组的大小。而loadFactor默认是0.75，建议不要设置比这个值更大，因为会显著增加冲突，降低存取性能。但是如果选择太小的负载因子，按照这个公式又会导致预设的capacity也应该调整变小，否则会导致更频繁的扩容。

        - 为啥要有复杂因子：因为它影响到了冲突产生的概率，参考[这里](https://zh.wikipedia.org/wiki/%E5%93%88%E5%B8%8C%E8%A1%A8#%E8%BD%BD%E8%8D%B7%E5%9B%A0%E5%AD%90)看下。

    - 树化（treeify）：如果小于MIN_TREEIFY_CAPACITY=64，只是resize，如果大于它则进行treeify。做树化的原因很容易想到，还是性能，因为当链表太长的时候，它的线性结构是无法保证存取速度的。

- 扩展：解决哈希冲突的办法有哪些（大学都学过的）

    - 开放定址法：hash(key)=h，冲突时再以p为基础，再去算出一个h1（计算方法可以是某个线性函数值为散列地址a*h+b或者在hash(h)等等），直到算出一个不冲突的pi作为地址。这种方法可能产生聚集。ThreadLocalMap据说用的是这个方法。

    - 再哈希法：同时构造多个哈希函数，hash1(key)=h1, hash2(key)=h2...hi，当h1冲突时，计算出h2，直到不产生冲突hi。这种方法不易产生聚集。

    - 链地址法：将所有hash地址相同的元素用一个单链表存起来，单链表头指针存在hash地址里。HashMap用的是这个方法。

    - 公共溢出区：将哈希表分为基本表和溢出表，产生了hash冲突的元素都丢到溢出表中。



## 10.如何保证集合是线程安全的？ConcurrentHashMap如何高效实现进程安全？



- 方法一：使用线程安全的集合容器类或者包装集合类：前面提到的大部分集合类，如ArrayList、LinkedList都是非线程安全的，而线程安全的Vector（或者后面的HashTable）容器类仅仅是对需要保护的方法（put、get、size）在方法上加了synchronized修饰，锁的粒度是实例对象，粒度非太粗。另外，java.util.Collections类提供了很多静态方法包装出线程安全的集合，比如`SynchronizedMap(Map<K,V> m)`等，同样也是对象粒度的锁（内部实现用的锁是this，仍然是对象粒度），在高并发情况时性能低下。

- 方法二：使用并发包`java.util.concurrent`包提供的线程安全容器：如ConcurrentHashMap、CopyOnWriteArrayList、ArrayBlockingQueue、SynchronousQueue等等。这些并发包中的线程安全容器，实现方式更加精细，比如基于分离锁实现的ConcurrentHashMap，远比上面的要优秀很多。

- ConcurrentHashMap的内部实现：它的实现一直在演化，从“早期实现->Java7->Java8”中都有不少变化，理解ConcurrentHashMap的实现对并发编程的知识要求比较高，感觉完全可以单独作为一大篇文章去讲（可以看看这篇对源码更详细的[分析](https://my.oschina.net/hosee/blog/675884)），后面也会对并发的更多知识有深入讲解。

    - 并发编程的3个概念：

        - 原子性：一个操作或者多个操作 要么全部执行并且执行的过程不会被任何因素打断，要么就都不执行。Java中只有简单的读取、赋值（将数字赋值给某个变量）是原子操作。如果要实现更大范围的原子操作，需要借助synchronized和Lock来实现。

        - 可见性：可见性是指当多个线程访问同一个变量时，一个线程修改了这个变量的值，其他线程能够立即看得到修改的值。Java中提供了volatile来保证可见性，它会保证修改的值会立即被更新到主存，当有其他线程需要读取时，它会去内存中读取新值（这里涉及Java内存模型：主存是指物理内存，而线程的内存通常都是CPU高速缓存L1/L2/L3，当volatile修饰的变量被一个线程修改时，会导致其他线程的工作内存中缓存的这个变量值无效，反映到硬件层就是CPU的L1或者L2缓存中行无效，这些线程要使用该变量的值时，需要从物理内存中再去读取）。另外synchronized和Lock来实现，因为它们可以保证同一时刻只有一个线程可以获得锁来执行同步代码，并且会在释放锁之前将变量的修改刷新到主存中。

        - 有序性：程序执行的顺序按照代码的先后顺序执行。在Java内存模型中，允许编译器和处理器对指令进行重排序，但是重排序过程不会影响到单线程程序的执行，却会影响到多线程并发执行的正确性。在Java里面，可以通过volatile关键字来保证一定的“有序性”（下面会讲）。另外可以通过synchronized和Lock来保证有序性，很显然，synchronized和Lock保证每个时刻是有一个线程执行同步代码，相当于是让线程顺序执行同步代码，自然就保证了有序性。

    - volatile：

        - 可见性：volatile是Java中用于保证**可见性**的关键字。当一个共享变量被volatile修饰时，它会保证修改的值会立刻被更新到主存，当有其他线程需要读取时，它会去主存（也就是内存）中读取新值（JDK1.5之前该关键字貌似不是很好用，运行结果不一定准确）。

        - 原子性：volatile是没法保证原子性的，只能保证可见性，但synchronized或者Lock是可以保证原子性的（后面有专门的对synchronized和Lock学习的整理）。

        - 有序性：volatile只能一定程度地保证有序，假设一段代码有5行，volatile修饰的变量在第3行，那么在volatile前面两行的代码一定会在第3行代码执行之前执行，后面两行代码一定会在第3行代码执行之后才执行。但第1、2行或者4、5行会不会发生指定重排就没法由第3行来保证了。

        - volatile用于什么场景呢？首先有了上面的知识，我们知道volatile可以保证可见性，一定有序性和原子性。volatile关键字在某些情况下性能是优于synchronized的，但它没法替代synchronized，因为无法保证原子性。**如果想让volatile满足原子性**，需要满足2个条件：（1）运算结果并不依赖于变量的当前值，或者能够确保只有一个线程修改变量的值；（2）变量不需要与其他的状态变量共同参与不变约束。实际应用中，用得最多的就是状态标记（一个线程初始化某些资源的时候，flag=true/inited=true之类的），以及单例的double-check实现中volatile修饰的instance对象（双重检查锁写法为什么用volatile修饰instance呢，因为外层的if(instance==null)可能会看到没有完整实例化的instance对象引用，构造函数还没有真正创建完整的单例对象。据说JDK1.5之后的volatile是可以保证instance是完整的）。

    - CAS（Compare and Swap）：直译过来就是比较并替换，是一种CPU直接支持的指令，是**无锁操作的核心**。CAS机制中有3个基本操作数，内存地址V，旧的预期值A，要修改的新值B。更新一个变量的时候，只有当内存地址V中的实际值等于变量的预期值A时，才会把内存地址V中的值改为B（原子操作）。如果内存地址V中的值不等于A时，会重新去读取地址V中的值，然后再次进行CAS操作。这个重新尝试的过程叫做**自旋**。从思想上说，synchronized属于**悲观锁**，悲观的认为程序中并发情况严重，需要严防死守。CAS属于**乐观锁**，认为并发竞争情况不那么严重，让线程不断尝试更新（不用阻塞线程，开销小）。不过CAS也有缺点，就是当反复尝试更新却一直更新不成功时，CPU消耗过大；也无法保证代码块的原子性，它只能保证一个变量的原子性，但多个变量的原子性不得不用synchronized；还有无法处理的ABA问题。在JDK中，`java.util.concurrent.atomic`包下面的类以及Lock类的底层实现其实用到了CAS（例如AtomicInteger#compareAndSet(int, int)方法用的是下面提到的`Unsafe.compareAndSwapXXX`。），JDK1.6之后，也对synchronized进行了优化，在转变为重量级锁之前，也会采用CAS机制。

    - Unsafe：`sun.misc.Unsafe`提供了CAS的实现，而且ConcurrentHashMap主要用到了它的volatile读写功能，**可以实现对非volatile修饰的变量执行volatile读写，保证可见性（**Unsafe提供了`public native Object getObjectVolatile(Object obj, long offset);`这种通过偏移量来读取一个对象中某一个变量的值的能力）。一般不推荐我们在编程时使用Unsafe，只有在框架级别（Spring、Netty）或者JDK内部使用比较多，且JDK9还做了一些限制。需要注意这个类是一个单例，获取实例的静态方法`public static Unsafe getUnsafe()`无法由我们的代码调用，因为它内部会检查调用类的ClassLoader，如果不是BootstrapClassLoader，就无法返回Unsafe实例，因此JDK中的rt.jar中的类调用它是可以的，但是我们不行，因为我们的类是AppClassLoader加载的。不过我们可以反射调用它。（这里不过多深入研究了，可以看看[这篇博客](https://www.infoq.cn/article/A-Post-Apocalyptic-sun.misc.Unsafe-World)对它的介绍）

    - 早期实现&JDK7比较新的实现：[JDK7源码](http://hg.openjdk.java.net/jdk7/jdk7/jdk/file/9b8c96f96a0f/src/share/classes/java/util/concurrent/ConcurrentHashMap.java)。

        - 分离锁：内部进行分段（每一段叫segment），每一段是一个HashEntry数组，然后冲突时也是拉链表解决。segment数量由concurrentcyLevel决定，默认16，可以设置，必须是2的幂，方便进行寻址的时候移位运算。

        - 每一个HashEntry内部，使用volatile保证value字段的可见性。也利用了不可变对象的机制，以利用Unsafe提供的底层读写能力，优化性能。

        - get：只需要保证可见性，所以用的是`UNSAFE.getObjectVolatile(segments, u))`来获得segment，然后找到里面的Entry返回。

        - put：（1）`UNSAFE.getObject(segments, (j << SSHIFT) + SBASE))`Unsafe方式先获取segment，由于是nonvolatile方式，所以还要再次check，再次check用的是`ensuereSegment`方法，而这个方法里面用到了`UNSAFE.getObjectVolatile`和`UNSAFE.compareAndSwapObject`；（2）**得到segment**之后，再进行**线程安全**的`segment.put(K key, int hash, V value, boolean onlyIfAbsent)`操作，Segment本身就是extends ReentrantLock，因此进行put时，segment是锁定的。在这里进行了重复扫描和检测冲突，决定是更新还是放置操作，如果容量不够了会单独对Segment扩容，而不是整个map。

        - size：由于分段，需要计算每个segment的size，但是同步锁定所有的segment，效率太低，因此它是通过重试机制（RETRIES_BEFORE_LOCK，指定重试2次），来试图获得可靠值，如果没有监控到变化（对比Segment.modCount），就返回，否则获取锁进行操作。

    - JDK8实现：总体结构也是分段（大的桶bucket数组），内部也是一个个的链表，但同步粒度更细致，也不再使用Segment，简化了初始化开销，修改为lazy_load。数据存储利用volatile保证可见性（key是final的，val是volatile的），使用CAS实现无所并发操作，使用Unsafe、LongAdder进行优化。

        - get：其实也是用的`Unsafe.getObjectVolatile(Object obj, long offset)`方法实现。

        - put：代码不贴了，可以去看JDK。

            - init：初始化数组`initTable()`在这里实现。利用了一个sizeCtl座位互斥手段，实现了CAS。当sizeCtl为负数时，说明有其他线程正在初始化，那么先spin，等到条件恢复了，再利用CAS设置sizeCtl实现排它，设置成功后进行初始化。

            - 放置新值：不需要加锁，直接用CAS实现compareAndSwapObject。

            - 替换旧值：用头元素节点`Node<K,V> f`对象的锁（内部代码是synchronized(f) {同步修改value}）来保证修改value时的同步，粒度更细，而不是JDK1.7当中Segment整段都锁，效率大大提高。**这个思路上的突破非常关键。**

            - size：基于一个叫做CounterCell的东西，它的操作基于了`java.util.concurrent.atomic.LongAdder`实现，是一种JVM利用空间换取更高效率的方法，利用了[Striped64](http://hg.openjdk.java.net/jdk/jdk/file/12fc7bf488ec/src/java.base/share/classes/java/util/concurrent/atomic/Striped64.java)内部的复杂逻辑。

    - 应用场景：内存级别的缓存，数据量不大，且不要求多机器之间必须实时同步的时候。但是要明白一点，ConcurrentHashMap不保证两原子操作之间的原子性，比如get之后，又put，是没法保证这两个操作线程安全的！





## 11.Java提供了哪些IO方式？NIO如何实现多路复用？



- IO方式

    - BIO：**同步阻塞**方式的IO，主要是指java.io包下面那些流式的IO功能，也包括java.net下面部分网络API，如Socket、ServerSocket、HttpURLConnection等。(https://juejin.im/post/5b97e5f75188255c8d0fb0c0)。

    - NIO：Java 1.4中引入了NIO框架（java.nio包，n可以理解为new/non-blocking），提供了Channel、Selector、Buffer等抽象，可以构建多路复用、**同步非阻塞**的IO程序。

    - NIO2：也叫AIO（Asynchronous IO），Java 7中，进一步引入了NIO2，是一种**异步非阻塞**的IO方式。异步IO操作基于事件和回调机制，可以简单理解为应用操作直接返回，而不会阻塞在那里，当后台处理完成，操作系统会通知相应的线程进行后续工作。



下面分别看看各自的特点和使用场景。



- 标准IO（BIO）：简单来说，BIO就是指请求的连接、数据的读写都是在一个独立用户线程中完成的。由于线程的创建、切换都比较昂贵，且如果网络连接比较慢，导致每个线程都阻塞的话，服务器可能就会变得很慢甚至不可用。对于BIO这种面向流的标准IO，主要需要了解Reader/Writer、InputStream/OutputStream以及它们的实现类，要熟悉[各种Stream的基本用法]。流的基本分类如下：

    - 按流向：输入流和输出流

    - 按操作单元：字节流和字符流

    - 按流的角色：节点流和处理流



这里有两张Java标准IO类的结构图，来自这篇[文章](https://juejin.im/post/5af79bcc51882542ad771546)，这篇文章也是一篇对Java标准IO知识点的大汇总，值得好好学习。



按照操作方式分类：

<img src="https://ws4.sinaimg.cn/large/006tNbRwly1fxd8ugmqh7j30k00u0go6.jpg" width="640" />



按照操作对象分类：

<img src="https://ws2.sinaimg.cn/large/006tNbRwly1fxd8ulc1ruj30k00evabp.jpg" width="640" />



- NIO：由于BIO存在上面的问题，JDK1.4带来了同步非阻塞的NIO，它可以实现一条线程管理多个通道，实现对请求连接的管理（非阻塞的），当连接准备好数据后，再将数据的读写交给具体的应用线程（IO仍然是同步的），剥离了请求的等待时间。由于一条线程可以通过多个通道管理连接，也避免了线程创建和切换的大量开销。（这里有非常棒的一篇[NIO教程](http://tutorials.jenkov.com/java-nio/overview.html)），下面看下NIO的核心组成部分。

    - Channel(通道)&Buffer(缓冲区)：这两者都是NIO中提出来的，用于IO处理的重要接口。标准IO是面向字节/字符流的，而NIO是面向通道和缓冲的。数据是从Channel（A Channel is a bit like a stream. `inChannel.read(buffer)`）读到Buffer中的，或者从Buffer写入到Channel（`outChannel.write(buffer)`）。学习NIO需要学习使用Channel和Buffer提供的方法读写数据。

        - Channel：与标准IO的流很像，但它是双向的。流是单向的（Input/Output）；支持异步读写，可以读取数据到Buffer，或者由Buffer写入数据。在NIO中实现了Channel接口的类有FileChannel、DatagramChannel(UDP)、SocketChannel(TCP)、ServerSocketChannel(Server listen TCP connections)，可以像标准IO那样可以支持文件、网络的IO需求了。

        - Buffer：Buffer是标准IO中完全没有的东西，本质上就是一块内存，提供了一系列的方法来操作内存数据。它是一个抽象类，继承它的子类有ByteBuffer、CharBuffer、LongBuffer等等，覆盖了IO操作中所有的基础数据类型。Buffer可以通过flip方法切换读写模式。Buffer中有3个重要属性：position、limit、capacity，它们的值决定了当前的Buffer是读还是写模式，其实从名字上就可以看出它们的含义，写模式时，limit等于capacity，读模式limit代表Buffer中有多少数据，position指的是操作的内存位置。[Buffer的方法](http://tutorials.jenkov.com/java-nio/buffers.html)需要好好掌握，我们常用Buffer的`rewind`、`clear`、`flip`来达到需要的功能。

    - Selector(多路复用器)：Selector允许一条线程处理多个Channel通道，如果应用程序中有很多的通道（每个通道对应一个连接，每个连接的数据量都较小），会非常方便，例如聊天服务器这样的场景。这是NIO与标准IO相比最牛逼的地方，因为单线程意味着更小的开销，在Java中线程是很重的，需要占用更多的资源，而且线程切换上下文是一件很昂贵的操作（哈哈哈“协程”你们好啊），所以the less threads the better（不过现代操作系统已经对多线程处理得越来越好，开销也会越来越小，事实上对于多核CPU如果不使用多线程反而是一种浪费）。

        - 创建selector：静态方法`Selector selector = Selector.open();`

        - 把channel注册到selector：Channel都需要注册到Selector之后才可以被Selector选中(`SelectionKey key = channel.register(selector, SelectionKey.OP_READ);`)，select()方法会一直block直到某一个注册Channel准备好了，有事件就绪。一旦这个方法返回，线程就可以处理这些事件了。不过需要注意，只有可切换为non-blocking模式的SelectableChannel才可以注册到Selector，所以SocketChannel可以，但FileChannel不行。`register`方法会返回一个channel和selector的关系`SelectionKey`。`register`方法的第二个参数表明selector监听channel时对什么事件（一共有4种：`Connect`、`Accept`、`Read`、`Write`）感兴趣，当一个channel准备好了处理感兴趣的事件时，就会返回xxxEvent ready。

        - selector选择channel：通过select方法(这个方法有3个重载)。其中不带参数的方法调用时返回的是本次调用和上次调用之间已经ready的channel个数。当select方法返回了大于1时，就可以调用`selector.selectedKeys();`来拿到一个SelectionKey集合，进而拿到每一个channel去操作。[这篇文章](https://mp.weixin.qq.com/s?__biz=MzU4NDQ4MzU5OA==&mid=2247483970&idx=1&sn=d5e2b133313b1d0f32872d54fbdf0aa7&chksm=fd985423caefdd354b587e57ce6cf5f5a7bec48b9ab7554f39a8d13af47660cae793956e0f46#rd)末尾有一份实际NIO服务器与客户端的例子，里面介绍了如何用ServerSocketChannel和SocketChannel实现NIO通信。



- NIO2（AIO）：Java7中新增，虽然NIO在处理网络连接时是non-blocking的（**非阻塞**），但是IO（数据的读写）行为仍然是同步的。具体来讲，在NIO中，业务线程是在IO操作准备好时，得到通知，接着再由这个线程进行IO操作，IO操作本身是同步的（**同步的**）。但AIO则更进一步，它不是在IO准备好时就通知线程，而是在IO操作已经异步完成后，再给线程发出通知。因此AIO的数据读写其实异步的，此时业务逻辑将变成一个回调函数，等待IO操作完成后，由系统触发。 

    - 新增的`AsynchronousServerSocketChannel`，`server = AsynchronousServerSocketChannel.open().bind( new InetSocketAddress (PORT));`，使用server提供的accept方法`public abstract <A> void accept(A attachment,CompletionHandler<AsynchronousSocketChannel,? super A> handler);`。

    - 新增的`AsynchronousFileChannel`，支持异步读写文件。一起来的还有类似`File`的`Path`API。`AsynchronousFileChannel`支持返回一个`Future<Integer> operation`或者设置`CompletionHandler` callback来获得异步读写的结果。



- BIO、NIO、AIO的区别：

    - BIO：面向流，每次从流中读取一个或多个字节直到读完，没有缓存。标准IO中各种流都是阻塞的，当一个线程调用流的read或者write时，整个线程就会阻塞，知道数据完全处理完，在此期间线程什么也不能做了。**BIO相当于一个线程管理一个通道，**适合于那种连接数量不多，但每个连接的数据量比较大的场景。

    - NIO：面向缓冲，可以前后移动数据，面向缓冲区。NIO使用一条线程从Channel通道中读取/发送数据，它仅能得到目前可用的数据，如果目前没有数据，就什么都不会获取。**NIO这种让一个线程（或几个）管理多个通道，**所以一个通道的操作并不会阻塞整个线程，不过解析数据的代价可能会比从一个阻塞的流中更复杂。因此NIO更适合处理请求数量很多，但每个请求的数据量很小的场景，NIO不适合读写过程长的场景。

    - AIO：除了网络连接的管理是非阻塞的，**连数据的读写也变成了异步**，数据读写完成后再通过回调等方式通知业务线程处理，适合那种读写过程长的任务。

    - 总结：BIO与NIO最大的区别就是如何读取请求的数据，BIO从流中读取数据时是阻塞的，直到全部准备好、并读取完成（即使没有数据或者只需要部分数据）才可以处理，且一个线程只能处理一个Stream，因此这样的服务器在处理百万级别的并发请求时内存和CPU压力会很大。通常我们会在服务器设置一个合理线程数量的线程池，并设计一个请求连接队列，线程池中的线程会从这个队列中依次处理每一个连接中的流。想象一下，如果有大量长时间不活跃的连接时，数据一直没法准备好，那么将占用所有的线程，所有的线程都被会阻塞，服务器将因此变得非常缓慢甚至不可用；NIO则可以使用一条线程除了不同请求连接中的流，但这里的流是non-blocking模式工作的channel，non-blocking模式意味着一个channel可以返回0或者1+的bytes，通过Selector来select一个准备好了、有数据的channel进行处理（处理时一般还是丢到一个独立的业务线程），因此这样就不会阻塞。由于NIO里，数据的读写还是在业务线程中，所以不适合处理那些读写时间长的场景；AIO对于IO的操作采用异步的方式，等到读写数据完成，再通知业务线程处理数据，因此可以胜任那些读写耗时的场景。（感觉[这篇文章](http://www.importnew.com/21341.html)对BIO、NIO、AIO讲的比较形象）最后再用一个类比来解释：BIO，快递员通知你有一份快递会在今天送到某某地方，你需要在某某地方一致等待快递员的到来。NIO，快递员通知你有一份快递会送到你公司的前台，你需要每隔一段时间去前台询问是否有你的快递。AIO，快递员通知你有一份快递会送到你公司的前台，并且前台收到后会给你打电话通知你过来取。





## 12.Java中有哪些文件拷贝方式？哪一种最高效？



- java.io：利用FileInputStream和FileOutputStream，循环读取，写入。

- java.nio：利用Channel的transferTo/tranferFrom方法拷贝（Buffer）。

- 两种方式的对比：了解操作系统层面的概念

    - 用户态空间：User Space，普通应用和服务运行在用户态空间。当使用IO流读写时，进行了多次上下文切换，先在内核态将数据从磁盘读取到内核缓存，再切换到用户态，从内核缓存读取到用户缓存。写入时顺序相反，如下面左图所示。这种方式带来了额外开销，可能降低IO效率。

    - 内核态空间：Kernel Space，操作系统内核、硬件驱动运行在内核态空间，具有相对高的特权。NIO的transferTo实现时，在Linux/Unix会用到**零拷贝(ZeroCopy)**技术，数据传入不需要用户态参与，省去了上下文切换的开销和内存拷贝（下面右图）。这种能力还可以用在网络IO中。



<img src="https://ws1.sinaimg.cn/large/006tNbRwly1fxfuja3dwmj312i0u0q7i.jpg" width="320"/> <img src="https://ws1.sinaimg.cn/large/006tNbRwly1fxfujhlz6dj310f0u0q78.jpg" width="320" />



- java.nio.file.Files.copy：JDK中自带的文件拷贝方法，内部实现是怎样的。然而虽然是有3种方法，但是其实都是在用户态的读写。

    - `public static Path copy(Path source, Path target, CopyOption... options) throws IOException`，虽然是native实现，但也是用户态拷贝。

    - `public static long copy(InputStream in, Path target, CopyOption... options) throws IOException`

    - `public static long copy(Path source, OutputStream out) throws IOException`



- 掌握NIO Buffer用法：前一节的NIO中其实有介绍，详细的用法示例可以看[这里](http://tutorials.jenkov.com/java-nio/buffers.html)。此外对于Buffer里面的[ByteBuffer](https://zhuanlan.zhihu.com/p/29675083)，需要了解一下它分为堆内（HeapByteBuffer，allocate创建）和堆外（DirectByteBuffer，allocateDirect创建）两种。

    - DirectByteBuffer：Buffer类中定义了`isDirect()`方法判断是否为堆外内存。实际使用中，Java会尽量对DirectBuffer仅做本地IO，对于大数据量的IO密集操作，可能会带来非常大的性能优势。因为Direct Buffer生命周期内内存地址不变，内核可以安全地对其访问，IO操作高效；减小了堆内内存对象存储的额外维护，访问效率可能提高。但是DirectBuffer的创建销毁比一般的Buffer开销更大，建议长时间使用的大数据量场景下才使用，而且一般要fullGC时才会销毁它，使用不当容易导致OOM。

    - MappedByteBuffer(abstract class，具体实现是DirectByteBuffer)：将文件按照指定大小直接映射为内存区域（虚拟内存并非是物理内存，如果文件巨大可以分段映射），当程序访问这个内存区域时将直接操作这块文件数据，省去了从内核空间向用户控件传输的损耗。除了`allocateDirect`方法，它还可以用`FileChannel.map`（`FileChannel.map(FileChannel.MapMode.READ_ONLY, 0, len);`）方法来创建，其实返回的实例是`DirectByteBuffer`(class DirectByteBuffer extends MappedByteBuffer implements DirectBuffer)。需要知道MappedByteBuffer不受JVM的`-Xmx`参数限制，而是受`-XX:MaxDirectMemorySize=512M`限制，这里要注意`-Xmx`是其默认大小，“不受限制”是指设置`-XX:MaxDirectMemorySize`可以让它更大。[这篇文章](https://blog.csdn.net/fcbayernmunchen/article/details/8635427)有一个用Channel+Buffer读取文件和MappedByteBuffer读取文件的对比测试。在Netty、Kafka、Grizzy里面都用到了这个技术。



Buffer、ByteBuffer的继承关系

<img src="https://ws4.sinaimg.cn/large/006tNbRwly1fxgmhiiy9sj30sq0m2gq7.jpg" width="400"/>



- 这里我提一个问题，ByteBuffer和byte[]有什么区别：ByteBufer的内部实现就是一个`final byte[] hb`，`allocate`方法就是new一个指定容量的byte数组赋值给hb（这里不分析创建堆外缓冲区的`allocateDirect`方法了）。然后提供了一系列方便的方法来操作这个byte数组，当然，ByteBuffer也可以与byte[]直接转换。

    - byte array -> ByteBuffer：`ByteBuffer buf = ByteBuffer.wrap(bytes);`

    - ByteBuffer -> byte array：`bytes = new byte[buf.remaining()]; byteBuffer.get(bytes, 0, bytes.length);`





## 13.接口和抽象类有什么区别？



接口和抽象都是Java面向对象设计的两个基础机制。

- 接口（interface）：是对**行为的抽象**（like-a），抽象方法的集合，利用接口可以达到API定义与实现分离。接口不能实例化，不能包含非常量（所有的field都是public static final），没有非静态方法的实现（要么是静态方法要么是抽象方法），例如java.util.List。

    - Java中一个类可以实现（implements）多个interface。

    - 接口主要是为了定义一种/同一类抽象行为，子类必须实现接口中定义的方法（否则无法编译）。

    - 接口不光是限于定义抽象方法，有一类接口没有任何方法，Marker Interface，它的目的是为了声明某些语义，例如Cloneable、Serializable。从表面看似乎与注解（Annotation）类似，但是它要更简单，因为注解是可以有参数和值的，在表达力上更强大。

    - functional interface：Java8增加了函数式编程支持，定义所谓的functional interface，就是**只有一个抽象方法的接口**，通常使用@FunctionalInterface Annotation来标记。Lambda表达式本身也可以看作是一类functional interface，这在某种程度上和面向对象是两码事。我们熟悉的Runnable、Callable、Comparator都是functional interface，它们可以用匿名类语法来实例化，并且可以作为参数传递给另一个方法，并在方法中调用。Java8中将很多可重用的方法设计成了functional interface，在`java.util.function`包中。以`java.util.Function`为例，它定义了一个`R apply(T t);`方法，我们可以`Function<String, Integer> stringToInt = x -> Integer.valueOf(x);`这样来创建一个Fuction对象，然后这个`stringToInt`可以作为一个对象传递，并且可以调用它的`apply(String)`方法来把一个字符串转成Integer。了解Scala和JS ES6的同学对这个应该再熟悉不过了。

    - default(virtual) method：有了上面的functional interface，Java具有了一定的函数式编程能力。但是接口和实现类之间的耦合度还是太高了，当一个接口添加方法时，所有的实现类都要实现这个新方法。default method的出现解决了这个问题，**它允许接口中有具体实现的方法，**这样的话在接口中添加一个方法不会破坏原有方法的实现。这样就使得Java 8以前就存在的接口也可以很好的兼容升级了（Java9甚至可以定义`private default method`），default method可以与旧版本编写的代码保持二进制兼容。**比如`java.util.Collection`，它是collection体系的root interface，在Java8中添加了一系列default method，增加了Lambda、Stream相关的功能，而以前实现过这个接口的类不需要更改。由于有了默认方法，接口的继承也会使得默认方法被继承，另外由于一个类可以实现多个接口，这就使得**默认方法出现了多继承**，有几种情况要注意（[这篇文章](http://ebnbin.com/2015/12/20/java-8-default-methods/)讲得不错），不过我感觉这里有点像C++了。

        - 接口继承时，默认方法的获取有下面3种情况

            - 不重写默认方法，直接从父接口中获得方法的默认实现。

            - 重写默认方法，这与类继承之间的重写规则类似。

            - 重写默认方法，将其重写声明为抽象方法。这样新接口的子类，必须实现这个默认方法。

        - 类实现多个接口时，默认方法的多继承出现的冲突有：

            - 冲突1：接口继承，由于Interface可以有default method，所以一个接口另一个接口时可以重写默认方法（@Override，相当于对后代屏蔽了父接口的方法）。当一个类实现了这两个有继承关系的Interface时，它可以获得default method（类自己不必实现了）。但是这个方法的实现，取决于它更靠近的那个接口（The fundamental rule is that the closest concrete implementation to the subclass wins the inherited behavior over others.），且这个类无法调用到最上层的接口的默认方法（因为被最靠近的直接父接口重写后屏蔽了）。可以看下面的例子，有注释。

            - 冲突2：接口之间没有继承，但有相同签名的默认方法，一个类实现了这两个接口时，必须由这个类实现具体的default method。设想一下，不可能让一个类同时从两个Interface种继承同一个方法对吧？因为这个子类从两个接口中都获得了默认方法，必须指定用哪一个才行（两个父接口的默认方法这个子类都可以访问），否则编译也不会通过的。可以看下面例子，有注释。



```

// 多继承冲突1

// Person interface with a concrete implementation of name
interface Person{
  default String getName(){
    return "Person";
  }
}
// Faculty interface extending Person but with its own name implementation
interface Faculty extends Person{

  @Override
  default public String getName(){
    return "Faculty";
  }
}



// The Student inherits Faculty's name rather than Person

// 可以这么理解，假设Java有多继承的话，Class A extends B, Object，那么B肯定离A更近，而不是Object类，因为B extends Object，B相当于A的直接父类。
class Student implements Faculty, Person{ .. }
// the getName() prints Faculty
private void test() {
  String name = new Student().getName();
  System.out.println("Name is "+name);
}
output: Name is Faculty



// 多继承冲突2

// 如果两个functional interface没有继承关系，那么一定要由子类实现default method

interface Person{ .. }
// Notice that the faculty is NOT implementing Person
interface Faculty { .. }
// As there's a conflict, out Student class must explicitly declare whose name it's going to inherit!
class Student implements Faculty, Person{
  @Override
  public String getName() {
    return Person.super.getName(); // 可以采用InterfaceName.super.methodName();调用需要的接口默认方法
  }
}

```



- 抽象类（abstract class）：用abstract修饰的class，目的是**代码复用**（is-a）。抽象类也不能实例化，除此之外与一般的类没有太多区别。可以有一个或多个抽象方法，也可以没有。抽象类大多用于抽取相关Java类的共用方法或共用成员变量，然后通过继承的方法达到代码复用，标准库中的Collection框架，很多通用部分就被抽取为抽象类，例如java.uitl.AbstractList。

    - Java中一个类只能extends一个抽象类，也就是说Java不支持多继承。

    - 抽象类主要是为了代码复用，但是有时会需要抽象出**与实例化无关**的逻辑，如果此时也写到抽象类中，会陷入单继承的窘境。因为不可能让无关的两个实例去继承多个抽象类，这时一般用静态方法组成的工具类来实现，例如`Utils`（java.util.Collections）。

    - 抽象类可以定义非抽象方法，这样子类可以享受到这个方法的能力，但是不需要实现这个方法。

- 面向对象OO的理解：

    - 封装：隐藏事务内部细节，提高安全性和便捷性。封装提供了合理的边界，避免外部调用者接触到内部细节。例如假设在多线程环境暴露内部状态，导致并发修改。

    - 继承：继承是代码复用的基础，但是继承其实是一种非常紧耦合的关系，父类代码修改、子类也会变化。在实践中，过度使用可能带来反效果。

    - 多态：

        - 重载：Overload（没有这个注解），相同方法名字，但参数不同。本质上这是不同的方法签名。**方法签名由方法名称和一个参数列表（方法的参数的顺序和类型）组成。注意，方法签名不包括方法的返回类型。不包括返回值和访问修饰符。**

        - 重写：Override（@Override），父子类中相同的方法名字和参数，实现不同。重写在本质上，方法签名是相同的，子类覆盖了父类的方法，修饰符的范围相同或者比父类的范围大。因此如果父类中的方法是private的话，子类中相同签名的方法是不能加override注解的，因为**private不会被继承，没有继承关系也就不存在重写，**子类中只是加了一个方法而已。如果父类中的方法是protected或者public的话，子类可以override它，但修饰符必须是protected或者public，如果子类用private修饰这个override方法，编译器会报错。此外，如果override修改了返回类型的话，也是无法编译通过的。

        - 向上转型：`Animal bird = new Bird();`，这里的`Bird extends Animal`，这就是一种向上转型，这种方式的好处是，可以用抽象父类来代替子类，作为方法的参数，例如用一个`void eat(Animal animal)`方法实现`eat(bird)`、`eat(tiger)`而不用写多个`eat`方法（体现了OO的编程思想）。但也带来了损失，假如`bird`类有一个`fly`方法，`tiger`有一个`run`方法，那么在`void eat(Animal animal)`方法中没有办法去调用。

        - 向下转型：仍然以`Animal a = new Bird();`为例，向下转型就是`Bird bird = (Bird) a;`，将第一句代码创建的a对象转型为子类对象。接下来就可以调用`Bird`类独有的`fly`方法了（`bird.fly();`）。但是如果这么转型`Tiger tiger = (Tiger) a;`，程序在编译时不会报错，但是运行时会抛出`java.lang.ClassCastException`；或者说如果创建a对象用的是`Animal a = new Animal();`方法，也是不可以将a对象向下转型为`Bird`类型的，在编程时转型最好用`A instanceof B`做下判断。为了解决这个问题，Java引入了泛型(Generic)。

- OO设计原则：S.O.L.I.D.原则，这里有[一篇实现支付功能的例子](http://www.cnblogs.com/wuyuegb2312/p/7011708.html)讲得不错。SOLID不仅适用于类的设计，也适用于软件组件和微服务的设计。

    - Single Responsibility：单一职责，类或者对象最好只有单一职责，如果某个类承担着多个义务，可以考虑拆分。最常见的一个例子就是我们不推荐在Bean类中增加属性getter、setter以外的方法，例如save。因为如果需要修改save的行为，而属性以及属性的操作方法也会需要重新编译以适应这种变化。更好的方式是增加一个类，实现这个save方法。引用Steve Fenton的话就是在设计我们的类时，我们应该把相关的特性放在一起，这样，每当它们需要改变的时候，它们都是因为同样的原因而改变。如果它们因不同的原因而改变，我们就应该尝试将它们分开。遵循这条原则，我们的程序也会变成高内聚。

    - Open-Close：开闭，设计要对扩展开放，对修改关闭。避免因为新增同类功能时，修改类的已有实现。实际编程的时候，尽量不要用很多if...else if...else，避免因为增加同类行为时，又来加一个分支，不断修改原有类的实现。而是应该把同类行为抽象成接口。

    - Liskov Substitution：里氏替换，进行继承关系抽象时，凡是可以用父类或者基类的地方，都可用子类替换。这个原则可以这么理解，当子类可以在任意地方替换基类且软件功能不受影响时，这种继承关系的设计才是合理的。

    - Interface Segregation：接口分离，一个接口不要定义太多方法，因为子类不一定都要实现这些抽象的行为（实际编程的时候可能就是一个空方法实现了）。可以拆分接口，每个接口功能单一，将来如果某个接口有变化，也不会影响其他实现了其他接口的子类。

    - Dependency Inversion：依赖倒置，指的是实体应该依赖抽象而不是实现，高层次模块不应该依赖低层次模块，而是应该依赖抽象。保证产品代码之间适当的耦合度。那么什么是高层次，什么是低层次？可以这么理解（李智慧老师的大数据课程中也提到这点），简单讲就是调用链中处于前面的是高层，后面的是低层。实际上，依赖倒置是实现开闭原则的方法。

- IOC和DI（Spring的核心概念）

    - IOC：控制反转，创建实例的控制权由一个实例的代码剥离到IOC容器中，如xml配置。原先是一个类去创建另一个类，IOC之后变成这个类被动等待另一个类的注入。

    - DI：依赖注入，一个类依赖另一个类的的功能，那么就通过注入，如构造器，setter方法将这个类的实例引入。



## 14.谈一谈你知道的设计模式？



- 什么是设计模式：人们为软件开发中相同表征的问题，抽象出的可重复利用的解决方案。在某种程度上，设计模式代表了一些特定情况的最佳实践。设计模式可以帮助我们很好地理解JDK以及一些框架中，一些看似非常复杂繁多的源码和类的设计。



- 按照应用目标，进行分类：

    - 创建型：对对象创建过程的各种问题和解决方案的总结，常见的有工厂（Factory、Abstract Factory）、单例（Singleton）、构建器（Builder）、原型（ProtoType）。

    - 结构型：对软件设计结构的总结，关注类、对象继承、组合。常见的有桥接模式（Bridge）、适配器（Adapter）、装饰者模式（Decorator）、代理（Proxy）、组合（Composite）、外观（Facade）、享元（Flyweight）。

    - 行为型：从类或者对象之间的交互、职责划分等角度总结。常见的有策略（Strategy）、解释器（Interpreter）、命令（Command）、观察者（Observer）、迭代器（Iterator）、模板方法（Template Method）、访问者（Visitor）等。



[这里](http://www.runoob.com/design-pattern/design-pattern-tutorial.html)是一个网上比较简明的设计模式教程，可以作为基本资料迅速查阅。



### 说说几种常见的设计模式理解



- Builder模式：是一种创建型模式，在很多SDK或者框架中都能看到，Builder模式通常会被实现为链式调用的方法。这种模式可以**优雅地解决构建复杂对象的麻烦，避免实现各种参数组合的构造函数**。所以如果一个类的构造函数特别多的时候，我们可以考虑实现Builder模式。不过这种方式一旦build出对象之后，不再容易实现对它的属性修改，实际应用时，如果需要修改，可能需要实现一个“回炉再造”的方法。或者，我们需要仔细思考对象构建出来以后，**是否允许**其修改，毕竟选择实现builder模式而不是去写一堆`“构造方法+setter方法”`本身可能就是希望对象的构造是一个连续行为并且构造出来以后不希望它变化。

```

// 以一段我曾经比较熟悉的OkHttp为例，构造请求的时候，可以这么写。

Request request = new Request.Builder()
        .url("http://publicobject.com/helloworld.txt")
        .build();

// Request类实现的简化版，实际实现不只有url和method两个属性。
public final class Request {
  final String url;
  final String method;

  Request(Builder builder) {
    //构造函数，省略属性赋值操作
  }
  public Builder newBuilder() {
    return new Builder(this);
  }

//省略部分代码
public static class Builder {
    HttpUrl url;
    String method;

    //省略部分代码
    Builder(Request request) {
      //构造函数，省略属性赋值操作
    }

    public Builder url(String url) {
      this.url = url;
      return this;
    }

    //省略部分代码
     public Request build() {
      if (url == null) throw new IllegalStateException("url == null");
         return new Request(this);
    }
}
```

    

- 装饰器：是一种结构型模式，JDK标准IO流就是它的实现。允许向现有对象添加新的功能（包装/装饰），同时不改变其结构。这种模式创建了一个装饰类，用来包装原有的类，并且在**保持类方法签名完整的前提下，提供新的功能。**比如说我们有一个下载器downloader，可以实现一个文件的多线程下载。然后我们希望它还可以自动重试，这时可以把downloader包装成了一个AutoRetryDownloader。其实**这种模式可以看作是继承的替代，**动态地扩展了一个实现类的功能，但是比继承要更灵活，不过如果真的进行了多层装饰的话，也会比较复杂。

    - Java IO：InputStream是一个抽象类，FileInputStream（文件输入流）、ByteArrayInputStream（字节输入流）各种子类从不同角度对InputStream进行扩展，实现了对不同类型的输入来源读取的功能。装饰器模式中，很典型的就是FileInputStream的构造函数的参数就是InputStream类型。然后BufferedInputStream又可以对FileInputStream进行扩展，实现带缓存的文件输入流（`BufferedInputStream bis = new BufferedInputStream(new FileInputStream("file://path"));`）。

    - 与代理模式的区别：装饰器模式为了增强功能，而代理模式是为了加以控制（控制被代理对象的访问，比如AOP）。从构造方法上来说，装饰器模式里面，被装饰对象一般在提前在外面创建，然后传给装饰类的构造方法。而代理模式的构造方法上一般不需要传入被代理对象，而是代理对象在创建时，自己去创建被代理对象。不过很多时候，其实代理模式中，被代理对象也是外面创建的，因此这种代码组织上区别并不是这种两种设计模式的区别，更多还是需要从目的上去理解区分。

    

- 单例：创建型的模式，提供一种创建唯一对象的方法，外部可以直接访问这个对象，不需要调用这个累的实例化方法（3要素，只有一个实例，自己通过私有构造方法创建实例，对外提供这一个实例）。需要重点关注如何写出线程安全的单例，这个模式虽然简单，但是有很多的点可以考察，比如实现单例的方式有几种，线程安全的方式有几种。

    - 懒汉式1：线程不安全。因为没有加锁，当多线程并发调用getInstance时，不安全。

	```
	public class Singleton {  
    	private static Singleton instance;  
    	private Singleton (){}  

    	public static Singleton getInstance() {  
    	  if (instance == null) {  
          instance = new Singleton();  
    	  }  
    	  return instance;  
    	}  
	}

	```

    - 懒汉式2：线程安全，getInstance加锁，线程安全，但是获取对象时效率低，99%的情况下是不需要同步的，因为对象一旦被创建后，就不再需要锁住getInstance方法了。

	```
	public class Singleton {  
    	private static Singleton instance;  
    	private Singleton (){}  
    	public static synchronized Singleton getInstance() {  
    	if (instance == null) {  
        	instance = new Singleton();  
    	  }  
    	  return instance;  
    	}  
	}
	```

    - 饿汉式：线程安全，无锁效率很高。但是会产生无用的对象，浪费内存。**它的同步是基于ClassLoader实现线程安全的，JVM会保证一个类的<clinit>()方法在多线程环境中被正确加锁、同步。当ClassLoader在加载类的Singleton类时候就会实例化instance对象了。**其实JDK中的[`java.lang.Runtime
`](http://hg.openjdk.java.net/jdk/jdk/file/18fba780c1d1/src/java.base/share/classes/java/lang/Runtime.java)就是这种方式。不过要注意，它的对象还被声明为了`final`，一定程度上保证了实例不被篡改，也保证执行顺序的语义。

	```
	public class Singleton {  
    	private static Singleton instance = new Singleton();  
    	private Singleton (){}  
    	public static Singleton getInstance() {  
    	  return instance;  
    	}  
	}

	// java.lang.Runtime
	private static final Runtime currentRuntime = new Runtime();
	private static Version version;
	// …
	public static Runtime getRuntime() {
    	return currentRuntime;
	}
	/** Don't let anyone else instantiate this class */
	private Runtime() {}
	```

    - 双重检查锁：双重检查加锁，线程安全，获取对象时性能高。前面提到过JDK1.5之后的`volatile`关键字在这里起到的**可见性 + 一定程度的有序**作用，如果没有它的话，那么第一个null检查，可能会看到初始化了一半的instance，然后返回从而造成问题。为什么会出现没有完全初始化的instance，是因为JVM会进行指令重排，假设构造函数里面的工作很多，虽然synchronized保证了原子性，但重排可能会导致实例还没有构造出来，instance就被赋值了（instance就是个引用，赋值就是给它分配个内存空间）。同样的，在我们自己写代码的时候，如果说构造函数是一个异步操作，那就得小心了，要保证异步操作初始化完成了，才可以返回instance对象。

	> 杨老师也提到：在现代 Java 中，内存排序模型（JMM）已经非常完善，通过 volatile 的 write 或者 read，能保证所谓的 happen-before，也就是避免常被提到的指令重排。换句话说，构造对象的 store 指令能够被保证一定在 volatile read 之前。

	```
	public class Singleton {  
    	private volatile static Singleton singleton;  
    	private Singleton (){}  
    	public static Singleton getSingleton() {  
    	if (singleton == null) {  
        	synchronized (Singleton.class) {  
        	if (singleton == null) {  
            	singleton = new Singleton();  
          }  
        }  
      	}  
      	return singleton;  
    	}  
	}
	```

    - 静态内部类：这个方法可以达到和双重检查锁一样的效果，但是实现方式更简单，同样利用了ClassLoader。不过这里的lazy-load依靠的是内部类，内部类不会在Singleton被加载时被加载，而是会等到调用getInstance时，SingletonHolder类才被加载。

	```
	public class Singleton {  
    	private static class SingletonHolder {  
    	  private static final Singleton INSTANCE = new Singleton();  
    	}  
    	private Singleton (){}  
    	public static final Singleton getInstance() {  
    	  return SingletonHolder.INSTANCE;  
    	}  
	}
	```

    - 枚举（单元素枚举）：这种方式很少见到用，但是据说这是最佳实现方法。更简洁（直接调用Singleton.INSTANCE即可），自动支持序列化，线程安全。EffectiveJava作者提倡，它不仅能避免多线程同步问题，而且还自动支持序列化机制，防止反序列化重新创建新的对象，绝对防止多次实例化。需要注意的是enum是JDK1.5之后才出现。JDK5中提供了大量语法糖，枚举就是其中一种。语法糖（Syntactic Sugar），指在计算机语言中添加的某种语法，这种语法对语言的功能并没有影响，但是但是更方便程序员使用。只是在编译器上做了手脚，却没有提供对应的指令集来处理它。`enum`就是一种普通的类，继承自`java.lang.Enum`，就是说我们写的一个`public enum XXX`编译完了之后其实是一个类`public final class XXX extends Enum<XXX>`，其中的每一个属性都是`static`的，所以INSTANCE也是static的，这就和饿汉式的单例很像了有木有（线程安全），所以这种方式不是lazy-load的。

	```
	public enum Singleton {  
    	INSTANCE;  
    	public void whateverMethod() {  
    	}  
	}

	// 字节码反编译之后

	public final class Singleton extends Enum<Singleton> {
      public static final Singleton INSTANCE;
      public static Singleton[] values();
      public static Singleton valueOf(String s);
      static {};
	}
	```

- Facade模式：Facade外观模式，属于结构型模式。它隐藏了系统的复杂性，向外部客户提供一个访问系统的简化接口。这种模式有利于将真实的客户端与系统解耦，比如我们使用HttpClient时，可以有OkHttpClient、HttpClient、HttpUrlConnection等等，按照外观模式，我们把它们封装成简单的Get、Post接口形式，给业务层使用，那么将来如果想要切换HttpClient的实现也是很容易的，业务层代码完全不需要修改。



- Spring等框架中使用了哪些模式，这里考察到了Spring框架的知识，在实际工作中使用到的话，应该都不会陌生。

    - BeanFactory、ApplicationContext，工厂模式。

    - Bean创建时的Scope定义（单例、原型），实现原理。

    - JdbcTemplate，模板模式。

    - AOP，代理模式、装饰器模式、适配器模式。


到这里，把杨晓峰老师的《Java核心知识》的基础篇提纲挈领地过了一遍，深刻感受到了温故而知新的道理，当年在大学里面向对象虽然学的是C++，后来从实验室到工作一直都是Java，基础的知识的确是越品越有味道。