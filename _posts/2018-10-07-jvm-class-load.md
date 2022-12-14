---
title: JVM 类加载机制
author: zhangzangqian
date: 2018-10-07 21:00:00 +0800
categories: [技术]
tags: [Java, 类加载]
math: true
mermaid: true
---

## 概述

虚拟机把描述类的数据从 Class 文件加载到内存，并对数据进行校验、转换解析和初始化，最终形成可以被虚拟机直接使用的 Java 类型，这就是虚拟机的类加载机制。

**Java 语言里面，类的加载、连接和初始化过程都是在程序运行期间完成的**，这种策略虽然会令类加载时稍微增加一些性能开销，但是会为 Java 应用程序提供了高度的灵活性，Java 里天生可以动态扩展的语言特性（我理解就是多态）就是依赖运行期动态加载和动态连接这个特点实现的。

**类被加载到虚拟机内存中开始，到卸载出内存为止，它的整个生命周期包括：加载（Loading）、验证（Verification）、准备（Preparation）、解析（Resolution）、初始化（Initialization）、使用（Useing）和卸载（Unloading）7个阶段。其中验证、准备、解析3个部分统称为连接。**

![类的生命周期](/assets/img/class_lifecycle.jpeg)

> 约定：第一，在实际情况中，每个 Class 文件都可能代表着 Java 语言中的一个类活着接口，后文直接对“类”的描述都包括了类和接口的可能性，而对于类和接口需要分开描述的场景，会特别指明；第二，所提到的“Class 文件”并非特指某个存在于具体磁盘中的文件，这里所说的“Class 文件”应当是一串二进制的字节流，无论以何种形式存在都可以。
{: .prompt-tip}

## 类加载时机

加载、验证、准备、初始化和卸载这5个阶段的顺序是确定的，类的加载过程必须按照这种顺序按部就班的开始，而解析阶段则不一定：它在某些情况下可以再初始化阶段再开始，这是为了支持 Java 语言的运行时绑定（也称为动态绑定和晚期绑定）。这些阶段通常是互相交叉地混合进行的，通常会在一个阶段执行的过程中调用、激活另外一个阶段。

Java 虚拟机规范中并没有进行强制约束何时进行类加载过程的第一个阶段：加载，而是交给虚拟机的具体实现自有把握，但是对于初始化阶段，虚拟机规范则严格要求有且只有5中情况必须立即对类进行“初始化”（而加载、验证、准备自然需要在此之前开始）：

- **遇到 new、getstatic、putstatic 或 invokestatic 这4条字节码指令时，如果类没有进行过初始化，则需要先触发其初始化。声场这4条指令的具体场景就是：**
    
    - **使用 new 关键字实例化对象**
    - **读取一个类的静态字段（被 final 修饰、已在编译器把结果放入常量池的静态字段除外）**
    - **设置一个类的静态字段**
    - **调用一个类的静态方法**

- **使用 java.lang.reflect 包的方法对类进行反射调用的时候，如果类没有进行过初始化，需要先触发其初始化。**
- **当初始化一个类的时候，如果发现其父类还没有进行过初始化，则需要先触发其父类初始化；但是一个接口在初始化时，并不要求其父类全部都完成了初始化，只有在真正使用到父接口的时候（如引用接口中定义的常量）才会初始化。**
- **当虚拟机启动时，用户需要指定一个要执行的主类（包含 main() 方法的那个类），虚拟机会先初始化这个主类。**
- **当使用 JDK 1.7 的动态语言支持时，如果一个 java.lang.invoke.MethodHandle 实例最后的解析结果 REF_getStatic、REF_putStatic、REF_invokeStatic 的方法的句柄，并且这个方法句柄所对应的类没有进行过初始化，则需要先触发其初始化。**

以上5种场景中的行为称为对一个类的主动引用。除此之外，所有引用类的方式都不会触发初始化，称为被动引用。以下三段代码清单说明何为被动引用。

- 通过子类引用父类的静态字段, 不会导致子类初始化。

    ```java
    package io.boom.classloading;

    public class SuperClass {

        static {
            System.out.println("SuperClass init!");
        }
        
        public static int value = 123;

    }

    public class SubClass extends SuperClass {

        static {
            System.out.println("SubClass init!");
        }

    }

    public class NotInitialization {

        public static void main(String[] args) {
            System.out.println(SubClass.value);
        }

    }
    ```

    以上代码只会输出 “SuperClass init!”，而不会输出 “SubClass init!”。**只有直接定义这个字段的类才会被初始化**，因此通过其子类来引用父类中的静态字段，只会导致父类初始化而不会触发子类的初始化。

- 通过数组定义引用类，不会触发此类的初始化。

    ```java
    package io.boom.classloading;

    public class NotInitialization {
        
        public static void main(String[] args) {
            SuperClass[] sca = new SuperClass[10];
        }

    }
    ```
    以上代码运行后没有输出“SuperClass init!”，说明没有触发 io.boom.classloading.SuperClass 类的初始化阶段。但是这段代码里面会触发另外一个类名为 “[Lio.boom.classloading.SuperClass” 的类初始化阶段，它是一个有虚拟机自动生成的、直接继承于 java.lang.Object 的子类，创建动作由字节码指令 newarray 触发。

- 常量在编译阶段存入调用类的常量池中，本质上并没有直接引用到定义常量的类，因此不会触发定义常量类的初始化。

    ```java
    package io.boom.classloading;

    public class ConstantClass {

        static {
            System.out.println("ConstantClass init!");
        }

        public static final String HELLOWORLD = "hello world";

    }

    public class NotInitialization {

        public static void main(String[] args) {
            System.out.println(ConstantClass.HELLOWORLD);
        }

    }
    ```
    以上代码运行后也不会输出“ConstantClass init!”，因为虽然在 Java 源码中引用了 ConstantClass 类中的常量 HELLOWORLD，但其实在编译阶段通过常量传播优化，已经将此常量的值“hello world”存储到了 NotInitialization 类的常量池中，以后 NotInitialization 对常量 ConstantClass.HELLOWORLD 的引用实际都被转化为 NotInitialization 类对自身常量池的引用了。

## 类加载的过程

接下来我们具体学习下加载、验证、准备、解析和初始化这5个阶段所执行的具体动作。

### 加载

“加载”是“类加载”（Class Loading）过程的一个阶段，在加载阶段，虚拟机会完成以下三件事：

- **通过一个类的全限定名来获取定义此类的二进制字节流。**
- **将这个二进制字节流所代表的静态存储结构转化为方法区的运行时数据结构。**
- **在内存中生成一个代表这个类的 java.lang.Class 对象，作为方法区这个类的各种数据的访问入口。**

虚拟机规范的这3点要求其实并不算具体实现，因此虚拟机与具体应用的灵活度都是相当大的。例如“通过一个类的全限定名来获取定义此类的二进制字节流”这条，他没有指明二进制字节流要从一个 Class 文件中获取，准确的说是根本没有指明要从哪里获取、怎么获取。虚拟机设计团队在加载阶段搭建了一个相当开放的、广阔的“舞台”，Java 发展历程中，充满创造力的开发人员则在这个“舞台”上玩出了各种花样，需要举足轻重的 Java 技术都建立在这一基础之上，例如：

- 从 zip 包中读取，这很常见，最终成为日后 jar、ear、war 格式的基础。
- 从网络中获取，这种场景最经典的应用就是 Applet。
- 运行中计算生成，这种场景使用的最多的就是动态代理技术，在 java.lang.reflect.Proxy 中，就是用了 ProxyGenerator.generateProxyClass 来为特定接口生成形式为“*$Proxy”的代理类的二进制字节流。
- 由其他文件生成，典型场景是 JSP 应用，即由 JSP 文件生成对应的 Class 类。

等等。。。。

一个非数组类的加载阶段（准确的说，是加载阶段中获取类的二进制字节流的动作）是开发人员可控性最强的，因为**加载阶段既可以使用系统提供的引导类加载器完成，也可以有用户自定义的类加载器去完成，开发人员可以通过定义自己的类加载器去控制字节流的获取方式（即重写一个类加载器的 loadClass() 方法）。**

对于数组而言，数组本身不是通过类加载器创建，它是由 Java 虚拟机直接创建的。但数组类与类加载器仍然有很密切关系，因为数组类的元素类型（Element Type，指的是数组去掉所有维度的类型）最终是要靠类加载器去创建，一个数组类（下面简称为 C）的创建过程就遵循以下规则：

- 如果数组累的组件类型（Component Type，指的是数组去掉一个维度的类型）是引用类型，那就递归采用本文中定义的加载过程去加载这个组件，数组 C 将在加载该组件类型的类加载器的类名称空间上被标识（这点很重要，后面会提到一个类必须与类加载器一起确定唯一性）。
- 如果数组的组件类型不是引用类型（例如 int[] 数组），Java 虚拟机将会把数组 C 标记为与引导类加载器关联。
- 数组类的可见性与他的组件类型的可见性一致，如果组件类型不是引用类型那数组的可见性将默认为 public。

加载阶段完成后，虚拟机外部的二进制字节流就按照虚拟机所需的格式存储在方法区之中，然后在内存中实例化一个 java.lang.Class 类的对象（并没有明确规定是在 Java 堆中，对于 HotSpot 虚拟机而言，Class 对象比较特殊，它虽然是对象，但是存放在方法区里面），将这个对象作为称为方位方法区中的这些类型数据的外部接口。

### 验证

**验证是连接的第一步，这一阶段的目的是为了确保 Class 文件的字节流中包含的信息符合当前虚拟机的要求，并且不会危害虚拟机自身的安全。**加载阶段和验证阶段是交叉进行的，加载阶段尚未完成，验证阶段可能已经开始。

前面已经说过，Class 文件并不一定要求用 Java 源码编译而来，可以使用任何途径产生，甚至包括用十六进制编辑器直接编写来产生 Class 文件。在字节码语言层面上，上述 Java 代码无法做到的事情都是可以实现的，至少语义上是可以表达出来的。虚拟机如果不检查输入流的字节流，对其完全信任的话，很可能会因为载入了有害的字节流而导致系统崩溃，所以验证是虚拟机对自身保护的一项重要工作。

**验证阶段大致上会完成下面4个阶段的检验动作：文件格式验证、元数据验证、字节码验证、符号引用验证。**

#### 文件格式验证

第一阶段要验证字节流是否符合 Class 文件格式的规范，并且能被当前版本的虚拟机处理。可能会包括但不限于以下这些点：

- 是否以魔法值 0xCAFEBABE 开头。
- 主、次版本号是否在当前虚拟机处理的范围内。
- 常量池的常量中是否有不被支持的常量类型（检查常量 tag 标志）。

#### 元数据验证

主要目的是对类的元数据信息进行语义校验，保证不存在不符合 Java 语言规范的元数据信息。验证内容包括但不限于一下几点：

- 这个类是否有父类（除了 java.lang.Object 之外，所有的类都应当有父类）。
- 这个类的父类是否集成了不允许被继承的类（被 final 修饰的类）。
- 如果这个类不是抽象类，是否实现了其他父类或接口之中要求实现的所有方法。

#### 字节码验证

**主要目的是通过数据流和控制流分析，确定程序语义是否是合法的、符合逻辑的。这个阶段将对类的方法体进行分析，保证被校验的类的方法在运行时不会做出危害虚拟机安全的时间**，例如：

- 保证任意时刻操作数栈的数据类型与指令代码序列都能配合工作，例如不会出现类似这样的情况：在操作栈放置了一个 int 类型的数据，使用时却按 long 类型类加载入本地变量表中。
- 保证跳转指令不会跳转到本地方法以外的字节码指令上。
- 保证方法体中的类型转换是有效的，如何可以把一个子类对象赋值给父类型数据类型，这是安全的，但是把父类对象赋值给子类数据类型，甚至把对象赋值给与它毫无继承关系、完全不相干的一个数据类型，则是危险和不合法的。

#### 符号引用验证

**符号引用验证可以看做是对类自身以外（常量池中的各种符号引用）的信息进行匹配性校验**，同时需要校验以下但不限于的内容：

- 符合引用中通过字符串描述的全限定名是否能找到对应的类。
- 在指定类中是否存在符合方法的字段描述符以及简单名称所描述的方法和字段。
- 符号引用中的类、字段、方法的访问性（private、protected、public、default）是否可被当前类方法。

### 准备

准备阶段是正式为类变量（被 static 修饰的变量）分配并设置类变量初始值的阶段，这些变量所使用的内存都将在方法区中分配。实例变量将会在对象实例化的时候随着对象一起分配在 Java 堆中。这里所说的初始值“通常情况下”是数据类型的默认值，假设一个类变量的定义为 `public static int value = 123`，那变量在准备阶段过后的初始值是0而不是123，因为这时候尚未开始执行任何 Java 方法，而把 value 赋值为 123 的 putstatic 指令是程序被编译后，存放于类构造器 &lt;clint&gt;() 方法之中，所以把 value 赋值为 123 的动作将在初始化阶段才会执行。下表列出了 Java 中所有基本数据类型的默认值。

|数据类型|	默认值|
|:---|:---|
|int	    |0|
|long	    |0L|
|short	    |(short)0|
|char	    |‘\u0000’|
|byte	    |(byte)0|
|boolean	|false|
|float	    |0.0f|
|double	    |0.0d|
|reference	|null|

**如果类字段在字段属性表中存在 ConstantValue 属性，那在准备阶段 value 就会被初始化为 ConstantValue 属性所指的值**，假设上面类变量 value 的定义为： `public static final int value = 123`，编译时 javac 会将 value 生成 ConstantValue 属性，在准备阶段虚拟机就会根据 ConstantValue 的设置将 value 赋值为 123。

### 解析

**解析阶段是虚拟机将常量池内的符号引用[^fhyy]替换为直接引用[^zjyy]的过程**。符号引用就是 Class 文件的常量池内的表类型的复合数据类型，例如：CONSTANT_Class_info、CONSTANT_Fidldref_info、CONSTANT_Method_info 等类型的常量出现。

**虚拟机规范之中并未规定解析阶段发生的具体时间，只要求了在执行 anewarray、checkcast、getfield、getstatic、instanceof、invokedynamic、invokeinterface、invokespcial、invokestatic、invokevirtual、ldc、ldc_w、multianewarray、new、putfiled 和 putstatic 这16个操作符号引用的字节码指令之前，先对他们所使用的符号引用进行解析。**所以虚拟机实现可以根据需要来判断到底是在类被加载器加载时就对常量池中的符号引用进行解析，还是等到一个符号引用要被使用前才去解析它。

### 初始化

**到了初始化阶段，才真正开始执行类中定义的 Java 程序代码（或者说是字节码）。在准备阶段，变量已经赋过一次系统要求的初始值，而在初始化阶段，则根据程序员通过程序定制的主观计划去初始化变量和其他资源。或者可以从另外一个角度来表达：初始化阶段是执行类构造器 &lt;clinit&gt;() 方法的过程。**

- &lt;clinit&gt;方法是有编译器自动收集类中的所有类变量的赋值动作和静态语句块（static{}）中的语句合并产生的，**编译器收集的顺序是由语句在源文件中出现的顺序决定的，静态语句块中只能访问到定义在静态语句块之前的变量，定义在它之后的变量，在前面的语句块可以赋值，但是不能访问。**

    ```java
    /**
    * 非法向前引用变量示例
    */
    public class Test {
        static {
            i = 0;                  // 给变量赋值可以正常编译通过
            System.out.println(i);  // 这句编译器会提示 “非法向前引用”
        }
        static int i = 1;
    }
    ```

- **&lt;clinit&gt;() 方法与类的构造器函数（或者说实例构造器 &lt;init&gt;() 方法）不同，他不需要显示的调用父类构造器，虚拟机保证在子类的 &lt;clinit&gt;() 方法执行之前，父类的 &lt;clinit&gt;() 方法已经执行完毕。因此虚拟机中第一个被执行的 &lt;clinit&gt;() 方法的类肯定是 java.lang.Object。** 由于父类的 &lt;clinit&gt;() 方法先执行，这也就意味着父类中定义的静态语句块要优先于之类的变量赋值操作。

```java
/**
* &lt;clinit&gt;() 方法执行顺序示例，B 的值将会是 2 而不是 1.
*/
static class Parent {
    public static int A = 1;
    static {
        A = 2;
    }
}

static class Sub extends Parent {
    public static int B = A;
}

public static void main(String[] args) {
    System.out.println(Sub.B);
}
```

- &lt;clinit&gt;() 方法对于类或者接口来说并不是必须的，如果一个类中没有静态语句块，也没有变量的赋值操作，那么编译器可以不为这个类生成 &lt;clinit&gt;() 方法。

- 接口中不能使用静态语句块，但仍然有变量初始化的赋值操作，因此接口与类一样都会生成 &lt;clinit&gt;() 方法。但是接口与类不同的是，**执行接口的 &lt;clinit&gt;() 方法不需要先执行父接口的 &lt;clinit&gt;() 方法。只有当父接口中定义的变量使用时，父接口才会初始化。另外，接口的实现类初始化时也一样不会执行接口的 &lt;clinit&gt;() 方法。**

- 虚拟机会保证一个类的 &lt;clinit&gt;() 方法在多线程环境中被正确的加锁、同步，如果多个线程同时去初始化一个类，那么只有一个线程去执行这个类的 &lt;clinit&gt;() 方法，其他线程都需要阻塞等待，知道活动线程执行 &lt;clinit&gt;() 方法完毕。如果一个类中的 &lt;clinit&gt;() 方法中有耗时很长的操作，就可能造成多个进行阻塞，在实际应用中这中阻塞往往是很隐蔽的。

    ```java
    static class DeadLoopClass {
        static {
            // 如果不加上 if 语句，编译器将提示  “Initializer does not complete normally” 并拒绝编译。
            if (true) {
                System.out.println(Thread.currentThread() + "init DeadLoopClass");
                while (true) {

                }
            }
        }
    }

    public static void main(String[] args) {
        Runnable script = new Runnable() {
            public void run() {
                System.out.println("Thread.currentThread()" + "start");
                DeadLoopClass dlc = new DeadLoopClass();
                System.out.println("Thread.currentThread()" + "run over");
            }
        }

        Thread t1 = new Thread(script);
        Thread t2 = new Thread(script);

        t1.start();
        t2.start();

    }
    ```

    运行结果如下，即另一条线程在死循环以模拟长时间操作，另外一条线程在阻塞等待。

    ```console
    Thread[Thread-0,5,main]start

    Thread[Thread-1,5,main]start

    Thread[Thread-0,5,main]init DeadLoopClass
    ```

[^fhyy]: 符号引用（Symbolic References）以一组符号来描述所引用的目标，符号引用可以是任何形式的字面量，只要使用时能无歧义的定位到目标即可。符号引用与虚拟机的实现无关，引用的目标不一定已经加载到内存中。各种虚拟机实现的内存布局可以各不相同，但它们能接受的符号引用必须都是一致的！因为符号引用的字面量形式明确定义在 Java 虚拟机规范的 Class 文件格式中。

[^zjyy]: 直接引用（Direct References）可以是直接指向目标的指针、相对偏移量或是一个能间接定位到目标的句柄。直接引用是和虚拟机实现的内存布局相关的，同一个符号引用在不同的虚拟机上翻译出来的直接引用一般不会相同。如果有了直接引用，那引用的目标必须已经在内存中存在了。