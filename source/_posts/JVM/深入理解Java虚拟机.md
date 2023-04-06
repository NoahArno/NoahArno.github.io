---
title: 深入理解Java虚拟机
category_bar: true
date: 2023-04-06 23:29:30
tags: [JVM]
categories:
  - JVM
---

# 第一章 类加载子系统

![image-20220504183302402](深入理解Java虚拟机/image-20220504183302402.png)

1. Class Loader SubSystem（类加载子系统）负责从文件系统或网络中加载Class Files，Class Files在文件开头有特定的文件标识
2. ClassLoader只负责class文件的加载，至于它是否可以允许，则有Execution Engine（执行引擎）决定。（可以理解为媒婆负责介绍对象，能不能成就得看自己的努力了）
3. 加载的类信息存放于一块称为方法区（Method Area）的内存空间。除了类的信息外，方法区（Method Area）中还会存放运行时常量池信息，可能还包括字符串字面量和数字常量（这部分信息是Class文件中常量池部分的内存映射）

## 什么是ClassLoader

![image-20220504183404624](深入理解Java虚拟机/image-20220504183404624.png)

## 类的加载过程

![image-20220504183421452](深入理解Java虚拟机/image-20220504183421452.png)

### 加载

1. 通过一个类的全限定名获取定义此类的二进制字节流
2. 将这个字节流所代表的静态存储结构转换为方法区的运行时数据结构
3. **在内存中生成一个代表这个类的java.lang.Class文件**，作为方法区这个类的各种数据的访问入口

### 链接

**1、验证**

- 目的在于确保Class文件的字节流中包含信息符合当前虚拟机要求，保证被加载类的正确性，不会危害虚拟机自身安全
- 主要包括四种验证：文件格式验证、元数据验证、字节码验证、符号引用验证

**2、准备**

- 为类变量分配内存并且设置该类变量的默认初始值，即零值（引用类型为null、布尔值为false等）
- **这里不包含用final修饰的static，因为final在编译的时候就会分配了，准备阶段会显示初始化**
- **这里不会为实例变量分配初始化**（因为还没有初始化），类变量会分配在方法区中，而实例变量是会随着对象一起分配到Java堆中

**3、解析**

- 将常量池内的符号引用转换为直接引用的过程
- 事实上，解析操作往往会伴随着JVM在执行完初始化之后再执行
- 符号引用就是一组符号来描述所引用的目标。符号引用的字面量形式明确定义在《java虚拟机规范》的Class文件格式中，直接引用和就是直接指向目标的指针、相对偏移量或一个间接定位到目标的句柄
- 解析动作主要针对类或接口、字段、类方法、接口方法、方法类型等。对应常量池中的CONSTANT_Class_info、CONSTANT_Fieldref_info、CONSTANT_Methodref_info等。
- 符号引用就是一个字符串，它给出了被引用的内容的名字并且可能会包含一些其他关于这个被引用项的信息--这些信息必须足以唯一的识别一个类、字段、方法。这样对于其他类的符号引用必须给出类的全名。对于其他类的字段必须给出类名、字段名以及字段描述符。对于其他类的方法的引用必须给出类名、方法名以及方法的描述符。这样我们就能根据符号引用锁定唯一的类，方法或者字段了。
- 符号引用主要包含下面三类常量：类和接口的全限定名、字段的名称和描述符、方法的名称和描述符

### 初始化

- 初始化阶段就是执行类构造器方法< clinit>()的过程。

- 这个方法不需要定义，是javac编译器自动收集类中的所有类变量的赋值动作和静态代码块中的语句合并而来。也就是说如果一个类没有类变量和静态代码块，那么该类在初始化的时候是不会有< Clinit>()方法的。

- 构造器方法中指定按语句在源文件中出现的顺序执行

  ```java
  public class ClassInitTest {
      private static int num = 1;
  
      static{
          num = 2;
          number = 20;
          System.out.println(num);
          //System.out.println(number);//报错：非法的前向引用。可以赋值，但是不能调用
      }
  
      private static int number = 10;  
      //linking之prepare: number = 0 --> initial: 20 --> 10
  
      public static void main(String[] args) {
          System.out.println(ClassInitTest.num);//2
          System.out.println(ClassInitTest.number);//10
      }
  }
  ```

- < clinit>()不同于类的构造器。（构造器是虚拟机视角下的< init>()）

- 若该类具有父类，JVM会保证子类的< clinit>()执行前，父类的< clinit>()已经执行完毕。

  ```java
  public class ClinitTest1 {
      static class Father{
          public static int A = 1;
          static{
              A = 2;
          }
      }
  
      static class Son extends Father{
          public static int B = A;
      }
  
      public static void main(String[] args) {
          //加载Father类，其次加载Son类。
          System.out.println(Son.B);//2
      }
  }
  ```

  要加载Son类的时候，首先去初始化Father类，初始化的时候会自动的执行类中的类变量和静态代码块，因此B的值为2

- 虚拟机必须保证一个类的< clinit>()方法在多线程下被同步加锁。

  ```java
  public class DeadThreadTest {
      public static void main(String[] args) {
          Runnable r = () -> {
              System.out.println(Thread.currentThread().getName() + "开始");
              DeadThread dead = new DeadThread();
              System.out.println(Thread.currentThread().getName() + "结束");
          };
  
          Thread t1 = new Thread(r,"线程1");
          Thread t2 = new Thread(r,"线程2");
  
          t1.start();
          t2.start();
      }
  }
  
  class DeadThread{
      static{
          if(true){
              System.out.println(Thread.currentThread().getName() + "初始化当前类");
              while(true){
  
              }
          }
      }
  }
  ```

  在对DeadThread进行初始化的时候，由于静态代码块里面是死循环，因此会一直初始化。此时另一个线程也要初始化DeadThread，但是由于第一个DeadThread并未完成初始化，因此它无法初始化成功。

## 类加载器的分类

- JVM支持两种类型的类加载器，分别为引导类加载器（Bootstrap ClassLoader）和自定义类加载器（User-Defined ClassLoader）。
- 从概念上来讲，自定义类加载器一般指的是程序中由开发人员自定义的一类加载器，但是Java虚拟机规范却没有这么定义，而是将**所有派生于抽象类ClassLoader的类加载器都划分为自定义加载器**。
- 无论类加载器的类型如何分类，我们在程序中最常见的类加载器始终只有三个

![image-20220504184253091](深入理解Java虚拟机/image-20220504184253091.png)

也就是说，JVM将Bootstarp Class Loader 分为一类，将扩展类加载器（Extension ClassLoader）、系统类加载器（SystemClassLoader）、还有其余的用户自定义的类加载器同一称为自定义类加载器

**1、BootstrapClassLoader**

- 这个类加载使用C/C++语言实现的，嵌套在JVM内部
- 它用来加载Java的核心库（JAVA_HOME/jre/lib/rt.jar、resources.jar或sun.boot.class.path路径下的内容），用于提供JVM自身所需要的类
- 并不继承自java.lang.ClassLoader，因此没有父加载器
- 加载扩展类和应用程序类加载器，并指定为他们的父类加载器
- 出于安全考虑，**Bootstrap启动类加载器只加载包名为java、javax、sun等开头的类**，比如java.lang.String。

**2、ExtensionClassLoader**

- java语言编写，由sun.misc.Launcher$ExtClassLoader实现。
- 派生于ClassLoader类
- 父类加载器为启动类加载器
- 从java.ext.dirs系统属性所指定的目录中加载类库，或从JDK的安装目录的jre/lib/ext子目录（扩展目录）下加载类库。如果用户创建的JAR放在此目录下，也会自动由扩展类加载器加载

**3、AppClassLoader**

- java语言编写，由sun.misc.Launcher&AppClassLoader实现
- 派生于ClassLoader类
- 父类加载器为扩展类加载器
- 它负责加载环境变量classpath或系统属性java.class.path指定路径下的类库
- **该类加载器是程序中默认的类加载器**，一般来说，java应用的类都是由它来完成加载
- 通过ClassLoader#getSystemClassLoader()方法可以获取到该类加载器

**4、用户自定义类加载器**

- 为什么要自定义类加载器？
  - 隔离加载类
  - 修改类加载的方式
  - 扩展加载源
  - 防止源码泄露
- 自定义类加载器实现步骤
  1. 开发人员可以通过继承抽象类java.lang.ClassLoader类的方式，实现自己的类加载器，以满足一些特殊的需求
  2. 在JDK1.2之前，在自定义类加载器时，总会去继承ClassLoader类并重写loadClass方法，从而实现自定义的类加载器，但在JDK1.2之后，已不再建议用户去覆盖loadClass方法，而是建议把自定义的类加载逻辑写在findClass方法中
  3. 在编写自定义类加载器时，如果没有太过于复杂的需求，可以直接继承URLClassLoader类，这样就可以避免自己去编写findClass方法级其获取字节码流的方式，使自定义类加载器编写更加简洁。

## ClassLoader的使用说明

ClassLoader类是一个抽象类，其后所有的类加载器都继承自ClassLoader，不包括BootstrapClassLoader

![image-20210821122951746](深入理解Java虚拟机/image-20210821122951746.png)

**获取ClassLoader的途径**

![tupu](深入理解Java虚拟机/image-20210821123048947.png)

## 双亲委派机制

java虚拟机对class文件采用的是**按需加载**方式，也就是说当需要使用该类时才会将它的class文件加载到内存生成class对象。而且加载某个类的class文件时，java虚拟机采用的是双亲委派模式，**即把请求交由父类处理**，它是一种任务委派模式

**工作原理**

1. 如果一个类加载器收到了类加载请求，它并不会自己先去加载，而是把这个请求委托给父类的加载器去加载
2. 如果父类加载器还存在其父类加载器，则进一步向上委托，依次递归请求，最终到达顶层的启动类加载器；
3. 如果父类加载器可以完成类加载任务就成功返回，倘若父类加载器无法完成此加载任务，子加载器才会尝试自己去加载，这就是双亲委派模式

**优势**

- 避免类的重复加载

- 保护程序安全，防止核心API被随意串改

  - 自定义类：java.lang.String
  - 自定义类：java.lang.MyString

  这两个类都无法被加载，就是因为双亲委派机制

**沙箱安全机制**

自定义String类，但是在加载自定义String类时会率先使用引导类加载器加载，而引导类加载器在加载过程中会加载jdk自带的文件（rt.jar包中的java\lang\String.class），报错信息说没有main方法，就是因为加载的是rt.jar包中的String类，这样可以保证对java核心源代码的保护，这就是沙箱安全机制

测试类如下：

```java
package com.atguigu.java1;

public class StringTest {

    public static void main(String[] args) {
        java.lang.String str = new java.lang.String();
        System.out.println("hello,atguigu.com");
    }
}
```

自定义java.lang包下的String类：

```java
package java.lang;

public class String {
    static{
        System.out.println("我是自定义的String类的静态代码块");
    }
    //错误: 在类 java.lang.String 中找不到 main 方法
    public static void main(String[] args) {
        System.out.println("hello,String");
    }
}
```

运行StringTest的main方法，只会打印hello,atguigu.com ，不会打印自定义的String类中的静态代码块中的内容，说明我们自定义的String类没有进行初始化

运行自定义String类中的main方法，会报错：

![image-20210821124341862](深入理解Java虚拟机/image-20210821124341862.png)

这就是上面说的沙箱安全机制

## 其他

- 在JVM中表示两个class对象是否为同一类存在两个必要条件：
  - 类的完整包名必须一致，包括包名
  - 加载这个类的ClassLoader（指ClassLaoder实例对象）必须相同
- 换句话说，在JVM中，即使这两个类对象来源同一个Class文件，被同一个虚拟机所加载，只要加载它们的ClassLoader实例对象不是同一个，那么这两个类对象也不是相等的

比如说我们自定义的String类和系统的String类，都是定义在java.lang下的，但是他们两个的ClassLoader不一样，因此就不是同一个类

JVM必须知道一个类型是由启动加载器加载的还是由用户类加载器加载的。如果一个类型是由用户类加载器加载的，那么JVM会**将这个类加载器的一个引用作为类型信息的一部分保存在方法区中**。当解析一个类型到另一个类型的引用时，JVM需要保证这两个类型的类加载器是相同的

**Java程序对类的使用方式：类的主动使用和被动使用**

**主动使用**，有7种情况：

- 创建类的实例
- 访问某个类的接口或静态变量，或者对该静态变量赋值
- 调用类的静态方法
- 反射（如Class.forName（“com.tomasCarry.Test”））
- 初始化一个类的子类
- Java虛拟机启动时被标明为启动类的类
- JDK 7开始提供的动态语言支持：java.lang.invoke.MethodHandle实例的解析结果 REF_getStatic、REF_putStatic、REF_invokeStatic句柄对应的类没有初始化，则初始化

除了以上7种情况，其他的使用都是**被动使用**，都不会导致类的初始化

# 第二章 运行时数据区

内存是非常重要的资源，是硬盘和CPU的中间仓库及桥梁，承载着操作系统和应用程序的实时运行。JVM内存布局规定了Java在运行过程种内存申请、分配、管理的策略，保证了JVM的高效稳定运行。**不同的JVM对于内存的划分方式和管理机制存在部分差异**。

![image-20210821190828499](深入理解Java虚拟机/image-20210821190828499.png)

在Java8的时候，方法区变成了元空间。

Java虚拟机定义了若干种程序运行期间会使用到的运行时数据区，其中有一些会随着虚拟机启动而创建，随着虚拟机退出而销毁。另外一些则是与线程一一对应的，这些与线程对应的数据区域会随着线程开始和结束创建和销毁

灰色的为单独线程私有的，红色的为多个线程共享的。

- 每个线程：独立包括程序计数器（ProgramCounterRegister）、本地方法栈（NativeMethodStack）、虚拟机栈（VirtualMachineStack）
- 线程间共享：堆（Heap）、堆外内存（永久代或元空间、代码缓存）

![IMG/03、运行时数据区概述及线程.assets/第03章_线程共享和私有的结构.jpg](深入理解Java虚拟机/第03章_线程共享和私有的结构.jpg)

每个JVM只有一个Runtime实例。即为运行时环境，相当于内存结构的中间的那个框框“运行时环境。

## 线程

- 线程是一个程序里的运行单元。JVM允许一个应用有多个线程并行的执行。
- 在Hotspot JVM里，每个线程都与操作系统的本地线程直接映射：当一个Java线程准备好执行以后，此时一个操作系统的本地线程也同时创建。Java线程执行终止后，本地线程也会回收。
- 操作系统负责所有线程的安排调度到任何一个可用的CPU上。一旦本地线程初始化成功，它就会调用Java线程种的run方法。
- 如果使用jconsole或者其他的调式工具，能看到后台有许多线程在运行，这些后台线程不包括调用psvm的main线程以及所有这个main线程自己创建的线程。
- 这些主要的后台线程在HotspotJVM里面主要是以下几个：
  - **虚拟机线程**：这种线程的操作是需要JVM达到安全点才会出现。这些操作必须在不同的线程中发生的原因是他们都需要JVM达到安全点，这样堆才不会变化。这种线程的执行类型包括”stop-the-world“的垃圾收集、线程栈收集、线程挂起以及偏向锁撤销
  - **周期任务线程**：这种线程是时间周期时间的体现（比如中断），他们一般用于周期性操作的调度执行
  - **GC线程**：这种线程对在JVM里不同种类的垃圾收集行为提供了支持
  - **编译线程**：在运行时将字节码编译成到本地代码
  - **信号调度线程**：这种线程接受信号并发送给JVM，它在内部通过调用适当的方法进行处理

## PC计数器

### PC寄存器的介绍

1. JVM中的程序计数寄存器（Program Counter Register）中，Register的命名源于CPU的寄存器，**寄存器存储指令相关的现场信息**。CPU只有把数据装载到寄存器才能够运行。
2. 这里，并非是广义上所指的物理寄存器，或许将其翻译为PC计数器（或指令计数器）会更加贴切（也称为程序钩子），并且也不容易引起一些不必要的误会。**JVM中的PC寄存器是对物理PC寄存器的一种抽象模拟**。
3. 它是一块很小的内存空间，几乎可以忽略不记。也是运行速度最快的存储区域。
4. 在JVM规范中，每个线程都有它自己的程序计数器，是线程私有的，生命周期与线程的生命周期保持一致。
5. 任何时间一个线程都只有一个方法在执行，也就是所谓的**当前方法**。程序计数器会存储当前线程正在执行的Java方法的JVM指令地址；或者，如果是在执行native方法，则是未指定值（undefned）。
6. 它是**程序控制流**的指示器，分支、循环、跳转、异常处理、线程恢复等基础功能都需要依赖这个计数器来完成。
7. 字节码解释器工作时就是通过改变这个计数器的值来选取下一条需要执行的字节码指令。
8. 它是**唯一一个**在Java虚拟机规范中没有规定任何OutofMemoryError情况的区域。

### PC寄存器的作用

PC寄存器用来存储指向下一条指令的地址，也即将 要执行的指令代码。由执行引擎读取下一条指令，并执行该指令。

![IMG/04、程序计数器.assets/image-20210821200606763.png](深入理解Java虚拟机/image-20210821200606763.png)

测试代码如下，通过字节码文件层次进行分析：

```java
public class PCRegisterTest {

    public static void main(String[] args) {
        int i = 10;
        int j = 20;
        int k = i + j;

        String s = "abc";
        System.out.println(i);
        System.out.println(k);

    }
}
```

左边的数字代表**指令地址（指令偏移）**，即 PC 寄存器中可能存储的值，然后执行引擎读取 PC 寄存器中的值，并执行该指令

![img](深入理解Java虚拟机/0009.png)

### 两个常见面试题

**使用PC寄存器存储字节码指令地址有什么用呢？**或者问**为什么使用PC寄存器来记录当前线程的执行地址**

1. 因为CPU需要不停的切换各个线程，这时候切换回来以后，就得知道接着从哪开始继续执行
2. JVM的字节码解释器就需要通过改变PC寄存器的值来明确下一条应该执行什么样的字节码指令

**PC寄存器为什么被设定为私有的**

1. 我们都知道所谓的多线程在一个特定的时间段内只会执行其中某一个线程的方法，CPU会不停地做任务切换，这样必然导致经常中断或恢复，如何保证分毫无差呢？**为了能够准确地记录各个线程正在执行的当前字节码指令地址，最好的办法自然是为每一个线程都分配一个PC寄存器**，这样一来各个线程之间便可以进行独立计算，从而不会出现相互干扰的情况。
2. 由于CPU时间片轮限制，众多线程在并发执行过程中，任何一个确定的时刻，一个处理器或者多核处理器中的一个内核，只会执行某个线程中的一条指令。
3. 这样必然导致经常中断或恢复，如何保证分毫无差呢？每个线程在创建后，都会产生自己的程序计数器和栈帧，程序计数器在各个线程之间互不影响。

### CPU时间片

1. CPU时间片即CPU分配给各个程序的时间，每个线程被分配一个时间段，称作它的时间片。
2. 在宏观上：我们可以同时打开多个应用程序，每个程序并行不悖，同时运行。
3. 但在微观上：由于只有一个CPU，一次只能处理程序要求的一部分，如何处理公平，一种方法就是引入时间片，**每个程序轮流执行**。

![img](深入理解Java虚拟机/0011.png)

## 虚拟机栈

### 简介

**虚拟机栈出现的背景**

1. 由于跨平台性的设计，Java的指令都是根据栈来设计的。不同平台CPU架构不同，所以不能设计为基于寄存器的【如果设计成基于寄存器的，耦合度高，性能会有所提升，因为可以对具体的CPU架构进行优化，但是跨平台性大大降低】。
2. 优点是跨平台，指令集小，编译器容易实现，缺点是性能下降，实现同样的功能需要更多的指令。

**内存中的栈与堆**

1. 首先栈是运行时的单位，而堆是存储的单位。
2. 即：栈解决程序的运行问题，即程序如何执行，或者说如何处理数据。堆解决的是数据存储的问题，即数据怎么放，放哪里

**虚拟机栈基本内容**

- Java虚拟机栈是什么？
  - Java虚拟机栈（Java Virtual Machine Stack）早期也叫Java栈。每个线程在创建时都会创建一个虚拟机栈，其内部保存一个个的栈帧（StackFrame），对应着一次次的Java方法调用，栈是线程私有的
- 虚拟机栈的生命周期
  - 和线程一致，也就是线程结束了，该虚拟机栈也就被销毁了
- 作用
  - 主管Java程序的运行，它保存方法的局部变量（8种基本数据类型、对象的引用地址）、部分结果，并参与方法的调用和返回。
    - 局部变量，它是相比于成员变量来说的（或属性）
    - 基本数据类型 VS 引用类型变量（类、数组、接口）

**虚拟机栈的特点**

- 栈是一种快速有效的分配存储方式，访问速度仅次于程序计数器
- JVM直接对Java栈的操作只有两个：
  - 每个方法执行，伴随着进栈（入栈、压栈）
  - 方法执行结束后出栈工作
- **对于栈来说不存在垃圾回收问题，但是可能存在OOM**

**虚拟机栈种的异常**

Java虚拟机规范允许Java栈的大小是动态的或者是固定不变的

- 如果采用固定大小的Java虚拟机栈，那么每一个线程的Java虚拟机栈容量可以在线程创建的时候独立选定。如果线程请求分配的栈容量超过Java虚拟机栈允许的最大容量，Java虚拟机将会抛出一个**StackoverflowError**的异常
- 如果Java虚拟机栈可以动态扩展，并且在尝试扩展的时候无法申请到足够的内存，或者在创建新的线程时没有足够的内存去创建对应的虚拟机栈，那么Java虚拟机将会抛出一个**OutOfMemoryError** 异常

**设置栈内存大小**

我们可以使用参数 **-Xss** 选项来设置线程的最大栈空间，栈的大小直接决定了函数调用的最大可达深度。

### 栈的存储单位

#### 栈中存储什么？

1. 每个线程都有自己的栈，栈中的数据都是以**栈帧**的格式存在
2. 在这个线程上正在执行的每个方法都各自对应着一个栈帧
3. 栈帧是一个内存区块，是一个数据集，维系着方法执行过程中的各种数据信息

#### 栈运行原理

1. JVM直接对Java栈的操作只有两个，就是对栈帧的压栈和出栈
2. 在一条活动线程中，一个时间点上，只会有一个活动的栈帧。即只有当前正在执行的方法的栈帧（即栈顶栈帧）是有效的。这个栈帧被称为**当前栈帧（Current Frame）**，与当前栈帧相对应的方法就是**当前方法（（Current Method）**，定义这个方法的类就是**当前类（Current Class）**
3. 执行引擎运行的所有字节码指令都针对当前栈帧进行操作。
4. 如果该方法调用了其他方法，对应的新的栈帧会被创建出来，放在栈的顶端，成为新的当前栈帧

![image-20210825093704619](深入理解Java虚拟机/image-20210825093704619.png)

1. **不同的线程中所包含的栈帧是不允许存在相互引用的**，即不可能在一个栈帧之中引用另外一个线程的栈帧。
2. 如果当前方法调用了其他方法，在方法返回之际，当前栈帧会传回此方法的执行结果给前一个栈帧，接着，虚拟机会丢弃当前栈帧，使得前一个栈帧重新成为当前栈帧
3. Java方法有两种返回函数的方式
   - 一种是正常的函数返回，使用return指令
   - 另一种是方法执行中出现未捕获处理的异常，以抛出异常的方式结束
   - 但是不管是哪种方式，都会导致栈帧被弹出

#### 栈帧的内部结构

- **局部变量表（Local Variables）**
- **操作数栈（Operand Stack）（或表达式栈）**
- 动态链接（Dynamic Linking）或指向运行时常量池的方法引用
- 方法返回地址（Return Address）或方法正常退出或者异常退出的定义
- 一些附加信息

![image-20210825095220429](深入理解Java虚拟机/image-20210825095220429.png)

### 局部变量表（有疑问）

- 局部变量表也被称为局部变量数组或本地变量表

- **定义为一个数字数组，主要用于存储方法参数和定义在方法体内的局部变量**，这些数据类型包括各类基本数据类型、对象引用（reference），以及returnAddress返回值类型。

  > ![image-20210825183353317](深入理解Java虚拟机/image-20210825183353317.png)

  > 注意这里说的，在Java8中，通过jclasslib我并没有看到局部变量表中有关于returnAddress返回值类型的影子，通过在尚硅谷群内询问，这位老哥告诉我returnAddress已经由异常表代替了，但是现在我还没有学到异常表相关信息（好像后序的章节会说明，暂时标注一下）

- 由于局部变量表是建立在线程的栈上，是线程的私有数据，因此**不存在数据安全问题**

- **局部变量表所需的容量大小是在编译期确定下来的**，并保存在方法的Code属性的**maximum local variables**数据项中。在方法运行期间是不会改变局部变量表的大小的。

- 方法嵌套调用的次数由栈的大小决定。一般来说，栈越大，方法嵌套调用次数越多。

  - 对一个函数而言，它的参数和局部变量越多，使得局部变量表膨胀，它的栈帧就越大，以满足方法调用所需传递的信息增大的需求。
  - 进而函数调用就会占用更多的栈空间，导致其嵌套调用次数就会减少。

- 局部变量表中的变量只在当前方法调用中有效。

  - 在方法执行时，虚拟机通过使用局部变量表完成参数值到参数变量列表的传递过程。
  - 当方法调用结束后，随着方法栈帧的销毁，局部变量表也会随之销毁。

#### 关于Slot的理解

1. 参数值的存放总是从局部变量数组索引 0 的位置开始，到数组长度-1的索引结束。
2. 局部变量表，**最基本的存储单元是Slot（变量槽）**，局部变量表中存放编译期可知的各种基本数据类型（8种），引用类型（reference），returnAddress类型的变量。
3. 在局部变量表里，32位以内的类型只占用一个slot（包括returnAddress类型），64位的类型占用两个slot（1ong和double）。
   - byte、short、char在储存前被转换为int，boolean也被转换为int，0表示false，非0表示true
   - long和double则占据两个slot
4. JVM会为局部变量表中的每一个Slot都分配一个访问索引，通过这个索引即可成功访问到局部变量表中指定的局部变量值
5. 当一个实例方法被调用的时候，它的方法参数和方法体内部定义的局部变量将会**按照顺序被复制**到局部变量表中的每一个slot上
6. 如果需要访问局部变量表中一个64bit的局部变量值时，只需要使用前一个索引即可。（比如：访问long或double类型变量）
7. 如果当前帧是由构造方法或者实例方法创建的，那么**该对象引用this将会存放在index为0的slot处**，其余的参数按照参数表顺序继续排列。（this也相当于一个变量）

#### Slot的重复利用

栈帧中的局部变量表中的槽位是可以重用的，如果一个局部变量过了其作用域，那么在其作用域之后申明新的局部变量变就很有可能会复用过期局部变量的槽位，从而达到节省资源的目的。

```java
public void localVar2() {
    {
        int a = 0;
        System.out.println(a);
    }
    //此时的b就会复用a的槽位
    int b = 0;
}
```

通过jclasslib可以看到b复用了a所处的slot

![IMG/05、虚拟机栈.assets/image-20210825184400287.png](深入理解Java虚拟机/image-20210825184400287.png)

#### 静态变量和局部变量的对比

```java
变量的分类：
1、按照数据类型分：① 基本数据类型  ② 引用数据类型
2、按照在类中声明的位置分：
  2-1、成员变量：在使用前，都经历过默认初始化赋值
       2-1-1、类变量: linking的prepare阶段：给类变量默认赋值
              ---> initial阶段：给类变量显式赋值即静态代码块赋值
       2-1-2、实例变量：随着对象的创建，会在堆空间中分配实例变量空间，并进行默认赋值
  2-2、局部变量：在使用前，必须要进行显式赋值的！否则，编译不通过。
```

1. 参数表分配完毕之后，再根据方法体内定义的变量的顺序和作用域分配。
2. 我们知道成员变量有两次初始化的机会**，**第一次是在“准备阶段”，执行系统初始化，对类变量设置零值，另一次则是在“初始化”阶段，赋予程序员在代码中定义的初始值。
3. 和类变量初始化不同的是，**局部变量表不存在系统初始化的过程**，这意味着一旦定义了局部变量则必须人为的初始化，否则无法使用。

#### 补充说明

1. 在栈帧中，与性能调优关系最为密切的部分就是前面提到的局部变量表。在方法执行时，虚拟机使用局部变量表完成方法的传递。
2. 局部变量表中的变量也是重要的垃圾回收根节点，只要被局部变量表中直接或间接引用的对象都不会被回收。

### 操作数栈

#### 操作数栈的特点

1. 每一个独立的栈帧除了包含局部变量表以外，还包含一个后进先出（Last - In - First -Out）的 操作数栈，也可以称之为**表达式栈**（Expression Stack）

2. 操作数栈，在方法执行过程中，**根据字节码指令，往栈中写入数据或提取数据**，即入栈（push）和 出栈（pop）

- 某些字节码指令将值压入操作数栈，其余的字节码指令将操作数取出栈。使用它们后再把结果压入栈，

- 比如：执行复制、交换、求和等操作

  ![image-20210825185025622](深入理解Java虚拟机/image-20210825185025622.png)

#### 作用

1. 操作数栈，**主要用于保存计算过程的中间结果，同时作为计算过程中变量临时的存储空间**。
2. 操作数栈就是JVM执行引擎的一个工作区，当一个方法刚开始执行的时候，一个新的栈帧也会随之被创建出来，这时方法的操作数栈是空的。
3. 每一个操作数栈都会拥有一个明确的栈深度用于存储数值，其所需的最大深度在编译期就定义好了，保存在方法的Code属性中，为**maxstack**的值。
4. 栈中的任何一个元素都是可以任意的Java数据类型
   - 32bit的类型占用一个栈单位深度
   - 64bit的类型占用两个栈单位深度
5. 操作数栈并非采用访问索引的方式来进行数据访问的，而是只能通过标准的入栈和出栈操作来完成一次数据访问。**只不过操作数栈是用数组这个结构来实现的而已**
6. 如果被调用的方法带有返回值的话，其返回值将会被压入当前栈帧的操作数栈中，并更新PC寄存器中下一条需要执行的字节码指令。
7. 操作数栈中元素的数据类型必须与字节码指令的序列严格匹配，这由编译器在编译器期间进行验证，同时在类加载过程中的类检验阶段的数据流分析阶段要再次验证。
8. 另外，**我们说Java虚拟机的解释引擎是基于栈的执行引擎，其中的栈指的就是操作数栈**。

### 栈顶缓存技术

Top Of Stack Cashing

1. 前面提过，基于栈式架构的虚拟机所使用的零地址指令更加紧凑，但完成一项操作的时候必然需要使用更多的入栈和出栈指令，这同时也就意味着将需要更多的指令分派（instruction dispatch）次数（也就是你会发现指令很多）和导致内存读/写次数多，效率不高。
2. 由于操作数是存储在内存中的，因此频繁地执行内存读/写操作必然会影响执行速度。为了解决这个问题，HotSpot JVM的设计者们提出了栈顶缓存（Tos，Top-of-Stack Cashing）技术，**将栈顶元素全部缓存在物理CPU的寄存器中，以此降低对内存的读/写次数，提升执行引擎的执行效率。**
3. 寄存器的主要优点：指令更少，执行速度快，但是指令集（也就是指令种类）很多
4. 也就是说为了效率更高

### 动态链接（指向运行时常量池的方法引用）

1. 每一个栈帧内部都包含**一个指向运行时常量池中该栈帧所属方法的引用**。包含这个引用的目的就是**为了支持当前方法的代码能够实现动态链接**（Dynamic Linking），比如：invokedynamic指令
2. 在Java源文件被编译到字节码文件中时，所有的变量和方法引用都作为符号引用（Symbolic Reference）保存在class文件的常量池里。比如：描述一个方法调用了另外的其他方法时，就是通过常量池中指向方法的符号引用来表示的，那么**动态链接的作用就是为了将这些符号引用转换为调用方法的直接引用**

```java
public class DynamicLinkingTest {

    int num = 10;

    public void methodA(){
        System.out.println("methodA()....");
    }

    public void methodB(){
        System.out.println("methodB()....");

        methodA();

        num++;
    }

}
```

对应字节码如下：

```java
Classfile /F:/IDEAWorkSpaceSourceCode/JVMDemo/out/production/chapter05/com/atguigu/java1/DynamicLinkingTest.class
  Last modified 2020-11-10; size 712 bytes
  MD5 checksum e56913c945f897c7ee6c0a608629bca8
  Compiled from "DynamicLinkingTest.java"
public class com.atguigu.java1.DynamicLinkingTest
  minor version: 0
  major version: 52
  flags: ACC_PUBLIC, ACC_SUPER
Constant pool:
   #1 = Methodref          #9.#23         // java/lang/Object."<init>":()V
   #2 = Fieldref           #8.#24         // com/atguigu/java1/DynamicLinkingTest.num:I
   #3 = Fieldref           #25.#26        // java/lang/System.out:Ljava/io/PrintStream;
   #4 = String             #27            // methodA()....
   #5 = Methodref          #28.#29        // java/io/PrintStream.println:(Ljava/lang/String;)V
   #6 = String             #30            // methodB()....
   #7 = Methodref          #8.#31         // com/atguigu/java1/DynamicLinkingTest.methodA:()V
   #8 = Class              #32            // com/atguigu/java1/DynamicLinkingTest
   #9 = Class              #33            // java/lang/Object
  #10 = Utf8               num
  #11 = Utf8               I
  #12 = Utf8               <init>
  #13 = Utf8               ()V
  #14 = Utf8               Code
  #15 = Utf8               LineNumberTable
  #16 = Utf8               LocalVariableTable
  #17 = Utf8               this
  #18 = Utf8               Lcom/atguigu/java1/DynamicLinkingTest;
  #19 = Utf8               methodA
  #20 = Utf8               methodB
  #21 = Utf8               SourceFile
  #22 = Utf8               DynamicLinkingTest.java
  #23 = NameAndType        #12:#13        // "<init>":()V
  #24 = NameAndType        #10:#11        // num:I
  #25 = Class              #34            // java/lang/System
  #26 = NameAndType        #35:#36        // out:Ljava/io/PrintStream;
  #27 = Utf8               methodA()....
  #28 = Class              #37            // java/io/PrintStream
  #29 = NameAndType        #38:#39        // println:(Ljava/lang/String;)V
  #30 = Utf8               methodB()....
  #31 = NameAndType        #19:#13        // methodA:()V
  #32 = Utf8               com/atguigu/java1/DynamicLinkingTest
  #33 = Utf8               java/lang/Object
  #34 = Utf8               java/lang/System
  #35 = Utf8               out
  #36 = Utf8               Ljava/io/PrintStream;
  #37 = Utf8               java/io/PrintStream
  #38 = Utf8               println
  #39 = Utf8               (Ljava/lang/String;)V
{
  int num;
    descriptor: I
    flags:

  public com.atguigu.java1.DynamicLinkingTest();
    descriptor: ()V
    flags: ACC_PUBLIC
    Code:
      stack=2, locals=1, args_size=1
         0: aload_0
         1: invokespecial #1                  // Method java/lang/Object."<init>":()V
         4: aload_0
         5: bipush        10
         7: putfield      #2                  // Field num:I
        10: return
      LineNumberTable:
        line 7: 0
        line 9: 4
      LocalVariableTable:
        Start  Length  Slot  Name   Signature
            0      11     0  this   Lcom/atguigu/java1/DynamicLinkingTest;

  public void methodA();
    descriptor: ()V
    flags: ACC_PUBLIC
    Code:
      stack=2, locals=1, args_size=1
         0: getstatic     #3                  // Field java/lang/System.out:Ljava/io/PrintStream;
         3: ldc           #4                  // String methodA()....
         5: invokevirtual #5                  // Method java/io/PrintStream.println:(Ljava/lang/String;)V
         8: return
      LineNumberTable:
        line 12: 0
        line 13: 8
      LocalVariableTable:
        Start  Length  Slot  Name   Signature
            0       9     0  this   Lcom/atguigu/java1/DynamicLinkingTest;

  public void methodB();
    descriptor: ()V
    flags: ACC_PUBLIC
    Code:
      stack=3, locals=1, args_size=1
         0: getstatic     #3                  // Field java/lang/System.out:Ljava/io/PrintStream;
         3: ldc           #6                  // String methodB()....
         5: invokevirtual #5                  // Method java/io/PrintStream.println:(Ljava/lang/String;)V
         8: aload_0
         9: invokevirtual #7                  // Method methodA:()V
        12: aload_0
        13: dup
        14: getfield      #2                  // Field num:I
        17: iconst_1
        18: iadd
        19: putfield      #2                  // Field num:I
        22: return
      LineNumberTable:
        line 16: 0
        line 18: 8
        line 20: 12
        line 21: 22
      LocalVariableTable:
        Start  Length  Slot  Name   Signature
            0      23     0  this   Lcom/atguigu/java1/DynamicLinkingTest;
}
SourceFile: "DynamicLinkingTest.java"
```

所谓的**将这些符号引用转换为调用方法的直接引用**，可以看到代码我们是方法B中调用了方法A，然后在方法B的字节码文件中，可以看到通过invokevirtual指令调用了方法A，而#7所代表的东西，我们可以去看Constant Pool。也就是我们可以将这些符号引用转化为调用方法的直接引用

![image-20210825195923604](深入理解Java虚拟机/image-20210825195923604.png)

**为什么要用常量池呢？**

1. 因为在不同的方法，都可能调用常量或者方法，所以只需要存储一份即可，然后记录其引用即可，节省了空间。
2. 常量池的作用：就是为了提供一些符号和常量，便于指令的识别

### 方法的调用

在JVM中，将符号引用转换为调用方法的直接引用与方法的绑定机制相关

#### 静态链接与动态链接

- **静态链接**：

当一个字节码文件被装载进JVM内部时，如果被调用的目标方法在编译期确定，且运行期保持不变时，这种情况下将调用方法的符号引用转换为直接引用的过程称之为静态链接

- **动态链接**：

如果被调用的方法在编译期无法被确定下来，也就是说，只能够在程序运行期将调用的方法的符号转换为直接引用，由于这种引用转换过程具备动态性，因此也被称之为动态链接。

#### 早期绑定与晚期绑定

静态链接和动态链接对应的方法的绑定机制为：早期绑定（Early Binding）和晚期绑定（Late Binding）。**绑定是一个字段、方法或者类在符号引用被替换为直接引用的过程**，这仅仅发生一次。

- **早期绑定**

早期绑定就是指被调用的目标方法如果在编译期可知，且运行期保持不变时，即可将这个方法与所属的类型进行绑定，这样一来，由于明确了被调用的目标方法究竟是哪一个，因此也就**可以使用静态链接的方式将符号引用转换为直接引用**。

- **晚期绑定**

如果被调用的方法在编译期无法被确定下来，**只能够在程序运行期根据实际的类型绑定相关的方法**，这种绑定方式也就被称之为晚期绑定。

#### 多态与绑定

1. 随着高级语言的横空出世，类似于Java一样的基于面向对象的编程语言如今越来越多，尽管这类编程语言在语法风格上存在一定的差别，但是它们彼此之间始终保持着一个共性，那就是都支持封装、继承和多态等面向对象特性，既然这一类的编程语言具备多态特性，那么自然也就具备早期绑定和晚期绑定两种绑定方式。
2. Java中任何一个普通的方法其实都具备虚函数的特征，它们相当于C++语言中的虚函数（C++中则需要使用关键字virtual来显式定义）。如果在Java程序中不希望某个方法拥有虚函数的特征时，则可以使用关键字final来标记这个方法。

#### 虚方法与非虚方法

1. 如果方法在编译期就确定了具体的调用版本，这个版本在运行时是不可变的。这样的方法称为**非虚方法**。
2. **静态方法、私有方法、final方法、实例构造器、父类方法都是非虚方法**。
3. 其他方法称为虚方法。

---------------------

**虚拟机中调用方法的指令**

- **普通指令：**

1. invokestatic：调用静态方法，解析阶段确定唯一方法版本

2. invokespecial：调用`<init>`方法、私有及父类方法，解析阶段确定唯一方法版本

3. invokevirtual：调用所有虚方法

4. invokeinterface：调用接口方法

- **动态调用指令**

invokedynamic：动态解析出需要调用的方法，然后执行

前四条指令固化在虚拟机内部，方法的调用执行不可人为干预。而invokedynamic指令则支持由用户确定方法版本。其中**invokestatic指令和invokespecial指令调用的方法称为非虚方法，其余的（final修饰的除外）称为虚方法**。

#### 关于invokedynamic指令

1. JVM字节码指令集一直比较稳定，一直到Java7中才增加了一个invokedynamic指令，这是Java为了实现【动态类型语言】支持而做的一种改进。
2. 但是在Java7中并没有提供直接生成invokedynamic指令的方法，需要借助ASM这种底层字节码工具来产生invokedynamic指令。直到Java8的Lambda表达式的出现，invokedynamic指令的生成，在Java中才有了直接的生成方式。
3. Java7中增加的动态语言类型支持的本质是对Java虚拟机规范的修改，而不是对Java语言规则的修改，这一块相对来讲比较复杂，增加了虚拟机中的方法调用，最直接的受益者就是运行在Java平台的动态语言的编译器。

------

**动态语言和静态语言**

1. 动态类型语言和静态类型语言两者的区别就在于**对类型的检查是在编译期还是在运行期**，满足前者就是静态类型语言，反之是动态类型语言。
2. 说的再直白一点就是，静态类型语言是判断变量自身的类型信息；动态类型语言是判断变量值的类型信息，变量没有类型信息，变量值才有类型信息，这是动态语言的一个重要特征。

```
Java：String info="atguigu"; //静态类型语言，编译期间就会进行类型检查
JS：var name="test";  //运行时才检查，动态语言
Python：info=120.3 //运行时才会检查
```

### 方法返回地址

> 在一些帖子里，方法返回地址、动态链接、一些附加信息也叫做帧数据区

1. 存放调用该方法的pc寄存器的值。一个方法的结束，有两种方式：
   - 正常执行完成
   - 出现未处理的异常，非正常退出
2. 无论通过哪种方式退出，在方法退出后都返回到该方法被调用的位置。方法正常退出时，**调用者的pc计数器的值作为返回地址，即调用该方法的指令的下一条指令的地址**。而通过异常退出的，返回地址是要通过异常表来确定，栈帧中一般不会保存这部分信息。
3. 本质上，方法的退出就是当前栈帧出栈的过程。此时，需要恢复上层方法的局部变量表、操作数栈、将返回值压入调用者栈帧的操作数栈、设置PC寄存器值等，让调用者方法继续执行下去。
4. 正常完成出口和异常完成出口的区别在于：通过异常完成出口退出的不会给他的上层调用者产生任何的返回值。

---

当一个方法开始执行后，只有两种方式可以退出这个方法，

**正常退出：**

1. 执行引擎遇到任意一个方法返回的字节码指令（return），会有返回值传递给上层的方法调用者，简称**正常完成出口**；
2. 一个方法在正常调用完成之后，究竟需要使用哪一个返回指令，还需要根据方法返回值的实际数据类型而定。
3. 在字节码指令中，返回指令包含：
   - ireturn：当返回值是boolean，byte，char，short和int类型时使用
   - lreturn：Long类型
   - freturn：Float类型
   - dreturn：Double类型
   - areturn：引用类型
   - return：返回值类型为void的方法、实例初始化方法、类和接口的初始化方法

**异常退出：**

1. 在方法执行过程中遇到异常（Exception），并且这个异常没有在方法内进行处理，也就是只要在本方法的异常表中没有搜索到匹配的异常处理器，就会导致方法退出，简称**异常完成出口**。
2. 方法执行过程中，抛出异常时的异常处理，存储在一个异常处理表，方便在发生异常的时候找到处理异常的代码

![img](深入理解Java虚拟机/0040.png)

异常处理表：

- 反编译字节码文件，可得到 Exception table
- from ：字节码指令起始地址
- to ：字节码指令结束地址
- target ：出现异常跳转至地址为 11 的指令执行
- type ：捕获异常的类型

所谓的异常处理表，包含的内容其实就是Java代码层次的try catch，上面说的，只要在本方法的异常表中没有搜索到匹配的异常处理器，就会导致方法退出，也就是方法执行过程中抛出的异常没有被catch到，就会终止该方法

### 一些附加信息

栈帧中还允许携带与Java虚拟机实现相关的一些附加信息。例如：对程序调试提供支持的信息。

## 本地方法接口

![img](深入理解Java虚拟机/0012.png)

1. 简单地讲，**一个Native Method是一个Java调用非Java代码的接囗**，一个Native Method是这样一个Java方法：该方法的实现由非Java语言实现，比如C。这个特征并非Java所特有，很多其它的编程语言都有这一机制，比如在C++中，你可以用extern 告知C++编译器去调用一个C的函数。
2. “A native method is a Java method whose implementation is provided by non-java code.”（本地方法是一个非Java的方法，它的具体实现是非Java代码的实现）
3. 在定义一个native method时，并不提供实现体（有些像定义一个Java interface），因为其实现体是由非java语言在外面实现的。
4. 本地接口的作用是融合不同的编程语言为Java所用，它的初衷是融合C/C++程序。

需要注意的是：**标识符native可以与其它java标识符连用，但是abstract除外**

**为什么要使用Native Method**

1. Java使用起来非常方便，然而有些层次的任务用Java实现起来不容易，或者我们对程序的效率很在意时，问题就来了。
2. **与Java环境外交互**：
   1. **有时Java应用需要与Java外面的硬件环境交互，这是本地方法存在的主要原因**。你可以想想Java需要与一些**底层系统**，如操作系统或某些硬件交换信息时的情况。本地方法正是这样一种交流机制：它为我们提供了一个非常简洁的接口，而且我们无需去了解Java应用之外的繁琐的细节。
3. **与操作系统的交互**：
   1. JVM支持着Java语言本身和运行时库，它是Java程序赖以生存的平台，它由一个解释器（解释字节码）和一些连接到本地代码的库组成。
   2. 然而不管怎样，它毕竟不是一个完整的系统，它经常依赖于一底层系统的支持。这些底层系统常常是强大的操作系统。
   3. **通过使用本地方法，我们得以用Java实现了jre的与底层系统的交互，甚至JVM的一些部分就是用C写的**。
   4. 还有，如果我们要使用一些Java语言本身没有提供封装的操作系统的特性时，我们也需要使用本地方法。
4. **Sun's Java**
   1. Sun的解释器是用C实现的，这使得它能像一些普通的C一样与外部交互。jre大部分是用Java实现的，它也通过一些本地方法与外界交互。
   2. 例如：类java.lang.Thread的setPriority()方法是用Java实现的，但是它实现调用的是该类里的本地方法setPriority0()。这个本地方法是用C实现的，并被植入JVM内部在Windows 95的平台上，这个本地方法最终将调用Win32 setpriority() API。这是一个本地方法的具体实现由JVM直接提供，更多的情况是本地方法由外部的动态链接库（external dynamic link library）提供，然后被JVM调用。

**本地方法的现状**：

目前该方法使用的越来越少了，除非是与硬件有关的应用，比如通过Java程序驱动打印机或者Java系统管理生产设备，在企业级应用中已经比较少见。因为现在的异构领域间的通信很发达，比如可以使用Socket通信，也可以使用Web Service等等，不多做介绍。

## 本地方法栈

1. **Java虚拟机栈于管理Java方法的调用，而本地方法栈用于管理本地方法的调用**。
2. 本地方法栈，也是线程私有的。
3. 允许被实现成固定或者是可动态扩展的内存大小（在内存溢出方面和虚拟机栈相同）
   - 如果线程请求分配的栈容量超过本地方法栈允许的最大容量，Java虚拟机将会抛出一个stackoverflowError 异常。
   - 如果本地方法栈可以动态扩展，并且在尝试扩展的时候无法申请到足够的内存，或者在创建新的线程时没有足够的内存去创建对应的本地方法栈，那么Java虚拟机将会抛出一个outofMemoryError异常。
4. 本地方法一般是使用C语言或C++语言实现的。
5. 它的具体做法是Native Method Stack中登记native方法，在Execution Engine 执行时加载本地方法库。

![image-20210828183928535](深入理解Java虚拟机/image-20210828183928535.png)

**注意事项**

1. 当某个线程调用一个本地方法时，它就进入了一个全新的并且不再受虚拟机限制的世界。它和虚拟机拥有同样的权限。
   - 本地方法可以通过本地方法接口来访问虚拟机内部的运行时数据区
   - 它甚至可以直接使用本地处理器中的寄存器
   - 直接从本地内存的堆中分配任意数量的内存
2. 并不是所有的JVM都支持本地方法。因为Java虚拟机规范并没有明确要求本地方法栈的使用语言、具体实现方式、数据结构等。如果JVM产品不打算支持native方法，也可以无需实现本地方法栈。
3. 在Hotspot JVM中，直接将本地方法栈和虚拟机栈合二为一。

## 堆

### 堆的核心概述

1. 堆针对一个JVM进程来说是唯一的。也就是**一个进程只有一个JVM实例**，一个JVM实例中就有一个运行时数据区，一个运行时数据区只有一个堆和一个方法区。
2. 但是**进程包含多个线程，他们是共享同一堆空间的**。
3. 一个JVM实例只存在一个堆内存，堆也是Java内存管理的核心区域。
4. Java堆区在JVM启动的时候即被创建，其空间大小也就确定了，堆是JVM管理的最大一块内存空间，并且堆内存的大小是可以调节的。
5. 《Java虚拟机规范》规定，堆可以处于物理上不连续的内存空间中，但在逻辑上它应该被视为连续的。
6. 所有的线程共享Java堆，在这里还可以划分线程私有的缓冲区（Thread Local Allocation Buffer，**TLAB**）。
7. 《Java虚拟机规范》中对Java堆的描述是：**所有的对象实例以及数组都应当在运行时分配在堆上**。（The heap is the run-time data area from which memory for all class instances and arrays is allocated）
   - 从实际使用角度看：“几乎”所有的对象实例都在堆分配内存，但并非全部。因为还有一些对象是在栈上分配的（逃逸分析，标量替换）
8. 数组和对象可能永远不会存储在栈上（**不一定**），因为栈帧中保存引用，这个引用指向对象或者数组在堆中的位置。
9. 在方法结束后，堆中的对象不会马上被移除，仅仅在垃圾收集的时候才会被移除。
   - 也就是触发了GC的时候，才会进行回收
   - 如果堆中对象马上被回收，那么用户线程就会收到影响，因为有stop the word
10. 堆，是GC（Garbage Collection，垃圾收集器）执行垃圾回收的重点区域。

> 随着JVM的迭代升级，原来一些绝对的事情，在后续版本中也开始有了特例，变的不再那么绝对。

```java
public class SimpleHeap {
    private int id;//属性、成员变量

    public SimpleHeap(int id) {
        this.id = id;
    }

    public static void main(String[] args) {
        SimpleHeap sl = new SimpleHeap(1);
        SimpleHeap s2 = new SimpleHeap(2);
    }
}
```

![IMG/07、堆.assets/0002.png](深入理解Java虚拟机/0002.png)

### 堆的内存细分

![IMG/07、堆.assets/image-20210827185554979.png](深入理解Java虚拟机/image-20210827185554979.png)

如图所示，在Java7及之前堆内存逻辑上分为三部分：新生代+老年代+永久代，而在Java8以及之后，永久代被替换成了元空间。

#### 设置堆内存大小

1. Java堆区用于存储Java对象实例，那么堆的大小在JVM启动时就已经设定好了，大家可以通过选项"-Xms"和"-Xmx"来进行设置。

   - **-Xms**用于表示堆区的起始内存，等价于**-XX:InitialHeapSize**
   - **-Xmx**则用于表示堆区的最大内存，等价于**-XX:MaxHeapSize**

2. 一旦堆区中的内存大小超过“-Xmx"所指定的最大内存时，将会抛出OutofMemoryError异常。

3. 通常会将-Xms和-Xmx两个参数配置相同的值

- 原因：假设两个不一样，初始内存小，最大内存大。在运行期间如果堆内存不够用了，会一直扩容直到最大内存。如果内存够用且多了，也会不断的缩容释放。频繁的扩容和释放造成不必要的压力，避免在GC之后调整堆内存给服务器带来压力。

- 如果两个设置一样的就少了频繁扩容和缩容的步骤。内存不够了就直接报OOM

1. 默认情况下:
   - 初始内存大小：物理电脑内存大小/64
   - 最大内存大小：物理电脑内存大小/4

如果我们给我们的程序设置xms和xmx都是600m，但是通过Java代码：

` Runtime.getRuntime().totalMemory()/1024/1024`

得到Java虚拟机中的堆内存总量发现，只有575m，比预计的少了25m，那是因为我们堆的S0和S1区只有一个能用，这是由于垃圾回收的需要导致的。

#### 新生代和老年代

![IMG/07、堆.assets/image-20210827190632387.png](深入理解Java虚拟机/image-20210827190632387.png)

1. 一般来说，新生代和老年代的大小比例为1：2，而新生代里面的Eden：S0：S1=8：1：1
   - **-XX:NewRatio**=4，表示新生代占1，老年代占4，新生代占整个堆的1/5
   - **-XX:SurvivorRatio=8**，调整新生代内部结构空间比例
2. **几乎所有的Java对象都是在Eden区被new出来的**
3. 绝大部分的Java对象的销毁都在新生代进行了（有些大的对象在Eden区无法存储时候，将直接进入老年代），IBM公司的专门研究表明，新生代中80%的对象都是“朝生夕死”的。
4. 可以使用选项"-Xmn"设置新生代最大内存大小，但这个参数一般使用默认值就可以了。

#### 一般情况下的对象分配

1. 我们创建的对象，一般都是放在Eden区的，**当我们Eden区满了之后，就会触发GC操作**，此时的GC操作一般称为YGC/Minor GC
2. 当YGC开始的时候，在Eden区过期的对象会被回收，然后没有过期的话就会存放在S0区。同时我们会给每一个对象都设置了一个年龄计数器，如果经过一次YGC之后还没有被回收，就会加1。
3. 如果年龄计数器到达了15次，就可以进入老年代，可以通过JVM参数设置：**-XX:MaxTenuringThreshold**=N。
4. 同时Eden继续存放对象，当Eden再次满的时候，就会触发YGC操作，此时YGC会将Eden和S0都进行一次垃圾回收，并且将存活的对象放入S1区，然后年龄计数器+1。此时S0区是空的，再下一次垃圾回收的时候，会将存活的对象放入S0区，然后清空S1，以此循环
5. 如果经过不停的对象生成和垃圾回收，当新生代中的对象的年龄计数器到达15次之后，会触发一次Promotion晋升的操作，将该对象晋升到老年代里面。

#### 特殊情况下的对象分配

1. 如果来了一个新对象，要先看Eden区是否放得下
   - 如果放得下，就直接放到Eden区
   - 如果放不下，就触发YGC，执行垃圾回收，再看能否放进去
2. 如果Eden执行了YGC后还是放不到Eden区，就只能看老年代能否放进去，如果老年代也放不进就 进行一次FGC。如果这个时候还是放不下，就只能报OOM错误了
3. 如果Eden区满了之后触发YGC，将对象放入S0或者S1，但是发现也放不下，就只能晋升至老年代了

![IMG/07、堆.assets/0019.png](深入理解Java虚拟机/0019.png)

### GC分类

1. 我们都知道，JVM的调优的一个环节，也就是垃圾收集，我们需要尽量的避免垃圾回收，因为在垃圾回收的过程中，容易出现STW（Stop the World）的问题，**而 Major GC 和 Full GC出现STW的时间，是Minor GC的10倍以上**
2. JVM在进行GC时，并非每次都对上面三个内存区域一起回收的，大部分时候回收的都是指新生代。针对Hotspot VM的实现，它里面的GC按照回收区域又分为两大种类型：一种是部分收集（Partial GC），一种是整堆收集（FullGC）

- 部分收集：不是完整收集整个Java堆的垃圾收集。其中又分为：
  - **新生代收集**（Minor GC/Young GC）：只是新生代（Eden，s0，s1）的垃圾收集
  - **老年代收集**（Major GC/Old GC）：只是老年代的圾收集。
  - 目前，只有CMS GC会有单独收集老年代的行为。
  - 注意，很多时候Major GC会和Full GC混淆使用，需要具体分辨是老年代回收还是整堆回收。
  - 混合收集（Mixed GC）：收集整个新生代以及部分老年代的垃圾收集。目前，只有G1 GC会有这种行为
- **整堆收集**（Full GC）：收集整个java堆和方法区的垃圾收集。

> 由于历史原因，外界各种解读，majorGC和Full GC有些混淆。

#### YoungGC

1. 当年轻代空间不足时，就会触发Minor GC，这里的年轻代满指的是Eden代满。Survivor满不会主动引发GC，在Eden区满的时候，会顺带触发s0区的GC，也就是被动触发GC（每次Minor GC会清理年轻代的内存）
2. 因为Java对象大多都具备朝生夕灭的特性，所以Minor GC非常频繁，一般回收速度也比较快。
3. Minor GC会引发STW（Stop The World），暂停其它用户的线程，等垃圾回收结束，用户线程才恢复运行

#### Major/Full GC

**老年代GC（MajorGC）触发机制**

1. 指发生在老年代的GC，对象从老年代消失时，我们说 “Major Gc” 或 “Full GC” 发生了
2. 出现了MajorGc，经常会伴随至少一次的Minor GC。（但非绝对的，在Parallel Scavenge收集器的收集策略里就有直接进行MajorGC的策略选择过程）
   - 也就是在老年代空间不足时，会先尝试触发Minor GC，如果之后空间还不足，则触发Major GC
3. Major GC的速度一般会比Minor GC慢10倍以上，STW的时间更长。
4. 如果Major GC后，内存还不足，就报OOM了

**Full GC 触发机制（后面细讲）**

**触发Full GC执行的情况有如下五种：**

1. 调用System.gc()时，系统建议执行FullGC，但是不必然执行
2. 老年代空间不足
3. 方法区空间不足
4. 通过Minor GC后进入老年代的平均大小大于老年代的可用内存
5. 由Eden区、survivor space0（From Space）区向survivor space1（To Space）区复制时，对象大小大于To Space可用内存，则把该对象转存到老年代，且老年代的可用内存小于该对象大小

说明：Full GC 是开发或调优中尽量要避免的。这样STW时间会短一些

### 堆空间分代思想

**分代的原因和作用**

1. 优化GC性能
2. 如果没有分代的话，就相当于将一个学校的人都关在一个教室里面，不方便管理。GC的时候如果要找到哪些对象没用，这样就会对堆内所有区域进行扫描，导致性能低下
3. 据研究表明，很多对象都是朝生夕死的，如果分代的话，将新创建的对象放到某一个地方，当GC的时候优先将这块区域进行回收，就会腾出很大的空间来。（多回收新生代，少回收老年代，这样能提升很多性能)。

### 对象内存分配策略

1. 如果对象在Eden出生并经过第一次Minor GC后仍然存活，并且能被Survivor容纳的话，将被移动到Survivor空间中，并将对象年龄设为1。
2. 对象在Survivor区中每熬过一次MinorGC，年龄就增加1岁，当它的年龄增加到一定程度（默认为15岁，其实每个JVM、每个GC都有所不同）时，就会被晋升到老年代
3. 对象晋升老年代的年龄阀值，可以通过选项**-XX:MaxTenuringThreshold**来设置

**针对不同年龄段的对象分配原则如下所示：**

1. **优先分配到Eden**：开发中比较长的字符串或者数组，会直接存在老年代，但是因为新创建的对象都是朝生夕死的，所以这个大对象可能也很快被回收，但是因为老年代触发Major GC的次数比 Minor GC要更少，因此可能回收起来就会比较慢
2. **大对象直接分配到老年代**：尽量避免程序中出现过多的大对象
3. **长期存活的对象分配到老年代**
4. **动态对象年龄判断**：如果Survivor区中相同年龄的所有对象大小的总和大于Survivor空间的一半，年龄大于或等于该年龄的对象可以直接进入老年代，无须等到MaxTenuringThreshold中要求的年龄。
5. **空间分配担保**： -XX:HandlePromotionFailure 。

### TLAB（Thread Local Allocation Buffer）

1. 堆区是线程共享区域，任何线程都可以访问到堆区中的共享数据
2. 由于对象实例的创建在JVM中非常频繁，因此在并发环境下从堆区中划分内存空间是线程不安全的
3. 为避免多个线程操作同一地址，需要使用**加锁等机制**，进而影响分配速度。

![IMG/07、堆.assets/image-20210828175838380.png](深入理解Java虚拟机/image-20210828175838380.png)

1. 从内存模型而不是垃圾收集的角度，对Eden区域继续进行划分，**JVM为每个线程分配了一个私有缓存区域，它包含在Eden空间内**。
2. 多线程同时分配内存时，使用TLAB可以避免一系列的非线程安全问题，同时还能够提升内存分配的吞吐量，因此我们可以将这种内存分配方式称之为**快速分配策略**。
3. 据我所知所有OpenJDK衍生出来的JVM都提供了TLAB的设计。

> TLAB只是让每个线程有私有的分配指针，但底下存对象的内存空间还是给所有线程访问的，只是其它线程无法在这个区域分配而已。

**TLAB再说明**

1. 尽管不是所有的对象实例都能够在TLAB中成功分配内存，但**JVM确实是将TLAB作为内存分配的首选**。
2. 在程序中，开发人员可以通过选项“**-XX:UseTLAB**”设置是否开启TLAB空间。
3. 默认情况下，TLAB空间的内存非常小，仅占有整个Eden空间的1%，当然我们可以通过选项“**-XX:TLABWasteTargetPercent**”设置TLAB空间所占用Eden空间的百分比大小。
4. 一旦对象在TLAB空间分配内存失败时，JVM就会尝试着通过**使用加锁机制确保数据操作的原子性**，从而直接在Eden空间中分配内存。

![image-20210828180331700](深入理解Java虚拟机/image-20210828180331700.png)

### 堆空间常用参数设置

```java
/**
 * 测试堆空间常用的jvm参数：
 * -XX:+PrintFlagsInitial : 查看所有的参数的默认初始值
 * -XX:+PrintFlagsFinal  ：查看所有的参数的最终值（可能会存在修改，不再是初始值）
 *      具体查看某个参数的指令： jps：查看当前运行中的进程
 *                             jinfo -flag SurvivorRatio 进程id
 *
 * -Xms：初始堆空间内存 （默认为物理内存的1/64）
 * -Xmx：最大堆空间内存（默认为物理内存的1/4）
 * -Xmn：设置新生代的大小。(初始值及最大值)
 * -XX:NewRatio：配置新生代与老年代在堆结构的占比
 * -XX:SurvivorRatio：设置新生代中Eden和S0/S1空间的比例
 * -XX:MaxTenuringThreshold：设置新生代垃圾的最大年龄
 * -XX:+PrintGCDetails：输出详细的GC处理日志
 * 打印gc简要信息：① -XX:+PrintGC   ② -verbose:gc
 * -XX:HandlePromotionFailure：是否设置空间分配担保
 */
```

### 空间分配担保

1、在发生Minor GC之前，虚拟机会检查老年代最大可用的连续空间是否大于新生代所有对象的总空间。

- 如果大于，则此次Minor GC是安全的
- 如果小于，则虚拟机会查看**-XX:HandlePromotionFailure**设置值是否允担保失败。
  - 如果HandlePromotionFailure=true，那么会继续检查老年代最大可用连续空间是否大于历次晋升到老年代的对象的平均大小。
    - 如果大于，则尝试进行一次Minor GC，但这次Minor GC依然是有风险的；
    - 如果小于，则进行一次Full GC。
  - 如果HandlePromotionFailure=false，则进行一次Full GC。

**历史版本**

1. 在JDK6 Update 24之后，HandlePromotionFailure参数不会再影响到虚拟机的空间分配担保策略，观察openJDK中的源码变化，虽然源码中还定义了HandlePromotionFailure参数，但是在代码中已经不会再使用它。
2. JDK6 Update 24之后的规则变为**只要老年代的连续空间大于新生代对象总大小或者历次晋升的平均大小就会进行Minor GC**，否则将进行Full GC。即 HandlePromotionFailure=true

### 堆是分配对象的唯一选择吗

**在《深入理解Java虚拟机》中关于Java堆内存有这样一段描述：**

1. 随着JIT编译期的发展与**逃逸分析技术**逐渐成熟，**栈上分配、标量替换**优化技术将会导致一些微妙的变化，所有的对象都分配到堆上也渐渐变得不那么“绝对”了。
2. 在Java虚拟机中，对象是在Java堆中分配内存的，这是一个普遍的常识。但是，有一种特殊情况，那就是**如果经过逃逸分析（Escape Analysis）后发现，一个对象并没有逃逸出方法的话，那么就可能被优化成栈上分配**。这样就无需在堆上分配内存，也无须进行垃圾回收了。这也是最常见的堆外存储技术。
3. 此外，前面提到的基于OpenJDK深度定制的TaoBao VM，其中创新的GCIH（GC invisible heap）技术实现off-heap，将生命周期较长的Java对象从heap中移至heap外，并且GC不能管理GCIH内部的Java对象，以此达到降低GC的回收频率和提升GC的回收效率的目的。

#### 逃逸分析

1. 如何将堆上的对象分配到栈，需要使用逃逸分析手段。
2. 这是一种可以有效减少Java程序中同步负载和内存堆分配压力的跨函数全局数据流分析算法。
3. 通过逃逸分析，Java Hotspot编译器能够分析出一个新的对象的引用的使用范围从而决定是否要将这个对象分配到堆上。
4. 逃逸分析的基本行为就是分析对象动态作用域：
   - 当一个对象在方法中被定义后，对象只在方法内部使用，则认为没有发生逃逸。
   - 当一个对象在方法中被定义后，它被外部方法所引用，则认为发生逃逸。例如作为调用参数传递到其他地方中。

**逃逸分析举例**

1、没有发生逃逸的对象，则可以分配到栈（无线程安全问题）上，随着方法执行的结束，栈空间就被移除（也就无需GC）

```java
public void my_method() {
    V v = new V();
    // use v
    // ....
    v = null;
}
```

2、下面代码中的 StringBuffer sb 发生了逃逸，不能在栈上分配

```java
public static StringBuffer createStringBuffer(String s1, String s2) {
    StringBuffer sb = new StringBuffer();
    sb.append(s1);
    sb.append(s2);
    return sb;
}
```

3、如果想要StringBuffer sb不发生逃逸，可以这样写

```java
public static String createStringBuffer(String s1, String s2) {
    StringBuffer sb = new StringBuffer();
    sb.append(s1);
    sb.append(s2);
    return sb.toString();
}
/**
 * 逃逸分析
 *
 *  如何快速的判断是否发生了逃逸分析，大家就看new的对象实体是否有可能在方法外被调用。
 */
public class EscapeAnalysis {

    public EscapeAnalysis obj;

    /*
    方法返回EscapeAnalysis对象，发生逃逸
     */
    public EscapeAnalysis getInstance(){
        return obj == null? new EscapeAnalysis() : obj;
    }
    /*
    为成员属性赋值，发生逃逸
     */
    public void setObj(){
        this.obj = new EscapeAnalysis();
    }
    //思考：如果当前的obj引用声明为static的？仍然会发生逃逸。

    /*
    对象的作用域仅在当前方法中有效，没有发生逃逸
     */
    public void useEscapeAnalysis(){
        EscapeAnalysis e = new EscapeAnalysis();
    }
    /*
    引用成员变量的值，发生逃逸
     */
    public void useEscapeAnalysis1(){
        EscapeAnalysis e = getInstance();
        //getInstance().xxx()同样会发生逃逸
    }
}
```

**逃逸分析参数设置**

1. 在JDK 1.7 版本之后，HotSpot中默认就已经开启了逃逸分析
2. 如果使用的是较早的版本，开发人员则可以通过：
   - 选项“-XX:+DoEscapeAnalysis"显式开启逃逸分析
   - 通过选项“-XX:+PrintEscapeAnalysis"查看逃逸分析的筛选结果

**总结**

开发中能使用局部变量的，就不要使用在方法外定义。

#### 代码优化

使用逃逸分析，编译器可以对代码做如下优化：

1. **栈上分配**：将堆分配转化为栈分配。如果一个对象在子程序中被分配，要使指向该对象的指针永远不会发生逃逸，对象可能是栈上分配的候选，而不是堆上分配
2. **同步省略**：如果一个对象被发现只有一个线程被访问到，那么对于这个对象的操作可以不考虑同步。
3. **分离对象或标量替换**：有的对象可能不需要作为一个连续的内存结构存在也可以被访问到，那么对象的部分（或全部）可以不存储在内存，而是存储在CPU寄存器中。

#### 栈上分配

1. JIT编译器在编译期间根据逃逸分析的结果，发现如果一个对象并没有逃逸出方法的话，就可能被优化成栈上分配。分配完成后，继续在调用栈内执行，最后线程结束，栈空间被回收，局部变量对象也被回收。这样就无须进行垃圾回收了。
2. 常见的栈上分配的场景：在逃逸分析中，已经说明了，分别是给成员变量赋值、方法返回值、实例引用传递。

#### 同步省略（同步消除）

1. 线程同步的代价是相当高的，同步的后果是降低并发性和性能。
2. 在动态编译同步块的时候，JIT编译器可以借助逃逸分析来**判断同步块所使用的锁对象是否只能够被一个线程访问而没有被发布到其他线程**。
3. 如果没有，那么JIT编译器在编译这个同步块的时候就会取消对这部分代码的同步。这样就能大大提高并发性和性能。这个**取消同步的过程就叫同步省略，也叫锁消除**。

例如下面的代码

```java
public void f() {
    Object hollis = new Object();
    synchronized(hollis) {
        System.out.println(hollis);
    }
}
```

代码中对hollis这个对象加锁，但是hollis对象的生命周期只在f()方法中，并不会被其他线程所访问到，所以在JIT编译阶段就会被优化掉，优化成：

```java
public void f() {
    Object hellis = new Object();
    System.out.println(hellis);
}
```

字节码文件中并没有进行优化，可以看到加锁和释放锁的操作依然存在，**同步省略操作是在解释运行时发生的**

#### 标量替换

**分离对象或标量替换**

1. 标量（scalar）是指一个无法再分解成更小的数据的数据。Java中的原始数据类型就是标量。
2. 相对的，那些还可以分解的数据叫做聚合量（Aggregate），Java中的对象就是聚合量，因为他可以分解成其他聚合量和标量。
3. 在JIT阶段，如果经过逃逸分析，发现一个对象不会被外界访问的话，那么经过JIT优化，就会把这个对象拆解成若干个其中包含的若干个成员变量来代替。这个过程就是标量替换。

**标量替换举例**

代码

```java
public static void main(String args[]) {
    alloc();
}
private static void alloc() {
    Point point = new Point(1,2);
    System.out.println("point.x" + point.x + ";point.y" + point.y);
}
class Point {
    private int x;
    private int y;
}
```

以上代码，经过标量替换后，就会变成

```java
private static void alloc() {
    int x = 1;
    int y = 2;
    System.out.println("point.x = " + x + "; point.y=" + y);
}
```

1. 可以看到，Point这个聚合量经过逃逸分析后，发现他并没有逃逸，就被替换成两个聚合量了。
2. 那么标量替换有什么好处呢？就是可以大大减少堆内存的占用。因为一旦不需要创建对象了，那么就不再需要分配堆内存了。
3. 标量替换为栈上分配提供了很好的基础。

**标量替换参数设置**

参数 -XX:+ElimilnateAllocations：开启了标量替换（默认打开），允许将对象打散分配在栈上。

#### 逃逸分析的不足

1. 关于逃逸分析的论文在1999年就已经发表了，但直到JDK1.6才有实现，而且这项技术到如今也并不是十分成熟的。
2. 其根本原因就是无法保证逃逸分析的性能消耗一定能高于他的消耗。虽然经过逃逸分析可以做标量替换、栈上分配、和锁消除。但是逃逸分析自身也是需要进行一系列复杂的分析的，这其实也是一个相对耗时的过程。
3. 一个极端的例子，就是经过逃逸分析之后，发现没有一个对象是不逃逸的。那这个逃逸分析的过程就白白浪费掉了。
4. 虽然这项技术并不十分成熟，但是它也是即时编译器优化技术中一个十分重要的手段。
5. 注意到有一些观点，认为通过逃逸分析，JVM会在栈上分配那些不会逃逸的对象，这在理论上是可行的，但是取决于JVM设计者的选择。据我所知，**Oracle Hotspot JVM中并未这么做**（刚刚演示的效果，是因为HotSpot实现了标量替换），这一点在逃逸分析相关的文档里已经说明，**所以可以明确在HotSpot虚拟机上，所有的对象实例都是创建在堆上**。
6. 目前很多书籍还是基于JDK7以前的版本，JDK已经发生了很大变化，intern字符串的缓存和静态变量曾经都被分配在永久代上，而永久代已经被元数据区取代。但是**intern字符串缓存和静态变量并不是被转移到元数据区，而是直接在堆上分配**，**所以这一点同样符合前面一点的结论：对象实例都是分配在堆上**。

## 方法区

### 方法区的理解

![img](深入理解Java虚拟机/0002-16516620314131.png)

#### 方法区在哪里？

1. 《Java虚拟机规范》中明确说明：尽管所有的方法区在逻辑上是属于堆的一部分，但一些简单的实现可能不会选择去进行垃圾收集或者进行压缩。但对于HotSpotJVM而言，方法区还有一个别名叫做Non-Heap（非堆），目的就是要和堆分开。
2. 所以，**方法区可以看作是一块独立于Java堆的内存空间**。

#### 方法区的基本理解

**方法区中主要存放的是Class，而堆中主要存放的是实例化对象**

1. 方法区（Method Area）与Java堆一样，是各个线程共享的内存区域。多个线程同时加载统一个类时，只能有一个线程能加载该类，其他线程只能等等待该线程加载完毕，然后直接使用该类，即类只能加载一次。

2. 方法区在JVM启动的时候被创建，并且它的实际的物理内存空间中和Java堆区一样都可以是不连续的。

3. 方法区的大小，跟堆空间一样，可以选择固定大小或者可扩展。

4. 方法区的大小决定了系统可以保存多少个类，如果系统定义了太多的类，导致方法区溢出，虚拟机同样会抛出内存溢出错误：

   ```
   java.lang.OutofMemoryError:PermGen space
   ```

   或者

   ```
   java.lang.OutOfMemoryError:Metaspace
   ```

   - 加载大量的第三方的jar包
   - Tomcat部署的工程过多（30~50个）
   - 大量动态的生成反射类

5. 关闭JVM就会释放这个区域的内存。

#### HotSpot方法区演进

1. 在 JDK7 及以前，习惯上把方法区，称为永久代。JDK8开始，使用元空间取代了永久代。我们可以将方法区类比为Java中的接口，将永久代或元空间类比为Java中具体的实现类
2. 本质上，方法区和永久代并不等价。仅是对Hotspot而言的可以看作等价。《Java虚拟机规范》对如何实现方法区，不做统一要求。例如：BEAJRockit / IBM J9 中不存在永久代的概念。
   - 现在来看，当年使用永久代，不是好的idea。导致Java程序更容易OOm（超过-XX:MaxPermsize上限）
3. 而到了JDK8，终于完全废弃了永久代的概念，改用与JRockit、J9一样在本地内存中实现的元空间（Metaspace）来代替
4. 元空间的本质和永久代类似，都是对JVM规范中方法区的实现。不过元空间与永久代最大的区别在于：**元空间不在虚拟机设置的内存中，而是使用本地内存**。
5. 永久代、元空间二者并不只是名字变了，内部结构也调整了
6. 根据《Java虚拟机规范》的规定，如果方法区无法满足新的内存分配需求时，将抛出OOM异常

#### 设置方法区大小

方法区的大小不必是固定的，JVM可以根据应用的需要动态调整。

##### JDK7及以前（永久代）

1. 通过-XX:Permsize来设置永久代初始分配空间。默认值是20.75M
2. -XX:MaxPermsize来设定永久代最大可分配空间。32位机器默认是64M，64位机器模式是82M
3. 当JVM加载的类信息容量超过了这个值，会报异常OutofMemoryError:PermGen space。

##### JDK8及以后（元空间）

1. 元数据区大小可以使用参数 **-XX:MetaspaceSize** 和 **-XX:MaxMetaspaceSize** 指定
2. 默认值依赖于平台，Windows下，-XX:MetaspaceSize 约为21M，-XX:MaxMetaspaceSize的值是-1，即没有限制。
3. 与永久代不同，如果不指定大小，默认情况下，虚拟机会耗尽所有的可用系统内存。如果元数据区发生溢出，虚拟机一样会抛出异常OutOfMemoryError:Metaspace
4. -XX:MetaspaceSize：设置初始的元空间大小。对于一个 64位 的服务器端 JVM 来说，其默认的 -XX:MetaspaceSize值为21MB。这就是初始的高水位线，一旦触及这个水位线，Full GC将会被触发并卸载没用的类（即这些类对应的类加载器不再存活），然后这个高水位线将会重置。新的高水位线的值取决于GC后释放了多少元空间。如果释放的空间不足，那么在不超过MaxMetaspaceSize时，适当提高该值。如果释放空间过多，则适当降低该值。
5. 如果初始化的高水位线设置过低，上述高水位线调整情况会发生很多次。通过垃圾回收器的日志可以观察到Full GC多次调用。为了避免频繁地GC，建议将-XX:MetaspaceSize设置为一个相对较高的值。

##### 如何解决OOM

1. 要解决OOM异常或heap space的异常，一般的手段是首先通过内存映像分析工具（如Ec1ipse Memory Analyzer）对dump出来的堆转储快照进行分析，重点是确认内存中的对象是否是必要的，也就是要先分清楚到底是出现了内存泄漏（Memory Leak）还是内存溢出（Memory Overflow）
2. **内存泄漏**就是有大量的引用指向某些对象，但是这些对象以后不会使用了，但是因为它们还和GC ROOT有关联，所以导致以后这些对象也不会被回收，这就是内存泄漏的问题
3. 如果是内存泄漏，可进一步通过工具查看泄漏对象到GC Roots的引用链。于是就能找到泄漏对象是通过怎样的路径与GC Roots相关联并导致垃圾收集器无法自动回收它们的。掌握了泄漏对象的类型信息，以及GC Roots引用链的信息，就可以比较准确地定位出泄漏代码的位置。
4. 如果不存在内存泄漏，换句话说就是内存中的对象确实都还必须存活着，那就应当检查虚拟机的堆参数（-Xmx与-Xms），与机器物理内存对比看是否还可以调大，从代码上检查是否存在某些对象生命周期过长、持有状态时间过长的情况，尝试减少程序运行期的内存消耗。

### 方法区的内部结构

#### 方法区存储什么

![image-20210830104459102](深入理解Java虚拟机/image-20210830104459102.png)

《深入理解Java虚拟机》书中对方法区（Method Area）存储内容描述如下：它用于存储已被虚拟机加载的**类型信息、常量、静态变量、即时编译器编译后的代码缓存**等。

![image-20210830104538625](深入理解Java虚拟机/image-20210830104538625.png)

**类型信息**

对每个加载的类型（类class、接口interface、枚举enum、注解annotation），JVM必须在方法区中存储以下类型信息：

1. 这个类型的完整有效名称（全名=包名.类名）
2. 这个类型直接父类的完整有效名（对于interface或是java.lang.Object，都没有父类）
3. 这个类型的修饰符（public，abstract，final的某个子集）
4. 这个类型直接接口的一个有序列表

**域（Field）信息**

> 也就是我们常说的成员变量，域信息是比较官方的称呼

1. JVM必须在方法区中保存类型的所有域的相关信息以及域的声明顺序。
2. 域的相关信息包括：域名称，域类型，域修饰符（public，private，protected，static，final，volatile，transient的某个子集）

**方法（Method）信息**

JVM必须保存所有方法的以下信息，同域信息一样包括声明顺序：

1. 方法名称
2. 方法的返回类型（包括 void 返回类型），void 在 Java 中对应的为 void.class
3. 方法参数的数量和类型（按顺序）
4. 方法的修饰符（public，private，protected，static，final，synchronized，native，abstract的一个子集）
5. 方法的字节码（bytecodes）、操作数栈、局部变量表及大小（abstract和native方法除外）
6. 异常表（abstract和native方法除外），异常表记录每个异常处理的开始位置、结束位置、代码处理在程序计数器中的偏移地址、被捕获的异常类的常量池索引

#### non-final类型的类变量

1. 静态变量和类关联在一起，随着类的加载而加载，他们成为类数据在逻辑上的一部分
2. 类变量被类的所有实例共享，即使没有类实例时，你也可以访问它

**举例**

1. 如下代码所示，即使我们把order设置为null，也不会出现空指针异常
2. 这更加表明了 static 类型的字段和方法随着类的加载而加载，并不属于特定的类实例

```java
public class MethodAreaTest {
    public static void main(String[] args) {
        Order order = null;
        order.hello();
        System.out.println(order.count);
    }
}

class Order {
    public static int count = 1;
    public static final int number = 2;


    public static void hello() {
        System.out.println("hello!");
    }
}
```

输出结果：

```
hello!
1
```

##### 全局常量：static final

1. 全局常量就是使用 static final 进行修饰
2. 被声明为final的类变量的处理方法则不同，每个全局常量在编译的时候就会被分配了。

查看上面代码，这部分的字节码指令

```java
class Order {
    public static int count = 1;
    public static final int number = 2;
    ...
}    
public static int count;
    descriptor: I
    flags: ACC_PUBLIC, ACC_STATIC

  public static final int number;
    descriptor: I
    flags: ACC_PUBLIC, ACC_STATIC, ACC_FINAL
    ConstantValue: int 2
```

可以发现 staitc和final同时修饰的number 的值在编译上的时候已经写死在字节码文件中了。

#### 运行时常量池

##### 运行时常量池VS常量池

1. 方法区，内部包含了运行时常量池
2. 字节码文件，内部包含了常量池。（之前的字节码文件中已经看到了很多Constant pool的东西，这个就是常量池）
3. 要弄清楚方法区，需要理解清楚ClassFile，因为加载类的信息都在方法区。
4. 要弄清楚方法区的运行时常量池，需要理解清楚ClassFile中的常量池。

##### 常量池

1. 一个有效的字节码文件中除了包含类的版本信息、字段、方法以及接口等描述符信息外。还包含一项信息就是**常量池表**（**Constant Pool Table**），包括各种字面量和对类型、域和方法的符号引用。
2. 字面量： 10 ， “我是某某”这种数字和字符串都是字面量

![image-20210830105414117](深入理解Java虚拟机/image-20210830105414117.png)

**为什么需要常量池？**

1. 一个java源文件中的类、接口，编译后产生一个字节码文件。而Java中的字节码需要数据支持，通常这种数据会很大以至于不能直接存到字节码里，换另一种方式，可以存到常量池。这个字节码包含了指向常量池的引用。在动态链接的时候会用到运行时常量池，之前有介绍

比如：如下的代码：

```java
public class SimpleClass {
    public void sayHello() {
        System.out.println("hello");
    }
}
```

1. 虽然上述代码只有194字节，但是里面却使用了String、System、PrintStream及Object等结构。
2. 比如说我们这个文件中有6个地方用到了"hello"这个字符串，如果不用常量池，就需要在6个地方全写一遍，造成臃肿。我们可以将"hello"等所需用到的结构信息记录在常量池中，并通过**引用的方式**，来加载、调用所需的结构
3. 这里的代码量其实很少了，如果代码多的话，引用的结构将会更多，这里就需要用到常量池了。

**常量池中有啥？**

1. 数量值
2. 字符串值
3. 类引用
4. 字段引用
5. 方法引用

**常量池总结**

常量池、可以看做是一张表，虚拟机指令根据这张常量表找到要执行的类名、方法名、参数类型、字面量等类型。

##### 运行时常量池

1. 运行时常量池（Runtime Constant Pool）是方法区的一部分。
2. 常量池表（Constant Pool Table）是Class字节码文件的一部分，用于存放编译期生成的各种字面量与符号引用，**这部分内容将在类加载后存放到方法区的运行时常量池中**。（运行时常量池就是常量池在程序运行时的称呼）
3. 运行时常量池，在加载类和接口到虚拟机后，就会创建对应的运行时常量池。
4. JVM为每个已加载的类型（类或接口）都维护一个常量池。池中的数据项像数组项一样，是通过索引访问的。
5. 运行时常量池中包含多种不同的常量，包括编译期就已经明确的数值字面量，也包括到运行期解析后才能够获得的方法或者字段引用。**此时不再是常量池中的符号地址了，这里换为真实地址**。

- 运行时常量池，相对于Class文件常量池的另一重要特征是：具备动态性。

1. 运行时常量池类似于传统编程语言中的符号表（symbol table），但是它所包含的数据却比符号表要更加丰富一些。
2. 当创建类或接口的运行时常量池时，如果构造运行时常量池所需的内存空间超过了方法区所能提供的最大值，则JVM会抛OutofMemoryError异常。

### 方法区演进细节

#### 永久代演进过程

1. 首先明确：只有Hotspot才有永久代。BEA JRockit、IBMJ9等来说，是不存在永久代的概念的。原则上如何实现方法区属于虚拟机实现细节，不受《Java虚拟机规范》管束，并不要求统一
2. Hotspot中方法区的变化：

| JDK1.6及以前 | 有永久代（permanent generation），静态变量存储在永久代上     |
| ------------ | ------------------------------------------------------------ |
| JDK1.7       | 有永久代，但已经逐步 “去永久代”，**字符串常量池，静态变量移除，保存在堆中** |
| JDK1.8       | 无永久代，类型信息，字段，方法，常量保存在本地内存的元空间，但字符串常量池、静态变量仍然在堆中。 |

**JDK6**

方法区由永久代实现，使用 JVM 虚拟机内存（虚拟的内存）

![image-20210830110159922](深入理解Java虚拟机/image-20210830110159922.png)

**JDK7**

方法区由永久代实现，使用 JVM 虚拟机内存

![image-20210830110221095](深入理解Java虚拟机/image-20210830110221095.png)

**JDK8**

方法区由元空间实现，使用物理机本地内存

![image-20210830110251768](深入理解Java虚拟机/image-20210830110251768.png)

#### 永久代为什么要被元空间替代

1. 随着Java8的到来，HotSpot VM中再也见不到永久代了。但是这并不意味着类的元数据信息也消失了。这些数据被移到了一个与堆不相连的本地内存区域，这个区域叫做元空间（Metaspace）。

2. 由于类的元数据分配在本地内存中，元空间的最大可分配空间就是系统可用内存空间。

3. 这项改动是很有必要的，原因有：

   1. 为永久代设置空间大小是很难确定的。在某些场景下，如果动态加载类过多，容易产生Perm区的OOM。比如某个实际Web工程中，因为功能点比较多，在运行过程中，要不断动态加载很多类，经常出现致命错误。`Exception in thread 'dubbo client x.x connector' java.lang.OutOfMemoryError:PermGen space`而元空间和永久代之间最大的区别在于：元空间并不在虚拟机中，而是使用本地内存。 因此，默认情况下，元空间的大小仅受本地内存限制。

   2. 对永久代进行调优是很困难的。方法区的垃圾收集主要回收两部分内容：常量池中废弃的常量和不再用的类型，方法区的调优主要是为了降低Full GC

      1. 有些人认为方法区（如HotSpot虚拟机中的元空间或者永久代）是没有垃圾收集行为的，其实不然。《Java虚拟机规范》对方法区的约束是非常宽松的，提到过可以不要求虚拟机在方法区中实现垃圾收集。事实上也确实有未实现或未能完整实现方法区类型卸载的收集器存在（如JDK11时期的ZGC收集器就不支持类卸载）。

4. 一般来说这个区域的回收效果比较难令人满意，尤其是类型的卸载，条件相当苛刻。但是这部分区域的回收有时又确实是必要的。以前Sun公司的Bug列表中，曾出现过的若干个严重的Bug就是由于低版本的HotSpot虚拟机对此区域未完全回收而导致内存泄漏。

#### 字符串常量池

**字符串常量池 StringTable 为什么要调整位置？**

- JDK7中将StringTable放到了堆空间中。因为永久代的回收效率很低，在Full GC的时候才会执行永久代的垃圾回收，而Full GC是老年代的空间不足、永久代不足时才会触发。
- 这就导致StringTable回收效率不高，而我们开发中会有大量的字符串被创建，回收效率低，导致永久代内存不足。放到堆里，能及时回收内存。

#### 静态变量放在哪

从《Java虚拟机规范》所定义的概念模型来看，所有Class相关的信息都应该存放在方法区之中，但方法区该如何实现，《Java虚拟机规范》并未做出规定，这就成了一件允许不同虚拟机自己灵活把握的事情。JDK7及其以后版本的HotSpot虚拟机选择把静态变量与类型在Java语言一端的映射Class对象存放在一起，**存储于Java堆之中**。

### 方法区的垃圾回收

1. 有些人认为方法区（如Hotspot虚拟机中的元空间或者永久代）是没有垃圾收集行为的，其实不然。《Java虚拟机规范》对方法区的约束是非常宽松的，提到过可以不要求虚拟机在方法区中实现垃圾收集。事实上也确实有未实现或未能完整实现方法区**类型卸载**的收集器存在（如JDK11时期的ZGC收集器就不支持类卸载）。

2. 一般来说这个区域的回收效果比较难令人满意，尤其是类型的卸载，条件相当苛刻。但是这部分区域的回收有时又确实是必要的。以前sun公司的Bug列表中，曾出现过的若干个严重的Bug就是由于低版本的HotSpot虚拟机对此区域未完全回收而导致内存泄漏。

3. 方法区的垃圾收集主要回收两部分内容：**常量池中废弃的常量和不再使用的类型**。

4. 先来说说方法区内常量池之中主要存放的两大类常量：字面量和符号引用。字面量比较接近Java语言层次的常量概念，如文本字符串、被声明为final的常量值等。而符号引用则属于编译原理方面的概念，包括下面三类常量：

   - 类和接口的全限定名
   - 字段的名称和描述符
   - 方法的名称和描述符

5. HotSpot虚拟机对常量池的回收策略是很明确的，只要常量池中的常量没有被任何地方引用，就可以被回收。

6. 回收废弃常量与回收Java堆中的对象非常类似。（关于常量的回收比较简单，重点是类的回收）

下面也称作**类卸载**

1、判定一个常量是否“废弃”还是相对简单，而要判定一个类型是否属于“不再被使用的类”的条件就比较苛刻了。需要同时满足下面三个条件：

- 该类所有的实例都已经被回收，也就是Java堆中不存在该类及其任何派生子类的实例。
- 加载该类的类加载器已经被回收，这个条件除非是经过精心设计的可替换类加载器的场景，如OSGi、JSP的重加载等，否则通常是很难达成的。
- 该类对应的java.lang.Class对象没有在任何地方被引用，无法在任何地方通过反射访问该类的方法。

2、Java虚拟机被允许对满足上述三个条件的无用类进行回收，这里说的仅仅是“被允许”，而并不是和对象一样，没有引用了就必然会回收。关于是否要对类型进行回收，HotSpot虚拟机提供了`-Xnoclassgc`参数进行控制，还可以使用`-verbose:class` 以及 `-XX：+TraceClass-Loading`、`-XX：+TraceClassUnLoading`查看类加载和卸载信息

3、在大量使用反射、动态代理、CGLib等字节码框架，动态生成JSP以及OSGi这类频繁自定义类加载器的场景中，通常都需要Java虚拟机具备类型卸载的能力，以保证不会对方法区造成过大的内存压力。

![image-20210830124820775](深入理解Java虚拟机/image-20210830124820775.png)

# 第三章 对象的实例化内存布局与访问定位

## 对象的实例化

![image-20210830125356389](深入理解Java虚拟机/image-20210830125356389.png)

## 对象创建的方式和步骤

![image-20210830125950217](深入理解Java虚拟机/image-20210830125950217.png)

1. new：最常见的方式、单例类中调用getInstance的静态类方法，XXXFactory的静态方法
2. Class的newInstance方法：在JDK9里面被标记为过时的方法，因为只能调用空参构造器，并且权限必须为 public
3. Constructor的newInstance(Xxxx)：反射的方式，可以调用空参的，或者带参的构造器
4. 使用clone()：不调用任何的构造器，要求当前的类需要实现Cloneable接口中的clone方法
5. 使用序列化：从文件中，从网络中获取一个对象的二进制流，序列化一般用于Socket的网络传输
6. 第三方库 Objenesis

> **从字节码看待对象的创建过程**

```java
public class ObjectTest {
    public static void main(String[] args) {
        Object obj = new Object();
    }
}

public static void main(java.lang.String[]);
    descriptor: ([Ljava/lang/String;)V
    flags: ACC_PUBLIC, ACC_STATIC
    Code:
      stack=2, locals=2, args_size=1
         0: new           #2                  // class java/lang/Object
         3: dup           
         4: invokespecial #1                  // Method java/lang/Object."<init>":()V
         7: astore_1
         8: return
      LineNumberTable:
        line 9: 0
        line 10: 8
      LocalVariableTable:
        Start  Length  Slot  Name   Signature
            0       9     0  args   [Ljava/lang/String;
            8       1     1   obj   Ljava/lang/Object;
}
```

**1、判断对象对应的类是否加载、链接、初始化**

1. 虚拟机遇到一条new指令，首先去检查这个指令的参数能否在Metaspace的常量池中定位到一个类的符号引用，并且检查这个符号引用代表的类是否已经被加载，解析和初始化。（即判断类元信息是否存在）。
2. 如果该类没有加载，那么在双亲委派模式下，使用当前类加载器以ClassLoader + 包名 + 类名为key进行查找对应的.class文件，如果没有找到文件，则抛出ClassNotFoundException异常，如果找到，则进行类加载，并生成对应的Class对象。

**2、为对象分配内存**

1. 首先计算对象占用空间的大小，接着在堆中划分一块内存给新对象。如果实例成员变量是引用变量，仅分配引用变量空间即可，即4个字节大小
2. 如果内存规整：采用指针碰撞分配内存
   - 如果内存是规整的，那么虚拟机将采用的是指针碰撞法（Bump The Point）来为对象分配内存。
   - 意思是所有用过的内存在一边，空闲的内存放另外一边，中间放着一个指针作为分界点的指示器，分配内存就仅仅是把指针往空闲内存那边挪动一段与对象大小相等的距离罢了。
   - 如果垃圾收集器选择的是Serial ，ParNew这种基于压缩算法的，虚拟机采用这种分配方式。一般使用带Compact（整理）过程的收集器时，使用指针碰撞。
   - 标记压缩（整理）算法会整理内存碎片，堆内存一存对象，另一边为空闲区域
3. 如果内存不规整
   - 如果内存不是规整的，已使用的内存和未使用的内存相互交错，那么虚拟机将采用的是空闲列表来为对象分配内存。
   - 意思是虚拟机维护了一个列表，记录上哪些内存块是可用的，再分配的时候从列表中找到一块足够大的空间划分给对象实例，并更新列表上的内容。这种分配方式成为了 “空闲列表（Free List）”
   - 选择哪种分配方式由Java堆是否规整所决定，而Java堆是否规整又由所采用的垃圾收集器是否带有压缩整理功能决定
   - 标记清除算法清理过后的堆内存，就会存在很多内存碎片。

**3、处理并发问题**

1. 采用CAS+失败重试保证更新的原子性
2. 每个线程预先分配TLAB - 通过设置 -XX:+UseTLAB参数来设置（区域加锁机制）
3. 在Eden区给每个线程分配一块区域

**4、初始化分配到的空间**

- 所有属性设置默认值，保证对象实例字段在不赋值可以直接使用
- 给对象属性赋值的顺序：

1. 属性的默认值初始化
2. 显示初始化/代码块初始化（并列关系，谁先谁后看代码编写的顺序）
3. 构造器初始化

**5、设置对象的对象头**

将对象的所属类（即类的元数据信息）、对象的HashCode和对象的GC信息、锁信息等数据存储在对象的对象头中。这个过程的具体设置方式取决于JVM实现。

**6、执行init方法进行初始化**

1. 在Java程序的视角看来，初始化才正式开始。初始化成员变量，执行实例化代码块，调用类的构造方法，并把堆内对象的首地址赋值给引用变量
2. 因此一般来说（由字节码中跟随invokespecial指令所决定），new指令之后会接着就是执行init方法，把对象按照程序员的意愿进行初始化，这样一个真正可用的对象才算完成创建出来。

## 内存布局

![image-20210830130203892](深入理解Java虚拟机/image-20210830130203892.png)

```java
public class Customer{
    int id = 1001;
    String name;
    Account acct;

    {
        name = "匿名客户";
    }
    public Customer(){
        acct = new Account();
    }
    public static void main(String[] args) {
        Customer cust = new Customer();
    }
}
class Account{

}
```

![image-20210830130225530](深入理解Java虚拟机/image-20210830130225530.png)

## 对象的访问内存

**JVM是如何通过栈帧中的对象引用访问到其内部的对象实例呢？**

定位，通过栈上reference访问

**对象的两种访问方式：句柄访问和直接指针**

**1、句柄访问**

1. 缺点：在堆空间中开辟了一块空间作为句柄池，句柄池本身也会占用空间；通过两次指针访问才能访问到堆中的对象，效率低
2. 优点：reference中存储稳定句柄地址，对象被移动（垃圾收集时移动对象很普遍）时只会改变句柄中实例数据指针即可，reference本身不需要被修改

![image-20210830130431500](深入理解Java虚拟机/image-20210830130431500.png)

**2、直接指针（HotSpot采用）**

1. 优点：直接指针是局部变量表中的引用，直接指向堆中的实例，在对象实例中有类型指针，指向的是方法区中的对象类型数据
2. 缺点：对象被移动（垃圾收集时移动对象很普遍）时需要修改 reference 的值

![image-20210830130447943](深入理解Java虚拟机/image-20210830130447943.png)

# 第四章 直接内存

## 直接内存概述（了解）

1. 不是虚拟机运行时数据区的一部分，也不是《Java虚拟机规范》中定义的内存区域。
2. 直接内存是在Java堆外的、直接向系统申请的内存区间。
3. 来源于NIO，通过存在堆中的DirectByteBuffer操作Native内存
4. 通常，访问直接内存的速度会优于Java堆。即读写性能高。
5. 因此出于性能考虑，读写频繁的场合可能会考虑使用直接内存。
6. Java的NIO库允许Java程序使用直接内存，用于数据缓冲区

## BIO与NIO

**非直接缓存区（BIO）**

原来采用BIO的架构，在读写本地文件时，我们需要从用户态切换成内核态

![img](深入理解Java虚拟机/0038.png)

**直接缓冲区（NIO）**

NIO 直接操作物理磁盘，省去了中间过程

![img](深入理解Java虚拟机/0039.png)

## 直接内存与 OOM

1. 直接内存也可能导致OutofMemoryError异常
2. 由于直接内存在Java堆外，因此它的大小不会直接受限于-Xmx指定的最大堆大小，但是系统内存是有限的，Java堆和直接内存的总和依然受限于操作系统能给出的最大内存。
3. 直接内存的缺点为：
   - 分配回收成本较高
   - 不受JVM内存回收管理
4. 直接内存大小可以通过MaxDirectMemorySize设置
5. 如果不指定，默认与堆的最大值-Xmx参数值一致

# 第五章 执行引擎

## 执行引擎概述

![image-20210904154047916](深入理解Java虚拟机/image-20210904154047916.png)

1. 执行引擎是Java虚拟机核心的组成部分之一
2. “虚拟机”是要给相对于“物理机”的概念，这两种机器都有代码执行能力，其区别是物理机的执行能力，其区别是物理机的执行引擎是直接建立在处理器、缓存、指令集和操作系统层面上的。而**虚拟机的执行引擎则是由软件自行实现的**，因此可以不受物理条件制约地定制指令集与执行引擎的结构体系，**能够执行那些不被硬件直接支持的指令集格式**。
3. JVM的主要任务是负责**装载字节码到其内部**，但字节码并不能够直接运行在操作系统之上，因为字节码指令并非等价于本地机器指令，它内部包含的仅仅只是一些能够被JVM所识别的字节码指令、符号表，以及其他辅助信息。
4. 那么，如果想要让一个Java程序运行起来，执行引擎（Execution Engine）的任务就是**将字节码指令解释/编译为对应平台上的本地机器指令才可以**。简单来说，JVM中的执行引擎充当了将高级语言翻译为机器语言的译者。

### 执行引擎工作过程

![img](深入理解Java虚拟机/0003.png)

1. 执行引擎在执行的过程中究竟需要执行什么样的字节码指令完全依赖于PC寄存器。
2. 每当执行完一项指令操作后，PC寄存器就会更新下一条需要被执行的指令地址。
3. 当然方法在执行的过程中，执行引擎有可能会通过存储在局部变量表中的对象引用准确定位到存储在Java堆区中的对象实例信息，以及通过对象头中的元数据指针定位到目标对象的类型信息。
4. 从外观上来看，所有的Java虚拟机的执行引擎输入、处理、输出都是一致的：输入的是字节码二进制流，处理过程是字节码解析执行、即时编译的等效过程，输出的是执行过程。

## Java代码编译和执行过程

![image-20210905112032806](深入理解Java虚拟机/image-20210905112032806.png)

大部分的程序代码转换成物理机的目标代码或虚拟机能执行的指令集之前，都需要经过上图中的各个步骤。

1. 前面橙色部分是编译生成字节码文件的过程（javac编译器来完成，也就是前端编译器），和JVM没有关系
2. 后面绿色（解释执行）和蓝色（即时编译）才是JVM需要考虑的过程。

Java代码编译是由Java源码编译器来完成，也就是上图橙色所属，流程图如下所示：

![image-20210905112236173](深入理解Java虚拟机/image-20210905112236173.png)

Java字节码的执行是由JVM执行引擎来完成，流程图如下图所示：

![image-20210905112340252](深入理解Java虚拟机/image-20210905112340252.png)

### 什么是解释器？什么是JIT编译器？

1. 解释器：当Java虚拟机启动时会根据预定义的规范对字节码采用**逐行**解释的方式**执行**，将每条字节码文件中的内容“翻译”为对应平台的本地机器指令执行。
2. JIT编译器：也叫即时编译器，就是虚拟机将源代码**一次性直接**编译成和本地机器平台相关的机器语言，**但并不是马上执行**。

**为什么Java是半编译半解释型语言？**

1. JDK1.0时代，将Java语言定位为“解释执行”开始比较准确的。再后来，Java也发展出可以直接生成本地代码的编译器。
2. 现在JVM在执行Java代码的时候，通常都会将解释执行与编译执行二者结合起来进行。
3. JIT编译器将字节码翻译成本地代码后，就可以做一个缓存操作，存储在方法区的JIT代码缓存中（执行效率更高了），并且在翻译成本地代码的过程中可以做优化。

![image-20210905113306187](深入理解Java虚拟机/image-20210905113306187.png)

## 解释器

### 为什么要有解释器

1. JVM设计者们的初衷仅仅只是单纯地为了满足Java程序实现跨平台特性，因此避免采用静态编译的方式由高级语言直接生成本地机器指令，从而诞生了实现解释器在运行时采用逐行解释字节码执行程序的想法（也就是产生了一个中间产品**字节码**）
2. 解释器真正的意义上所承担的角色就是一个运行时“翻译者”，将字节码文件中的内容“翻译”为对应平台的本地机器指令执行。
3. 当一条字节码指令被解释执行完成后，接着再根据PC寄存器中记录的下一条需要被执行的字节码指令执行解释操作。

### 解释器的分类

1. 在Java的发展历史里，一共有两套解释执行器，即古老的**字节码解释器**、现在普遍使用的**模板解释器**
   - 字节码解释器在执行时通过纯软件代码模拟字节码的执行，效率非常低下。
   - 而模板解释器将每一条字节码和一个模板函数相关联，模板函数中直接产生这条字节码执行时的机器码，从而很大程度上提高了解释器的性能。
2. 在HotSpot VM中，解释器主要由Interpreter模块和Code模块构成。
   - Interpreter模块：实现了解释器的核心功能
   - Code模块：用于管理HotSpot VM在运行时生成的本地机器指令

### 解释器的现状

1. 由于解释器在设计和实现上非常简单，因此除了Java语言之外，还有许多高级语言同样也是基于解释器执行的，比如Python、Perl、Ruby等。但是在今天，基于解释器执行已经沦落为低效的代名词，并且时常被一些C/C++程序员所调侃。
2. 为了解决这个问题，JVM平台支持一种叫作即时编译的技术。即时编译的目的是避免函数被解释执行，而是将整个函数体编译成为机器码，每次函数执行时，只执行编译后的机器码即可，这种方式可以使执行效率大幅度提升。
3. 不过无论如何，基于解释器的执行模式仍然为中间语言的发展做出了不可磨灭的贡献。

## JIT编译器

### Java代码执行的分类

1. 第一种是将源代码编译成字节码文件，然后在运行时通过解释器将字节码文件转为机器码执行

2. 第二种是编译执行（直接编译成机器码）。现代虚拟机为了提高执行效率，会使用即时编译技术（JIT，Just In Time）将方法编译成机器码后再执行

3. HotSpot VM是目前市面上高性能虚拟机的代表作之一。**它采用解释器与即时编译器并存的架构**。在Java虚拟机运行时，解释器和即时编译器能够相互协作，各自取长补短，尽力去选择最合适的方式来权衡编译本地代码的时间和直接解释执行代码的时间。

4. 在今天，Java程序的运行性能早已脱胎换骨，已经达到了可以和C/C++ 程序一较高下的地步。

### 为什么我们还需要解释器

1. 有些开发人员会感觉到诧异，既然HotSpot VM中已经内置JIT编译器了，那么为什么还需要再使用解释器来“拖累”程序的执行性能呢？比如JRockit VM内部就不包含解释器，字节码全部都依靠即时编译器编译后执行。
2. JRockit虚拟机是砍掉了解释器，也就是只采及时编译器。那是因为呢JRockit只部署在服务器上，一般已经有时间让他进行指令编译的过程了，对于响应来说要求不高，等及时编译器的编译完成后，就会提供更好的性能

**首先明确两点：**

1. 当程序启动后，解释器可以马上发挥作用，**响应速度快**，省去编译的时间，立即执行。
2. 编译器要想发挥作用，把代码编译成本地代码，**需要一定的执行时间**，但编译为本地代码后，执行效率高。

**所以：**

1. 尽管JRockit VM中程序的执行性能会非常高效，但程序在启动时必然需要花费更长的时间来进行编译。对于服务端应用来说，启动时间并非是关注重点，但对于那些看中启动时间的应用场景而言，或许就需要采用解释器与即时编译器并存的架构来换取一个平衡点。
2. 在此模式下，在Java虚拟器启动时，解释器可以首先发挥作用，而不必等待即时编译器全部编译完成后再执行，这样可以省去许多不必要的编译时间。随着时间的推移，编译器发挥作用，把越来越多的代码编译成本地代码，获得更高的执行效率。
3. 同时，解释执行在编译器进行激进优化不成立的时候，作为编译器的“逃生门”（后备方案）。

### 案例

- 当虚拟机启动的时候，解释器可以首先发挥作用，而不必等待即时编译器全部编译完成再执行，这样可以省去许多不必要的编译时间。随着程序运行时间的推移，即时编译器逐渐发挥作用，根据热点探测功能，将有价值的字节码编译为本地机器指令，以换取更高的程序执行效率。

1. 注意解释执行与编译执行在线上环境微妙的辩证关系。**机器在热机状态（已经运行了一段时间叫热机状态）可以承受的负载要大于冷机状态（刚启动的时候叫冷机状态）**。如果以热机状态时的流量进行切流，可能使处于冷机状态的服务器因无法承载流量而假死。
2. 在生产环境发布过程中，以分批的方式进行发布，根据机器数量划分成多个批次，每个批次的机器数至多占到整个集群的1/8。曾经有这样的故障案例：某程序员在发布平台进行分批发布，在输入发布总批数时，误填写成分为两批发布。如果是热机状态，在正常情况下一半的机器可以勉强承载流量，但由于刚启动的JVM均是解释执行，还没有进行热点代码统计和JIT动态编译，导致机器启动之后，当前1/2发布成功的服务器马上全部宕机，此故障说明了JIT的存在。—**阿里团队**

### JIT编译器相关概念

1. Java 语言的“编译期”其实是一段“不确定”的操作过程，因为它可能是指一个前端编译器（其实叫“编译器的前端”更准确一些）把.java文件转变成.class文件的过程。
2. 也可能是指虚拟机的后端运行期编译器（JIT编译器，Just In Time Compiler）把字节码转变成机器码的过程。
3. 还可能是指使用静态提前编译器（AOT编译器，Ahead of Time Compiler）直接把.java文件编译成本地机器代码的过程。（可能是后续发展的趋势）

**典型的编译器：**

1. 前端编译器：Sun的javac、Eclipse JDT中的增量式编译器（ECJ）。
2. JIT编译器：HotSpot VM的C1、C2编译器。
3. AOT 编译器：GNU Compiler for the Java（GCJ）、Excelsior JET。

### 热点代码及探测方式

1. 当然是否需要启动JIT编译器将字节码直接编译为对应平台的本地机器指令，则需要根据代码被调用**执行的频率**而定。
2. 关于那些需要被编译为本地代码的字节码，也被称之为**“热点代码”**，JIT编译器在运行时会针对那些频繁被调用的“热点代码”做出**深度优化**，将其直接编译为对应平台的本地机器指令，以此提升Java程序的执行性能。
3. 一个被多次调用的方法，或者是一个方法体内部循环次数较多的循环体都可以被称之为“热点代码”，因此都可以通过JIT编译器编译为本地机器指令。由于这种编译方式发生在方法的执行过程中，因此也被称之为栈上替换，或简称为OSR (On StackReplacement)编译。
4. 一个方法究竟要被调用多少次，或者一个循环体究竟需要执行多少次循环才可以达到这个标准？必然需要一个明确的阈值，JIT编译器才会将这些“热点代码”编译为本地机器指令执行。这里主要依靠热点探测功能。
5. **目前HotSpot VM所采用的热点探测方式是基于计数器的热点探测**。
6. 采用基于计数器的热点探测，HotSpot VM将会为每一个方法都建立2个不同类型的计数器，分别为方法调用计数器（Invocation Counter）和回边计数器（Back Edge Counter）。
   1. 方法调用计数器用于统计方法的调用次数
   2. 回边计数器则用于统计循环体执行的循环次数

#### 方法调用计数器

1. 这个计数器就用于统计方法被调用的次数，它的默认阀值在Client模式下是1500次，在Server模式下是10000次。超过这个阈值，就会触发JIT编译。
2. 这个阀值可以通过虚拟机参数 -XX:CompileThreshold 来人为设定。
3. 当一个方法被调用时，会先检查该方法是否存在被JIT编译过的版本
   - 如果存在，则优先使用编译后的本地代码来执行
   - 如果不存在已被编译过的版本，则将此方法的调用计数器值加1，然后判断方法调用计数器与回边计数器值之和是否超过方法调用计数器的阀值。
     - 如果已超过阈值，那么将会向即时编译器提交一个该方法的代码编译请求。
     - 如果未超过阈值，则使用解释器对字节码文件解释执行

![img](深入理解Java虚拟机/0013.png)

#### 热度衰减

1. 如果不做任何设置，方法调用计数器统计的并不是方法被调用的绝对次数，而是一个相对的执行频率，即**一段时间之内方法被调用的次数**。当超过一定的时间限度，如果方法的调用次数仍然不足以让它提交给即时编译器编译，那这个方法的调用计数器就会被减少一半，这个过程称为方法调用计数器热度的衰减（Counter Decay），而这段时间就称为此方法统计的半衰周期（Counter Half Life Time）（半衰周期是化学中的概念，比如出土的文物通过查看C60来获得文物的年龄）
2. 进行热度衰减的动作是在虚拟机进行垃圾收集时顺便进行的，可以使用虚拟机参数 -XX:-UseCounterDecay 来关闭热度衰减，让方法计数器统计方法调用的绝对次数，这样的话，只要系统运行时间足够长，绝大部分方法都会被编译成本地代码。
3. 另外，可以使用-XX:CounterHalfLifeTime参数设置半衰周期的时间，单位是秒。

#### 回边计数器

它的作用是统计一个方法中循环体代码执行的次数，在字节码中遇到控制流向后跳转的指令称为“回边”（Back Edge）。显然，建立回边计数器统计的目的就是为了触发OSR编译。

![image-20210905140917696](深入理解Java虚拟机/image-20210905140917696.png)

### HotSpotVM可以设置程序执行方法

缺省情况下HotSpot VM是采用解释器与即时编译器并存的架构，当然开发人员可以根据具体的应用场景，通过命令显式地为Java虚拟机指定在运行时到底是完全采用解释器执行，还是完全采用即时编译器执行。如下所示：

1. -Xint：完全采用解释器模式执行程序；
2. -Xcomp：完全采用即时编译器模式执行程序。如果即时编译出现问题，解释器会介入执行
3. -Xmixed：采用解释器+即时编译器的混合模式共同执行程序。

![img](深入理解Java虚拟机/0015.png)

**HotSpotVM JIT 分类**

在HotSpot VM中内嵌有两个JIT编译器，分别为Client Compiler和Server Compiler，但大多数情况下我们简称为C1编译器 和 C2编译器。开发人员可以通过如下命令显式指定Java虚拟机在运行时到底使用哪一种即时编译器，如下所示：

1. -client：指定Java虚拟机运行在Client模式下，并使用C1编译器；
   - C1编译器会对字节码进行简单和可靠的优化，耗时短，以达到更快的编译速度。
2. -server：指定Java虚拟机运行在server模式下，并使用C2编译器。
   - C2进行耗时较长的优化，以及激进优化，但优化的代码执行效率更高。（使用C++）

### C1和C2编译器不同的优化策略

1. 在不同的编译器上有不同的优化策略，C1编译器上主要有方法内联，去虚拟化、元余消除。
   - 方法内联：将引用的函数代码编译到引用点处，这样可以减少栈帧的生成，减少参数传递以及跳转过程
   - 去虚拟化：对唯一的实现樊进行内联
   - 冗余消除：在运行期间把一些不会执行的代码折叠掉
2. C2的优化主要是在全局层面，逃逸分析是优化的基础。基于逃逸分析在C2上有如下几种优化：
   - 标量替换：用标量值代替聚合对象的属性值
   - 栈上分配：对于未逃逸的对象分配对象在栈而不是堆
   - 同步消除：清除同步操作，通常指synchronized

> 也就是说之前的逃逸分析，只有在C2（server模式下）才会触发。那是否说明C1就用不了了？

### 分层编译策略

1. 分层编译（Tiered Compilation）策略：程序解释执行（不开启性能监控）可以触发C1编译，将字节码编译成机器码，可以进行简单优化，也可以加上性能监控，C2编译会根据性能监控信息进行激进优化。

2. 不过在Java7版本之后，一旦开发人员在程序中显式指定命令“-server"时，默认将会开启分层编译策略，由C1编译器和C2编译器相互协作共同来执行编译任务。

3. 一般来讲，JIT编译出来的机器码性能比解释器解释执行的性能高

4. C2编译器启动时长比C1慢，系统稳定执行以后，C2编译器执行速度远快于C1编译器

#### Graal编译器

- 自JDK10起，HotSpot又加入了一个全新的即时编译器：Graal编译器

- 编译效果短短几年时间就追平了G2编译器，未来可期（对应还出现了Graal虚拟机，是有可能替代Hotspot的虚拟机的）

- 目前，带着实验状态标签，需要使用开关参数去激活才能使用

  -XX:+UnlockExperimentalvMOptions -XX:+UseJVMCICompiler

#### AOT编译器

1. jdk9引入了AoT编译器（静态提前编译器，Ahead of Time Compiler）

2. Java 9引入了实验性AOT编译工具jaotc。它借助了Graal编译器，将所输入的Java类文件转换为机器码，并存放至生成的动态共享库之中。

3. 所谓AOT编译，是与即时编译相对立的一个概念。我们知道，即时编译指的是**在程序的运行过程中**，将字节码转换为可在硬件上直接运行的机器码，并部署至托管环境中的过程。而AOT编译指的则是，**在程序运行之前**，便将字节码转换为机器码的过程。

   .java -> .class -> (使用jaotc) -> .so

**AOT编译器编译器的优缺点**

**最大的好处：**

1. Java虚拟机加载已经预编译成二进制库，可以直接执行。
2. 不必等待即时编译器的预热，减少Java应用给人带来“第一次运行慢” 的不良体验

**缺点：**

1. 破坏了 java “ 一次编译，到处运行”，必须为每个不同的硬件，OS编译对应的发行包
2. 降低了Java链接过程的动态性，加载的代码在编译器就必须全部已知。
3. 还需要继续优化中，最初只支持Linux X64 java base

# 第六章 StringTable

## String的基本特性

1. String被声明为final的，不可被继承

2. String实现了Serializable接口：表示字符串是支持序列化的。实现了Comparable接口：表示String可以比较大小

3. String在jdk8以及以前内部定义了`final char value[]` 用于存储字符串数据。JDK9时改为`byte[]`

## 为什么JDK9改变了String的结构

**为什么改为byte[]存储**

1. String类的当前实现将字符串存储在char数组中，每个字符使用两个字节（16位）
2. 从许多不同的应用程序收集的数据表明，字符串是堆使用的主要组成部分，而且大多数字符串对象只包含拉丁字符（Latin-1）。这些字符只需要一个字节的存储空间，因此这些字符串对象的内部char数组中有一半的空间将不会使用，产生了大量浪费
3. 之前String类使用UTF-16的char[] 数组存储，现在改为byte[]数组外加 一个编码标识存储。该编码表示如果你的字符是ISO-8859-1或者Latin-1，那么只需要一个字节存。如果你是使用其它字符集，比如UTF-8，你仍然用两个字节存。
4. 结论：**String再也不用char来存储，改为了byte加上编码标记**，节约了 一些空间
5. 同时基于String的数据结构，例如StringBuffer和StringBuilder也同样做了修改

### String的基本特性

- String：代表不可变的字符序列。简称：不可变性
  1. 当对字符串重新赋值时，需要重写指定内存区域赋值，不能使用原有的value进行赋值。
  2. 当对现有的字符串进行连接操作时，也需要重新指定内存区域赋值，不能使用原有的value进行赋值
  3. 当调用String的replace方法修改指定字符或字符串时，也需要重新指定内存区域赋值，不能使用原有的value进行赋值
- 通过字面量的方式（区别于new）给一个字符串赋值，此时的字符串值声明在字符串常量池中。

### String的底层结构

**字符串常量池是不会存储相同内容的字符串的**

1. String的StringPool是一个固定大小的HashTable，默认大小为1009。如果放进StringPool的String非常多，就会造成Hash冲突严重，从而导致链表会很长，而链表长了之后会造成当前调用String.intern()方法是性能会大幅度下降
2. 使用-XX:StringTablesize可以设置StringTable的长度
3. 在JDK6中StringTable是固定的，就是1009的长度，所以如果常量池中的字符串过多就会导致效率下降很快，StringTablesize设置没有要求
4. 在JDK7中，StringTable的长度默认值都是60013，StringTablesize设置没有要求
5. 在JDK8中，StringTable的长度默认值是60013，StringTable可以设置的最小值为1009

## String的内存分配

1. 在Java语言中有八种基本数据类型和一种比较特殊的类型String。这些类型为了使它们在运行过程中速度更快、更节省内存，都提供了一种常量池的概念
2. 常量池就类似一个Java系统级别提供的缓存。8种基本数据结构类型的常量池都是系统协调的，String类型的常量池比较特殊。他的主要使用方法有两种。
   1. 直接使用双引号声明出来的String对象会直接存储在常量池中
   2. 如果不是用双引号声明的String对象，可以使用String提供的intern方法。
3. Java6以及之前，字符串常量池存放在永久代
4. Java7中Oracle的工程师对字符串池的逻辑做了很大的改变，即将字符串常量池的位置调整到Java堆内
   - 所有的字符串都保存在堆中，和其他普通对象一样，这样可以让你在进行调优应用时仅需要调整堆大小就可以了。
   - 字符串常量池概念原本使用得比较多，但是这个改动使得我们有足够的理由让我们重新考虑在Java7中使用String.intern()
5. Java8元空间，字符串常量在堆

### StringTable为什么要调整？

> In JDK 7, interned strings are no longer allocated in the permanent generation of the Java heap, but are instead allocated in the main part of the Java heap (known as the young and old generations), along with the other objects created by the application. This change will result in more data residing in the main Java heap, and less data in the permanent generation, and thus may require heap sizes to be adjusted. Most applications will see only relatively small differences in heap usage due to this change, but larger applications that load many classes or make heavy use of the method will see more significant differences.`String.intern()`

1. 为什么要调整位置？
   - 永久代的默认空间大小比较小
   - 永久代垃圾回收频率低，大量的字符串无法及时回收，容易进行Full GC产生STW或者容易产生OOM：PermGen Space
2. 在JDK7中，interned字符串不再在Java堆的永久代中分配，而是在Java堆的主要部分（称为年轻代和老年代）中分配，与应用程序创建的其他对象一起分配
3. 此更改导致驻留在主Java堆中的数据更多，驻留在永久代中的数据更少，因此可能需要调整堆大小。

## 字符串拼接操作

1. 常量与常量的拼接结果在常量池中，原理是编译期优化
2. 常量池中不会存在相同内容的变量
3. 拼接前后，只要其中有一个是变量，结果就在堆中。变量拼接的原理是StringBuilder
4. 如果拼接的结果调用intern方法，根据该字符串是否在常量池中存在，分为：
   1. 如果存在，则返回字符串在常量池中的地址
   2. 如果字符串常量池中不存在该字符串，则在常量池中创建一份，并返回此对象的地址

```java
public void test2(){
    String s1 = "javaEE";
    String s2 = "hadoop";

    String s3 = "javaEEhadoop";
    String s4 = "javaEE" + "hadoop";//编译期优化
    //如果拼接符号的前后出现了变量，则相当于在堆空间中new String()，具体的内容为拼接的结果：javaEEhadoop
    String s5 = s1 + "hadoop";
    String s6 = "javaEE" + s2;
    String s7 = s1 + s2;

    System.out.println(s3 == s4);//true
    System.out.println(s3 == s5);//false
    System.out.println(s3 == s6);//false
    System.out.println(s3 == s7);//false
    System.out.println(s5 == s6);//false
    System.out.println(s5 == s7);//false
    System.out.println(s6 == s7);//false
    //intern():判断字符串常量池中是否存在javaEEhadoop值，如果存在，则返回常量池中javaEEhadoop的地址；
    //如果字符串常量池中不存在javaEEhadoop，则在常量池中加载一份javaEEhadoop，并返回次对象的地址。
    String s8 = s6.intern();
    System.out.println(s3 == s8);//true
}
```

### 字符串拼接的底层细节

```java
@Test
public void test3(){
    String s1 = "a";
    String s2 = "b";
    String s3 = "ab";
    /*
        如下的s1 + s2 的执行细节：(变量s是我临时定义的）
        ① StringBuilder s = new StringBuilder();
        ② s.append("a")
        ③ s.append("b")
        ④ s.toString()  --> 约等于 new String("ab")，但不等价

        补充：在jdk5.0之后使用的是StringBuilder,在jdk5.0之前使用的是StringBuffer
         */
    String s4 = s1 + s2;//
    System.out.println(s3 == s4);//false
}
```

```java
/*
    1. 字符串拼接操作不一定使用的是StringBuilder!
       如果拼接符号左右两边都是字符串常量或常量引用，则仍然使用编译期优化，即非StringBuilder的方式。
    2. 针对于final修饰类、方法、基本数据类型、引用数据类型的量的结构时，能使用上final的时候建议使用上。
     */
@Test
public void test4(){
    final String s1 = "a";
    final String s2 = "b";
    String s3 = "ab";
    String s4 = s1 + s2;
    System.out.println(s3 == s4);//true
}
```

### 拼接操作与append操作的效率对比

```java
@Test
public void test6(){
    long start = System.currentTimeMillis();
    //        method1(100000);//4014
    method2(100000);//7
    long end = System.currentTimeMillis();
    System.out.println("花费的时间为：" + (end - start));
}

public void method1(int highLevel){
    String src = "";
    for(int i = 0;i < highLevel;i++){
        src = src + "a";//每次循环都会创建一个StringBuilder、String
    }
}

public void method2(int highLevel){
    //只需要创建一个StringBuilder
    StringBuilder src = new StringBuilder();
    for (int i = 0; i < highLevel; i++) {
        src.append("a");
    }
}
```

1. 通过StringBuilder的append()方式添加字符串的效率要远高于使用String的字符串拼接方式
2. 原因：
   1. StringBuilder的append方式自始自终只创建过一个StringBuilder的对象
   2. 使用String的字符串拼接方式创建过多个StringBuilder和String（调的toString方法）的对象，内存占用要更大；如果进行GC，需要花费额外的时间（在拼接的过程中产生的一些中间字符串可能永远也用不到，会产生大量垃圾字符串）
3. 改进的空间：
   1. 在实际开发中，如果基本确定要前前后后添加的字符串长度不高于某个限定值highLevel的情况下，建议使用构造器实例化：
   2. `StringBuilder s = new StringBuilder(highLevel); //new char[highLevel]`
   3. 这样可以避免频繁扩容

## intern的使用

### intern()方法的说明

```java
public native String intern();
```

1. intern是一个native方法，调用的是底层C的方法
2. 字符串常量池最初是空的，由String类私有地维护。在调用intern方法时，如果池中已经包含了由equals(object)方法确定的与该字符串内容相等的字符串，则返回池中的字符串地址。否则该字符串对象将被添加到常量池中，并返回对该字符串对象的地址。
3. 如果不是用双引号声明的String对象，可以使用String提供的intern方法：intern方法会从字符串常量池中查询当前字符串是否存在，若不存在就会将当前字符串放入常量池中。
4. 也就是说，如果在任意字符串上调用String.intern方法，那么其返回结果所指向的那个类实例，必须和直接以常量形式出现的字符串实例完全相同。
5. 通俗点讲，interned String就是确保字符串在内存中只有一份拷贝，这样就可以节约内存空间，加快字符串操作任务的执行速度。注意，这个值会被放到字符串内部池（Stirng intern Pool）

### new String()的说明

**new String("ab")会创建几个对象？**

```java
/**
两个：
    一个是new关键字在堆空间创建的
    另一个对象是：字符串常量池中的对象“ab”。字节码指令：ldc
**/
public class StringNewTest{
    public static void main(String[] args){
        String str = new String("ab");
    }
}
```

**new String("a") + new String("b") 会创建几个对象？**

```java
/**
 * 思考：
 * new String("a") + new String("b")呢？
 *  对象1：new StringBuilder()
 *  对象2： new String("a")
 *  对象3： 常量池中的"a"
 *  对象4： new String("b")
 *  对象5： 常量池中的"b"
 *
 *  深入剖析： StringBuilder的toString():
 *      对象6 ：new String("ab")
 *       强调一下，toString()的调用，在字符串常量池中，没有生成"ab"
 *
 */
public class StringNewTest {
    public static void main(String[] args) {

        String str = new String("a") + new String("b");
    }
}
```

### 有点难的面试题

**执行s3.intern()，而在JDK7的后续版本中，字符串常量池被移动到了堆中，此时堆里已经有new String（"11"）了，出于节省空间的目的，直接将堆中的字符串的引用地址储存在字符串常量池中。没错，字符串常量池中存的是new String（"11"）在堆中的地址**

```java
/**
 * 如何保证变量s指向的是字符串常量池中的数据呢？
 * 有两种方式：
 * 方式一： String s = "shkstart";//字面量定义的方式
 * 方式二： 调用intern()
 *         String s = new String("shkstart").intern();
 *         String s = new StringBuilder("shkstart").toString().intern();
 *
 */
public class StringIntern {
    public static void main(String[] args) {

        String s = new String("1");
        s.intern();//调用此方法之前，字符串常量池中已经存在了"1"
        String s2 = "1";
        System.out.println(s == s2);//jdk6：false   jdk7/8：false

        /*
         1、s3变量记录的地址为：new String("11")
         2、经过上面的分析，我们已经知道执行完pos_1的代码，
             在堆中有了一个new String("11")
             这样的String对象。但是在字符串常量池中没有"11"
         3、接着执行s3.intern()，在字符串常量池中生成"11"
           3-1、在JDK6的版本中，字符串常量池还在永久代，
                   所以直接在永久代生成"11",也就有了新的地址
           3-2、而在JDK7的后续版本中，字符串常量池被移动到了堆中，
                   此时堆里已经有new String（"11"）了
                   出于节省空间的目的，直接将堆中的字符串的引用地址储存在字符串常量池中。
                   没错，字符串常量池中存的是new String（"11"）在堆中的地址
         4、所以在JDK7后续版本中，s3和s4指向的完全是同一个地址。
         */
        String s3 = new String("1") + new String("1");//pos_1
        s3.intern();

        //s4变量记录的地址：使用的是上一行代码代码执行时，在常量池中生成的"11"的地址
        String s4 = "11";
        System.out.println(s3 == s4);//jdk6：false  jdk7/8：true
    }
}
```

**内存分析**

JDK6：

![img](深入理解Java虚拟机/0013-16516626421232.png)

JDK7以及以后：

![img](深入理解Java虚拟机/0014.png)

### 面试题的拓展

```java
public class StringIntern1 {
    public static void main(String[] args) {
        String s3 = new String("1") + new String("1");
        // 执行完上一行代码后，字符串常量池中并没有出现“11”
        String s4 = "11";
        String s5 = s3.intern();

        // s3是堆中的"11"，s4是字符串常量池中的"11"
        System.out.println(s3 == s4); //false

        System.out.println(s5 == s4);// true
    }
}
```

如果将s5和s4的语句交换位置，就会导致两个都输出true

### intern方法的练习题

![img](深入理解Java虚拟机/0017.png)

### intern的效率测试（空间角度）

结论：

1、**对于程序中大量存在的字符串，尤其其中存在很多重复的字符串时，使用intern**

2、大的网站平台，需要内存中存储大量的字符串。比如社交网站，很多人都存储：北京市、海淀区等信息。这时候如果字符串都调用intern() 方法，就会很明显降低内存的大小。

## G1中的String去重操作

**String去重操作的背景**

> 注意不是字符串常量池的去重操作，字符串常量池本身就没有重复的

1. 背景：对许多Java应用（有大的也有小的）做的测试得出以下结果：
   - 堆存活数据集合里面String对象占了25%
   - 堆存活数据集合里面重复的String对象有13.5%
   - String对象的平均长度是45
2. 许多大规模的Java应用的瓶颈在于内存，测试表明，在这些类型的应用里面，Java堆中存活的数据集合差不多25%是String对象。更进一步，这里面差不多一半String对象是重复的，重复的意思是说：`str1.equals(str2)= true`。堆上存在重复的String对象必然是一种内存的浪费。这个项目将在G1垃圾收集器中实现自动持续对重复的String对象进行去重，这样就能避免浪费内存。

**String 去重的的实现**

1. 当垃圾收集器工作的时候，会访问堆上存活的对象。对每一个访问的对象都会检查是否是候选的要去重的String对象。
2. 如果是，把这个对象的一个引用插入到队列中等待后续的处理。一个去重的线程在后台运行，处理这个队列。处理队列的一个元素意味着从队列删除这个元素，然后尝试去重它引用的String对象。
3. 使用一个Hashtable来记录所有的被String对象使用的不重复的char数组。当去重的时候，会查这个Hashtable，来看堆上是否已经存在一个一模一样的char数组。
4. 如果存在，String对象会被调整引用那个数组，释放对原来的数组的引用，最终会被垃圾收集器回收掉。
5. 如果查找失败，char数组会被插入到Hashtable，这样以后的时候就可以共享这个数组了。

**命令行选项**

1. UseStringDeduplication(bool) ：开启String去重，默认是不开启的，需要手动开启。
2. PrintStringDeduplicationStatistics(bool) ：打印详细的去重统计信息
3. stringDeduplicationAgeThreshold(uintx) ：达到这个年龄的String对象被认为是去重的候选对象

# 第七章 垃圾回收

## 垃圾回收概述

1. Java 和 C++语言的区别，就在于垃圾收集技术和内存动态分配上，C++语言没有垃圾收集技术，需要程序员手动的收集。
2. 垃圾收集，不是Java语言的伴生产物。早在1960年，第一门开始使用内存动态分配和垃圾收集技术的Lisp语言诞生。
3. 关于垃圾收集有三个经典问题：
   - 哪些内存需要回收？
   - 什么时候回收？
   - 如何回收？
4. 垃圾收集机制是Java的招牌能力，极大地提高了开发效率。如今，垃圾收集几乎成为现代语言的标配，即使经过如此长时间的发展，Java的垃圾收集机制仍然在不断的演进中，不同大小的设备、不同特征的应用场景，对垃圾收集提出了新的挑战，这当然也是面试的热点。

### 什么是垃圾

1. 垃圾是指**在运行程序中没有任何指针指向的对象**，这个对象就是需要被回收的垃圾。
2. 如果不及时堆内存中的垃圾进行清理，那么这些垃圾对象所占用的内存空间就会一直保留到应用程序结束，被保留的空间无法被其他对象使用。甚至可能导致内存溢出。

### 为什么需要GC

1. 对于高级语言来说，一个基本认知就是如果不进行垃圾回收，**内存迟早会被消耗完**，因为不断地分配内存空间而不进行回收，就好像不停地生产生活垃圾而从来不进行打扫一样。
2. 除了释放没有用的对象，垃圾回收也可以清除内存里的记录碎片。碎片整理将所占用的堆内存移到堆的一端，**以便JVM将整理出的内存分配给新的对象**。
3. 随着应用程序所应付的业务越来越庞大、复杂、用户越来越多，**没有GC就不能保证应用程序的正常运行**。而经常造成STW的GC又跟不上实际的需求，所以才会不断地尝试堆GC进行优化。

### 早期垃圾回收

1. 在早期的C/C++时代，垃圾回收基本上是手工进行的。开发人员可以使用new关键字进行内存申请，并使用delete关键字进行内存释放。
2. 这种方式可以灵活控制内存释放的时间，但是会给开发人员带来**频繁申请和释放内存的管理负担。**倘若有一处内存区间由于程序员编码的问题忘记被回收，那么就会产生**内存泄漏**，垃圾对象永远无法被清除，随着系统运行时间的不断增长，垃圾对象所耗费的内存可能持续上升，知道出现内存溢出并造成**应用程序崩溃**。
3. 现在，除了Java以外，C#、Python、Ruby等语言都使用了自动垃圾回收的思想，也是未来发展趋势，可以说这种自动化的内存分配和垃圾回收方式已经成为了现代开发语言必备的标准。

### Java垃圾回收机制

#### 自动内存管理

**自动内存管理的优点**

1. 自动内存管理，无需开发人员手动参与内存的分配与回收，这样降低内存泄漏和内存溢出的风险
2. 没有垃圾回收器，java也会和cpp一样，各种悬垂指针，野指针，泄露问题让你头疼不已。
3. 自动内存管理机制，将程序员从繁重的内存管理中释放出来，可以更专心地专注于业务开发

**关于自动内存管理的担忧**

1. 对于Java开发人员而言，自动内存管理就像是一个黑匣子，如果过度依赖于“自动”，那么这将会是一场灾难，最严重的就会**弱化Java开发人员在程序出现内存溢出时定位问题和解决问题的能力**。
2. 此时，了解JVM的自动内存分配和内存回收原理就显得非常重要，只有在真正了解JVM是如何管理内存后，我们才能够在遇见OutofMemoryError时，快速地根据错误异常日志定位问题和解决问题。
3. 当需要排查各种内存溢出、内存泄漏问题时，当垃圾收集成为系统达到更高并发量的瓶颈时，我们就必须对这些“自动化”的技术**实施必要的监控和调节**。

#### 应该关系哪些区域的回收

1. 垃圾收集器可以对年轻代回收，也可以对老年代回收，甚至是全栈和方法区的回收，
2. 其中，**Java堆是垃圾收集器的工作重点**
3. 从次数上讲：
   1. 频繁收集Young区
   2. 较少收集Old区
   3. 基本不收集Perm区（元空间）

## 垃圾回收相关算法

### 标记阶段：引用计数算法

#### 标记阶段的目的

**垃圾标记阶段：主要是为了判断对象是否存活**

1. 在堆里存放着几乎所有的Java对象实例，在GC执行垃圾回收之前，首先**需要区分出内存中哪些是存活对象，哪些是已经死亡的对象**。只有被标记为已经死亡的对象，GC才会在执行垃圾回收的时候，释放掉其所占用的内存空间，因此这个过程我们可以称为**垃圾标记阶段**。
2. 那么在JVM中究竟是如何标记一个死亡对象呢？简单来说，当一个对象已经不再被任何的存活对象继续引用时，就可以宣判为已经死亡。
3. 判断对象存活一般有两种方式：**引用计数算法**和**可达性分析算法**

#### 引用计数算法

1. 引用计数算法（Reference Counting）比较简单，对每个对象保存一个整型的引用计数器属性。用于记录对象被引用的情况。

2. 对于一个对象A，只要有任何一个对象引用了A，则A的引用计数器就加1；当引用失效时，引用计数器就减1。只要对象A的引用计数器的值为0，即表示对象A不可能再被使用，可进行回收。

3. 优点：实现简单，垃圾对象便于辨识；判定效率高，回收没有延迟性

4. 缺点：

   1. 它需要单独的字段存储计数器，这样的做法增加了**存储空间的开销**。
   2. 每次赋值都需要更新计数器，伴随着加法和减法，这增加了**时间开销**
   3. 引用计数器有一个严重的问题，即**无法处理循环引用的问题**，这是一条致命缺陷，导致在Java的垃圾回收器中没有使用这类算法

#### 循环引用

![img](深入理解Java虚拟机/0004.png)

当p的指针断开的时候，内部的引用形成一个循环，计数器都还算1，无法被回收，这就是循环引用，从而造成内存泄漏。

这里的内存泄露并不是Java中的内存泄露。面试中不可以用这个例子进行举例

#### 小结

1. 引用计数算法，是很多语言的资源回收选择，例如因人工智能而更加火热的Python，它更是同时支持引用计数和垃圾收集机制。
2. 具体哪种最优是要看场景的，业务有大规模实践中仅保留引用计数机制，以提高吞吐量的尝试。
3. Java并没有选择引用计数，是因为其存在一个基本的难题，也就是很难处理循环引用关系
4. Python如何解决循环引用？
   - 手动解除：很好理解，就是在合适的时机，解除引用关系。
   - 使用弱引用weakref，weakref是Python提供的标准库，旨在解决循环引用

### 标记阶段：可达性分析算法

**可达性分析算法：也可以称为根搜索算法、追踪性垃圾收集**

1. 相对于引用计数算法而言，可达性分析算法不仅同样具备实现简单和执行高效等特点，最重要的是该算法可以有效地**解决在引用计数算法中循环引用的问题，防止内存泄漏的发生**。
2. 相较于引用计数算法，这里的可达性分析就是Java、C#选择的。这种类型的垃圾收集通常也叫做**追踪性垃圾收集**（Tracing Garbage Collection）

#### 可达性分析实现思路

- 所谓“GCRoots”根集合就是一组必须活跃的引用
- 其基本思路如下：
  1. 可达性分析算法是以跟对象集合（GCRoots）为起始点，按照从上至下的方式**搜索被根对象集合所连接的目标对象是否可达**。
  2. 使用可达性分析算法后，内存中的存活对象都会被根对象集合直接或间接连接着，搜索走过的路径称为**引用链**（Reference Chain）
  3. 如果目标对象没有任何引用链相连，则是不可达的，就意味着该对象已经死亡，可以标记为垃圾对象。
  4. 在可达性分析算法中，只有能够被根对象集合直接或间接连接的对象才是存活对象。

![IMG/13、垃圾回收概述和垃圾回收算法.assets/image-20210909162516777.png](深入理解Java虚拟机/image-20210909162516777.png)

#### GCRoots可以是哪些元素？

1. 虚拟机栈中引用的对象：各个线程被调用的方法中使用到的参数、局部变量等。
2. 本地方法栈内JNI（通常说的本地方法）引用的对象
3. 方法区中类静态属性引用的对象：Java类的引用类型静态变量
4. 方法区中常量引用的对象：字符串常量池里的引用
5. 所有被同步锁synchronized持有的对象
6. Java虚拟机内部的引用：基本数据类型对应的Class对象，一些常驻的异常对象（如：NullPointerException、OutOfMemoryError），系统类加载器
7. 反映java虚拟机内部情况的JMXBean、JVMTI中注册的回调、本地代码缓存等

----

1. 总结一句话就是，除了堆空间的周边，比如：虚拟机栈、本地方法栈、方法区、字符串常量池等地方对堆空间进行引用的，都可以作为GC Roots进行可达性分析
2. 除了这些固定的GC Roots集合以外，根据用户所选用的垃圾收集器以及当前回收的内存区域不同，还可以有其他对象“**临时性**”地加入，共同构成完整GC Roots集合。比如：**分代收集**和局部回收（PartialGC）。
   - 如果只针对Java堆中的某一块区域进行垃圾回收（比如：典型的只针对新生代），必须考虑到内存区域是虚拟机自己的实现细节，更不是孤立封闭的，这个区域的对象完全有可能被其他区域的对象所引用，这时候就需要一并将关联的区域对象也加入GC Roots集合中去考虑，才能保证可达性分析的准确性。

**小技巧**

由于Root采用栈方式存放变量和指针，所以如果一个指针，它保存了堆内存里面的对象，但是自己又不存放在堆内存里面，那它就是一个Root。

#### 注意

1. 如果要使用可达性分析算法来判断内存是否可回收，那么分析工作必须在一个能保障一致性的快照中进行。这点不满足的话分析结果的准确性就无法保证。
2. 这点也是导致GC进行时必须“Stop The World”的一个重要原因。即使是号称（几乎）不会发生停顿的CMS收集器中，**枚举根节点时也是必须要停顿的**。

### 对象的finalization机制

#### finalize() 方法机制

**对象销毁前的回调函数：finalize()**

1. Java语言提供了对象终止（finalization）机制来允许开发人员提供**对象被销毁之前的自定义处理逻辑**。
2. 当垃圾回收器发现没有引用指向一个对象，即：垃圾回收此对象之前，总会先调用这个对象的finalize()方法。
3. finalize() 方法允许在子类中被重写，**用于在对象被回收时进行资源释放**。通常在这个方法中进行一些资源释放和清理的工作，比如关闭文件、套接字和数据库连接等。

Object 类中 finalize() 源码

```java
// 等待被重写
protected void finalize() throws Throwable { }
```

1. 永远不要主动调用某个对象的finalize()方法，应该交给垃圾回收机制调用。理由包括下面三点：
   1. 在finalize()时可能会导致对象复活。
   2. finalize()方法的执行时间是没有保障的，它完全由GC线程决定，极端情况下，若不发生GC，则finalize()方法将没有执行机会。
   3. 一个糟糕的finalize()会严重影响GC的性能。比如finalize是个死循环
2. 从功能上来说，finalize()方法与C++中的析构函数比较相似，但是Java采用的是基于垃圾回收器的自动内存管理机制，所以finalize()方法在**本质上不同于C++中的析构函数**。
3. finalize()方法对应了一个finalize线程，因为优先级比较低，即使主动调用该方法，也不会因此就直接进行回收

#### 生存还是死亡？

由于finalize()方法的存在，**虚拟机中的对象一般处于三种可能的状态。**

1. 如果从所有的根节点都无法访问到某个对象，说明对象己经不再使用了。一般来说，此对象需要被回收。但事实上，也并非是“非死不可”的，这时候它们暂时处于“缓刑”阶段。

   一个无法触及的对象有可能在某一个条件下“复活”自己

   ，如果这样，那么对它立即进行回收就是不合理的。为此，定义虚拟机中的对象可能的三种状态。如下：

   1. 可触及的：从根节点开始，可以到达这个对象。
   2. 可复活的：对象的所有引用都被释放，但是对象有可能在finalize()中复活。
   3. 不可触及的：对象的finalize()被调用，并且没有复活，那么就会进入不可触及状态。不可触及的对象不可能被复活，**因为finalize()只会被调用一次**。

2. 以上3种状态中，是由于finalize()方法的存在，进行的区分。只有在对象不可触及时才可以被回收。

#### 具体过程

判定一个对象objA是否可回收，至少要经历两次标记过程：

1. 如果对象objA到GC Roots没有引用链，则进行第一次标记。
2. 进行筛选，判断此对象是否有必要执行finalize()方法
   1. 如果对象objA没有重写finalize()方法，或者finalize()方法已经被虚拟机调用过，则虚拟机视为“没有必要执行”，objA被判定为不可触及的。
   2. 如果对象objA重写了finalize()方法，且还未执行过，那么objA会被插入到F-Queue队列中，由一个虚拟机自动创建的、低优先级的Finalizer线程触发其finalize()方法执行。
   3. finalize()方法是对象逃脱死亡的最后机会，稍后GC会对F-Queue队列中的对象进行第二次标记。如果objA在finalize()方法中与引用链上的任何一个对象建立了联系，那么在第二次标记时，objA会被移出“即将回收”集合。之后，对象会再次出现没有引用存在的情况。在这个情况下，finalize()方法不会被再次调用，对象会直接变成不可触及的状态，也就是说，一个对象的finalize()方法只会被调用一次。

### 清除阶段：标记-清除算法

**垃圾清除阶段**

- 当成功区分出内存中存活对象和死亡对象后，GC接下来的任务就是执行垃圾回收，释放掉无用对象所占用的内存空间，以便有足够的可用内存空间为新对象分配内存。目前在JVM中比较常见的三种垃圾收集算法是

1. 标记-清除算法（Mark-Sweep）
2. 复制算法（Copying）
3. 标记-压缩算法（Mark-Compact）

**背景**

标记-清除算法（Mark-Sweep）是一种非常基础和常见的垃圾收集算法，该算法被J.McCarthy等人在1960年提出并并应用于Lisp语言。

**执行过程**

当堆中的有效内存空间（available memory）被耗尽的时候，就会停止整个程序（也被称为stop the world），然后进行两项工作，第一项则是标记，第二项则是清除

1. 标记：Collector从引用根节点开始遍历，标记所有被引用的对象。一般是在对象的Header中记录为可达对象。
   - 注意：标记的是被引用的对象，也就是可达对象，并非标记的是即将被清除的垃圾对象
2. 清除：Collector对堆内存从头到尾进行线性的遍历，如果发现某个对象在其Header中没有标记为可达对象，则将其回收

![IMG/13、垃圾回收概述和垃圾回收算法.assets/0029.png](深入理解Java虚拟机/0029.png)

**标记-清除算法的缺点**

1. 标记清除算法的效率不算高
2. 在进行GC的时候，需要停止整个应用程序，用户体验较差
3. 这种方式清理出来的空闲内存是不连续的，产生内碎片，需要维护一个空闲列表

**注意：何为清除？**

这里所谓的清除并不是真的置空，而是把需要清除的对象地址保存在空闲的地址列表里。下次有新对象需要加载时，判断垃圾的位置空间是否够，如果够，就存放（也就是覆盖原有的地址）。

关于空闲列表是在为对象分配内存的时候提过：

1. 如果内存规整
   - 采用指针碰撞的方式进行内存分配
2. 如果内存不规整
   - 虚拟机需要维护一个空闲列表
   - 采用空闲列表分配内存

### 清除阶段：复制算法

**背景**

1. 为了解决标记-清除算法在垃圾收集效率方面的缺陷，M.L.Minsky于1963年发表了著名的论文，“使用双存储区的Lisp语言垃圾收集器CA LISP Garbage Collector Algorithm Using Serial Secondary Storage）”。M.L.Minsky在该论文中描述的算法被人们称为复制（Copying）算法，它也被M.L.Minsky本人成功地引入到了Lisp语言的一个实现版本中。

**核心思想**

将活着的内存空间分为两块，每次只使用其中一块，在垃圾回收时将正在使用的内存中的存活对象复制到未被使用的内存块中，之后清除正在使用的内存块中的所有对象，交换两个内存的角色，最后完成垃圾回收

![IMG/13、垃圾回收概述和垃圾回收算法.assets/0030.png](深入理解Java虚拟机/0030.png)

新生代里面就用到了复制算法，Eden区和S0区存活对象整体复制到S1区

**复制算法的优缺点**

**优点**

1. 没有标记和清除过程，实现简单，运行高效
2. 复制过去以后保证空间的连续性，不会出现“碎片”问题。

**缺点**

1. 此算法的缺点也是很明显的，就是需要两倍的内存空间。
2. 对于G1这种分拆成为大量region的GC，复制而不是移动，意味着GC需要维护region之间对象引用关系，不管是内存占用或者时间开销也不小

**复制算法的应用场景**

1. 如果系统中的垃圾对象很多，复制算法需要复制的存活对象数量并不会太大，效率较高
2. 老年代大量的对象存活，那么复制的对象将会有很多，效率会很低
3. 在新生代，对常规应用的垃圾回收，一次通常可以回收70% - 99% 的内存空间。回收性价比很高。所以现在的商业虚拟机都是用这种收集算法回收新生代。

### 清除阶段：标记-压缩算法

**标记-压缩（或标记-整理、Mark - Compact）算法**

**背景**

1. 复制算法的高效性是建立在存活对象少、垃圾对象多的前提下的。这种情况在新生代经常发生，但是在老年代，更常见的情况是大部分对象都是存活对象。如果依然使用复制算法，由于存活对象较多，复制的成本也将很高。因此，**基于老年代垃圾回收的特性，需要使用其他的算法。**
2. 标记-清除算法的确可以应用在老年代中，但是该算法不仅执行效率低下，而且在执行完内存回收后还会产生内存碎片，所以JVM的设计者需要在此基础之上进行改进。标记-压缩（Mark-Compact）算法由此诞生。
3. 1970年前后，G.L.Steele、C.J.Chene和D.s.Wise等研究者发布标记-压缩算法。在许多现代的垃圾收集器中，人们都使用了标记-压缩算法或其改进版本。

**执行过程**

1. 第一阶段和标记清除算法一样，从根节点开始标记所有被引用对象
2. 第二阶段将所有的存活对象压缩到内存的一端，按顺序排放。之后，清理边界外所有的空间。

![IMG/13、垃圾回收概述和垃圾回收算法.assets/0032.png](深入理解Java虚拟机/0032.png)

**标记-压缩算法与标记-清除算法的比较**

1. 标记-压缩算法的最终效果等同于标记-清除算法执行完成后，再进行一次内存碎片整理，因此，也可以把它称为标记-清除-压缩（Mark-Sweep-Compact）算法。
2. 二者的本质差异在于标记-清除算法是一种**非移动式的回收算法**，标记-压缩是**移动式的**。是否移动回收后的存活对象是一项优缺点并存的风险决策。
3. 可以看到，标记的存活对象将会被整理，按照内存地址依次排列，而未被标记的内存会被清理掉。如此一来，当我们需要给新对象分配内存时，JVM只需要持有一个内存的起始地址即可，这比维护一个空闲列表显然少了许多开销。

**标记-压缩算法的优缺点**

**优点**

1. 消除了标记-清除算法当中，内存区域分散的缺点，我们需要给新对象分配内存时，JVM只需要持有一个内存的起始地址即可。
2. 消除了复制算法当中，内存减半的高额代价。

**缺点**

1. 从效率上来说，标记-整理算法要低于复制算法。
2. 移动对象的同时，如果对象被其他对象引用，则还需要调整引用的地址（因为HotSpot虚拟机采用的不是句柄池的方式，而是直接指针）
3. 移动过程中，需要全程暂停用户应用程序。即：STW

### 垃圾回收算法小结

> **对比三种清除阶段的算法**

1. 效率上来说，复制算法是当之无愧的老大，但是却浪费了太多内存。
2. 而为了尽量兼顾上面提到的三个指标，标记-整理算法相对来说更平滑一些，但是效率上不尽如人意，它比复制算法多了一个标记的阶段，比标记-清除多了一个整理内存的阶段。

|              | 标记清除           | 标记整理         | 复制                                  |
| ------------ | ------------------ | ---------------- | ------------------------------------- |
| **速率**     | 中等               | 最慢             | 最快                                  |
| **空间开销** | 少（但会堆积碎片） | 少（不堆积碎片） | 通常需要活对象的2倍空间（不堆积碎片） |
| **移动对象** | 否                 | 是               | 是                                    |

### 分代收集算法

Q：难道就没有一种最优的算法吗？

A：无，没有最好的算法，只有最合适的算法

**为什么要使用分代收集算法**

1. 前面所有这些算法中，并没有一种算法可以完全替代其他算法，它们都具有自己独特的优势和特点。分代收集算法应运而生。
2. 分代收集算法，是基于这样一个事实：**不同的对象的生命周期是不一样的。因此，不同生命周期的对象可以采取不同的收集方式，以便提高回收效率。**一般是把Java堆分为新生代和老年代，这样就可以根据各个年代的特点使用不同的回收算法，以提高垃圾回收的效率。
3. 在Java程序运行的过程中，会产生大量的对象，其中有些对象是与业务信息相关:
   - 比如Http请求中的Session对象、线程、Socket连接，这类对象跟业务直接挂钩，因此生命周期比较长。
   - 但是还有一些对象，主要是程序运行过程中生成的临时变量，这些对象生命周期会比较短，比如：String对象，由于其不变类的特性，系统会产生大量的这些对象，有些对象甚至只用一次即可回收。

**目前几乎所有的GC都采用分代收集算法执行垃圾回收的**

在HotSpot中，基于分代的概念，GC所使用的内存回收算法必须结合年轻代和老年代各自的特点。

1. 年轻代（Young Gen）
   - 年轻代特点：区域相对老年代较小，对象生命周期短、存活率低，回收频繁。
   - 这种情况复制算法的回收整理，速度是最快的。复制算法的效率只和当前存活对象大小有关，因此很适用于年轻代的回收。而复制算法内存利用率不高的问题，通过hotspot中的两个survivor的设计得到缓解。
2. 老年代（Tenured Gen）
   - 老年代特点：区域较大，对象生命周期长、存活率高，回收不及年轻代频繁。
   - 这种情况存在大量存活率高的对象，复制算法明显变得不合适。一般是由标记-清除或者是标记-清除与标记-整理的混合实现。
     - Mark阶段的开销与存活对象的数量成正比。
     - Sweep阶段的开销与所管理区域的大小成正相关。
     - Compact阶段的开销与存活对象的数据成正比。
3. 以HotSpot中的CMS回收器为例，CMS是基于Mark-Sweep实现的，对于对象的回收效率很高。对于碎片问题，CMS采用基于Mark-Compact算法的Serial Old回收器作为补偿措施：当内存回收不佳（碎片导致的Concurrent Mode Failure时），将采用Serial Old执行Full GC以达到对老年代内存的整理。
4. 分代的思想被现有的虚拟机广泛使用。几乎所有的垃圾回收器都区分新生代和老年代

### 增量收集算法和分区算法

#### 增量收集算法

上述现有的算法，在垃圾回收过程中，应用软件将处于一种Stop the World的状态。在**Stop the World**状态下，应用程序所有的线程都会挂起，暂停一切正常的工作，等待垃圾回收的完成。如果垃圾回收时间过长，应用程序会被挂起很久，将严重影响用户体验或者系统的稳定性。为了解决这个问题，即对实时垃圾收集算法的研究直接导致了增量收集（Incremental Collecting）算法的诞生。

**增量收集算法基本思想**

1. 如果一次性将所有的垃圾进行处理，需要造成系统长时间的停顿，那么就可以让垃圾收集线程和应用程序线程交替执行。**每次，垃圾收集线程只收集一小片区域的内存空间，接着切换到应用程序线程。依次反复，直到垃圾收集完成。**
2. 总的来说，增量收集算法的基础仍是传统的标记-清除和复制算法。增量收集算法通过**对线程间冲突的妥善处理，允许垃圾收集线程以分阶段的方式完成标记、清理或复制工作**

**增量收集算法的缺点**

使用这种方式，由于在垃圾回收过程中，间断性地还执行了应用程序代码，所以能减少系统的停顿时间。但是，因为线程切换和上下文转换的消耗，会使得垃圾回收的总体成本上升，**造成系统吞吐量的下降**。

#### 分区算法

> 主要针对G1收集器来说的

1. 一般来说，在相同条件下，堆空间越大，一次GC时所需要的时间就越长，有关GC产生的停顿也越长。为了更好地控制GC产生的停顿时间，将一块大的内存区域分割成多个小块，根据目标的停顿时间，每次合理地回收若干个小区间，而不是整个堆空间，从而减少一次GC所产生的停顿。
2. 分代算法将按照对象的生命周期长短划分成两个部分，分区算法将整个堆空间划分成连续的不同小区间。每一个小区间都独立使用，独立回收。这种算法的好处是可以控制一次回收多少个小区间。

![IMG/13、垃圾回收概述和垃圾回收算法.assets/0033.png](深入理解Java虚拟机/0033.png)

### 写在最后

注意，这些只是基本的算法思路，实际GC实现过程要复杂的多，目前还在发展中的前沿GC都是复合算法，并且并行和并发兼备。

# 第八章 垃圾回收相关概念

## System.gc()的理解

1. 默认情况下，通过System.gc()者Runtime.getRuntime().gc()的调用，**会显示触发FullGC**，同时对老年代和新生代进行回收，尝试释放被丢弃对象占用的内存。
2. 然而System.gc()调用附带一个免责声明，无法保证对垃圾收集器的调用（不能确保立即生效）
3. JVM实现者可以通过System.gc()调用来决定JVM的GC行为。而一般情况下，垃圾回收应该是自动进行的，**无法手动触发，否则就太过于麻烦了**。在一些特殊情况下，如我们正在编写一个性能基准，我们可以在运行之间调用System.gc()；

### 手动GC理解不可达对象的回收行为

运行时的JVM参数设置

`-Xms256m -Xmx256m -XX:+PrintGCDetails -XX:PretenureSizeThreshold=15m`

第四个参数设置大对象直接进行老年代的阈值

1、buffer数组对象仅仅将年轻代的buffer数组对象放到了老年代，buffer对象仍然没有回收

```java
public void localvarGC1() {
    byte[] buffer = new byte[10 * 1024 * 1024]; // 10MB
    System.gc();
}
```

2、buffer对象没有引用指向它，可能会被gc回收

```java
public void localvarGC2() {
    byte[] buffer = new byte[10 * 1024 * 1024];
    buffer = null;
    System.gc();
}
```

3、虽然出了代码块的作用域，但是buffer数组对象并没有被回收

这就需要局部变量表的插槽机制了，虽然buffer超出了作用域范围，但是仍然占据了第1个插槽的位置（第0个是this），只是没有显示出来，执行gc的时候，实际上栈中还有buffer变量指向堆中的字节数组，所以没有gc

```java
public void localvarGC3() {
    {
        byte[] buffer = new byte[10 * 1024 * 1024];
    }
    System.gc();
}
```

4、这里的buffer会被回收，因为buffer的索引位置会被value所占用，因此就没有引用指向buffe所指的对象

```java
public void localvarGC4() {
    {
        byte[] buffer = new byte[10 * 1024 * 1024];
    }
    int value = 10;
    System.gc();
}
```

5、局部变量出了方法范围就是失效了，堆中的字节数组被回收

```java
public void localvarGC5() {
    localvarGC1();
    System.gc();
}
```

## 内存溢出与内存泄漏

### 内存溢出

1. 内存溢出相对于内存泄漏来说，尽管更容易被理解，但是同样的，内存溢出也是引发程序崩溃的罪魁祸首之一。
2. 由于GC一直在发展，所有一般情况下，除非应用程序占用的内存增长速度非常快，造成垃圾回收已经更不上内存消耗的速度，否则不太容易出现OOM的情况
3. 大多数情况下，GC会进行各种年龄段的垃圾回收，实在不行了就放大招，来一次独占式的FullGC操作，这时候会回收大量的内存，供应用程序继续使用。
4. Javadoc中对OutOfMemoryError的解释是，没有空闲内存，并且垃圾收集器也无法提供更多内存。

**内存溢出的原因分析**

首先说没有空闲内存的情况：说明Java虚拟机的堆内存不够

1. Java虚拟机的堆内存设置不够。

   - 比如：可能存在内存泄漏问题；也很有可能就是堆的大小不合理，比如我们要处理比较客观的数据量，但是没有显示指定JVM堆大小或者指定数值偏小。我们可以通过参数-Xms、-Xmx来调整

2. 代码中创建了大量大对象，并且长时间不能被垃圾收集器收集（存在被引用）

   - 对于老版本的Oracle JDK，因为永久代的大小是有限的，并且JVM对永久代垃圾回收（如常量池回收、卸载不再需要的类型）非常不积极，所以当我们不断添加新类型的时候，永久代出现OutOfMemoryError也非常多见。尤其是在运行时存在大量动态类型生成的场合；类似intern字符串缓存占用太多空间，也会导致OOM问题。对应的异常信息，会标记出现和永久代相关：“java.lang.OutOfMemoryError:PermGen space"。
   - 随着元数据区的引入，方法区内存已经不再那么窘迫，所以相应的OOM有所改观，出现OOM，异常信息变成了："java.lang.OutOfMemoryError:Metaspace"。直接内存不足，也会导致OOM

3. 这里隐含着一层意思是，在抛出OutOfMemoryError之前，通常垃圾收集器会被触发，尽其所能去清理出空间。

   1. 例如：在引用机制分析中，涉及到JVM去尝试**回收软引用指向的对象**等。
   2. 在java.nio.Bits.reserveMemory()方法中，我们能清除看到，System.gc()会被调用，以清理空间。

4. 当然，也不是在任何情况下垃圾收集器都会被触发的：比如我们去分配一个超大对象，类似一个超大数组超过堆的最大值，JVM可以判断出垃圾收集器并不能解决这个问题，所以直接抛出OOM

### 内存泄漏

1. 也称作”存储渗漏“，严格来说，**只有对象不会再被程序用到了，但是GC又不能回收他们的情况，才叫内存泄漏**。
2. 但实际情况很多时候一些不太好的实践（或疏忽）会导致对象的生命周期变得很长甚至导致OOM，也可以叫做宽泛意义上的“内存泄漏”。
3. 尽管内存泄漏并不会立即引起程序崩溃，但是一旦发生内存泄漏，程序中的可用内存就会被逐步蚕食，直至耗尽所有内存，最终出现OOM异常，导致程序崩溃。
4. 注意，这里的存储空间并不是值物理内存，而是指虚拟内存大小，这个虚拟内存大小取决于磁盘交换区设定的大小。

**内存泄漏官方例子**

左边的图：Java使用可达性分析算法，最上面的数据不可达，就是需要被回收的对象。

右边的图：后期有一些对象不用了，按道理应该断开引用，但是存在一些链没有断开（图示中的Forgotten Reference Memory Leak），从而导致没有办法被回收。

![IMG/14、垃圾回收相关概念.assets/0006.png](深入理解Java虚拟机/0006.png)

**常见例子**

1. 单例模式
   - 单例的生命周期和应用程序是一样长的，所以在单例程序中，如果持有对外部对象的引用的话，那么这个外部对象是不能被回收的，则会导致内存泄漏的产生。
2. 一些提供close()的资源未关闭导致内存泄漏
   - 数据库连接dataSource.getConnection()，网络连接socker和io连接必须手动close，否则是不能被回收的。

## Stop the World

1. 简称STW，指的是GC事件发生过程中，会产生应用程序的停顿。**停顿产生时整个应用程序线程都会被暂停，没有任何响应**，有点像卡死的感觉。

2. 可达性分析算法中枚举根节点（GCRoots）会导致所有Java执行线程停顿，为什么需要停顿所有Java执行线程呢？

   - 分析工作必须在一个能确保一致性的快照中进行
   - 一致性指整个分析期间整个执行系统看起来像被冻结在某个时间点上
   - **如果出现分析过程中对象引用关系还在不断变化，则分析结果准确性无法保证**

3. 被STW中断的应用程序线程会在完成GC之后恢复，频繁中断会让用户感觉像是网速不快造成电影卡带一样，所以我们需要减少STW的发生

4. STW事件和采用哪款GC无关，所有的GC都有这个事件。

5. 哪怕是G1也不能完全避免STW情况的发生，只能说垃圾回收器越来越优秀，回收效率越来越高，尽可能地缩短了暂停时间

6. STW是JVM在**后台自动发起和自动完成**的。在用户不可见的情况下，把用户正常的工作线程全部停掉。

7. 开发中不要用System.gc()，这会导致STW的发生

## 垃圾回收的并行与并发

### 并发的概念

1. 在操作系统中，是指**一个时间段**中有几个程序都处于已启动运行到运行完毕之间，且这几个程序都是在同一个处理器上运行。
2. 并发不是真正意义上的同时进行，只是CPU把一个时间段划分成几个时间片段（时间区间），然后在这几个时间区间之间来回切换。由于CPU处理的速度非常快，只要时间间隔处理得当，即可让用户感觉是多个应用程序同时进行

![IMG/14、垃圾回收相关概念.assets/image-20210910101637422.png](深入理解Java虚拟机/image-20210910101637422.png)

### 并行的概念

1. 当系统有一个以上的CPU时，当一个CPU执行一个进程时，另一个CPU可以执行另一个进程，两个进程互相不抢占CPU资源，可以**同时**进行，我们称之为并行
2. 其实决定并行的因素不是CPU的数量，而是CPU的核心数量，比如一个CPU多个核也可以并行
3. 适合科学计算，后台处理等弱交互场景

![IMG/14、垃圾回收相关概念.assets/image-20210910101832115.png](深入理解Java虚拟机/image-20210910101832115.png)

> 并行与并发的对比

1. 并行（Parallel）：指多条垃圾收集线程并行工作，但此时用户线程仍处于等待状态
   - 如ParNew、Parallel Scavenge、Parallel Old
2. 串行（Serial）
   - 相较于并行的概念，单线程执行。
   - 如果内存不够，则程序暂停，启动JVM垃圾回收器进行垃圾回收（单线程）

![IMG/14、垃圾回收相关概念.assets/image-20210910102113842.png](深入理解Java虚拟机/image-20210910102113842.png)

并行和并发，在谈论垃圾收集器的上下文语境中，他们可以解释如下：

1. 并发（Concurrent）：指**用户线程与垃圾收集线程同时执行**（但不一定是并行的，可能会交替执行），垃圾回收线程在执行时不会停顿用户程序的运行
   - 比如用户程序在继续运行，而垃圾收集器程序线程运行于另一个CPU上；
2. 典型垃圾回收器：CMS、G1

![IMG/14、垃圾回收相关概念.assets/image-20210910102356847.png](深入理解Java虚拟机/image-20210910102356847.png)

## HotSpot的算法实现细节

### 根节点枚举

1. 固定可作为GCRoots的节点主要在全局性的引用（例如常量或类静态属性）与执行上下文（例如栈帧中的本地变量表）中，尽管目标明确，但查找过程要做到高效并非一件容易的事情，现在Java应用越做越大，光是方法区的大小就常有数百上千兆，里面的类、常量等更是恒河沙数，若要逐个检查以这里为起源的引用肯定得消耗不少时间。

2. 迄今为止，**所有垃圾收集器在根节点枚举这一步骤时都是必须暂停用户线程的**，因此毫无疑问根节点枚举与之前提及的整理内存碎片一样会面临相似的STW的困扰。现在可达性分析算法耗时 最长的查找引用链的过程已经可以做到与用户线程一起并发，**但根节点枚举始终还 是必须在一个能保障一致性的快照中才得以进行**——这里“一致性”的意思是整个枚举期间执行子系统 看起来就像被冻结在某个时间点上，不会出现分析过程中，根节点集合的对象引用关系还在不断变化 的情况，若这点不能满足的话，分析结果准确性也就无法保证。这是导致垃圾收集过程必须停顿所有 用户线程的其中一个重要原因，即使是号称停顿时间可控，或者（几乎）不会发生停顿的CMS、G1、 ZGC等收集器，枚举根节点时也是必须要停顿的。

   3、由于目前主流Java虚拟机使用的都是**准确式垃圾收集**，所以当用户线程停顿下来之后，其实并不需要一个不漏地检查完所有 执行上下文和全局的引用位置，虚拟机应当是有办法直接得到哪些地方存放着对象引用的。在HotSpot 的解决方案里，是使用一组称为**OopMap的数据结构**来达到这个目的。一旦类加载动作完成的时候， HotSpot就会把对象内什么偏移量上是什么类型的数据计算出来，在即时编译过程中，也 会在特定的位置记录下栈里和寄存器里哪些位置是引用。这样收集器在扫描时就可以直接得知这些信 息了，**并不需要真正一个不漏地从方法区等GC Roots开始查找**。

   4、Exact VM因它使用**准确式内存管理**（Exact Memory Management，也可以叫Non-Con- servative/Accurate Memory Management）而得名。准确式内存管理是指虚拟机可以知道内存中某个位 置的数据具体是什么类型。譬如内存中有一个32bit的整数123456，虚拟机将有能力分辨出它到底是一 个指向了123456的内存地址的引用类型还是一个数值为123456的整数，准确分辨出哪些内存是引用类 型，这也是在垃圾收集时准确判断堆上的数据是否还可能被使用的前提。【**这个不是特别重要，了解一下即可**】

   > 常考面试：**在OopMap的协助下，HotSpot可以快速准确地完成GC Roots枚举**

### 安全点与安全区域

**安全点（Safepoint）**

1. 程序执行时并非在所有地方都能停顿下来开始GC，只有在特定的位置才能停顿下来开始GC，这些位置称为“安全点（Safepoint）”。
2. Safe Point的选择很重要，**如果太少可能导致GC等待的时间太长，如果太频繁可能导致运行时的性能问题**。大部分指令的执行时间都非常短暂，通常会根据“**是否具有让程序长时间执行的特征**”为标准。比如：选择一些执行时间较长的指令作为Safe Point，**如方法调用、循环跳转和异常跳转等**。

**如何在GC发生时，检查所有线程都跑到最近的安全点停顿下来呢？**

1. 抢先式中断：（目前没有虚拟机采用了）首先中断所有线程。如果还有线程不在安全点，就恢复线程，让线程跑到安全点。
2. 主动式中断：设置一个中断标志，各个线程运行到Safe Point的时候**主动轮询**这个标志，如果中断标志为真，则将自己进行中断挂起。

**安全区域（Safe Region）**

1. Safepoint 机制保证了程序执行时，在不太长的时间内就会遇到可进入GC的Safepoint。但是，程序“不执行”的时候呢？
2. 例如线程处于Sleep状态或Blocked 状态，这时候线程无法响应JVM的中断请求，“走”到安全点去中断挂起，JVM也不太可能等待线程被唤醒。对于这种情况，就需要安全区域（Safe Region）来解决。
3. **安全区域是指在一段代码片段中，对象的引用关系不会发生变化，在这个区域中的任何位置开始GC都是安全的**。我们也可以把Safe Region看做是被扩展了的Safepoint。

**安全区域的执行流程**

1. 当线程运行到Safe Region的代码时，首先标识已经进入了Safe Region，如果这段时间内发生GC，JVM会忽略标识为Safe Region状态的线程
2. 当线程即将离开Safe Region时，会检查JVM是否已经完成根节点枚举（即GC Roots的枚举），如果完成了，则继续运行，否则线程必须等待直到收到可以安全离开Safe Region的信号为止；

### 记忆集与卡表

#### 什么是跨代引用？

1、一般的垃圾回收算法至少会划分出两个年代，年轻代和老年代。但是单纯的分代理论在垃圾回收的时候存在一个巨大的缺陷：为了找到年轻代中的存活对象，却不得不遍历整个老年代，反过来也是一样的。

![IMG/14、垃圾回收相关概念.assets/image-20210910104835512.png](深入理解Java虚拟机/image-20210910104835512.png)

2、如果我们从年轻代开始遍历，那么可以断定N, S, P, Q都是存活对象。但是，V却不会被认为是存活对象，其占据的内存会被回收了。这就是一个惊天的大漏洞！因为U本身是老年代对象，而且有外部引用指向它，也就是说U是存活对象，而U指向了V，也就是说V也应该是存活对象才是！而这都是因为我们只遍历年轻代对象！

3、所以，为了解决这种跨代引用的问题，最笨的办法就是遍历老年代的对象，找出这些跨代引用来。这种方案存在极大的性能浪费。因为从两个分代假说里面，其实隐含了一个推论：跨代引用是极少的。也就是为了找出那么一点点跨代引用，我们却得遍历整个老年代！从上图来说，很显然的是，我们根本不必遍历R。

4、因此，为了避免这种遍历老年代的性能开销，通常的分代垃圾回收器会引入一种称为**记忆集**的技术。**简单来说，记忆集就是用来记录跨代引用的表。**

#### 记忆集与卡集

1、为解决对象跨代引用所带来的问题，垃圾收集器在新生代中建 立了名为**记忆集（Remembered Set）的数据结构**，用以避免把整个老年代加进GC Roots扫描范围。事实上并不只是新生代、老年代之间才有跨代引用的问题，所有涉及部分区域收集（Partial GC）行为的 垃圾收集器，典型的如G1、ZGC和Shenandoah收集器，都会面临相同的问题，因此我们有必要进一步 理清记忆集的原理和实现方式，以便在后续章节里介绍几款最新的收集器相关知识时能更好地理解。

2、记忆集是一种用于记录**从非收集区域指向收集区域的指针集合的抽象数据结构**。如果我们不考虑效率和成本的话，最简单的实现可以用非收集区域中所有含跨代引用的对象数组来实现这个数据结构。

> 比如说我们有老年代（非收集区域）和年轻代（收集区域）的对象之间有一条引用链

3、这种记录全部含跨代引用对象的实现方案，无论是空间占用还是维护成本都相当高昂。而在垃圾 收集的场景中，收集器只需要通过记忆集判断出某一块非收集区域是否存在有指向了收集区域的指针 就可以了，并不需要了解这些跨代指针的全部细节。那设计者在实现记忆集的时候，便可以选择更为 粗犷的记录粒度来节省记忆集的存储和维护成本，下面列举了一些可供选择（当然也可以选择这个范 围以外的）的记录精度：

- 字长精度：每个记录精确到一个机器字长（就是处理器的寻址位数，如常见的32位或64位，这个 精度决定了机器访问物理内存地址的指针长度），该字包含跨代指针。
- 对象精度：每个记录精确到一个对象，该对象里有字段含有跨代指针。
- 卡精度：每个记录精确到一块内存区域，该区域内有对象含有跨代指针。

4、其中，第三种“卡精度”所指的是用一种称为“卡表”（Card Table）的方式去实现记忆集，这也是 目前最常用的一种记忆集实现形式，一些资料中甚至直接把它和记忆集混为一谈。前面定义中提到记 忆集其实是一种“抽象”的数据结构，抽象的意思是只定义了记忆集的行为意图，并没有定义其行为的 具体实现。卡表就是记忆集的一种具体实现，它定义了记忆集的记录精度、与堆内存的映射关系等。 关于卡表与记忆集的关系，读者不妨按照Java语言中HashMap与Map的关系来类比理解。 卡表最简单的形式可以只是一个字节数组，而HotSpot虚拟机确实也是这样做的

> 读者只需要知道有这个东西，面试的时候能说出来，再细致一点的就需要看周志明老师的第三版书了

## 再谈引用概述

1. 我们希望能描述这样一类对象：当内存空间还足够时，则能保留在内存中；如果内存空间在进行垃圾收集后还是很紧张，则可以抛弃这些对象。
2. 既偏门又非常高频的面试题：强引用、软引用、弱引用、虚引用有什么区别？具体使用场景是什么？
3. 在JDK1.2版之后，Java对引用的概念进行了扩充，将引用分为：
   - 强引用（Strong Reference）
   - 软引用（Soft Reference）
   - 弱引用（Weak Reference）
   - 虚引用（Phantom Reference）
4. 这4种引用强度依次逐渐减弱。除强引用外，其他3种引用均可以在java.lang.ref包中找到它们的身影。如下图，显示了这3种引用类型对应的类，开发人员可以在应用程序中直接使用它们。

![img](深入理解Java虚拟机/0012-16516627146513.png)

Reference子类中只有终结器引用是包内可见的，其他3种引用类型均为public，可以在应用程序中直接使用

1. 强引用（StrongReference）：最传统的“引用”的定义，是指在程序代码之中普遍存在的引用赋值，即类似“`object obj=new Object()`”这种引用关系。无论任何情况下，只要强引用关系还存在，垃圾收集器就永远不会回收掉被引用的对象。宁可报OOM，也不会GC强引用
2. 软引用（SoftReference）：在系统将要发生内存溢出之前，将会把这些对象列入回收范围之中进行第二次回收。如果这次回收后还没有足够的内存，才会抛出内存溢出异常。
3. 弱引用（WeakReference）：被弱引用关联的对象只能生存到下一次垃圾收集之前。当垃圾收集器工作时，无论内存空间是否足够，都会回收掉被弱引用关联的对象。
4. 虚引用（PhantomReference）：一个对象是否有虚引用的存在，完全不会对其生存时间构成影响，也无法通过虚引用来获得一个对象的实例。为一个对象设置虚引用关联的唯一目的就是能在这个对象被收集器回收时收到一个系统通知。

## 再谈引用：强引用

1. 在Java程序中，最常见的引用类型是强引用（普通系统99%以上都是强引用），也就是我们最常见的普通对象引用，**也是默认的引用类型**。
2. 当在Java语言中使用new操作符创建一个新的对象，并将其赋值给一个变量的时候，这个变量就成为指向该对象的一个强引用。
3. **只要强引用的对象是可触及的，垃圾收集器就永远不会回收掉被引用的对象。**只要强引用的对象是可达的，jvm宁可报OOM，也不会回收强引用。
4. 对于一个普通的对象，如果没有其他的引用关系，只要超过了引用的作用域或者显式地将相应（强）引用赋值为null，就是可以当做垃圾被收集了，当然具体回收时机还是要看垃圾收集策略。
5. 相对的，软引用、弱引用和虚引用的对象是软可触及、弱可触及和虚可触及的，在一定条件下，都是可以被回收的。所以，强引用是造成Java内存泄漏的主要原因之一。

**总结**

1. 强引用可以直接访问目标对象。
2. 强引用所指向的对象在任何时候都不会被系统回收，虚拟机宁愿抛出OOM异常，也不会回收强引用所指向对象。
3. 强引用可能导致内存泄漏。

## 再谈引用：软引用

**软引用（Soft Reference）：内存不足即回收**

1. 软引用是用来描述一些还有用，但非必需的对象。只被软引用关联着的对象，在系统将要发生内存溢出异常前，会把这些对象列进回收范围之中进行第二次回收，如果这次回收还没有足够的内存，才会抛出内存溢出异常。注意，这里的第一次回收是不可达的对象
2. 软引用通常用来实现内存敏感的缓存。比如：高速缓存就有用到软引用。如果还有空闲内存，就可以暂时保留缓存，当内存不足时清理掉，这样就保证了使用缓存的同时，不会耗尽内存。
3. 垃圾回收器在某个时刻决定回收软可达的对象的时候，会清理软引用，并可选地把引用存放到一个引用队列（Reference Queue）。
4. 类似弱引用，只不过Java虚拟机会尽量让软引用的存活时间长一些，迫不得已才清理。
5. 一句话概括：当内存足够时，不会回收软引用可达的对象。内存不够时，会回收软引用的可达对象

在JDK1.2版之后提供了SoftReference类来实现软引用

```java
Object obj = new Object();// 声明强引用
SoftReference<Object> sf = new SoftReference<>(obj);
obj = null; //销毁强引用
```

## 再谈引用：弱引用

> **弱引用（Weak Reference）发现即回收**

1. 弱引用也是用来描述那些非必需对象，**只被弱引用关联的对象只能生存到下一次垃圾收集发生为止。在系统GC时，只要发现弱引用，不管系统堆空间使用是否充足，都会回收掉只被弱引用关联的对象**。
2. 但是，由于垃圾回收器的线程通常优先级很低，因此，并不一定能很快地发现持有弱引用的对象。在这种情况下，弱引用对象可以存在较长的时间。
3. 弱引用和软引用一样，在构造弱引用时，也可以指定一个引用队列，当弱引用对象被回收时，就会加入指定的引用队列，通过这个队列可以跟踪对象的回收情况。
4. 软引用、弱引用都非常适合来保存那些可有可无的缓存数据。如果这么做，当系统内存不足时，这些缓存数据会被回收，不会导致内存溢出。而当内存资源充足时，这些缓存数据又可以存在相当长的时间，从而起到加速系统的作用。

在JDK1.2版之后提供了WeakReference类来实现弱引用

```java
// 声明强引用
Object obj = new Object();
WeakReference<Object> sf = new WeakReference<>(obj);
obj = null; //销毁强引用Copy to clipboardErrorCopied
```

弱引用对象与软引用对象的最大不同就在于，当GC在进行回收时，需要通过算法检查是否回收软引用对象，而对于弱引用对象，GC总是进行回收。弱引用对象更容易、更快被GC回收。

**面试题：你开发中使用过WeakHashMap吗？**

## 再谈引用：虚引用

**虚引用（Phantom Reference）：对象回收跟踪**

1. 也称为“幽灵引用”或者“幻影引用”，是所有引用类型中最弱的一个
2. 一个对象是否有虚引用的存在，完全不会决定对象的生命周期。如果一个对象仅持有虚引用，那么它和没有引用几乎是一样的，随时都可能被垃圾回收器回收。
3. 它不能单独使用，也无法通过虚引用来获取被引用的对象。当试图通过虚引用的get()方法取得对象时，总是null 。**即通过虚引用无法获取到我们的数据**
4. **为一个对象设置虚引用关联的唯一目的在于跟踪垃圾回收过程。比如：能在这个对象被收集器回收时收到一个系统通知。**
5. 虚引用必须和引用队列一起使用。虚引用在创建时必须提供一个引用队列作为参数。当垃圾回收器准备回收一个对象时，如果发现它还有虚引用，就会在回收对象后，将这个虚引用加入引用队列，以通知应用程序对象的回收情况。
6. 由于虚引用可以跟踪对象的回收时间，因此，也可以将一些资源释放操作放置在虚引用中执行和记录。

在JDK1.2版之后提供了PhantomReference类来实现虚引用。

```java
// 声明强引用
Object obj = new Object();
// 声明引用队列
ReferenceQueue phantomQueue = new ReferenceQueue();
// 声明虚引用（还需要传入引用队列）
PhantomReference<Object> sf = new PhantomReference<>(obj, phantomQueue);
obj = null; 
```

## 再谈引用：终结器引用（了解）

1. 它用于实现对象的finalize() 方法，也可以称为终结器引用
2. 无需手动编码，其内部配合引用队列使用
3. 在GC时，终结器引用入队。由Finalizer线程通过终结器引用找到被引用对象调用它的finalize()方法，第二次GC时才回收被引用的对象

# 第九章 垃圾回收器

## GC分类与性能指标

### 垃圾回收器概述

1. 垃圾收集器没有在规范中进行过多的规定，可以由不同的厂商、不同的版本JVM来实现
2. 由于JDK版本处于高速迭代过程中，因此Java发展至今已经衍生了众多的GC版本
3. 从不同角度分析垃圾收集器，可以将GC分为不同的类型

### 垃圾回收器分类

**按线程数分（垃圾回收线程数），可以分为串行垃圾回收器和并行垃圾回收器**

1. 串行回收指的是在同一时间段只允许有一个CPU用于执行垃圾回收操作，此时工作线程被暂停，直至垃圾收集工作结束
   1. 在诸如单CPU处理器或者较小的应用内存等硬件平台不是特别优越的场合，串行回收器的性能表现可以超过并行回收器和并发回收器。所以串行回收默认被应用在客户端的Client模式下的JVM中
   2. 在并发能力较强的CPU上，并行回收器产生的停顿时间要短于串行回收器
2. 和串行回收器相反，并行收集可以运用多个CPU同时执行垃圾回收，因此提高了应用的吞吐量，不过并行回收依然与串行回收器一样，采用独占式，使用了STW机制

**按工作模式分，可以分为并发式垃圾回收器和独占式垃圾回收器**

1. 并发式垃圾回收器与应用程序线程交替工作，以尽可能减少应用程序的停顿时间。
2. 独占式垃圾回收器（STW）一旦运行，就停止应用程序中的所有用户线程，直到垃圾回收过程结束

![img](深入理解Java虚拟机/image-20210910185741502.png)

**按碎片处理方式分，可以分为压缩式垃圾回收器和非压缩式垃圾回收器**

1. 压缩式垃圾回收器会在回收完成后，对存活对象进行压缩整理，消除回收后的碎片。再分配对象空间。使用指针碰撞。
2. 非压缩式的垃圾回收器不会进行这步操作，分配对象空间。使用空闲列表

**按工作的内存区间分，可分为年轻代垃圾回收和老年代垃圾回收**

### 评估GC的性能指标

#### 指标

1. **吞吐量**：运行用户代码的时间占总运行时间的比例（总运行时间 = 程序的运行时间 + 内存回收的时间）
2. 垃圾收集开销：吞吐量的补数，垃圾收集所用时间与总运行时间的比例
3. **暂停时间**：执行垃圾收集时，程序的工作线程被暂停的时间
4. 收集频率：相对于应用程序的执行，收集操作发生的频率
5. **内存占用**：Java堆区所占的内存大小
6. 快速：一个对象从诞生到被回收所经历的时间

- 吞吐量、暂停时间、内存占用这三者共同构成一个“不可能三角”。三者总体的表现会随着技术进步而越来越好。一款优秀的收集器通常最多同时满足其中的两项。
- 这三项里，暂停时间的重要性日益凸显。因为随着硬件的发展，内存占用多些越来越能容忍，硬件性能的提升也有助于降低收集器运行时对应用程序的影响，即提高了吞吐量。而内存的扩大对延迟反而带来负面效果。
- 简单来说抓住两点：吞吐量、暂停时间

#### 吞吐量（throughput）

1. 吞吐量就是CPU用于运行用户代码的时间与CPU总消耗时间的比值，即吞吐量 = 运行用户代码时间 / （运行用户代码时间+垃圾收集时间）
2. 这种情况下，应用程序能容忍较高的暂停时间，因此，吞吐量的应用程序有更长的时间基准，快速响应是不必考虑的。
3. 吞吐量优先意味着在单位时间内，STW时间最短

![image-20210910191705132](深入理解Java虚拟机/image-20210910191705132.png)

#### 暂停时间（paus time）

1. 暂停时间是值一个时间段内应用程序线程暂停，让GC线程执行的状态。
2. 暂停时间优先意味着尽可能让单次STW的时间最短，但是总的GC时间可能会长

![img](深入理解Java虚拟机/0004-16516627404644.png)

#### 吞吐量vs暂停时间

1. **高吞吐量较好**：因为这会让应用程序的最终用户感觉只有应用程序线程在做“生产性”工作。直觉上，吞吐量越高程序运行越快
2. 低暂停时间（低延迟）较好，是从最终用户的角度来看，不管是GC还是其他原因导致一个应用被挂起始终是不好的。这取决于应用程序的类型，有时候甚至短暂的200毫秒暂停都可能打断终端用户体验。因此具有较低的暂停时间是非常重要的，特别是对于一个交互式应用程序（就是和用户交互比较多的场景）
3. 不幸的是高吞吐量和低暂停时间是一对相互竞争的目标
   - 因为如果选择以吞吐量优先，那么**必然需要降低内存回收的执行频率**，但是这样会导致GC需要更长时间的暂停时间来执行内存回收
   - 相反的，如果选择以低延迟优先为原则，那么为了降低每次执行内存回收时的暂停时间，也只能频繁地执行内存回收，但这又引起了年轻代内存的缩减和导致程序吞吐量的下降
4. 在设计或使用GC算法时，我们必须确定我们的目标：一个GC算法只可能针对两个目标之一、或者尝试找到有一个二者的折中
5. 现在的标准：**在最大吞吐量优先的情况下，降低停顿时间**

## 不同的垃圾回收器概述

1. 垃圾收集机制是Java的招牌能力，极大地提高了开发效率
2. 那么Java常见的垃圾收集器有哪些？

### 垃圾收集器发展史

有了虚拟机，就一定需要收集垃圾的机制，这就是Garbage Collection，对应的产品我们称为Garbage Collector

1. 1999年随JDK1.3.1一起来的是串行方式的Serial GC，它是第一款GC。ParNew垃圾收集器是Serial收集器的多线程版本
2. 2002年2月26日，Parallel GC和Concurrent Mark Sweep GC跟随JDK1.4.2一起发布·
3. **Parallel GC在JDK6之后成为HotSpot默认GC**。
4. 2012年，在JDK1.7u4版本中，G1可用。
5. **2017年，JDK9中G1变成默认的垃圾收集器，以替代CMS**。
6. 2018年3月，JDK10中G1垃圾回收器的并行完整垃圾回收，实现并行性来改善最坏情况下的延迟。
7. 2018年9月，JDK11发布。引入Epsilon 垃圾回收器，又被称为 "No-Op(无操作)“ 回收器。同时，引入ZGC：可伸缩的低延迟垃圾回收器（Experimental）
8. 2019年3月，JDK12发布。增强G1，自动返回未用堆内存给操作系统。同时，引入Shenandoah GC：低停顿时间的GC（Experimental）。
9. 2019年9月，JDK13发布。增强ZGC，自动返回未用堆内存给操作系统。
10. 2020年3月，**JDK14发布。删除CMS垃圾回收器**。扩展ZGC在macOS和Windows上的应用

### 7款经典的垃圾收集器

1. 串行回收器：Serial、Serial Old
2. 并行回收器：ParNew、Parallel Scavenge、Parallel Old
3. 并发回收器：CMS、G1

**7款经典回收器与垃圾分代之间的关系**

![img](深入理解Java虚拟机/0007.png)

1. 新生代收集器：Serial、ParNew、Parallel Scavenge；
2. 老年代收集器：Serial old、Parallel old、CMS；
3. 整堆收集器：G1；

### 垃圾收集器的组合关系

![img](深入理解Java虚拟机/0008.png)

1. 两个收集器间有连线，表明它们可以搭配使用：
   - Serial/Serial old
   - Serial/CMS （JDK9废弃）
   - ParNew/Serial Old （JDK9废弃）
   - ParNew/CMS
   - Parallel Scavenge/Serial Old （预计废弃）
   - Parallel Scavenge/Parallel Old
   - G1
2. 其中Serial Old作为CMS出现"Concurrent Mode Failure"失败的后备预案。
3. （红色虚线）由于维护和兼容性测试的成本，在JDK 8时将Serial+CMS、ParNew+Serial Old这两个组合声明为废弃（JEP173），并在JDK9中完全取消了这些组合的支持（JEP214），即：移除。
4. （绿色虚线）JDK14中：弃用Parallel Scavenge和Serial Old GC组合（JEP366）
5. （青色虚线）JDK14中：删除CMS垃圾回收器（JEP363）

- 为什么要有很多收集器，一个不够用吗？因为Java的使用场景很多，移动端、服务器等。所以就需要针对不同的场景，提供不同的垃圾收集器，提高垃圾收集的性能
- 虽然我们会对各个收集器进行比较，但并非为了挑选一个最好的收集器出来。没有一种放之四海而皆准、任何场景下都适用的完美收集器存在，更加没有万能的收集器。所以**我们选择的只是对具体应用最合适的收集器**

### 查看默认垃圾收集器

1. -XX:+PrintCommandLineFlags：查看命令行相关参数（包含使用的垃圾收集器）
2. 使用命令行指令：jinfo -flag 相关垃圾回收器参数 进程ID

## Serial回收器：串行回收

1. Serial收集器是最基本、历史最悠久的垃圾收集器了。JDK1.3之前回收新生代唯一的选择
2. Serial收集器作为HotSpot中Client模式下的默认新生代垃圾收集器
3. Serial收集器采用复制算法、串行回收和STW机制的方式执行内存回收。
4. 除了年轻代之外，Serial收集器还提供用于执行老年代垃圾收集的Serial Old收集器。Serial Old收集器同样也采用了串行回收和STW机制，只不过内存回收算法使用的是标记-压缩算法
5. Serial Old是运行在Client模式下默认的老年代的垃圾回收器，Serial Old在Server模式下主要有两个用途：① 与新生代的Parallel Scavenge配合使用 ② 作为老年代CMS收集器的后备垃圾收集方案

这个收集器是一个单线程的收集器，单线程的意义：他只会使用一个CPU（串行）或者一条收集线程去完成垃圾收集工作。更重要的是在它进行垃圾收集时，必须暂停其他所有的工作线程，直到它收集结束。

![img](深入理解Java虚拟机/0011-16516627404645.png)

**Serial 回收器的优势**

1. 优势：简单而高效（与其他收集器的单线程比），对于限定单个CPU的环境来说，Serial收集器由于没有线程交互的开销，专心做垃圾收集自然可以获得最高的单线程收集效率，运行在Client模式下的虚拟机是个不错的选择
2. 在用户的桌面应用场景中，可用内存一般不大，可以在较短时间内完成垃圾收集，只要不频繁发生，使用串行回收器是可以接受的。
3. 在HotSpot虚拟机中，使用-XX:UseSerialGC参数可以指定年轻代和老年代都是用串行收集器，等价于新生代用SerialGC，且老年代使用SerialOldGC

**总结**

1. 这种垃圾收集器大家了解，现在已经不用串行的了。而且在限定单核CPU才可以用。现在都不是单核的了
2. 对于交互较强的应用而言，这种垃圾收集器是不能接受的。一般在JavaWeb应用程序中是不会采用串行垃圾收集器的。

## ParNew回收器：并行回收

1. 如果说Serial GC是年轻代中的单线程垃圾收集器，那么ParNew收集器则是Serial收集器则是Serial收集器的多线程版本
   - Par是Parallel的缩写，New：只能处理新生代
2. ParNew收集器除了采用**并行回收**的方式执行内存回收外，两款垃圾收集器之间几乎没有任何区别，ParNew收集器在年轻代中同样也是采用复制算法、STW机制
3. ParNew是很多JVM运行在Server模式下新生代的默认垃圾收集器

![img](深入理解Java虚拟机/0012-16516627404646.png)

1. 对于新生代，回收次数频繁，使用并行方式高效
2. 对于老年代，回收次数少，使用串行方式节省资源。（CPU并行需要切换线程，串行可以省去切换线程的资源）

**ParNew回收器与Serial回收器比较**

Q：由于ParNew收集器基于并行回收，那么是否可以断定ParNew收集器的回收效率在任何场景下都会比Serial收集器更高效？

A：**不能**

1. ParNew收集器运行在多CPU的环境下，由于可以充分利用多CPU、多核心等物理硬件资源优势，可以更快速地完成垃圾收集，提升程序的吞吐量。
2. 但是在单个CPU的环境下，ParNew收集器不比Serial收集器更高效。虽然Serial收集器是基于串行回收，但是由于CPU不需要频繁地做任务切换，因此可以有效避免多线程交互过程中产生的一些额外开销。
3. 除Serial外，目前只有ParNew GC能与CMS收集器配合工作

**设置ParNew垃圾回收器**

1. 在程序中，开发人员可以通过选项"-XX:+UseParNewGC"手动指定使用ParNew收集器执行内存回收任务。它表示年轻代使用并行收集器，不影响老年代。
2. -XX:ParallelGCThreads限制线程数量，默认开启和CPU数据相同的线程数。

## Parallel回收器：吞吐量优先

HotSpot的年轻代中除了拥有ParNew收集器是基于并行回收的以外，ParallelScavenge收集器同样也采用了复制算法、并行回收和STW机制

那么Parallel收集器的出现是否多次一举？

- 和ParNew收集器不同，ParallelScavenge收集器的目标则是达到一个**可控制的吞吐量**，它也被称为吞吐量优先的垃圾收集器
- 自适应调节策略也是ParallelScavenge与ParNew一个重要的区别。（动态调整内存分配情况，以达到一个最优的吞吐量或低延迟）

高吞吐量则可以高效地利用CPU时间，尽快完成程序的运算任务，**主要适合在后台运算而不需要太多交互的任务**。因此常见在服务器环境中使用。例如，那些执行批量处理、订单处理、工资支付、科学计算的应用程序。

Paraller收集器在JDK1.6时提供了用于执行老年代垃圾收集的Parallel Old收集器，用来代替老年代的SerialOld收集器。

ParallelOld收集器采用了标记压缩算法，但同样也是基于并行回收和STW机制

![img](深入理解Java虚拟机/0013-16516627404647.png)

1. 在程序吞吐量优先的应用场景中，Parallel收集器和ParallelOld收集器的组合，在Server模式下的内存回收性能很不错
2. **在Java8中，默认是此垃圾收集器**

**Parallel Scavenge 回收参数设置**

1. -XX:+UseParallelGC 手动指定年轻代使用Parallel并行收集器执行内存回收任务。

2. -XX:+UseParallelOldGC：手动指定老年代都是使用并行回收收集器。

   - 分别适用于新生代和老年代
   - 上面两个参数分别适用于新生代和老年代。默认jdk8是开启的。默认开启一个，另一个也会被开启。（互相激活）

3. -XX:ParallelGCThreads：设置年轻代并行收集器的线程数。一般地，最好与CPU数量相等，以避免过多的线程数影响垃圾收集性能。

   1. 在默认情况下，当CPU数量小于8个，ParallelGCThreads的值等于CPU数量。
   2. 当CPU数量大于8个，ParallelGCThreads的值等于3+[5*CPU_Count]/8]

4. -XX:MaxGCPauseMillis 设置垃圾收集器最大停顿时间（即STW的时间）。单位是毫秒。

   1. 为了尽可能地把停顿时间控制在XX:MaxGCPauseMillis 以内，收集器在工作时会调整Java堆大小或者其他一些参数。
   2. 对于用户来讲，停顿时间越短体验越好。但是在服务器端，我们注重高并发，整体的吞吐量。所以服务器端适合Parallel，进行控制。
   3. 该参数使用需谨慎。

5. -XX:GCTimeRatio垃圾收集时间占总时间的比例，即等于 1 / (N+1) ，用于衡量吞吐量的大小。

   1. 取值范围(0, 100)。默认值99，也就是垃圾回收时间占比不超过1。
   2. 与前一个-XX:MaxGCPauseMillis参数有一定矛盾性，STW暂停时间越长，Radio参数就容易超过设定的比例。

6. -XX:+UseAdaptiveSizePolicy 设置Parallel Scavenge收集器具有**自适应调节策略**

   1. 在这种模式下，年轻代的大小、Eden和Survivor的比例、晋升老年代的对象年龄等参数会被自动调整，已达到在堆大小、吞吐量和停顿时间之间的平衡点。
   2. 在手动调优比较困难的场合，可以直接使用这种自适应的方式，仅指定虚拟机的最大堆、目标的吞吐量（GCTimeRatio）和停顿时间（MaxGCPauseMillis），让虚拟机自己完成调优工作。

## CMS回收器：低延迟

### CMS回收器

1. 在JDK1.5时期，HotSpot推出了一款在**强交互应用中（就是和用户打交道的应用）**，几乎可以认为具有划时代意义的垃圾收集器：CMS（ConcurrentMarkSweep）收集器，**这款收集器是HotSpot虚拟机中第一款真正意义上的并发收集器，它第一次实现了让垃圾收集线程与用户线程同时工作**
2. CMS收集器的关注点是尽可能缩短垃圾收集时用户线程的停顿时间。停顿时间越短就越适合与用户交互的程序，良好的响应速度能提升用户体验。
   - 目前很大一部分的Java应用集中在互联网站或者BS系统的服务端上，这类应用尤其重视服务的响应速度，希望系统停顿时间最短，以给用户带来较好的体验。CMS收集器就非常符合这类应用的需求。
3. CMS的垃圾收集算法采用标记清除算法，并且也会STW
4. 不幸的是，CMS作为老年代的收集器，却无法与JDK1.4中已经存在的新生代收集器ParallelScavenge配合工作（因为实现的框架不一样，没办法兼容使用），所以在JDK1.5中使用CMS来收集老年代的时候，新生代只能选择ParNew或者Serial收集器中的一个
5. 在G1出现之前，CMS使用还是非常广泛的。一直到今天，仍然还有很多系统使用CMSGC

### CMS工作原理（过程）

![img](深入理解Java虚拟机/0014-16516627404648.png)

CMS整个过程比之前的收集器要复杂，整个过程分为4个主要阶段，即初始标记阶段、并发标记阶段、重新标记阶段和并发清除阶段（涉及STW的阶段主要是：初始标记和重新标记）

1. 初始标记（Initial-Mark）阶段：在这个阶段中，程序中所有的工作线程都将会因为STW机制而出现短暂的暂停，**这个阶段的主要任务仅仅只是标记出GCRoots能直接关联到的对象**。一旦标记完成之后就会恢复之前被暂停的所有应用线程。由于直接关联的对象比较小，所以这里的**速度非常快**
2. 并发标记（Concurrent-Mark）阶段：从GC Roots的直接关联对象开始遍历整个对象图的过程，这个过程耗时较长但是**不需要停顿用户线程**，**可以与垃圾收集线程一起并发运行**。
3. 重新标记（Remark）阶段：由于在并发标记阶段中，程序的工作线程会和垃圾收集线程同时运行或者交叉运行，**因此为了修正并发标记期间，因用户程序继续运作而导致标记产生变动的那一部分对象的标记记录，**这个阶段的停顿时间通常会比初始标记阶段稍长一些，并且也会导致“Stop-the-World”的发生，但也远比并发标记阶段的时间短。
4. 并发清除（Concurrent-Sweep）阶段：此阶段清理删除掉标记阶段判断的已经死亡的对象，释放内存空间。**由于不需要移动存活对象，所以这个阶段也是可以与用户线程同时并发的**

### CMS分析

1. 尽管CMS收集器采用的是并发回收（非独占式），**但是在其初始化标记和再次标记这两个阶段中仍然需要执行“Stop-the-World”机制**暂停程序中的工作线程，不过暂停时间并不会太长，因此可以说明目前所有的垃圾收集器都做不到完全不需要“Stop-the-World”，只是尽可能地缩短暂停时间。
2. **由于最耗费时间的并发标记与并发清除阶段都不需要暂停工作，所以整体的回收是低停顿的**。
3. 另外，由于在垃圾收集阶段用户线程没有中断，所以在CMS回收过程中，还应该确保应用程序用户线程有足够的内存可用。因此，CMS收集器不能像其他收集器那样等到老年代几乎完全被填满了再进行收集，**而是当堆内存使用率达到某一阈值时，便开始进行回收**，以确保应用程序在CMS工作过程中依然有足够的空间支持应用程序运行。要是CMS运行期间预留的内存无法满足程序需要，就会出现一次**“Concurrent Mode Failure”** 失败，这时虚拟机将启动后备预案：临时启用Serial old收集器来重新进行老年代的垃圾收集，这样停顿时间就很长了。
4. CMS收集器的垃圾收集算法采用的是**标记清除算法**，这意味着每次执行完内存回收后，由于被执行内存回收的无用对象所占用的内存空间极有可能是不连续的一些内存块，**不可避免地将会产生一些内存碎片**。那么CMS在为新对象分配内存空间时，将无法使用指针碰撞（Bump the Pointer）技术，而只能够选择空闲列表（Free List）执行内存分配。

**为什么 CMS 不采用标记-压缩算法呢？**

答案其实很简答，因为当并发清除的时候，用Compact整理内存的话，原来的用户线程使用的内存还怎么用呢？要保证用户线程能继续执行，前提的它运行的资源不受影响嘛。Mark Compact更适合“stop the world”这种场景下使用

### CMS的优点与弊端

**优点**

1. 并发收集
2. 低延迟

**弊端**

1. **会产生内存碎片**，导致并发清除后，用户线程可用的空间不足。在无法分配大对象的情况下，不得不提案触发FullGC
2. **CMS收集器对CPU资源非常敏感**。在并发阶段，他虽然不会导致用户停顿，但是会因为占用了一部分线程而导致应用程序变慢，总吞吐量会降低
3. **CMS收集器无法处理浮动垃圾**。可能出现”Concurrent Mode Failure“失败而导致另一次Full GC的产生。在并发标记阶段由于程序的工作线程和垃圾收集线程是同时运行或者交叉运行的，**那么在并发标记阶段如果产生新的垃圾对象，CMS将无法对这些垃圾对象进行标记，最终会导致这些新产生的垃圾对象没有被及时回收**，从而只能在下一次执行GC时释放这些之前未被回收的内存空间。

### CMS参数配置

- -XX:+UseConcMarkSweepGC：手动指定使用CMS收集器执行内存回收任务。

  开启该参数后会自动将-XX:+UseParNewGC打开。即：ParNew（Young区）+CMS（Old区）+Serial Old（Old区备选方案）的组合。

- -XX:CMSInitiatingOccupanyFraction：设置堆内存使用率的阈值，一旦达到该阈值，便开始进行回收。

1. JDK5及以前版本的默认值为68，即当老年代的空间使用率达到68%时，会执行一次CMS回收。JDK6及以上版本默认值为92%

2. 如果内存增长缓慢，则可以设置一个稍大的值，大的阀值可以有效降低CMS的触发频率，减少老年代回收的次数可以较为明显地改善应用程序性能。反之，如果应用程序内存使用率增长很快，则应该降低这个阈值，以避免频繁触发老年代串行收集器。因此通过该选项便可以有效降低Full GC的执行次数。

- -XX:+UseCMSCompactAtFullCollection：用于指定在执行完Full GC后对内存空间进行压缩整理，以此避免内存碎片的产生。不过由于内存压缩整理过程无法并发执行，所带来的问题就是停顿时间变得更长了。

- -XX:CMSFullGCsBeforeCompaction：设置在执行多少次Full GC后对内存空间进行压缩整理。

- -XX:ParallelCMSThreads：设置CMS的线程数量。

1. CMS默认启动的线程数是 (ParallelGCThreads + 3) / 4，ParallelGCThreads是年轻代并行收集器的线程数，可以当做是 CPU 最大支持的线程数。当CPU资源比较紧张时，受到CMS收集器线程的影响，应用程序的性能在垃圾回收阶段可能会非常糟糕。

### 小结

HotSpot有这么多的垃圾回收器，那么如果有人问，Serial GC、Parallel GC、Concurrent Mark Sweep GC这三个GC有什么不同呢？

1. 如果你想要最小化地使用内存和并行开销，请选Serial GC；
2. 如果你想要最大化应用程序的吞吐量，请选Parallel GC；
3. 如果你想要最小化GC的中断或停顿时间，请选CMS GC。

### JDK后续版本中的CMS变化

1. JDK9新特性：CMS被标记为Deprecate了（JEP291）
   - 如果对JDK9及以上版本的HotSpot虚拟机使用参数-XX:+UseConcMarkSweepGC来开启CMS收集器的话，用户会收到一个警告信息，提示CMS未来将会被废弃。
2. JDK14新特性：删除CMS垃圾回收器（JEP363）移除了CMS垃圾收集器，
   - 如果在JDK14中使用XX:+UseConcMarkSweepGC的话，JVM不会报错，只是给出一个warning信息，但是不会exit。JVM会自动回退以默认GC方式启动JVM

## G1回收器：区域化分代式

### 为什么还需要G1

**既然我们已经有了前面几个强大的 GC ，为什么还要发布 Garbage First（G1）GC？**

1. 原因就在于应用程序所应对的业务越来越庞大、复杂，用户越来越多，没有GC就不能保证应用程序正常进行，而经常造成STW的GC又跟不上实际的需求，所以才会不断地尝试对GC进行优化。
2. G1（Garbage-First）垃圾回收器是在Java7 update4之后引入的一个新的垃圾回收器，是当今收集器技术发展的最前沿成果之一。
3. 与此同时，**为了适应现在不断扩大的内存和不断增加的处理器数量**，进一步降低暂停时间（pause time），同时兼顾良好的吞吐量。
4. 官方给G1设定的目标是在延迟可控的情况下获得尽可能高的吞吐量，所以才担当起“全功能收集器”的重任与期望。

### 为什么名字叫Garbage First呢？

1. 因为G1是一个并行回收器，它把堆内存分割为很多不相关的区域（Region）（物理上不连续的）。使用不同的Region来表示Eden、幸存者0区，幸存者1区，老年代等。
2. G1 GC有计划地避免在整个Java堆中进行全区域的垃圾收集。G1跟踪各个Region里面的垃圾堆积的价值大小（回收所获得的空间大小以及回收所需时间的经验值），在后台维护一个优先列表，**每次根据允许的收集时间，优先回收价值最大的Region。**
3. 由于这种方式的侧重点在于回收垃圾最大量的区间（Region），所以我们给G1一个名字：垃圾优先（Garbage First）。
4. G1（Garbage-First）是一款面向服务端应用的垃圾收集器，主要针对配备多核CPU及大容量内存的机器，以极高概率满足GC停顿时间的同时，还兼具高吞吐量的性能特征。
5. 在JDK1.7版本正式启用，移除了Experimental的标识，**是JDK9以后的默认垃圾回收器**，取代了CMS回收器以及Parallel+Parallel Old组合。被Oracle官方称为**“全功能的垃圾收集器”**。
6. 与此同时，CMS已经在JDK9中被标记为废弃（deprecated）。**G1在JDK8中还不是默认的垃圾回收器**，需要使用-XX:+UseG1GC来启用。

### G1回收器的优势

与其他GC收集器相比，G1使用了全新的分区算法，其特点如下所示：

1. 并行与并发兼备
   - 并行性：G1在回收期间，可以有多个GC线程同时工作，有效利用多核计算能力。此时用户线程STW
   - 并发性：G1拥有与应用程序交替执行的能力，部分工作可以和应用程序同时执行，因此，一般来说，不会在整个回收阶段发生完全阻塞应用程序的情况
2. 分代收集
   - 从分代上看，G1依然属于分代型垃圾回收器，它会区分年轻代和老年代，年轻代依然有Eden区和Survivor区。但从堆的结构上看，它不要求整个Eden区、年轻代或者老年代都是连续的，也不再坚持固定大小和固定数量。
   - 将堆空间分为若干个区域（Region），这些区域中包含了逻辑上的年轻代和老年代。
   - 和之前的各类回收器不同，它同时兼顾年轻代和老年代。对比其他回收器，或者工作在年轻代，或者工作在老年代；

G1的分代，已经不是下面这样的了

![img](深入理解Java虚拟机/0016.png)

G1的分区是这样的一个区域

![img](深入理解Java虚拟机/0017-16516627404649.png)

**空间整合**

1. CMS：“标记-清除”算法、内存碎片、若干次GC后进行一次碎片整理
2. G1将内存划分为一个个的region。内存的回收是以region作为基本单位的。**Region之间是复制算法，但整体上实际可看作是标记-压缩（Mark-Compact）算法**，两种算法都可以避免内存碎片。这种特性有利于程序长时间运行，分配大对象时不会因为无法找到连续内存空间而提前触发下一次GC。尤其是当Java堆非常大的时候，G1的优势更加明显。

### 可预测的停顿时间模型

**可预测的停顿时间模型（即：软实时soft real-time）**

这是G1相对于CMS的另一大优势，G1除了追求低停顿外，还能建立可预测的停顿时间模型，能让使用者明确指定在一个长度为M毫秒的时间片段内，消耗在垃圾收集上的时间不得超过N毫秒。

1. 由于分区的原因，G1可以只选取部分区域进行内存回收，这样缩小了回收的范围，因此对于全局停顿情况的发生也能得到较好的控制。
2. G1跟踪各个Region里面的垃圾堆积的价值大小（回收所获得的空间大小以及回收所需时间的经验值），在后台维护一个优先列表，**每次根据允许的收集时间，优先回收价值最大的Region**。保证了G1收集器在有限的时间内可以获取尽可能高的收集效率。
3. 相比于CMS GC，G1未必能做到CMS在最好情况下的延时停顿，但是最差情况要好很多。

### G1回收器的缺点

1. 相较于CMS，G1还不具备全方位、压倒性优势。比如在用户程序运行过程中，G1无论是为了垃圾收集产生的内存占用（Footprint）还是程序运行时的额外执行负载（overload）都要比CMS要高。
2. 从经验上来说，在小内存应用上CMS的表现大概率会优于G1，而G1在大内存应用上则发挥其优势。平衡点在6-8GB之间。

### G1参数设置

- -XX:+UseG1GC：手动指定使用G1垃圾收集器执行内存回收任务
- -XX:G1HeapRegionSize：设置每个Region的大小。值是2的幂，范围是1MB到32MB之间，目标是根据最小的Java堆大小划分出约2048个区域。默认是堆内存的1/2000。
- -XX:MaxGCPauseMillis：设置期望达到的最大GC停顿时间指标，JVM会尽力实现，但不保证达到。默认值是200ms
- -XX:+ParallelGCThread：设置STW工作线程数的值。最多设置为8
- -XX:ConcGCThreads：设置并发标记的线程数。将n设置为并行垃圾回收线程数（ParallelGcThreads）的1/4左右。
- -XX:InitiatingHeapOccupancyPercent：设置触发并发GC周期的Java堆占用率阈值。超过此值，就触发GC。默认值是45。

### G1收集器的常见操作步骤

G1的设计原则就是简化JVM性能调优，开发人员只需要简单的三步即可完成调优：

1. 第一步：开启G1垃圾收集器
2. 第二步：设置堆的最大内存
3. 第三步：设置最大的停顿时间

G1中提供了三种垃圾回收模式：YoungGC、Mixed GC和Full GC，在不同的条件下被触发。

### G1的适用场景

1. 面向服务端应用，针对具有大内存、多处理器的机器。（在普通大小的堆里表现并不惊喜）
2. 最主要的应用是需要低GC延迟，并具有大堆的应用程序提供解决方案；
3. 如：在堆大小约6GB或更大时，可预测的暂停时间可以低于0.5秒；（G1通过每次只清理一部分而不是全部的Region的增量式清理来保证每次GC停顿时间不会过长）。
4. 用来替换掉JDK1.5中的CMS收集器；在下面的情况时，使用G1可能比CMS好：
   - 超过50%的Java堆被活动数据占用；
   - 对象分配频率或年代提升频率变化很大；
   - GC停顿时间过长（长于0.5至1秒）
5. HotSpot垃圾收集器里，除了G1以外，其他的垃圾收集器均使用内置的JVM线程执行GC的多线程操作，而G1 GC可以采用应用线程承担后台运行的GC工作，即当JVM的GC线程处理速度慢时，系统会调用应用程序线程帮助加速垃圾回收过程。

### 分区Region

**分区 Region：化整为零**

1. 使用G1收集器时，它将整个Java堆划分成约2048个大小相同的独立Region块，每个Region块大小根据堆空间的实际大小而定，整体被控制在1MB到32MB之间，且为2的N次幂，即1MB，2MB，4MB，8MB，16MB，32MB。可以通过

2. XX:G1HeapRegionSize设定。**所有的Region大小相同，且在JVM生命周期内不会被改变。**

3. 虽然还保留有新生代和老年代的概念，但新生代和老年代不再是物理隔离的了，它们都是一部分Region（不需要连续）的集合。通过Region的动态分配方式实现逻辑上的连续。

4. 一个Region有可能属于Eden，Survivor或者Old/Tenured内存区域。但是一个Region只可能属于一个角色。图中的E表示该Region属于Eden内存区域，S表示属于Survivor内存区域，O表示属于Old内存区域。图中空白的表示未使用的内存空间。

5. G1垃圾收集器还增加了一种新的内存区域，叫做Humongous内存区域，如图中的H块。主要用于存储大对象，如果超过0.5个Region，就放到H。

   > 纠错：尚硅谷视频里这里写的是超过1.5个region。根据[官方文档](https://www.oracle.com/technetwork/tutorials/tutorials-1876574.html): **The G1 Garbage Collector Step by Step**
   >
   > As shown regions can be allocated into Eden, survivor, and old generation regions. In addition, there is a fourth type of object known as Humongous regions. These regions are designed to hold objects that are 50% the size of a standard region or larger. They are stored as a set of contiguous regions. Finally the last type of regions would be the unused areas of the heap.
   >
   > 翻译：
   >
   > 如图所示，可以将区域分配到Eden，幸存者和旧时代区域。 此外，还有第四种类型的物体被称为巨大区域。 这些区域旨在容纳标准区域大小的50％或更大的对象。 它们存储为一组连续区域。 最后，最后一种区域类型是堆的未使用区域。

   ![img](深入理解Java虚拟机/0018.png)

**设置 H 的原因**

对于堆中的大对象，默认直接会被分配到老年代，但是如果**它是一个短期存在的大对象**就会对垃圾收集器造成负面影响。为了解决这个问题，G1划分了一个Humongous区，它用来专门存放大对象。如**果一个H区装不下一个大对象，那么G1会寻找连续的H区来存储**。为了能找到连续的H区，有时候不得不启动Full GC。G1的大多数行为都把H区作为老年代的一部分来看待。

**Regio的细节**

![img](深入理解Java虚拟机/0019-165166274046510.png)

1. 每个Region都是通过指针碰撞来分配空间
2. G1为每一个Region设 计了两个名为TAMS（Top at Mark Start）的指针，把Region中的一部分空间划分出来用于并发回收过程中的新对象分配，并发回收时新分配的对象地址都必须要在这两个指针位置以上。
3. TLAB还是用来保证并发性

### G1垃圾回收流程

G1 GC的垃圾回收过程主要包括如下三个环节：

- 年轻代GC（Young GC）
- 老年代并发标记过程（Concurrent Marking）
- 混合回收（Mixed GC）
- （如果需要，单线程、独占式、高强度的Full GC还是继续存在的。它针对GC的评估失败提供了一种失败保护机制，即强力回收。）

![img](深入理解Java虚拟机/0020.png)

顺时针，Young GC --> Young GC+Concurrent Marking --> Mixed GC顺序，进行垃圾回收

**回收流程**

1. 应用程序分配内存，当年轻代的Eden区用尽时开始年轻代回收过程；G1的年轻代收集阶段是一个并行的独占式收集器。在年轻代回收期，G1 GC暂停所有应用程序线程，启动多线程执行年轻代回收。然后从年轻代区间移动存活对象到Survivor区间或者老年区间，也有可能是两个区间都会涉及。
2. 当堆内存使用达到一定值（默认45%）时，开始老年代并发标记过程。
3. 标记完成马上开始混合回收过程。对于一个混合回收期，G1 GC从老年区间移动存活对象到空闲区间，这些空闲区间也就成为了老年代的一部分。和年轻代不同，老年代的G1回收器和其他GC不同，**G1的老年代回收器不需要整个老年代被回收，一次只需要扫描/回收一小部分老年代的Region就可以了**。同时，这个老年代Region是和年轻代一起被回收的。
4. 举个例子：一个Web服务器，Java进程最大堆内存为4G，每分钟响应1500个请求，每45秒钟会新分配大约2G的内存。G1会每45秒钟进行一次年轻代回收，每31个小时整个堆的使用率会达到45%，会开始老年代并发标记过程，标记完成后开始四到五次的混合回收。

### Remembered Set（记忆集）

1. 一个对象被不同区域引用的问题
2. 一个Region不可能是孤立的，一个Region中的对象可能被其他任意Region中对象引用，判断对象存活时，是否需要扫描整个Java堆才能保证准确？
3. 在其他的分代收集器，也存在这样的问题（而G1更突出，因为G1主要针对大堆）
4. 回收新生代也不得不同时扫描老年代？这样的话会降低Minor GC的效率

**解决方法：**

1. 无论G1还是其他分代收集器，JVM都是使用Remembered Set来避免全堆扫描；
2. 每个Region都有一个对应的Remembered Set
3. 每次Reference类型数据写操作时，都会产生一个Write Barrier暂时中断操作；
4. 然后检查将要写入的引用指向的对象是否和该Reference类型数据在不同的Region（其他收集器：检查老年代对象是否引用了新生代对象）；
5. 如果不同，通过CardTable把相关引用信息记录到引用指向对象的所在Region对应的Remembered Set中；
6. 当进行垃圾收集时，在GC根节点的枚举范围加入Remembered Set；就可以保证不进行全局扫描，也不会有遗漏。

![img](深入理解Java虚拟机/0021.png)

1. 在回收 Region 时，为了不进行全堆的扫描，引入了 Remembered Set
2. Remembered Set 记录了当前 Region 中的对象被哪个对象引用了
3. 这样在进行 Region 复制时，就不要扫描整个堆，只需要去 Remembered Set 里面找到引用了当前 Region 的对象
4. Region 复制完毕后，修改 Remembered Set 中对象的引用即可

### G1回收过程一：年轻代GC

1. JVM启动时，G1先准备好Eden区，程序在运行过程中不断创建对象到Eden区，当Eden空间耗尽时，G1会启动一次年轻代垃圾回收过程。
2. 年轻代回收只回收Eden区和Survivor区
3. YGC时，首先G1停止应用程序的执行（Stop-The-World），G1创建回收集（Collection Set），回收集是指需要被回收的内存分段的集合，年轻代回收过程的回收集包含年轻代Eden区和Survivor区所有的内存分段。

![img](深入理解Java虚拟机/0022.png)

图的大致意思就是：

1、回收完E和S区，剩余存活的对象会复制到新的S区

2、S区达到一定的阈值可以晋升为O区

**细致过程：**

**然后开始如下回收过程：**

1. 第一阶段，扫描根

   根是指GC Roots，根引用连同RSet记录的外部引用作为扫描存活对象的入口。

2. 第二阶段，更新RSet

3. 第三阶段，处理RSet

   识别被老年代对象指向的Eden中的对象，这些被指向的Eden中的对象被认为是存活的对象。

4. 第四阶段，复制对象。

   - 此阶段，对象树被遍历，Eden区内存段中存活的对象会被复制到Survivor区中空的内存分段，Survivor区内存段中存活的对象
   - 如果年龄未达阈值，年龄会加1，达到阀值会被会被复制到Old区中空的内存分段。
   - 如果Survivor空间不够，Eden空间的部分数据会直接晋升到老年代空间。

5. 第五阶段，处理引用

   处理Soft，Weak，Phantom，Final，JNI Weak 等引用。最终Eden空间的数据为空，GC停止工作，而目标内存中的对象都是连续存储的，没有碎片，所以复制过程可以达到内存整理的效果，减少碎片。

**备注：**

1. 对于应用程序的引用赋值语句 oldObject.field（这个是老年代）=object（这个是新生代），JVM会在之前和之后执行特殊的操作以在dirty card queue中入队一个保存了对象引用信息的card。在年轻代回收的时候，G1会对Dirty Card Queue中所有的card进行处理，以更新RSet，保证RSet实时准确的反映引用关系。
2. 那为什么不在引用赋值语句处直接更新RSet呢？这是为了性能的需要，RSet的处理需要线程同步，开销会很大，使用队列性能会好很多。

### G1回收过程二：并发标记过程

1. 初始标记阶段：标记从根节点直接可达的对象。这个阶段是STW的，并且会触发一次年轻代GC。正是由于该阶段时STW的，所以我们只扫描根节点可达的对象，以节省时间。
2. 根区域扫描（Root Region Scanning）：G1 GC扫描Survivor区直接可达的老年代区域对象，并标记被引用的对象。这一过程必须在Young GC之前完成，因为Young GC会使用复制算法对Survivor区进行GC。
3. 并发标记（Concurrent Marking）：
   1. 在整个堆中进行并发标记（和应用程序并发执行），此过程可能被Young GC中断。
   2. **在并发标记阶段，若发现区域对象中的所有对象都是垃圾，那这个区域会被立即回收。**
   3. 同时，并发标记过程中，会计算每个区域的对象活性（区域中存活对象的比例）。
4. 再次标记（Remark）：由于应用程序持续进行，需要修正上一次的标记结果。是STW的。G1中采用了比CMS更快的原始快照算法：Snapshot-At-The-Beginning（SATB）。
5. 独占清理（cleanup，STW）：计算各个区域的存活对象和GC回收比例，并进行排序，识别可以混合回收的区域。为下阶段做铺垫。是STW的。这个阶段并不会实际上去做垃圾的收集
6. 并发清理阶段：识别并清理完全空闲的区域。

### G1回收过程三：混合回收过程

当越来越多的对象晋升到老年代Old Region时，为了避免堆内存被耗尽，虚拟机会触发一个混合的垃圾收集器，即Mixed GC，该算法并不是一个Old GC，除了回收整个Young Region，还会回收一部分的Old Region。这里需要注意：是一部分老年代，而不是全部老年代。可以选择哪些Old Region进行收集，从而可以对垃圾回收的耗时时间进行控制。也要注意的是Mixed GC并不是Full GC。

![img](深入理解Java虚拟机/0023.png)

**混合回收的细节**

1. 并发标记结束以后，老年代中百分百为垃圾的内存分段被回收了，部分为垃圾的内存分段被计算了出来。默认情况下，这些老年代的内存分段会分8次（可以通过-XX:G1MixedGCCountTarget设置）被回收。【意思就是一个Region会被分为8个内存段】
2. 混合回收的回收集（Collection Set）包括八分之一的老年代内存分段，Eden区内存分段，Survivor区内存分段。混合回收的算法和年轻代回收的算法完全一样，只是回收集多了老年代的内存分段。具体过程请参考上面的年轻代回收过程。
3. 由于老年代中的内存分段默认分8次回收，G1会优先回收垃圾多的内存分段。垃圾占内存分段比例越高的，越会被先回收。并且有一个阈值会决定内存分段是否被回收。XX:G1MixedGCLiveThresholdPercent，默认为65%，意思是垃圾占内存分段比例要达到65%才会被回收。如果垃圾占比太低，意味着存活的对象占比高，在复制的时候会花费更多的时间。
4. 混合回收并不一定要进行8次。有一个阈值-XX:G1HeapWastePercent，默认值为10%，意思是允许整个堆内存中有10%的空间被浪费，意味着如果发现可以回收的垃圾占堆内存的比例低于10%，则不再进行混合回收。因为GC会花费很多的时间但是回收到的内存却很少。

### G1回收可选的过程四：Full GC

1. G1的初衷就是要避免Full GC的出现。但是如果上述方式不能正常工作，G1会停止应用程序的执行（Stop-The-World），使用**单线程**的内存回收算法进行垃圾回收，性能会非常差，应用程序停顿时间会很长。
2. 要避免Full GC的发生，一旦发生Full GC，需要对JVM参数进行调整。什么时候会发生Ful1GC呢？比如堆内存太小，当G1在复制存活对象的时候没有空的内存分段可用，则会回退到Full GC，这种情况可以通过增大内存解决。

导致G1 Full GC的原因可能有两个：

1. EVacuation的时候没有足够的to-space来存放晋升的对象；
2. 并发处理过程完成之前空间耗尽。

### G1补充

从Oracle官方透露出来的信息可获知，回收阶段（Evacuation）其实本也有想过设计成与用户程序一起并发执行，但这件事情做起来比较复杂，考虑到G1只是回一部分Region，停顿时间是用户可控制的，所以并不迫切去实现，**而选择把这个特性放到了G1之后出现的低延迟垃圾收集器（即ZGC）中。**另外，还考虑到G1不是仅仅面向低延迟，停顿用户线程能够最大幅度提高垃圾收集效率，为了保证吞吐量所以才选择了完全暂停用户线程的实现方案。

**G1 回收器的优化建议**

1. 年轻代大小
   - 避免使用-Xmn或-XX:NewRatio等相关选项显式设置年轻代大小，因为固定年轻代的大小会覆盖可预测的暂停时间目标。我们让G1自己去调整
2. 暂停时间目标不要太过严苛
   - G1 GC的吞吐量目标是90%的应用程序时间和10%的垃圾回收时间
   - 评估G1 GC的吞吐量时，暂停时间目标不要太严苛。目标太过严苛表示你愿意承受更多的垃圾回收开销，而这些会直接影响到吞吐量。

## 垃圾回收器总结

### 7种垃圾回收器的比较

截止JDK1.8，一共有7款不同的垃圾收集器。每一款的垃圾收集器都有不同的特点，在具体使用的时候，需要根据具体的情况选用不同的垃圾收集器。

![img](深入理解Java虚拟机/0034.jpg) ![img](深入理解Java虚拟机/0024.png)

### 怎么选择垃圾回收器

Java垃圾收集器的配置对于JVM优化来说是一个很重要的选择，选择合适的垃圾收集器可以让JVM的性能有一个很大的提升。怎么选择垃圾收集器？

1. 优先调整堆的大小让JVM自适应完成。
2. 如果内存小于100M，使用串行收集器
3. 如果是单核、单机程序，并且没有停顿时间的要求，串行收集器
4. 如果是多CPU、需要高吞吐量、允许停顿时间超过1秒，选择并行或者JVM自己选择
5. 如果是多CPU、追求低停顿时间，需快速响应（比如延迟不能超过1秒，如互联网应用），使用并发收集器
6. 官方推荐G1，性能高。现在互联网的项目，基本都是使用G1。

最后需要明确一个观点：

1. 没有最好的收集器，更没有万能的收集算法
2. 调优永远是针对特定场景、特定需求，不存在一劳永逸的收集器

**面试**

1. 对于垃圾收集，面试官可以循序渐进从理论、实践各种角度深入，也未必是要求面试者什么都懂。但如果你懂得原理，一定会成为面试中的加分项。
2. 这里较通用、基础性的部分如下：
   - 垃圾收集的算法有哪些？如何判断一个对象是否可以回收？
   - 垃圾收集器工作的基本流程。
3. 另外，大家需要多关注垃圾回收器这一章的各种常用的参数

## GC日志分析

### 常用参数配置

> **GC 日志参数设置**

**通过阅读GC日志，我们可以了解Java虚拟机内存分配与回收策略。**

内存分配与垃圾回收的参数列表

1. -XX:+PrintGC ：输出GC日志。类似：-verbose:gc
2. -XX:+PrintGCDetails ：输出GC的详细日志
3. -XX:+PrintGCTimestamps ：输出GC的时间戳（以基准时间的形式）
4. -XX:+PrintGCDatestamps ：输出GC的时间戳（以日期的形式，如2013-05-04T21: 53: 59.234 +0800）
5. -XX:+PrintHeapAtGC ：在进行GC的前后打印出堆的信息
6. -Xloggc:…/logs/gc.log ：日志文件的输出路径

> **verbose:gc**

1、JVM 参数

```
-verbose:gc
```

2、这个只会显示总的GC堆的变化，如下：

![img](深入理解Java虚拟机/0025.png)

3、参数解析

![img](深入理解Java虚拟机/0026.png)

> **PrintGCDetails**

1、JVM 参数

```
-XX:+PrintGCDetails
```

2、输入信息如下

![img](深入理解Java虚拟机/0027.png)

3、参数解析

![img](深入理解Java虚拟机/0028.png)

> **PrintGCTimestamps 和 PrintGCDatestamps**

1、JVM 参数

```
-XX:+PrintGCTimeStamps -XX:+PrintGCDateStamps
```

2、输出信息如下

![img](深入理解Java虚拟机/0029-165166274046511.png)

3、说明：日志带上了日期和时间

### GC日志补充说明

1. “[GC"和”[Full GC"说明了这次垃圾收集的停顿类型，如果有"Full"则说明GC发生了"Stop The World"
2. 使用Serial收集器在新生代的名字是Default New Generation，因此显示的是"[DefNew"
3. 使用ParNew收集器在新生代的名字会变成"[ParNew"，意思是"Parallel New Generation"
4. 使用Parallel scavenge收集器在新生代的名字是”[PSYoungGen"
5. 老年代的收集和新生代道理一样，名字也是收集器决定的
6. 使用G1收集器的话，会显示为"garbage-first heap"
7. Allocation Failure表明本次引起GC的原因是因为在年轻代中没有足够的空间能够存储新的数据了。
8. [ PSYoungGen: 5986K->696K(8704K) ] 5986K->704K (9216K)
   - 中括号内：GC回收前年轻代大小，回收后大小，（年轻代总大小）
   - 括号外：GC回收前年轻代和老年代大小，回收后大小，（年轻代和老年代总大小）
9. user代表用户态回收耗时，sys内核态回收耗时，real实际耗时。由于多核线程切换的原因，时间总和可能会超过real时间

#### Young GC

![img](深入理解Java虚拟机/0030-165166274046612.png)

#### Full GC

![img](深入理解Java虚拟机/0031.png)

#### 举例

```java
/**
 * 在jdk7 和 jdk8中分别执行
 * -verbose:gc -Xms20M -Xmx20M -Xmn10M -XX:+PrintGCDetails -XX:SurvivorRatio=8 -XX:+UseSerialGC
 */
public class GCLogTest1 {
    private static final int _1MB = 1024 * 1024;

    public static void testAllocation() {
        byte[] allocation1, allocation2, allocation3, allocation4;
        allocation1 = new byte[2 * _1MB];
        allocation2 = new byte[2 * _1MB];
        allocation3 = new byte[2 * _1MB];
        allocation4 = new byte[4 * _1MB];
    }

    public static void main(String[] agrs) {
        testAllocation();
    }
}Copy to clipboardErrorCopied
```

**JDK7 中的情况**

1、首先我们会将3个2M的数组存放到Eden区，然后后面4M的数组来了后，将无法存储，因为Eden区只剩下2M的剩余空间了，那么将会进行一次Young GC操作，将原来Eden区的内容，存放到Survivor区，但是Survivor区也存放不下，那么就会直接晋级存入Old 区

![img](深入理解Java虚拟机/0032-165166274046613.png)

2、然后我们将4M对象存入到Eden区中

![img](深入理解Java虚拟机/0033-165166274046614.png)

老年代图画的有问题，free应该是4M

**JDK8 中的情况**

```java
com.atguigu.java.GCLogTest1
[GC (Allocation Failure) [DefNew: 6322K->668K(9216K), 0.0034812 secs] 6322K->4764K(19456K), 0.0035169 secs] [Times: user=0.00 sys=0.00, real=0.00 secs] 
Heap
 def new generation   total 9216K, used 7050K [0x00000000fec00000, 0x00000000ff600000, 0x00000000ff600000)
  eden space 8192K,  77% used [0x00000000fec00000, 0x00000000ff23b668, 0x00000000ff400000)
  from space 1024K,  65% used [0x00000000ff500000, 0x00000000ff5a71d8, 0x00000000ff600000)
  to   space 1024K,   0% used [0x00000000ff400000, 0x00000000ff400000, 0x00000000ff500000)
 tenured generation   total 10240K, used 4096K [0x00000000ff600000, 0x0000000100000000, 0x0000000100000000)
   the space 10240K,  40% used [0x00000000ff600000, 0x00000000ffa00020, 0x00000000ffa00200, 0x0000000100000000)
 Metaspace       used 3469K, capacity 4496K, committed 4864K, reserved 1056768K
  class space    used 381K, capacity 388K, committed 512K, reserved 1048576K

Process finished with exit code 0
Copy to clipboardErrorCopied
```

![img](深入理解Java虚拟机/0035.jpg)

与 JDK7 不同的是，JDK8 直接判定 4M 的数组为大对象，直接怼到老年区去了

### 常用日志分析工具

**保存日志文件**

**JVM参数**：`-XLoggc:./logs/gc.log`， ./ 表示当前目录，在 IDEA中程序运行的当前目录是工程的根目录，而不是模块的根目录

可以用一些工具去分析这些GC日志，常用的日志分析工具有：

GCViewer、GCEasy、GCHisto、GCLogViewer、Hpjmeter、garbagecat等

**推荐：GCeasy**

在线分析网址：gceasy.io

![img](深入理解Java虚拟机/0036.jpg) ![img](深入理解Java虚拟机/0037.png) ![img](深入理解Java虚拟机/0038-165166274046615.png)

## 垃圾回收器的新发展

### 垃圾回收器的发展过程

1. GC仍然处于飞速发展之中，目前的默认选项G1 GC在不断的进行改进，很多我们原来认为的缺点，例如串行的Full GC、Card Table扫描的低效等，都已经被大幅改进，例如，JDK10以后，Fu11GC已经是并行运行，在很多场景下，其表现还略优于ParallelGC的并行Ful1GC实现。
2. 即使是SerialGC，虽然比较古老，但是简单的设计和实现未必就是过时的，它本身的开销，不管是GC相关数据结构的开销，还是线程的开销，都是非常小的，所以随着云计算的兴起，在serverless等新的应用场景下，Serial Gc找到了新的舞台。
3. 比较不幸的是CMSGC，因为其算法的理论缺陷等原因，虽然现在还有非常大的用户群体，但在JDK9中已经被标记为废弃，并在JDK14版本中移除
4. 现在G1回收器已成为默认回收器好几年了。我们还看到了引入了两个新的收集器：ZGC（JDK11出现）和Shenandoah（Open JDK12），其特点：主打低停顿时间

### Shenandoah GC

**Open JDK12的Shenandoash GC：低停顿时间的GC（实验性）**

1. Shenandoah无疑是众多GC中最孤独的一个。是第一款不由Oracle公司团队领导开发的Hotspot垃圾收集器。不可避免的受到官方的排挤。比如号称openJDK和OracleJDK没有区别的Oracle公司仍拒绝在OracleJDK12中支持Shenandoah。
2. Shenandoah垃圾回收器最初由RedHat进行的一项垃圾收集器研究项目Pauseless GC的实现，旨在针对JVM上的内存回收实现低停顿的需求。在2014年贡献给OpenJDK。
3. Red Hat研发Shenandoah团队对外宣称，Shenandoah垃圾回收器的暂停时间与堆大小无关，这意味着无论将堆设置为200MB还是200GB，99.9%的目标都可以把垃圾收集的停顿时间限制在十毫秒以内。不过实际使用性能将取决于实际工作堆的大小和工作负载。

这是RedHat在2016年发表的论文数据，测试内容是使用ES对200GB的维基百科数据进行索引。从结果看：

1. 停顿时间比其他几款收集器确实有了质的飞跃，但也未实现最大停顿时间控制在十毫秒以内的目标。
2. 而吞吐量方面出现了明显的下降，总运行时间是所有测试收集器里最长的。

![img](深入理解Java虚拟机/0039-165166274046616.png)

总结

1. Shenandoah GC的弱项：高运行负担下的吞吐量下降。
2. Shenandoah GC的强项：低延迟时间。

### 令人震惊、革命性的ZGC

1. 官方文档：https://docs.oracle.com/en/java/javase/12/gctuning/
2. ZGC与Shenandoah目标高度相似，在尽可能对吞吐量影响不大的前提下，实现在任意堆内存大小下都可以把垃圾收集的停颇时间限制在十毫秒以内的低延迟。
3. 《深入理解Java虚拟机》一书中这样定义ZGC：ZGC收集器是一款基于Region内存布局的，（暂时）不设分代的，使用了读屏障、染色指针和内存多重映射等技术来实现可并发的标记-压缩算法的，以低延迟为首要目标的一款垃圾收集器。
4. ZGC的工作过程可以分为4个阶段：并发标记 - 并发预备重分配 - 并发重分配 - 并发重映射 等。
5. ZGC几乎在所有地方并发执行的，除了初始标记的是STW的。所以停顿时间几乎就耗费在初始标记上，这部分的实际时间是非常少的。

**吞吐量**

![img](深入理解Java虚拟机/0040-165166274046617.png)

max-JOPS：以低延迟为首要前提下的数据

critical-JOPS：不考虑低延迟下的数据

**低延迟**

![img](深入理解Java虚拟机/0041.png)

在ZGC的强项停顿时间测试上，它毫不留情的将Parallel、G1拉开了两个数量级的差距。无论平均停顿、95%停顿、998停顿、99. 98停顿，还是最大停顿时间，ZGC都能毫不费劲控制在10毫秒以内。

虽然ZGC还在试验状态，没有完成所有特性，但此时性能已经相当亮眼，用“令人震惊、革命性”来形容，不为过。未来将在服务端、大内存、低延迟应用的首选垃圾收集器。

![img](深入理解Java虚拟机/0042.png)

1. JDK14之前，ZGC仅Linux才支持。

2. 尽管许多使用ZGC的用户都使用类Linux的环境，但在Windows和macOS上，人们也需要ZGC进行开发部署和测试。许多桌面应用也可以从ZGC中受益。因此，ZGC特性被移植到了Windows和macOS上。

3. 现在mac或Windows上也能使用ZGC了，示例如下：

   -XX:+UnlockExperimentalVMOptions-XX：+UseZGC

### 面向大堆的AliGC

AliGC是阿里巴巴JVM团队基于G1算法，面向大堆（LargeHeap）应用场景。指定场景下的对比：

![image-20230406233123757](深入理解Java虚拟机/image-20230406233123757.png)
