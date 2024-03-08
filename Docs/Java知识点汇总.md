- [JVM](#jvm)
  - [JVM 工作流程](#jvm-工作流程)
  - [运行时数据区（Runtime Data Area）](#运行时数据区runtime-data-area)
    - [程序计数器](#程序计数器)
    - [Java 虚拟机栈](#java-虚拟机栈)
    - [本地方法栈](#本地方法栈)
    - [Java 堆](#java-堆)
    - [方法区](#方法区)
  - [方法指令](#方法指令)
  - [类加载器](#类加载器)
  - [垃圾回收 gc](#垃圾回收-gc)
    - [对象存活判断](#对象存活判断)
    - [垃圾收集算法](#垃圾收集算法)
    - [垃圾收集器](#垃圾收集器)
    - [内存模型与回收策略](#内存模型与回收策略)
- [Object](#object)
  - [equals 方法](#equals-方法)
  - [hashCode 方法](#hashcode-方法)
- [static](#static)
- [final](#final)
- [String、StringBuffer、StringBuilder](#stringstringbufferstringbuilder)
- [异常处理](#异常处理)
- [内部类](#内部类)
  - [匿名内部类](#匿名内部类)
- [多态](#多态)
- [抽象和接口](#抽象和接口)
- [集合框架](#集合框架)
  - [HashMap](#hashmap)
    - [结构图](#结构图)
    - [HashMap 的工作原理](#hashmap-的工作原理)
    - [HashMap 与 HashTable 对比](#hashmap-与-hashtable-对比)
  - [ConcurrentHashMap](#concurrenthashmap)
    - [Base 1.7](#base-17)
    - [Base 1.8](#base-18)
  - [ArrayList](#arraylist)
  - [LinkedList](#linkedlist)
  - [CopyOnWriteArrayList](#copyonwritearraylist)
- [反射](#反射)
- [单例](#单例)
  - [饿汉式](#饿汉式)
  - [双重检查模式](#双重检查模式)
  - [静态内部类模式](#静态内部类模式)
- [线程](#线程)
  - [属性](#属性)
  - [状态](#状态)
  - [状态控制](#状态控制)
- [volatile](#volatile)
- [synchronized](#synchronized)
  - [根据获取的锁分类](#根据获取的锁分类)
  - [原理](#原理)
- [Lock](#lock)
  - [锁的分类](#锁的分类)
    - [悲观锁、乐观锁](#悲观锁乐观锁)
    - [自旋锁、适应性自旋锁](#自旋锁适应性自旋锁)
    - [死锁](#死锁)
- [引用类型](#引用类型)
- [动态代理](#动态代理)
- [元注解](#元注解)
# JVM
## JVM 工作流程
![](https://user-gold-cdn.xitu.io/2019/6/23/16b833f4a4906226?w=448&h=592&f=jpeg&s=44057)

## 运行时数据区（Runtime Data Area）
![](https://user-gold-cdn.xitu.io/2019/6/23/16b833f4a499f6fe?w=868&h=497&f=webp&s=46378)

### 程序计数器
**程序计数器（Program Counter Register）** 是一块较小的内存空间，它可以看作是当前线程所执行的字节码的行号指示器。

字节码解释器工作时就是通过改变这个计数器的值来选取下一条需要执行的字节码指令，分支、循环、跳转、异常处理、线程恢复等基础功能都需要依赖这个计数器来完成。

由于 Java 虚拟机的多线程是通过线程轮流切换并分配处理器执行时间的方式来实现的，**在任何一个确定的时刻，一个处理器（对于多核处理器来说是一个内核）都只会执行一条线程中的指令**。

因此，为了线程切换后能恢复到正确的执行位置，每条线程都需要有一个独立的程序计数器，各条线程之间计数器互不影响，独立存储，我们称这类内存区域为“线程私有”的内存。

- 如果线程正在执行的是一个 Java 方法，这个计数器记录的是正在执行的**虚拟机字节码指令的地址**。
- 如果线程正在执行的是一个 Native 方法，这个计数器值则为空（Undefined）。
> 对于Native方法的线程状态保持和恢复，JVM使用栈帧和本地方法接口（JNI）等机制来管理。每个方法调用，无论是Java方法还是Native方法，都会在调用栈上创建一个栈帧，记录方法调用的状态信息。对于Native方法，JNI框架提供了一种方式来保证在调用和执行完成后，可以正确地恢复到Java环境中，确保线程的连续性和状态的正确管理。此外，操作系统的线程调度和上下文切换机制也保障了在执行Native方法期间线程状态的保存和恢复。

**此内存区域是唯一一个在Java虚拟机规范中没有规定任何OutOfMemoryError情况的区域。**

### Java 虚拟机栈
**Java 虚拟机栈（Java Virtual Machine Stacks**）也是线程私有的，它的生命周期与线程相同。虚拟机栈描述的是 Java 方法执行的内存模型，每个方法在执行的同时都会创建一个**栈帧（Stack Frame）** 用于存储**局部变量表、操作数栈、动态链接、方法出口**等消息。每一个方法从调用直至执行完成的过程，就对应着一个栈帧在虚拟机栈中入栈到出栈的过程。

**局部变量表**（Local Variable Array）存放了编译器可知的各种基本数据类型（boolean、byte、char、short、int、float、long、double）、对象引用（reference类型，它不等同于对象本身，可能是一个指向对象起始地址的引用指针，也可能是指向一个代表对象的句柄或其他与此对象相关的位置）和 returnAddress 类型（指向了一条字节码指令的地址）。

其中 64 位长度的 long 和 double 类型的数据会占用两个局部变量空间（Slot），其余的数据类型只占用一个。局部变量表所需的内存空间在编译期间完成分配，当进入一个方法时，这个方法需要在帧中分配多大的局部变量空间是完全确定的，在方法运行期间不会改变局部变量表的大小。

**操作数栈**（Operand Stack）：也称为操作栈，是一个后进先出（LIFO）栈。指令执行时，会将操作数压入操作数栈，指令执行后，会将结果弹出操作数栈。操作数栈的深度在编译期间就已经被确定。

**动态链接**（Dynamic Linking）：每个栈帧内部都包含一个指向运行时常量池中该栈帧所属方法的引用，以支持方法调用过程中的动态链接。这使得方法能够调用其他方法或者访问类的字段。

关于动态连接有更细致的说明：
> 类加载阶段：当Java虚拟机（JVM）加载类文件时，它首先会将类文件中的符号引用加载到运行时常量池中。这些符号引用包括类和接口的全限定名、字段的名称和描述符、方法的名称和描述符等。
符号引用转直接引用：在某个时间点，这些符号引用需要被转换为直接引用，这个过程称为解析（Resolution）。直接引用是指向方法区中类、方法和字段具体地址的指针。这个解析过程可以在类加载时（即静态解析）完成，也可以在首次使用时（即动态解析）完成。
执行过程中的动态链接：当执行到一个方法调用或字段访问指令时，JVM会使用栈帧中的符号引用去运行时常量池中查找对应的直接引用。如果这个符号引用还没有被解析为直接引用，JVM会先进行解析。一旦拿到直接引用，JVM就可以执行相应的操作了。
对于方法调用，这意味着JVM会根据直接引用找到方法的实际代码位置并执行。
对于字段访问，JVM会根据直接引用找到字段在对象中的具体位置或类中的静态字段位置，并进行读写操作。
这个机制支持了Java的多态性和动态绑定，因为直到运行时，JVM才最终确定调用哪个具体的方法版本或访问哪个字段。这也是Java动态语言特性的基础之一，允许Java程序在运行时加载、链接和修改类和方法。

**方法出口信息**（Method Return Address）：当一个方法调用完成后，需要返回到方法被调用的位置继续执行。方法出口信息包含了调用该方法之前的栈帧的状态，以便恢复到方法调用前的状态。

在 Java 虚拟机规范中，对这个区域规定了两种异常状态：
- 如果线程请求的栈深度大于虚拟机所允许的的深度，将抛出 **StackOverflowError** 异常。
- 如果虚拟机栈可以动态扩展（当前大部分的Java虚拟机都可动态扩展，只不过Java虚拟机规范中也允许固定长度的虚拟机栈），如果扩展时无法申请到足够的内存，就会抛出 **OutOfMemoryError** 异常。

### 本地方法栈
**本地方法栈（Native Method Stack）** 与虚拟机栈所发挥的作用是非常相似的，它们之间的区别不过是虚拟机栈为虚拟机执行Java方法（也就是字节码）服务，而本地方法栈则为虚拟机使用到的Native方法服务。

Native方法是用非Java语言（如C或C++）编写的方法，它们不由JVM直接执行，而是通过JNI（Java Native Interface）调用本地系统或其他语言的库函数。

在虚拟机规范中对本地方法栈中方法使用的语言、使用方式与数据结构并没有强制规定，但它通常包含了方法参数、局部变量、调用本地方法所需的任何环境信息等。具体的虚拟机可以自由实现它。甚至有的虚拟机（例如：Sun HotSpot虚拟机）直接就把虚拟机栈和本地方法栈合二为一。

与虚拟机栈一样，本地方法栈区域也会抛出 **StackOverflowError** 和 **OutOfMemoryError** 异常。

### Java 堆
对于大多数应用来说，**Java 堆（Java Heap）** 是 Java 虚拟机所管理的的内存中最大的一块。Java 堆是被**所有线程共享**的一块内存区域，在虚拟机启动时创建，其大小可以是固定的，也可以是动态扩展的，这取决于具体的JVM实现和配置参数（比如-Xms和-Xmx分别用来设置堆的初始大小和最大大小）。此内存区域的唯一目的就是存放对象实例，几乎所有的对象实例都在这里分配内存。

Java堆的结构可能因JVM的不同而有所不同，但通常包括以下几个部分：

**年轻代**（Young Generation）：存放新创建的对象。大部分情况下，对象首先在年轻代中分配。年轻代通常分为一个或多个Eden区和两个Survivor区（分别称为From和To）。
**老年代**（Old Generation）：存放经过多次垃圾回收仍然存活的对象。当对象在年轻代中存活足够长的时间后，就会被移动到老年代中。
**永久代/元空间**（PermGen/Metaspace，取决于JVM版本）：用于存储类的元数据信息。在Java 8及以后的版本中，永久代已被元空间（Metaspace）所替代。

Java堆是垃圾收集器工作的主要场所。垃圾收集器会自动管理堆内存的分配和释放，帮助开发者避免内存泄漏和溢出的问题。

堆的大小和垃圾收集策略的选择会直接影响到程序的性能，包括响应时间和吞吐量。

Thread Local Allocation Buffer（TLAB）是一种性能优化技术，它的主要作用是提高对象分配的效率。
Java虚拟机（JVM）在堆内存中为每个线程分配了一个私有的缓冲区域——TLAB，以减少线程之间在对象分配时的竞争；而且线程在缓冲区中可以进行连续内存分配，有利于提高内存分配效率；它可以动态调整分配缓冲区的大小，比如一个线程频繁分配对象，就可以适当增加这个线程分配缓冲区的大小。
线程分配缓冲区也有一些缺点，例如，TLAB占用的是堆内存的一部分，每个线程分配一个TLAB会占用额外的内存资源。如果有大量线程，但每个线程的对象分配量不大，这可能会导致内存的浪费。

### 方法区
**方法区（Method Area**）与 Java 堆一样，是各个**线程共享**的内存区域，它用于存储已被虚拟机加载的类信息、常量、静态变量、即时编译器编译后的代码等数据。

>在JDK 7及以前的版本中，方法区的实现通常被称为永久代（PermGen space）。由于永久代容易出现内存溢出的问题，从JDK 8开始，Oracle的JVM实现（HotSpot）用元空间（MetaSpace）替代了永久代。元空间不在虚拟机内存中，而是使用本地内存，这样做的目的是为了更好地支持动态生成大量类的应用场景，例如OSGi、动态代理等。

**类信息存储**：当类被加载到JVM时，JVM会将类的全限定名、类的直接父类的全限定名（除了Object类，它没有父类）、类的修饰符（public、abstract、final的标志），以及类的接口、字段和方法的信息存储在方法区中。

**运行时常量池（Runtime Constant Pool）** 方法区包含一个运行时常量池，用于存储编译期生成的各种字面量和符号引用，这些内容将被加载到方法区中。运行时常量池是方法区的一部分，它具有动态性，Java语言并不要求常量池中的内容在编译期间完全确定，也就是说，在程序运行期间也可能将新的常量放入池中。
- 字面量: 就是值，字面量通常在编译时期就确定，并存储在类文件的常量池中。加载类或接口到JVM时，这些字面量会被加载到运行时常量池中。
- 符号引用：就是名称和描述符，符号引用在编译期间生成，存储的是一些符号地址，这些地址在类加载的解析阶段会被转换成直接引用。直接引用是指向方法区中类结构、方法数据或者字段数据的指针或者偏移量。这种转换过程确保了对类、方法、字段的引用在运行时能够被JVM正确访问。

**静态变量**：类中定义的静态变量也会被存放在方法区中，所有实例共享这些变量，可以直接通过类来访问它们。

**即时编译器编译后的代码**：一些现代JVM实现了即时编译（JIT编译），将字节码编译成本地机器码以提高效率，这些编译后的代码也存储在方法区。

既然运行时常量池是方法区的一部分，自然受到方法区内存的限制，当常量池无法再申请到内存时就会抛出 OutOfMemoryError 异常。

>从Java 8开始，Oracle的JVM实现中，方法区的实现方式发生了变化。Java 8引入了元空间（Metaspace）来替代原有的永久代（PermGen），其中方法区的数据（如类的元数据、运行时常量池等）现在是存储在本地内存中，而不是JVM堆内存中。
这一变化的主要目的是为了解决永久代可能导致的内存溢出问题，并允许类元数据所占用的内存能够根据应用程序的需求动态扩展。由于本地内存的容量通常大于JVM堆内存的固定大小，这一策略提高了JVM的灵活性和扩展性，使得Java应用能更好地适应不同的运行环境和需求。
元空间的使用，使得Java的内存管理在处理大量类加载和卸载的情况下变得更加高效，特别是对于那些动态生成大量类的应用程序（如某些Web服务器和应用服务器），能够减少内存溢出的风险，提高应用程序的稳定性。

## 方法指令
JVM指令有很多种，如加载和存储指令、算术指令、类型转换指令、对象创建和操作指令、栈操作指令、控制转移指令、方法调用和返回指令、同步指令、异常处理指令、扩展指令等，方法指令只是其中一种：
| 指令 | 说明 |                     
|----------|-----|
| invokeinterface | 用以调用接口方法 |
| invokevirtual | 指令用于调用对象的实例方法 |
| invokestatic | 用以调用类/静态方法 |
| invokespecial | 用于调用一些需要特殊处理的实例方法，包括实例初始化方法、私有方法和父类方法 | 
| return | 用于从方法返回，它根据方法返回类型的不同有几个变种，如ireturn（返回整型）、lreturn（返回长整型）以及return（用于返回void类型方法）等|

## 类加载器
java中的类加载器是用于加载Java类到Java虚拟机中的部分。它们在Java运行时系统中起着至关重要的作用，负责将类的.class文件从各种来源（如文件系统、网络等）加载到运行时的数据区域中，以便Java程序可以使用这些类。Java中主要有以下几种类型的类加载器：

| 类加载器 | 说明 |                     
|----------|-----|
| BootstrapClassLoader（引导类加载器） | 它是虚拟机自带的类加载器，负责加载Java的核心库（JAVA_HOME/jre/lib/rt.jar或sun.boot.class.path路径下的内容）。**引导类加载器是用原生代码实现的**，并不继承自java.lang.ClassLoader。它主要负责加载Java的核心类库，为Java的运行提供最基本的类。 |
| ExtClassLoader (扩展类加载器) | 它是由Java实现的，并且继承自ClassLoader类。Extension 将加载类的请求先委托给它的父加载器，也就是Bootstrap，如果没有成功加载的话，再从 jre/lib/ext 目录下或者 java.ext.dirs 系统属性定义的目录下加载类。这允许Java的扩展库放在这个目录下，通过扩展类加载器加载到JVM中。Extension 加载器由 sun.misc.Launcher$ExtClassLoader 实现 |
| AppClassLoader(应用类加载器) | 第三种默认的加载器就是 System 类加载器（又叫作 Application 类加载器）了，它是程序中默认的类加载器，Java应用的类都是由它来加载的。它也是由Java实现的，并且继承自ClassLoader类。它负责从 classpath 环境变量中加载某些应用相关的类，classpath 环境变量通常由 -classpath 或 -cp 命令行选项来定义，或者是 JAR 中的 Manifest 的 classpath 属性。Application 类加载器是 Extension 类加载器的子加载器 |
| 自定义类加载器 | Java允许用户通过继承ClassLoader类的方式创建自己的类加载器。通过重写ClassLoader类的findClass(String name)方法，用户可以自定义类的加载方式。用户自定义加载器可以实现特殊的加载需求，比如从网络或其他非标准源加载类。|

| 工作原理 | 说明 |                     
|----------|------|
| 委托机制 | 加载任务委托交给父类加载器，如果不行就向下传递委托任务，由其子类加载器加载，这样做是为了避免重复加载以及保证 java 核心库的安全性 |
| 可见性机制 | 子加载器可以访问父加载器加载的类，而父加载器不能访问子加载器加载的类。|
| 单一性机制 | 由于委托机制的存在，同一个类在程序的一次执行过程中只会被加载一次，无论通过哪个加载器加载，确保了Java程序运行时的稳定性和安全性 |

当你调用String.class.getClassLoader()并得到null结果时，这实际上表示String类是由引导类加载器（Bootstrap Class Loader）加载的。引导类加载器是用于加载Java核心库的类加载器，它并不是Java语言实现的，而是由虚拟机的本地实现部分实现的。因此，它在Java代码中不表示为一个ClassLoader对象，而是以null来表示。

## 垃圾回收 gc
### 对象存活判断
- **引用计数**（Reference Counting）

每个对象有一个引用计数属性，新增一个引用时计数加1，引用释放时计数减1，计数为0时可以回收。
此方法简单，无法解决对象相互循环引用的问题。 

- **可达性分析**（Reachability Analysis）

这是目前主流JVM使用的算法。

从 GC Roots 开始向下搜索，搜索所走过的路径称为引用链。当一个对象到 GC Roots 没有任何引用链相连时，则证明此对象是不可用的。不可达对象。

> 在Java语言中，GC Roots包括：
> - 虚拟机栈中引用的对象。
> - 方法区中类静态属性引用的对象。
> - 方法区中常量引用的对象。
> - 本地方法栈中 JNI（即一般说的Native方法）引用的对象。

### 垃圾收集算法
- **标记 -清除算法**（Mark-Sweep）
  
“标记-清除”（Mark-Sweep）算法，如它的名字一样，算法分为“标记”和“清除”两个阶段：首先标记出所有需要回收的对象，在标记完成后统一回收掉所有被标记的对象。之所以说它是最基础的收集算法，是因为后续的收集算法都是基于这种思路并对其缺点进行改进而得到的。

它的主要缺点有两个：一个是效率问题，标记和清除过程的效率都不高；另外一个是空间问题，标记清除之后会产生大量不连续的内存碎片，空间碎片太多可能会导致，当程序在以后的运行过程中需要分配较大对象时无法找到足够的连续内存而不得不提前触发另一次垃圾收集动作。

- **复制算法**（Copying）
  
“复制”（Copying）的收集算法，它将可用内存按容量划分为大小相等的两块，每次只使用其中的一块。当这一块的内存用完了，就将还存活着的对象复制到另外一块上面，然后再把已使用过的内存空间一次清理掉。

这样使得每次都是对其中的一块进行内存回收，内存分配时也就不用考虑内存碎片等复杂情况，只要移动堆顶指针，按顺序分配内存即可，实现简单，运行高效。只是这种算法的代价是将内存缩小为原来的一半，内存使用率降低，持续复制长生存期的对象则导致效率降低。

- **标记-整理算法**（Mark-Compact）
  
复制收集算法在对象存活率较高时就要执行较多的复制操作，效率将会变低。更关键的是，如果不想浪费50%的空间，就需要有额外的空间进行分配担保，以应对被使用的内存中所有对象都100%存活的极端情况，所以在老年代一般不能直接选用这种算法。

根据老年代的特点，有人提出了另外一种“标记-整理”（Mark-Compact）算法，标记过程仍然与“标记-清除”算法一样，但后续步骤不是直接对可回收对象进行清理，而是让所有存活的对象都向一端移动，然后直接清理掉端边界以外的内存。

- **分代收集算法** （Generational Collection）

这不是一种具体的算法，而是一种垃圾回收的策略。

GC 分代的基本假设：绝大部分对象的生命周期都非常短暂，存活时间短。

把Java堆分为新生代和老年代，这样就可以根据各个年代的特点采用最适当的收集算法。在新生代中，每次垃圾收集时都发现有大批对象死去，只有少量存活，那就选用复制算法，只需要付出少量存活对象的复制成本就可以完成收集。而老年代中因为对象存活率高、没有额外空间对它进行分配担保，就必须使用“标记-清理”或“标记-整理”算法来进行回收。

### 垃圾收集器
- **CMS收集器**
  
> CMS（Concurrent Mark Sweep，并发标记清除垃圾回收器）收集器是一种以获取最短回收停顿时间为目标的收集器。目前很大一部分的 Java 应用都集中在互联网站或B/S系统的服务端上，这类应用尤其重视服务的响应速度，希望系统停顿时间最短，以给用户带来较好的体验。

从名字（包含“Mark Sweep”）上就可以看出CMS收集器是基于“标记-清除”算法实现的，它的运作过程相对于前面几种收集器来说要更复杂一些，整个过程分为4个步骤，包括：

- 初始标记（CMS initial mark）
- 并发标记（CMS concurrent mark）
- 重新标记（CMS remark）
- 并发清除（CMS concurrent sweep）

其中初始标记、重新标记这两个步骤仍然需要“Stop The World”。初始标记仅仅只是标记一下GC Roots能直接关联到的对象，速度很快，并发标记阶段就是进行GC Roots Tracing的过程，而重新标记阶段则是为了修正并发标记期间，因用户程序继续运作而导致标记产生变动的那一部分对象的标记记录，这个阶段的停顿时间一般会比初始标记阶段稍长一些，但远比并发标记的时间短。

由于整个过程中耗时最长的并发标记和并发清除过程中，收集器线程都可以与用户线程一起工作，所以总体上来说，CMS收集器的内存回收过程是与用户线程一起并发地执行。老年代收集器（新生代使用ParNew）

- **G1收集器**

G1: （Garbage-First垃圾回收器）

G1通过将堆划分为多个（通常是2048个）独立的小块（Region）来管理内存，采用一种优先列表（即首先回收价值最大的区域）的方式来进行垃圾收集，目标是提供一个更可预测的垃圾收集停顿时间，同时还不牺牲太多的吞吐量。

G1的新生代收集跟 ParNew 类似，当新生代占用达到一定比例的时候，开始出发收集。和 CMS 类似，G1 收集器收集老年代对象会有短暂停顿。

与CMS收集器相比G1收集器有以下特点：

1、空间整合，G1收集器采用标记整理算法，不会产生内存空间碎片。分配大对象时不会因为无法找到连续空间而提前触发下一次GC。

2、可预测停顿，这是G1的另一大优势，降低停顿时间是G1和CMS的共同关注点，但G1除了追求低停顿外，还能建立可预测的停顿时间模型，能让使用者明确指定在一个长度为N毫秒的时间片段内，消耗在垃圾收集上的时间不得超过N毫秒，这几乎已经是实时 Java（RTSJ）的垃圾收集器的特征了。

### 堆内存模型与回收策略
![](https://mmbiz.qpic.cn/mmbiz_png/qdzZBE73hWsbhfAng9ibqfcbjrqgyRWqAKiaJ2U75SGYwQhs2tuNbXtu8KIpaUsBOaHRKXf7esuuFoMjELFxibIVg/640?wx_fmt=png&tp=webp&wxfrom=5&wx_lazy=1&wx_co=1)

Java 堆（Java Heap）是JVM所管理的内存中最大的一块，堆又是垃圾收集器管理的主要区域，Java 堆主要分为2个区域-年轻代与老年代，其中年轻代又分 Eden 区和 Survivor 区，其中 Survivor 区又分 From 和 To 2个区。

- **Eden 区**
  
大多数情况下，对象会在新生代 Eden 区中进行分配，当 Eden 区没有足够空间进行分配时，虚拟机会发起一次 Minor GC，Minor GC 相比 Major GC 更频繁，回收速度也更快。
通过 Minor GC 之后，Eden 会被清空，Eden 区中绝大部分对象会被回收，而那些无需回收的存活对象，将会进到 Survivor 的 From 区（若 From 区不够，则直接进入 Old 区）。

- **Survivor 区**

Survivor 区相当于是 Eden 区和 Old 区的一个**缓冲**，类似于我们交通灯中的黄灯。Survivor 又分为2个区，一个是 From 区，一个是 To 区。每次执行 Minor GC，会将 Eden 区和 From 存活的对象放到 Survivor 的 To 区（如果 To 区不够，则直接进入 Old 区）。**Survivor 的存在意义就是减少被送到老年代的对象，进而减少 Major GC 的发生**。Survivor 的预筛选保证，只有经历16次 Minor GC 还能在新生代中存活的对象，才会被送到老年代。

- **Old 区**
  
老年代占据着2/3的堆内存空间，只有在 Major GC 的时候才会进行清理，**每次 GC 都会触发“Stop-The-World”**。内存越大，STW 的时间也越长，所以内存也不仅仅是越大就越好。由于复制算法在对象存活率较高的老年代会进行很多次的复制操作，效率很低，所以老年代这里采用的是标记——整理算法。


# Object
## equals 方法
对两个对象的地址值进行的比较（即比较引用是否相同）
```java
public boolean equals(Object obj) {
    return (this == obj);
}
```

## hashCode 方法
hashCode() 方法给对象返回一个 hash code 值。这个方法被用于 hash tables，例如 HashMap。

它的性质是：
- 一致性：在一个Java应用的执行期间，如果一个对象提供给 equals 做比较的信息没有被修改的话，该对象多次调用 hashCode() 方法，该方法必须始终如一返回同一个 integer。

- 与equals()方法的关系：如果两个对象根据 equals(Object) 方法是相等的，那么调用二者各自的 hashCode() 方法必须产生同一个 integer 结果。

- 并不要求根据 equals(Object) 方法不相等的两个对象，调用二者各自的 hashCode() 方法必须产生不同的 integer 结果。然而，程序员应该意识到对于不同的对象产生不同的 integer 结果，有可能会提高 hash table 的性能。

覆写equals时需要覆写hashCode：如果你覆写了对象的equals()方法，那么也应该覆写hashCode()方法，以保持上述equals方法和hashCode方法之间的一致性。

在 JDK 中，Object 的 hashcode 方法是本地方法，也就是用 c 语言或 c++ 实现的，该方法直接返回**对象的 内存地址**。在 String 类，重写了 hashCode 方法
```java
public int hashCode() {
    int h = hash;
    if (h == 0 && value.length > 0) {
        char val[] = value;

        for (int i = 0; i < value.length; i++) {
            h = 31 * h + val[i];
        }
        hash = h;
    }
    return h;
}
```

# static
- static关键字修饰的方法或者变量不需要依赖于对象来进行访问，只要类被加载了，就可以通过类名去进行访问。
- 静态变量被所有的对象所共享，在内存中只有一个副本，它当且仅当在类初次加载时会被初始化。
- 能通过 this 访问静态成员变量吗?
所有的静态方法和静态变量都可以通过对象访问（只要访问权限足够）。
- static是不允许用来修饰局部变量

# final
- 可以声明成员变量、方法、类以及局部变量
- final 成员变量必须在声明的时候初始化或者在构造器中初始化，否则就会报编译错误
- final 变量是只读的
- final 申明的方法不可以被子类的方法重写
- final 类通常功能是完整的，不能被继承
- final 变量可以安全的在多线程环境下进行共享，而不需要额外的同步开销
- final 关键字提高了性能，JVM 和 Java 应用都会缓存 final 变量，会对方法、变量及类进行优化
- 方法的内部类访问方法中的局部变量，但必须用 final 修饰才能访问（这些变量实际上是被内部类的实例复制了一份，存储在堆上，使得即使方法执行完毕局部变量消失后，内部类仍然可以无障碍地访问这些数据。声明为final，避免了复杂的同步问题，同时也简化了内部类的实现）

# String、StringBuffer、StringBuilder
- String 是 final 类，不能被继承。对于已经存在的 Stirng 对象，修改它的值，就是重新创建一个对象
- StringBuffer 是一个类似于 String 的字符串缓冲区，使用 append() 方法修改 Stringbuffer 的值，使用 toString() 方法转换为字符串，是线程安全的
- StringBuilder 用来替代于 StringBuffer，StringBuilder 是非线程安全的，速度更快

# 异常处理
- Exception、Error 是 Throwable 类的子类
- Error 类对象由 Java 虚拟机生成并抛出，不可捕捉，或者说不建议捕捉，因为系统已经发生严重的错误
- 不管有没有异常，finally 中的代码都会执行
- 当 try、catch 中有 return 时，finally 中的代码依然会继续执行，此时会先执行finally中的代码，最后返回；如果finally中也有return，此时会覆盖try-catch中的return，这不是好的编程实践

**检查异常**（Checked Exceptions）
含义：检查异常是那些在编译时就必须被捕获或声明抛出的异常。这种异常的设计初衷是让开发者在编写代码时就能明确地处理潜在的错误情况，如输入/输出（I/O）操作失败。
例子：IOException和SQLException。
**非检查异常**（Unchecked Exceptions）
含义：非检查异常包括运行时异常（RuntimeException）和错误（Error）。这些异常可以在编程时不被显式捕获或声明抛出。运行时异常通常是编程错误，如试图从数组中访问不存在的元素。错误通常是指Java虚拟机（JVM）无法恢复的严重问题，如OutOfMemoryError。
例子：NullPointerException和ArrayIndexOutOfBoundsException。

| 常见的Error | | |
|------|-----|-----|
| OutOfMemoryError | StackOverflowError | NoClassDeffoundError |

| 常见的Exception | | |
|------|-----|-----|
| 常见的非检查性异常 |  |
| ArithmeticException | ArrayIndexOutOfBoundsException | ClassCastException |
| IllegalArgumentException | IndexOutOfBoundsException | NullPointerException |
| NumberFormatException | SecurityException | UnsupportedOperationException |
| 常见的检查性异常 |  |
| IOException | CloneNotSupportedException | IllegalAccessException |
| NoSuchFieldException | NoSuchMethodException | FileNotFoundException |

有一个例子，比如Android开发中，内存中加载了一张占用内存图片，它可能出现OOM。如果我对这段逻辑进行try-catch并释放某些内存资源。那么，假设真的触发了错误，这里的异常捕获能否按照我期望的逻辑正常运行，并正确的执行后续的代码，不终止程序进程？

这个要分情况，如果这种操作是一次性或偶尔执行的内存密集型操作，那它确实可能有效；反之，可能事半功倍。
- 内存到达阈值，可能任何地方都会触发OOM，意义不大；
- 正确的识别并释放内存的操作并不容易；
- 如果没有解决根本原因，程序可能很快再次遇到OOM；

>Kotlin中淡化了“检查异常”的概念，这其实是语言设计者的理念问题。比方说让代码更加简洁，提供了其他的语法糖(如`runCatching`)来处理异常等。但不可否认它这样处理也带来了隐患，这就要求开发者有较高的意识，为了程序的严谨性，要恰当的手动处理异常。

# 内部类
Java 中的内部类可以大致分为三种：静态内部类（static nested class）、非静态内部类（inner class）以及匿名内部类（anonymous inner class）。

- 非静态内部类没法在外部类的静态方法中实例化。
- 非静态内部类的方法可以直接访问外部类的所有数据，包括私有的数据。
- 可以直接访问外部类的静态成员，而不需要外部类的实例。但是不能直接访问外部类的非静态成员。
- 创建静态内部类的对象可以直接通过外部类调用静态内部类的构造器；创建非静态的内部类的对象必须先创建外部类的对象，通过外部类的对象调用内部类的构造器。

## 匿名内部类
常用于监听器(listener)的实现或是简单的回调函数，以及在单元测试中快速实现接口或类的实例。

- 匿名内部类不能定义任何静态成员、方法
- 匿名内部类中的方法不能是抽象的
- 匿名内部类必须实现接口或抽象父类的所有抽象方法
- 匿名内部类不能定义构造器
- 匿名内部类访问的外部类成员变量或成员方法必须用 final 修饰

>Kotlin的内部类默认是静态的，Kotlin鼓励更好的编程实践，只有你明确需要使用某些特性的时候，才去声明：对于内部类来将，只有明确需要外部类引用的时候，才去主动声明inner class。再者，是为了避免内存泄漏问题，因为内部类会默认持有外部类的引用。

# 多态
- 父类的引用可以指向子类的对象
- 创建子类对象时，调用的方法为子类重写的方法或者继承的方法
- 如果我们在子类中编写一个独有的方法，此时就不能通过父类的引用创建的子类对象来调用该方法

### 多态实现的基础：
继承（实现）、重写、向上转型。

### 多态实现的流程和动态连接：
- 编译时的类型检查：Java编译器检查方法调用是否合法，检查基于引用变量的类型。
- 运行时的动态绑定：**在运行时，JVM根据对象的实际类型来确定应该调用哪个类的哪个方法。**这个过程是动态的，即“动态绑定”。

# 抽象和接口
- 抽象类有构造函数，目的是为了初始化某些逻辑以及被子类调用
- 抽象类不能有对象（不能用 new 关键字来创建抽象类的对象）
- 抽象类中的抽象方法必须在子类中被重写
- 接口中的所有属性默认为：public static final ****；
- 接口中的所有方法默认为：public abstract ****；

### Java 8及以后的新特性
- 默认方法：允许在接口中有默认实现的方法，使得接口有更大的灵活性。
- 静态方法：允许在接口中定义静态方法。
- 私有方法：Java 9引入了接口的私有方法，主要用于解决默认方法之间的代码重复问题。

# 集合框架
![](https://user-gold-cdn.xitu.io/2019/6/23/16b833f4a86db5e6?w=643&h=611&f=gif&s=22445)
- List接口存储一组不唯一，有序（插入顺序）的对象, Set接口存储一组唯一，无序的对象。

## HashMap
### 结构图
- **JDK 1.7 HashMap 结构图**

![](https://user-gold-cdn.xitu.io/2019/6/23/16b833f4ac8f44fd?w=1636&h=742&f=png&s=88323)

- **JDK 1.8 HashMap 结构图**

![](https://user-gold-cdn.xitu.io/2018/7/23/164c47f32f9650ba?imageView2/0/w/1280/h/960/format/webp/ignore-error/1)

```
HashMap
|
|-- Array (Hash Table)
    |
    |-- Bucket[0] --> Entry --> Entry --> ...
    |
    |-- Bucket[1] --> Entry (转换为红黑树)
    |                /    \
    |               Node  Node
    |               /  \   /  \
    |             ...  ... ... ...
    |
    |-- Bucket[2] --> Entry --> Entry
    |
    |-- ...
```

### 结构
- 数组（Hash Table）：HashMap 的核心是一个**动态数组**（Hash Table），数组的每个槽位（slot）称为“桶”（Bucket）。这个数组的默认初始大小为 **16**（这个值可以在创建 HashMap 实例时通过构造函数指定），并且每次扩展或缩减时，大小总是保持 2 的幂次方（每次扩展2倍）。
- 链表：当不同的键值对（Entries）通过散列函数产生了相同的散列码（Hash Code），并且被分配到同一个桶中时，它们会以链表的形式存储在该桶中。这种情况称为“冲突”（Collision）。
- 红黑树：为了提高在高冲突情况下的查询效率，当链表的长度超过某个阈值（JDK 1.8 默认为 8）时，链表会被转换为红黑树。这样可以保证在最坏情况下的查找时间复杂度为 **O(log n)**。

### HashMap 的工作原理
HashMap 基于 散列（hashing） 原理，我们通过 put() 和 get() 方法储存和获取对象。当我们将键值对传递给 put() 方法时，它调用键对象的 hashCode() 方法来计算 hashcode，让后找到 bucket 位置来储存 Entry 对象。当两个对象的 hashcode 相同时，它们的 bucket 位置相同，‘**碰撞**’会发生。因为 HashMap 使用链表存储对象，这个 Entry 会存储在链表中，当获取对象时，通过键对象的 equals() 方法找到正确的键值对，然后返回值对象。

**散列函数**
1. 使用Object的hashCode()得到一个int类型的值。
2. 使用了一种高位混合的方法，通过对散列码的高位和低位进行异或（XOR）操作来增加低位的随机性，从而减少冲突。
3. 与数组的长度减一进行按位与操作（index = hashCode & (n - 1)），以计算出键值对应该存储在数组的哪个位置（桶）。

**如果 HashMap 的大小超过了负载因子(load factor)定义的容量，怎么办？**  
默认的负载因子大小为 0.75，也就是说，当一个 map 填满了 75% 的 bucket 时候，和其它集合类(如 ArrayList 等)一样，将会创建原来 HashMap 大小的两倍的 bucket 数组，来重新调整 map 的大小，并将原来的对象放入新的 bucket 数组中。这个过程叫作 rehashing，因为它调用 hash 方法找到新的 bucket 位置。

**为什么 String, Interger 这样的 wrapper 类适合作为键?**  
因为 String 是不可变的，也是 final 的，而且已经重写了 equals() 和 hashCode() 方法了。其他的 wrapper 类也有这个特点。**不可变性是必要的**，因为为了要计算 hashCode()，就要防止键值改变，如果键值在放入时和获取时返回不同的 hashcode 的话，那么就不能从 HashMap 中找到你想要的对象。不可变性还有其他的优点如线程安全。如果你可以仅仅通过将某个 field 声明成 final 就能保证 hashCode 是不变的，那么请这么做吧。因为获取对象的时候要用到 equals() 和 hashCode() 方法，那么键对象正确的重写这两个方法是非常重要的。如果两个不相等的对象返回不同的 hashcode 的话，那么碰撞的几率就会小些，这样就能提高 HashMap 的性能。

### fail-fast
HashMap 的迭代器是快速失败（fail-fast）的，这意味着如果在迭代过程中集合被修改，除了通过迭代器自己的 remove 方法外，迭代器会立即抛出 ConcurrentModificationException。
这里的修改是指增删数据，导致数据结构发生变化。
HashMap内部会维护一个modCount字段，每次修改数据后，都会累加；创建迭代器的时候，当前modCount会同步到迭代器的expectedModCount；迭代器迭代过程中（如调用 next() 或 remove() 方法），会判断modCount和expectedModCount是否相等，如果不相等则证明数据结构发生了变化，则会直接抛出ConcurrentModificationException。

### HashMap 与 HashTable 对比

|条目| HashMap| HashTable|
|------|-----|-----|
|父类| 实现Map接口| 继承Dictionary |
|同步性|非 synchronized 的，单线程下性能更好| 线程安全的，多线程下性能较差 |
|null key/value| 允许一个null key和 多个null value| 不接受 null 的 key-value |
|迭代器| Iterator 是fail-fast的 | Iterator、Enumerator ， Enumerator不是fail-fast的 |

``HashMap.java``
```java
public class HashMap<K,V> extends AbstractMap<K,V>
    implements Map<K,V>, Cloneable, Serializable {
    ···

    public V put(K key, V value) {
        return putVal(hash(key), key, value, false, true);
    }
    ···

    public V get(Object key) {
        Node<K,V> e;
        return (e = getNode(hash(key), key)) == null ? null : e.value;
    }
    ···
}
```

``HashTable.java``
```java
public class Hashtable<K,V>
    extends Dictionary<K,V>
    implements Map<K,V>, Cloneable, java.io.Serializable {
    ···

    public synchronized V put(K key, V value) {
        // Make sure the value is not null
        if (value == null) {
            throw new NullPointerException();
        }

        ···
        addEntry(hash, key, value, index);
        return null;
    }
    ···

    public synchronized V get(Object key) {
        HashtableEntry<?,?> tab[] = table;
        int hash = key.hashCode();
        int index = (hash & 0x7FFFFFFF) % tab.length;
        for (HashtableEntry<?,?> e = tab[index] ; e != null ; e = e.next) {
            if ((e.hash == hash) && e.key.equals(key)) {
                return (V)e.value;
            }
        }
        return null;
    }
    ···
}
```

## ConcurrentHashMap

与传统的 Hashtable 和同步的 HashMap (通过 Collections.synchronizedMap 包装得到) 相比，ConcurrentHashMap 提供了更好的并发性能，并且在多线程环境中能够更高效地工作。
ConcurrentHashMap不允许null key & value。它的迭代器是弱一致性的，并非fail-fast，它允许并发遍历数据，但不保证可靠性。

### Base 1.7

- 细粒度的锁 （Segmentation）
- 锁分离（读写分离）
- 锁升级 （优先使用自旋锁，到达一定阈值再使用互斥锁）

ConcurrentHashMap 最外层不是一个大的数组，而是一个 Segment 的数组。每个 Segment 包含一个与 HashMap 数据结构差不多的链表数组。

![](http://www.jasongj.com/img/java/concurrenthashmap/concurrenthashmap_java7.png)

在读写某个 Key 时，先取该 Key 的哈希值。并将哈希值的高 N 位对 Segment 个数取模从而得到该 Key 应该属于哪个Segment，接着如同操作 HashMap 一样操作这个 Segment。

Segment 继承自 ReentrantLock，可以很方便的对每一个 Segment 上锁。

对于读操作，获取 Key 所在的 Segment 时，需要保证可见性。具体实现上可以使用volatile关键字，也可使用锁。但使用锁开销太大，而使用volatile时每次写操作都会让所有CPU内缓存无效，也有一定开销。ConcurrentHashMap 使用如下方法保证可见性，取得最新的Segment：

```java
Segment<K,V> s = (Segment<K,V>)UNSAFE.getObjectVolatile(segments, u)
```

获取 Segment 中的 HashEntry 时也使用了类似方法：

```java
HashEntry<K,V> e = (HashEntry<K,V>) UNSAFE.getObjectVolatile
  (tab, ((long)(((tab.length - 1) & h)) << TSHIFT) + TBASE)
```

这里的 UNSAFE 是 sun.misc.Unsafe 类的一个实例，这个类提供了一些底层操作，比如直接内存访问、线程调度等，它不是标准 Java API 的一部分，但在 JDK 的内部广泛使用。
UNSAFE.getObjectVolatile 方法：这个方法是 Unsafe 类提供的，用于获取给定对象（segments 数组）中某个偏移量（u）处的对象引用，并且保证了操作的内存可见性。这意味着，无论何时一个线程修改了 segments 数组中的某个 Segment 引用，其他线程通过这种方式读取该引用时都能看到最新的值。

这里并没有直接把segments数组声明为volatile的，而是让此次操作具有volatile的语义，这样可以更灵活的控制内存操作，降低性能开销。

对于写操作，并不要求同时获取所有 Segment 的锁，因为那样相当于锁住了整个Map。它会先获取该 Key-Value 对所在的 Segment 的锁，获取成功后就可以像操作一个普通的 HashMap 一样操作该 Segment，并保证该 Segment 的安全性。同时由于其它 Segment 的锁并未被获取，因此理论上可支持 concurrencyLevel（等于Segment的个数）个线程安全的并发读写。

获取锁时，并不直接使用 lock 来获取，因为该方法获取锁失败时会挂起。事实上，它使用了**自旋锁**，如果 tryLock 获取锁失败，说明锁被其它线程占用，此时通过循环再次以 tryLock 的方式申请锁。如果在循环过程中该 Key 所对应的链表头被修改，则重置 retry 次数。如果 retry 次数超过一定值，则使用 lock 方法申请锁。

这里使用自旋锁是因为自旋锁的效率比较高，但是它消耗 CPU 资源比较多，因此在自旋次数超过阈值时切换为互斥锁。

### Base 1.8  
1.7 已经解决了并发问题，并且能支持 N 个 Segment 这么多次数的并发，但依然存在 HashMap 在 1.7 版本中的问题：查询遍历链表效率太低。因此 1.8 做了一些数据结构上的调整。
1.8主要做了两个方面的改进：
- 去除分段锁（Segment）：Java 1.8 移除了分段锁的设计，改用了更细粒度的锁定机制，即直接在节点上进行同步。这减少了锁的竞争，提高了并发性能，尤其是在高并发读写的场景下。
- 引入红黑树：当链表中的节点数超过一定阈值时（默认为 8），链表会转换成红黑树，以提高搜索效率。这减少了在哈希冲突多的情况下的查询时间复杂度，从链表的 O(n) 降低到红黑树的 O(log n)。

![](https://user-gold-cdn.xitu.io/2018/7/23/164c47f3756eb206?imageView2/0/w/1280/h/960/format/webp/ignore-error/1)

其中抛弃了原有的 Segment 分段锁，而采用了 CAS + synchronized 来保证并发安全性。
- 使用 synchronized 关键字替代 Segment 锁：在节点上的同步操作现在使用的是 synchronized 关键字，而不是像以前版本中使用的 ReentrantLock。**这利用了 JVM 对 synchronized 的优化，如偏向锁、轻量级锁等，来减少同步的开销。**
- CAS 操作：Java 1.8 的 ConcurrentHashMap 在更多场合使用了 CAS（比较并交换）操作来实现无锁的更新，进一步提高并发性能。

#### 读写操作的安全性
- 无锁设计：读操作在 ConcurrentHashMap 中是完全无锁的。这得益于使用 volatile 关键字对节点中的变量进行声明，保证了内存可见性。即使在并发环境下，读取操作也能确保获得最新的数据，因为任何写入操作都会立即反映到主内存中，并且所有读操作都会从主内存中获取数据。

- 使用 CAS 和 synchronized：尽管读操作本身不需要锁，但更新操作（写操作）在修改节点时会使用 CAS（比较并交换）操作或者在必要时使用 synchronized 关键字。这确保了即使在进行写操作时，读操作仍然可以安全地执行，因为写操作会保持数据的一致性和可见性。


``ConcurrentHashMap.java``
```java
final V putVal(K key, V value, boolean onlyIfAbsent) {
    if (key == null || value == null) throw new NullPointerException();
    int hash = spread(key.hashCode());
    int binCount = 0;
    for (Node<K,V>[] tab = table;;) {
        Node<K,V> f; int n, i, fh;
        if (tab == null || (n = tab.length) == 0)
            tab = initTable();
        else if ((f = tabAt(tab, i = (n - 1) & hash)) == null) {
            if (casTabAt(tab, i, null,
                            new Node<K,V>(hash, key, value, null)))
                break;                   // no lock when adding to empty bin
        }
        else if ((fh = f.hash) == MOVED)
            tab = helpTransfer(tab, f);
        else {
            V oldVal = null;
            synchronized (f) {
                if (tabAt(tab, i) == f) {
                    if (fh >= 0) {
                        binCount = 1;
                        ···
                    }
                    else if (f instanceof TreeBin) {
                       ···
                    }
                    else if (f instanceof ReservationNode)
                        throw new IllegalStateException("Recursive update");
                }
            }
            ···
    }
    addCount(1L, binCount);
    return null;
}
```

## ArrayList
ArrayList 本质上是一个动态数组，第一次添加元素时，数组大小将变化为 DEFAULT_CAPACITY 10，不断添加元素后，会进行扩容。删除元素时，会按照位置关系把数组元素整体（复制）移动一遍。
它支持随机访问，效率为O(1)，但在数组中间增删元素效率较慢，复杂度为O(n)；

``ArrayList.java``
```java
public class ArrayList<E> extends AbstractList<E>
        implements List<E>, RandomAccess, Cloneable, java.io.Serializable
    ···

    // 增加元素
    public boolean add(E e) {
        ensureCapacityInternal(size + 1);  // Increments modCount!!
        elementData[size++] = e;
        return true;
    }
    ···

    // 删除元素
    public E remove(int index) {
        if (index >= size)
            throw new IndexOutOfBoundsException(outOfBoundsMsg(index));

        modCount++;
        E oldValue = (E) elementData[index];

        int numMoved = size - index - 1;
        if (numMoved > 0)
            System.arraycopy(elementData, index+1, elementData, index,
                             numMoved);
        elementData[--size] = null; // clear to let GC do its work

        return oldValue;
    }
    ···

    // 查找元素
    public E get(int index) {
        if (index >= size)
            throw new IndexOutOfBoundsException(outOfBoundsMsg(index));

        return (E) elementData[index];
    }
    ···
}
```

### 扩容机制
在 Java 中，ArrayList 的扩容机制是通过重新分配一个更大的数组并将旧数组中的元素复制到新数组中来实现的。ArrayList 默认情况下不会自动收缩其大小，但是可以通过调用 trimToSize() 方法来减少存储容量，以匹配列表的当前大小。这意味着如果你想减少 ArrayList 占用的空间，需要显式调用此方法。

当向 ArrayList 添加元素，而其内部数组已满时，ArrayList 会进行扩容操作。默认情况下，ArrayList 的初始容量是 10，但这可以在创建 ArrayList 时通过构造函数来指定。扩容的具体步骤如下：

1. 计算新的容量大小。新容量通常是旧容量的 1.5 倍（即旧容量的 50%增加）。这是通过右移一位（oldCapacity >> 1）来实现的，然后将这个增加的量加到旧容量上。

2. 使用新的容量大小创建一个新的数组。

3. 将旧数组中的元素复制到新数组中。

4. 将 ArrayList 的内部数组引用更新为新数组。


## LinkedList
LinkedList 本质上是一个双向链表的存储结构。
对于元素查询来说，ArrayList 优于 LinkedList，因为 LinkedList 要移动指针。对于新增和删除操作，LinedList 比较占优势，因为 ArrayList 要移动数据。
在链表中间插入或删除元素非常高效，时间复杂度为 O(1)，只要能够直接访问到目标节点（例如，通过迭代器）。但是，寻找特定位置的节点仍然需要 O(n) 的时间复杂度。

``LinkedList.java``
```java
public class LinkedList<E>
    extends AbstractSequentialList<E>
    implements List<E>, Deque<E>, Cloneable, java.io.Serializable
{
    ····

    private static class Node<E> {
        E item;
        Node<E> next;
        Node<E> prev;

        Node(Node<E> prev, E element, Node<E> next) {
            this.item = element;
            this.next = next;
            this.prev = prev;
        }
    }
    ···
    
    // 增加元素
    void linkLast(E e) {
        final Node<E> l = last;
        final Node<E> newNode = new Node<>(l, e, null);
        last = newNode;
        if (l == null)
            first = newNode;
        else
            l.next = newNode;
        size++;
        modCount++;
    }
    ···

    // 删除元素
    E unlink(Node<E> x) {
        final E element = x.item;
        final Node<E> next = x.next;
        final Node<E> prev = x.prev;

        if (prev == null) {
            first = next;
        } else {
            prev.next = next;
            x.prev = null;
        }

        if (next == null) {
            last = prev;
        } else {
            next.prev = prev;
            x.next = null;
        }

        x.item = null;
        size--;
        modCount++;
        return element;
    }
    ···

    // 查找元素
    Node<E> node(int index) {
        // assert isElementIndex(index);

        if (index < (size >> 1)) {
            Node<E> x = first;
            for (int i = 0; i < index; i++)
                x = x.next;
            return x;
        } else {
            Node<E> x = last;
            for (int i = size - 1; i > index; i--)
                x = x.prev;
            return x;
        }
    }
    ···
}
```

## CopyOnWriteArrayList
CopyOnWriteArrayList 是线程安全容器(相对于 ArrayList)，它通过一种称为“写时复制”（Copy-On-Write）的技术来实现线程安全，适用于**读多写少**的并发场景。
它使用对象锁来保证写的线程安全，每次写的时候都创建新的数组，所以存在大量写操作的时候，内存的占用量也会增大。
并发读写时，读操作不需要加锁，因为读取的是旧数组中的数据，所以它具有“弱一致性”。

``CopyOnWriteArrayList.java``
```java
public class CopyOnWriteArrayList<E>
    implements List<E>, RandomAccess, Cloneable, java.io.Serializable {
 
    final transient Object lock = new Object();
    ···

    // 增加元素
    public boolean add(E e) {
        synchronized (lock) {
            Object[] elements = getArray();
            int len = elements.length;
            Object[] newElements = Arrays.copyOf(elements, len + 1);
            newElements[len] = e;
            setArray(newElements);
            return true;
        }
    }
    ···

    // 删除元素
    public E remove(int index) {
        synchronized (lock) {
            Object[] elements = getArray();
            int len = elements.length;
            E oldValue = get(elements, index);
            int numMoved = len - index - 1;
            if (numMoved == 0)
                setArray(Arrays.copyOf(elements, len - 1));
            else {
                Object[] newElements = new Object[len - 1];
                System.arraycopy(elements, 0, newElements, 0, index);
                System.arraycopy(elements, index + 1, newElements, index,
                                 numMoved);
                setArray(newElements);
            }
            return oldValue;
        }
    }
    ···
    
    // 查找元素 不需要锁
    private E get(Object[] a, int index) {
        return (E) a[index];
    }
}
```

# 反射

Java中的反射（Reflection）是一种强大的机制，允许程序在运行时查询、访问和修改其自身结构（类、接口、字段、方法等）的信息。
- 在运行时获取类的信息
- 动态创建对象
- 访问和修改字段
- 调用方法
- 创建动态代理

动态代理非常适合在真正处理业务前后添加逻辑，屏蔽掉细节。比如，在Android中经常要动态判断权限，这种情况很适合使用动态代理：

```java
public class PermissionInvocationHandler implements InvocationHandler {
    private final Object target;

    public PermissionInvocationHandler(Object target) {
        this.target = target;
    }

    @Override
    public Object invoke(Object proxy, Method method, Object[] args) throws Throwable {
        // 在这里实现权限检查逻辑
        if (checkPermission()) {
            // 如果检查通过，则调用目标对象的方法
            return method.invoke(target, args);
        } else {
            // 如果检查不通过，抛出一个异常或进行其他处理
            throw new IllegalAccessException("Permission Denied");
        }
    }

    private boolean checkPermission() {
        // 实现具体的权限检查逻辑
        // 这里返回true表示权限检查通过，仅为示例
        return true;
    }
}

Service service = new ServiceImpl();
Service proxyService = (Service) Proxy.newProxyInstance(
    Service.class.getClassLoader(),
    new Class<?>[]{Service.class},
    new PermissionInvocationHandler(service)
);

proxyService.performAction(); // 调用时会进行权限检查
```

虽然反射提供了极大的灵活性，但它也有一些缺点：

- 性能开销：反射操作的性能比直接的Java代码调用要慢，因为它需要在运行时解析类的元数据。
- 安全问题：反射可以用来访问和修改私有字段和方法，这可能会破坏封装性，导致安全漏洞。
- 难维护：反射代码比直接的Java代码更难理解和维护。

```java
try {
    Class cls = Class.forName("com.jasonwu.Test");
    //获取构造方法
    Constructor[] publicConstructors = cls.getConstructors();
    //获取全部构造方法
    Constructor[] declaredConstructors = cls.getDeclaredConstructors();
    //获取公开方法
    Method[] methods = cls.getMethods();
    //获取全部方法
    Method[] declaredMethods = cls.getDeclaredMethods();
    //获取公开属性
    Field[] publicFields = cls.getFields();
    //获取全部属性
    Field[] declaredFields = cls.getDeclaredFields();
    Object clsObject = cls.newInstance(); (已过时，Java 9开始推荐使用getDeclaredConstructor().newInstance())
    Method method = cls.getDeclaredMethod("getModule1Functionality");
    Object object = method.invoke(null);
} catch (ClassNotFoundException e) {
    e.printStackTrace();
} catch (IllegalAccessException e) {
    e.printStackTrace();
} catch (InstantiationException e) {
    e.printStackTrace();
} catch (NoSuchMethodException e) {
    e.printStackTrace();
} catch (InvocationTargetException e) {
    e.printStackTrace();
}
```

# 单例

## 枚举
枚举可能式最方便、安全、能避免反射、序列化的单例实现方式。
它的缺点是不能传递初始化参数、不能继承，不适合构造复杂的对象。

```java
public enum Singleton {
    INSTANCE;

    public void doSomething() {
        // 执行操作
    }
}
```

## 懒汉式
懒汉式是一种线程安全的单例，但它没办法在构造对象的时候传递参数，并且某种程度上没有实现lazy loading的效果。
之所以说是某种程度上，是因为类加载的实际是在初次使用时，jvm某种程度上也做了'lazy loading'。
```java
public class Singleton {
    // 在类加载时就完成实例化，避免了线程同步问题
    private static final Singleton instance = new Singleton();

    // 私有化构造函数，防止外部实例化
    private Singleton() {}

    // 提供一个公共的静态方法，返回唯一实例
    public static Singleton getInstance() {
        return instance;
    }

    // 类中其他方法，尽量是static
    public void someMethod() {
        // do something
    }
}
```

## 饿汉式
饿汉式实现了lazy loading，能够传递初始化参数，但由于实际上是对getInstance整个方法加锁来保证多线程安全，所以某种程度上效率较低。
之所以说某种程度上，是因为jvm已经对`synchronized`做了很多优化。这些优缺点其实都是对比来讲。
```java
public class CustomManager {
    private CustomManager(){}
    private Context mContext;
    private static final Object mLock = new Object();
    private static CustomManager mInstance;

    public static CustomManager getInstance(Context context) {
        synchronized (mLock) {
            if (mInstance == null) {
                mInstance = new CustomManager(context);
            }

            return mInstance;
        }
    }

    private CustomManager(Context context) {
        this.mContext = context.getApplicationContext();
    }
}
```
## 双重检查模式
Double Check既保证了延迟初始化，也能初始化传参，同时保证了多线程安全性。只是实现相对复杂，注意volatile的作用。
```java
public class CustomManager {
    private CustomManager(){}
    private Context mContext;
    private volatile static CustomManager mInstance;

    public static CustomManager getInstance(Context context) {
        // 避免非必要加锁
        if (mInstance == null) {//这次检查是由volatile保证的可见性
            synchronized (CustomManger.class) {
                if (mInstance == null) {
                    mInstacne = new CustomManager(context);
                }
            }
        }

        return mInstacne;
    }

    private CustomManager(Context context) {
        this.mContext = context.getApplicationContext();
    }
}
```
## 静态内部类模式

```java
public class CustomManager{
    private CustomManager(){}
 
    private static class CustomManagerHolder {
        private static final CustomManager INSTANCE = new CustomManager();
    }
 
    public static CustomManager getInstance() {
        return CustomManagerHolder.INSTANCE;
    } 
}
```

静态内部类的原理是：

当 SingleTon 第一次被加载时，并不需要去加载 SingleTonHoler，只有当 getInstance() 方法第一次被调用时，才会去初始化 INSTANCE，这种方法不仅能确保线程安全，也能保证单例的唯一性，同时也延迟了单例的实例化。getInstance 方法并没有多次去 new 对象，取的都是同一个 INSTANCE 对象。

虚拟机会保证一个类的 ``<clinit>()`` 方法在多线程环境中被正确地加锁、同步，如果多个线程同时去初始化一个类，那么只会有一个线程去执行这个类的 ``<clinit>()`` 方法，其他线程都需要阻塞等待，直到活动线程执行 ``<clinit>()`` 方法完毕

缺点在于无法传递参数，如Context等

它不能避免**反射创建实例**，即使我们在构造函数中进行如下限制，反射也总能破坏这种限制：
```java
public class Singleton {
    // 用于跟踪实例是否已经被创建
    private static boolean instanceCreated = false;

    private Singleton() {
        // 在构造方法中进行检查
        synchronized (Singleton.class) {
            if (instanceCreated) {
                throw new IllegalStateException("Instance already created.");
            }
            instanceCreated = true;
        }
    }

    private static class SingletonHolder {
        private static final Singleton INSTANCE = new Singleton();
    }

    public static Singleton getInstance() {
        return SingletonHolder.INSTANCE;
    }
}

```

如果单例实现了**Serializable**接口，应该重写readResolve()方法避免重新创建对象：

```java
public class Singleton {
    private Singleton() {}

    private static class SingletonHolder {
        private static final Singleton INSTANCE = new Singleton();
    }

    public static Singleton getInstance() {
        return SingletonHolder.INSTANCE;
    }

    // 当实现了Serializable接口时，使用此方法防止反序列化破坏单例
    protected Object readResolve() {
        return getInstance();
    }
}

```

## 枚举的优势
Java语言和虚拟机对枚举类型提供了特殊的支持，包括在序列化和反序列化过程中确保枚举实例的唯一性。这不是通过开发者在代码中添加保护逻辑实现的，而是语言本身的特性，因此更加可靠。
- 反射保护：枚举类型在编译时就已经将其所有的实例定义完毕，在运行时不能通过构造器创建新的枚举实例。尝试这样做会导致`java.lang.reflect.Constructor#newInstance方法抛出IllegalArgumentException`异常。
- 序列化的机制：当一个枚举实例被序列化时，Java仅仅是保存枚举实例的名称。当反序列化时，Java会通过`Enum.valueOf()`方法来返回对应名称的枚举实例。如果这个序列化的对象被修改为一个不存在的枚举名称，那么在反序列化时将会抛出`java.lang.IllegalArgumentException`异常。这保证了即使是经过序列化和反序列化，返回的还是枚举类型中预定义的实例，维持了单例的唯一性。

# 线程
线程是进程中可独立执行的最小单位，也是 CPU 资源（时间片）分配的基本单位。同一个进程中的线程可以共享进程中的资源，如内存空间和文件句柄。

## 属性
| 属性 | 说明 |
|--|--|
| id | 线程 id 用于标识不同的线程。编号可能被后续创建的线程使用。编号是只读属性，不能修改 |
| name | 名字的默认值是 Thread-(id) |
| daemon | 分为**守护线程**和**用户线程**，我们可以通过 setDaemon(true) 把线程设置为守护线程。守护线程通常用于执行不重要的任务，比如监控其他线程的运行情况，GC 线程就是一个守护线程。setDaemon() 要在线程启动前设置，否则 JVM 会抛出非法线程状态异常，可被继承。 |
| priority | 线程调度器会根据这个值来决定优先运行哪个线程（不保证），优先级的取值范围为 1~10，默认值是 5，可被继承。Thread 中定义了下面三个优先级常量：<br>- 最低优先级：MIN_PRIORITY = 1<br>- 默认优先级：NORM_PRIORITY = 5<br>- 最高优先级：MAX_PRIORITY = 10 |

## 状态
![](https://pic2.zhimg.com/80/v2-326a2be9b86b1446d75b6f52f54c98fb_hd.jpg)

| 状态 | 说明 |
|--|--|
| New | 新创建了一个线程对象，但还没有调用start()方法。 |
| Runnable | Ready 状态 线程对象创建后，其他线程(比如 main 线程）调用了该对象的 start() 方法。该状态的线程位于可运行线程池中，等待被线程调度选中 获取 cpu 的使用权。Running 绪状态的线程在获得 CPU 时间片后变为运行中状态（running）。 |
| Blocked | 线程因为某种原因放弃了cpu 使用权（等待锁），暂时停止运行 |
| Waiting | 线程进入等待状态因为以下几个方法：<br>- Object#wait()<br>- Thread#join()<br>- LockSupport#park() |
| Timed Waiting | 有等待时间的等待状态。 |
| Terminated | 表示该线程已经执行完毕。 |

## 状态控制
- wait() / notify() / notifyAll()
  
``wait()``，``notify()``，``notifyAll()`` 是定义在Object类的实例方法，用于控制线程状态，三个方法都必须在synchronized 同步关键字所限定的作用域中调用，否则会报错 ``java.lang.IllegalMonitorStateException``。

| 方法 | 说明 |
|--|--|
| ``wait()`` | 线程释放锁，Running 变为 Waiting, 并将当前线程放入**等待队列**中 |
| ``notify()`` | notify() 方法是将等待队列中一个等待线程从等待队列移动到**同步队列**中 |
| ``notifyAll() `` | 则是将所有等待队列中的线程移动到同步队列中 |

虽然线程会从等待队列移动到同步队列，但其实状态的变化并不是直接从Watting直接变为Blocked，在线程获得锁之前，会首先变为Runnable，表示准备好运行。线程要等待调度器分配CPU时间片。如果在尝试重新获得锁时，锁被其他线程持有，那么该线程才会进入Blocked状态。

被移动的线程状态由 Runnable 变为 Blocked，notifyAll 方法调用后，等待线程依旧不会从 wait() 返回,需要调用 notify() 或者 notifyAll() 的线程**释放掉锁**后，等待线程才有机会从 wait() 返回。之所以是有机会，因为线程还是要等待cpu时间片，并且遵循锁竞争的规则（可重入、公平锁）来决定线程是否能最终获得锁。

- join() / sleep() / yield()
  
在很多情况，主线程创建并启动子线程，如果子线程中需要进行大量的耗时计算，主线程往往早于子线程结束。这时，如果**主线程想等待子线程执行结束**之后再结束，比如子线程处理一个数据，主线程要取得这个数据，就要用 ``join()`` 方法。

``sleep(long)`` 方法在睡眠时不释放对象锁，而 ``join()`` 方法在等待的过程中释放对象锁。

``yield()`` 方法会临时暂停当前正在执行的线程，来让有同样优先级的正在等待的线程有机会执行。如果没有正在等待的线程，或者所有正在等待的线程的优先级都比较低，那么该线程会继续运行。执行了yield方法的线程什么时候会继续运行由线程调度器来决定。

# volatile
volatile是一种轻量级的同步策略。当把变量声明为 volatile 类型后，编译器与运行时都会注意到这个变量是共享的，因此不会将该变量上的操作与其他内存操作一起重排序。volatile 变量不会被缓存在寄存器或者对其他处理器不可见的地方，JVM 保证了每次**读变量都从内存中读，跳过 CPU cache 这一步**，因此在读取 volatile 类型的变量时总会返回最新写入的值。

![](https://user-gold-cdn.xitu.io/2019/6/23/16b833f4a48b216e?w=550&h=429&f=png&s=21448)

当一个变量定义为 volatile 之后，将具备以下特性：
- 保证此变量对所有的线程的可见性，（可见性，是指线程之间的可见性，一个线程修改能被另一个线程立即看到）
- 禁止指令重排序优化，使得操作的顺序对其他线程是可预测的
- 非原子性的，如自增操作i++，它其实是一个复合操作(读取-修改-写入)，volatile不能保证这一系列的操作是作为一个整体原子性执行的。
- volatile 的**读性能消耗与普通变量几乎相同，但是写操作稍慢**，因为它需要在本地代码中插入许多内存屏障指令来保证处理器不发生乱序执行。

## 内存屏障
volatile正是在变量读写前后插入内存屏障，来保证可见性和有序性的。

内存屏障（Memory Barrier），又称为内存栅栏，是一种同步机制，用来确保指令执行的顺序性以及变量更新的可见性。内存屏障的主要作用是：
- 防止指令重排序：编译器和处理器为了优化指令执行效率，可能会对指令序列进行重排序。内存屏障可以禁止这种重排序，**确保在屏障之前的操作必须在屏障之后的操作之前完成**。
- 确保变量更新的可见性：通过在代码中插入适当的内存屏障，可以**确保在屏障之前对变量的修改在屏障之后对其他线程可见**，这对于实现多线程之间的同步非常重要。

# CAS

CAS（Compare-And-Swap）是一种低级同步机制，用于在并发编程中实现原子性操作。它是一种**系统原语**，通常直接由硬件支持。在 Java 中，sun.misc.Unsafe 类提供了 CAS 相关的方法。

CAS 操作涉及三个主要操作数：内存位置（V），预期原值（A），和新值（B）。如果内存位置的当前值与预期原值相匹配，那么处理器会自动将该位置值更新为新值。

## 应用场景
- 自定义同步工具：开发者可以直接使用 Unsafe 类（或在支持 CAS 的平台上使用相应机制）来构建自定义的同步工具或数据结构，比如无锁的数据结构。
- 并发算法实现：在一些高级的并发算法中，直接使用 CAS 可以提供更细粒度的控制，比如在实现一个高性能的并发队列或特定的锁算法时。
- 操作系统和编译器优化：在更底层，CAS 不仅仅限于 Java 或高级编程语言，它也被操作系统和编译器用于实现线程或进程同步，以及进行各种并发控制优化。

## Atomic

Java 的 Atomic 类通过封装 Unsafe 类中的 CAS 方法，提供了一套更高级、易用的 API，使得开发者可以在 Java 应用中方便地执行无锁的线程安全操作。volatile 修饰符保证了 value 在内存中其他线程可以看到其值的改变。CAS（Compare and Swap）操作保证了 AtomicInteger 可以安全的修改value 的值。

```java
//伪代码 其实是本地代码的逻辑
public final int incrementAndGet() {
    for (;;) {
        int current = get();//这里的值是通过volatile修饰的，保证读取最新的值
        int next = current + 1;
        //如果 CAS 操作失败，方法会再次从第一步开始重试，获取最新的当前值，然后尝试用新计算的下一个值进行更新，直到 CAS 操作成功为止
        if (compareAndSet(current, next))
        //compareAndSet 方法的实现依赖于底层硬件的支持，通常是通过一组原子指令，如 x86 架构的 CMPXCHG 指令。这些指令能够保证比较和设置值的操作是原子的，即不可被中断的。
            return next;
    }
}
```

# Monitor
Monitor对象存在于每个Java对象的对象头Mark Word中（指针），这就是Java中为什么任意对象可以作为锁的原因。Java对象中的wait/notify/notifyAll方法会使用到Monitor对象，所以必须在同步代码块中调用这些方法。

## 对象头的数据结构
一个Java对象的数据结构可分为三部分：对象头、实例数据、对齐填充，对象头通常由两部分构成：Mark Word和Class Pointer。
- Mark Word: 对象头中非常灵活的部分，内容根据对象状态和信息的不同而变化。通常包含：
    - hashCode：对象哈希码
    - 锁信息：锁状态信息，比如锁定状态、锁定的线程、锁的类型
    - GC分代年龄: 垃圾收集过程中，用于记录对象的年龄
    - 偏向线程ID: 使用偏向锁时，存储持有偏向锁的线程ID
    - 轻量级/重量级锁指针：如果对象被锁定，存储锁的指针
- Class Pointer: 指向对象所属类的元数据的指针，便于JVM快速访问对象的类型信息

## 锁信息的记录
对象头中的Mark Word部分是用来记录锁信息的主要区域。HotSpot JVM使用了一系列的锁优化技术，包括偏向锁、轻量级锁和重量级锁，这些锁状态都通过Mark Word中的位模式来表示。

- 无锁（Biased Locking未启用时）：Mark Word存储对象的哈希码、GC分代年龄等信息。
- 偏向锁：启用偏向锁后，Mark Word会记录偏向的线程ID，以及一些必要的锁信息和对象年龄。
- 轻量级锁：线程尝试获取锁时，会在自己的栈帧中创建一个锁记录（Lock Record），然后尝试使用CAS操作将对象头的Mark Word替换为指向锁记录的指针。
- 重量级锁：如果轻量级锁的CAS操作失败，表示有竞争，锁会膨胀为重量级锁。此时，Mark Word会被替换为指向一个监视器（Monitor）的指针，该监视器中包含锁的所有信息。

## ObjectMonitor的数据结构
ObjectMonitor 是 HotSpot JVM 中实现监视器（Monitor）功能的一个关键的 C++ 类。它用于实现 Java 对象的同步机制，比如在 synchronized 方法或 synchronized 块中使用。
- _owner：指向持有当前监视器锁的线程。如果监视器是空闲的，则此字段为 NULL。
- _WaitSet：等待集合，包含了所有调用了该对象的 wait() 方法并正在等待的线程。
- _EntryList：入口列表（或称为等待锁的线程列表），包含了试图获取这个监视器锁但未能成功的线程。
- _recursions：递归锁计数，表示重入次数。如果一个线程已经获得了锁，再次进入同步块时，递归计数会增加。
- _object：指向 Java 对象的指针，即该监视器所关联的对象。
- _count 和 _waiters：这些字段可能用于跟踪等待条件的线程数量以及监视器上的总等待线程数量。

# synchronized
当它用来修饰一个方法或者一个代码块的时候，能够保证在同一时刻最多只有一个线程执行该段代码。

## 根据获取的锁分类
**获取对象锁**
- synchronized(this|object) {}  
- 修饰非静态方法  

**获取类锁**
- synchronized(类.class) {}  
- 修饰静态方法

## 原理
**同步代码块：**
- monitorenter 和 monitorexit 指令实现的。
这两个指令的执行是JVM通过调用操作系统的互斥原语mutex来实现的，但是呢，随着 JVM 技术的发展，为了提高性能和减少同步开销，JVM 实现了多种锁优化策略：例如，轻量级锁和偏向锁都是试图在没有竞争的情况下避免昂贵的操作系统互斥原语调用。只有在竞争条件下，这些优化锁才会升级为重量级锁，可能会涉及到操作系统级的互斥原语。

Java 中的锁是可重入的，这意味着同一个线程可以多次获取同一个锁，而不会导致死锁。所以有计数器的概念，线程竞争到锁进入当前对象的同步代码块后，计数+1，退出同步代码块后，计数器-1；当计数器为0时，释放锁，同步队列的线程被唤醒竞争锁。

**同步方法**
- 方法修饰符上的 ACC_SYNCHRONIZED 实现
当调用方法时，调用指令会先检查ACC_SYNCHRONIZED标志是否被设置，如果被设置了，需要先竞争锁。本质上是和同步代码块的原理是相同的，只是方法的同步是一种隐式的方式来实现，无需通过字节码来完成。

# Lock
Java中的Lock对象是并发编程中用于管理对共享资源的访问的一种机制。Lock提供了比synchronized关键字更复杂的锁定操作，它允许更灵活的结构，可以有不同的属性（如可重入性、公平性），并且可以显式地获取和释放锁。
特点：
 - 可重入性
 - 中断能力
 - 尝试获取锁
 - 定时锁
 - 公平锁

```java
public interface Lock {
    void lock();
    void lockInterruptibly() throws InterruptedException;  
    boolean tryLock();  
    boolean tryLock(long time, TimeUnit unit) throws InterruptedException;  
    void unlock();  
    Condition newCondition();
}
```

使用模板：
```java
import java.util.concurrent.locks.Lock;
import java.util.concurrent.locks.ReentrantLock;

public class LockExample {
    private final Lock lock = new ReentrantLock();
    private int count = 0;

    public void increment() {
        lock.lock();
        try {
            count++;
        } finally {
            //确保释放
            lock.unlock();
        }
    }
}
```

| 方法 | 说明 |
|--|--|
| ``lock()`` | 用来获取锁，如果锁被其他线程获取，处于等待状态。如果采用 Lock，必须主动去释放锁，并且在发生异常时，不会自动释放锁。因此一般来说，使用Lock必须在 try{}catch{} 块中进行，并且将释放锁的操作放在finally块中进行，以保证锁一定被被释放，防止死锁的发生。 |
| ``lockInterruptibly()`` | 通过这个方法去获取锁时，如果线程正在等待获取锁，则这个线程能够响应中断，即中断线程的等待状态。 |
| ``tryLock()`` | tryLock 方法是有返回值的，它表示用来尝试获取锁，如果获取成功，则返回 true，如果获取失败（即锁已被其他线程获取），则返回 false，也就说这个方法无论如何都会立即返回。在拿不到锁时不会一直在那等待。 |
| ``tryLock(long，TimeUnit)`` | 与 tryLock 类似，只不过是有等待时间，在等待时间内获取到锁返回 true，超时返回 false。 |

常见实现：
- ReentrantLock
- ReadWriteLock
- ReentrantReadWriteLock

## 锁的分类
![](https://user-gold-cdn.xitu.io/2019/6/18/16b69b50c9d340a5?w=1372&h=1206&f=png&s=142754)

### 悲观锁、乐观锁
悲观锁认为自己在使用数据的时候一定有别的线程来修改数据，因此在获取数据的时候会先加锁，确保数据不会被别的线程修改。Java 中，synchronized 关键字和 Lock 的实现类都是悲观锁。**悲观锁适合写操作多的场景**，先加锁可以保证写操作时数据正确。

而乐观锁认为自己在使用数据时不会有别的线程修改数据，所以不会添加锁，只是在更新数据的时候去判断之前有没有别的线程更新了这个数据。如果这个数据没有被更新，当前线程将自己修改的数据成功写入。如果数据已经被其他线程更新，则根据不同的实现方式执行不同的操作（例如报错或者自动重试）。乐观锁在 Java 中是通过使用无锁编程来实现，最常采用的是 CAS 算法，Java 原子类中的递增操作就通过 CAS 自旋实现。**乐观锁适合读操作多的场景**，不加锁的特点能够使其读操作的性能大幅提升。

### 自旋锁、适应性自旋锁
阻塞或唤醒一个 Java 线程需要操作系统切换 CPU 状态来完成，这种状态转换需要耗费处理器时间。如果同步代码块中的内容过于简单，状态转换消耗的时间有可能比用户代码执行的时间还要长。

在许多场景中，同步资源的锁定时间很短，为了这一小段时间去切换线程，线程挂起和恢复现场的花费可能会让系统得不偿失。如果物理机器有多个处理器，能够让两个或以上的线程同时并行执行，我们就可以让后面那个请求锁的线程不放弃CPU的执行时间，看看持有锁的线程是否很快就会释放锁。

而为了让当前线程“稍等一下”，我们需让当前线程进行自旋，如果在自旋完成后前面锁定同步资源的线程已经释放了锁，那么当前线程就可以不必阻塞而是直接获取同步资源，从而避免切换线程的开销。这就是自旋锁。

自旋锁本身是有缺点的，它不能代替阻塞。自旋等待虽然避免了线程切换的开销，但它要占用处理器时间。如果锁被占用的时间很短，自旋等待的效果就会非常好。反之，如果锁被占用的时间很长，那么自旋的线程只会白浪费处理器资源。所以，自旋等待的时间必须要有一定的限度，如果自旋超过了限定次数（默认是 10 次，可以使用 -XX:PreBlockSpin 来更改）没有成功获得锁，就应当挂起线程。

自旋锁的实现原理同样也是 CAS，AtomicInteger 中调用 unsafe 进行自增操作的源码中的 do-while 循环就是一个自旋操作，如果修改数值失败则通过循环来执行自旋，直至修改成功。

### 死锁
当前线程拥有其他线程需要的资源，当前线程等待其他线程已拥有的资源，都不放弃自己拥有的资源。
死锁的四个必要条件：
- 互斥条件：资源被一个线程占用，别的线程不能使用。
- 非抢占：资源请求者不能强制从占用者中夺取资源，资源只能由占用者主动释放。
- 保持并等待：资源请求者请求其他资源的时候保持对原有资源的占有。
- 循环等待：存在一个等待队列，比如，P1占有P2的资源，P2占有P3的资源，P3占有P1的资源。这样就形成了一个等待环路。

死锁检测：
- Jstack命令
- JConsole工具

死锁避免：
- 按照顺序获取锁
- 超时放弃

其他形式的死锁：
- 单线程池中，任务A等待任务B执行完成
- 网络连接池死锁

# 引用类型
强引用 > 软引用 > 弱引用 

| 引用类型 | 说明 |
|------|------|
| StrongReferenc（强引用）| 当一个对象具有强引用，那么垃圾回收器是绝对不会的回收和销毁它的，**非静态内部类会在其整个生命周期中持有对它外部类的强引用**|
| SoftReference（软引用）| 如果一个对象只具有软引用，若内存空间足够，垃圾回收器就不会回收它；如果内存空间不足了，才会回收这些对象的内存|
| WeakReference （弱引用）| 在垃圾回收器运行的时候，如果对一个对象的所有引用**都是弱引用**的话，该对象会被回收 |
| PhantomReference（虚引用） | 一个只被虚引用持有的对象可能会在任何时候被 GC 回收。虚引用对对象的生存周期完全没有影响，也**无法通过虚引用来获取对象实例**，当垃圾回收器准备回收一个对象，如果这个对象注册了虚引用，垃圾回收器会在回收对象内存之前，把这个虚引用加入到一个引用队列（ReferenceQueue）中。|

# 动态代理
动态代理的处理过程大概有以下几步：
1. 检查Proxy.newProxyInstance的入参
2. 生成代理类的二进制字节码，如java中的ProxyGenerator.generateProxyClass或者android中的native方法 generateProxy
3. 加载并定义代理类，这是通过ClassLoader的defineClass方法完成的，它将字节码转换为Class对象
4. 创建代理实例，newInstance，这个构造函数接收一个InvocationHandler作为参数，实例中所有方法的调用，其实调用的是InvocationHandler的invoke方法
5. 返回代理实例

示例：

```java
// 定义相关接口
public interface BaseInterface {
    void doSomething();
}

// 接口的相关实现类
public class BaseImpl implements BaseInterface {
    @Override
    public void doSomething() {
        System.out.println("doSomething");
    }
}

public static void main(String args[]) {
    BaseImpl base = new BaseImpl();
    // Proxy 动态代理实现
    BaseInterface proxyInstance = (BaseInterface) Proxy.newProxyInstance(base.getClass().getClassLoader(), base.getClass().getInterfaces(), new InvocationHandler() {
        @Override
        public Object invoke(Object proxy, Method method, Object[] args) throws Throwable {
            if (method.getName().equals("doSomething")) {
                method.invoke(base, args);
                System.out.println("do more");
            }
            return null;
        }
    });

    proxyInstance.doSomething();
}
```

``Proxy.java``
```java
public class Proxy implements java.io.Serializable {

    // 代理类的缓存
    private static final WeakCache<ClassLoader, Class<?>[], Class<?>>
        proxyClassCache = new WeakCache<>(new KeyFactory(), new ProxyClassFactory());
    ···

    // 生成代理对象方法入口
    public static Object newProxyInstance(ClassLoader loader,
                                        Class<?>[] interfaces,
                                        InvocationHandler h)
    throws IllegalArgumentException
    {
        Objects.requireNonNull(h);

        final Class<?>[] intfs = interfaces.clone();
        // 找到并生成相关的代理类
        Class<?> cl = getProxyClass0(loader, intfs);

        // 调用代理类的构造方法生成代理类实例
        try {
            final Constructor<?> cons = cl.getConstructor(constructorParams);
            final InvocationHandler ih = h;
            if (!Modifier.isPublic(cl.getModifiers())) {
                cons.setAccessible(true);
            }
            return cons.newInstance(new Object[]{h});
        } 
        ···
    }
    ···
    
    // 定义和返回代理类的工厂类
    private static final class ProxyClassFactory
        implements BiFunction<ClassLoader, Class<?>[], Class<?>>
    {
        // 所有代理类的前缀
        private static final String proxyClassNamePrefix = "$Proxy";

        //  用于生成唯一代理类名称的下一个数字
        private static final AtomicLong nextUniqueNumber = new AtomicLong();

        @Override
        public Class<?> apply(ClassLoader loader, Class<?>[] interfaces) {

            Map<Class<?>, Boolean> interfaceSet = new IdentityHashMap<>(interfaces.length);
            ···

            String proxyPkg = null;     // 用于定义代理类的包名
            int accessFlags = Modifier.PUBLIC | Modifier.FINAL;

            // 确保所有 non-public 的代理接口在相同的包里
            for (Class<?> intf : interfaces) {
                int flags = intf.getModifiers();
                if (!Modifier.isPublic(flags)) {
                    accessFlags = Modifier.FINAL;
                    String name = intf.getName();
                    int n = name.lastIndexOf('.');
                    String pkg = ((n == -1) ? "" : name.substring(0, n + 1));
                    if (proxyPkg == null) {
                        proxyPkg = pkg;
                    } else if (!pkg.equals(proxyPkg)) {
                        throw new IllegalArgumentException(
                            "non-public interfaces from different packages");
                    }
                }
            }

            if (proxyPkg == null) {
                // 如果没有 non-public 的代理接口，使用默认的包名
                proxyPkg = "";
            }

            {
                List<Method> methods = getMethods(interfaces);
                Collections.sort(methods, ORDER_BY_SIGNATURE_AND_SUBTYPE);
                validateReturnTypes(methods);
                List<Class<?>[]> exceptions = deduplicateAndGetExceptions(methods);

                Method[] methodsArray = methods.toArray(new Method[methods.size()]);
                Class<?>[][] exceptionsArray = exceptions.toArray(new Class<?>[exceptions.size()][]);

                // 生成代理类的名称
                long num = nextUniqueNumber.getAndIncrement();
                String proxyName = proxyPkg + proxyClassNamePrefix + num;

                // Android 特定修改：直接调用 native 方法生成代理类
                return generateProxy(proxyName, interfaces, loader, methodsArray,
                                     exceptionsArray);

                // JDK 使用的 ProxyGenerator.generateProxyClas 方法创建代理类
                byte[] proxyClassFile = ProxyGenerator.generateProxyClass(
                    proxyName, interfaces, accessFlags);
                try {
                    return defineClass0(loader, proxyName,
                                        proxyClassFile, 0, proxyClassFile.length);
                } ···
        }
    }
    ···

    // 最终调用 native 方法生成代理类
    @FastNative
    private static native Class<?> generateProxy(String name, Class<?>[] interfaces,
                                                 ClassLoader loader, Method[] methods,
                                                 Class<?>[][] exceptions);

}
```

``ProxyGenerator.java``
```java
public static byte[] generateProxyClass(final String name,
                                        Class[] interfaces)
{
    ProxyGenerator gen = new ProxyGenerator(name, interfaces);
    final byte[] classFile = gen.generateClassFile();

    if (saveGeneratedFiles) {
        java.security.AccessController.doPrivileged(
        new java.security.PrivilegedAction<Void>() {
            public Void run() {
                try {
                    FileOutputStream file =
                        new FileOutputStream(dotToSlash(name) + ".class");
                    file.write(classFile);
                    file.close();
                    return null;
                } catch (IOException e) {
                    throw new InternalError(
                        "I/O exception saving generated file: " + e);
                }
            }
        });
    }

    return classFile;
}
```

# 元注解
@Retention：保留的范围，可选值有三种。

| RetentionPolicy | 说明 |
|----|----|
| SOURCE | 注解将被编译器丢弃（该类型的注解信息只会保留在源码里，源码经过编译后，注解信息会被丢弃，不会保留在编译好的class文件里），如 @Override |
| CLASS | 注解在class文件中可用，但会被 VM 丢弃（该类型的注解信息会保留在源码里和 class 文件里，在执行的时候，不会加载到虚拟机中），请注意，当注解未定义 Retention 值时，默认值是 CLASS。 |
| RUNTIME | 注解信息将在运行期 (JVM) 也保留，因此可以通过反射机制读取注解的信息（源码、class 文件和执行的时候都有注解的信息），如 @Deprecated |

@Target：可以用来修饰哪些程序元素，如 TYPE, METHOD, CONSTRUCTOR, FIELD, PARAMETER等，未标注则表示可修饰所有

@Inherited：是否可以被继承，默认为false  

@Documented：是否会保存到 Javadoc 文档中

# 对象逸出
对象逸出是指没有在对象初始化完毕之前，就暴露了对象的引用，导致不能正确访问到对象的正确状态。
比如，在构造函数中启动子线程，在子线程中访问外部对象时，对象构造函数可能还没有初始化完毕。
避免对象逸出要做到：
- 避免在构造函数中启动线程
- 最小化构造函数的逻辑
- 使用工厂方法或者私有构造函数
