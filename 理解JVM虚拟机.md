# 理解JVM虚拟机

## 1. Java是怎么实现从源代码到机器码的

![](https://img2018.cnblogs.com/blog/595137/201812/595137-20181212194443807-900872025.png)

字节码与机器码：

机器码就是说计算机能读懂的代码,简单点说就是给计算机执行的二进制代码.
字节码,是JAVA语言专有的,它是让JVM来执行的二进制代码
虽然都是二进制代码,但是由于执行它的环境不一样,所以它们存在一些指令集上的区别

### 1.1 前端编译器（源代码到字节码）

通过我们熟悉的javac实现，把Java语言规范转换为字节码语言规范

- 第一阶段：词法，语法分析。编译器分析我们写的Java语言，像英文中分析主谓宾一样，分析我们要表达什么
- 第二阶段：填充符号表。由于类之间是有互相引用的，但是在编译阶段我们不知道引用类的具体地址，所以我们用符号去替代。等到类加载阶段，JVM会将符号替换成具体的内存地址。
- 第三阶段：注解处理，根据注解的作用将其还原成具体的指令集。
- 第四阶段：分析与字节码生成。根据上面三阶段的分析，我们的字节码就可以生成了，最终输出为class文件

### 1.2 JIT编译器（字节码到机器码）

![](https://img2018.cnblogs.com/blog/595137/201812/595137-20181212194503259-888638210.png)

由上图可知，如果我们得到了字节码之后，想要运行程序，其实有两种选择

1. 通过Java interpreter（Java解释器）直接执行字节码

   **解释器**会对字节码采用**逐行解释**方式执行，将字节码文件中的内容转化为对应平台的机器码

   这种方式启动速度快，但是运行速度慢

2. 通过JIT编译器将字节码转化为本地机器码

   **编译器**是直接将用的比较多的代码编译成本地代码，生成缓存（`cached`）放在`方法区`中，这样可以减少解释器的中间消耗，获得更高的执行效率

   这种方式启动速度慢，但是运行速度快

**为什么会出现这种启动速度和运行速度的差异呢？**

​	因为解释器不需要像编译器一样需要将字节码转换为机器码，自然就少去了部分时间，可以直接运行，启动速度快。但是编译器完成第一次编译之后，编译完成的机器码就会被保留下来，下次可以直接使用。众所周知，机器码的运行效率肯定是高于Java解释器的，所以这种方式的运行速度更快。

​	在实际应用中，我们为了运行速度及效率，通常采用两者结合的方式进行Java代码的编译执行。

### 1.3 JIT的两种编译器及其两种编译模式

​	在HotSpot虚拟机内置了两个即时编译器，分别为Client Compiler和Server Compiler。这两种编译器分别对应着两种编译模式，我们分别称之为C1编译模式，C2编译模式。

​	*注意：现在许多人习惯上将 Client Compiler 称为 C1 编译器，将 Server Compiler 称为 C2 编译器，但在 Oracle 官方文档中将其描述为 compiler mode（编译模式）。所以说 C1 编译器、C2 编译器只是我们自己的习惯性称呼，并不是官方的说法。这点需要特别注意。*

**C1编译模式和C2编译模式的区别在哪呢？**

​	C1编译模式会在编译过程中进行简单可靠的优化，如果有必要会加入性能监控的功能。

​								-----------C1编译模式优化保守，速度较快

​	C2编译模式会启用一些编译耗时较长的优化，甚至会根据性能监控信息进行一些不可靠激进优化。

​								------------C2编译模式会进行激进优化，会根据性能监控做针对性优化，编译质量相对较好，但是速度慢

​	**现代编译器（Java7开始），是混合使用C1与C2的，在启动时使用C1编译模式，以提供更好的启动性能，一旦程序预热完成，C2编译模式就会接管编译，以提供更好的优化和性能。Java8之后，这种模式变成默认模式**

> 引用原文：
>
> ​	Compilation Modes
>  Inside Java HotSpot VM, there are actually two separate JIT compiler modes, which are known as C1 and C2. C1 is used for applications where quick startup and rock-solid optimization are required; GUI applications are often good candidates for this compiler. C2, on the other hand, was originally intended for long-running, predominantly server-side applications. Prior to some of the later Java SE 7 releases, these two modes were available using the -client and -server switches, respectively.
>
> ​	The two compiler modes use different techniques for JIT compilation, and they can output very different machine code for the same Java method. Modern Java applications, however, can usually make use of both compilation modes. To take advantage of this fact, starting with some of the later Java SE 7 releases, a new feature called tiered compilation became available. This feature uses the C1 compiler mode at the start to provide better startup performance. Once the application is properly warmed up, the C2 compiler mode takes over to provide more-aggressive optimizations and, usually, better performance. With the arrival of Java SE 8, tiered compilation is now the default behavior.

### 1.4 三种运行模式

- 混合模式（默认模式）

![](https://img-blog.csdnimg.cn/20200523161213259.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L0FscGhy,size_16,color_FFFFFF,t_70)

如上图可以看出，整个java应用程序的执行过程如下：

1、源代码经javac编译成字节码，class文件

2、程序字节码经过JIT环境变量进行判断，是否属于“热点代码”（多次调用的方法，或循环等）

3、如是，走JIT编译为具体硬件处理器（如sparc、intel）机器码

4、如否，则直接由解释器解释执行

5、操作系统及类库调用

6、硬件

以上实际上是JVM的“混合模式”对java程序的执行方式。

- 解释模式

	不经过jit直接由解释器解释执行所有字节码，执行效率不高。

- 编译模式

	​	不加筛选的将全部代码进行编译机器码不论其执行频率是否有编译价值，在程序响应时间的限制下，编译器没法采用编译耗时较高的优化技术（因为JIT的编译是首次运行或启动的时候进行的！），所以，在纯编译执行模式下的java程序执行效率跟C/C++也是具有较大差距的。

## 2. Java虚拟机内存结构

​	Java虚拟机的内存结构可以分为共有和私有两部分。公有部分指的是所有线程都共享的部分，私有部分指的是每个线程的私有数据

​	![](https://upload-images.jianshu.io/upload_images/10006199-a4108d8fb7810a71.jpeg?imageMogr2/auto-orient/strip|imageView2/2/w/1200/format/webp)

### 2.1 公有部分：Java堆，方法区，常量池

​	**Java堆**

​	这块区域专门用于Java实例对象的内存分配，几乎所有的实例对象都会在这里进行内存的分配。之所以我们说“几乎”，是因为有些时候小对象会直接在栈上进行分配，就是所谓的“栈上分配”，后面会介绍。

​	**方法区**

​	这块区域指的是储存**Java类**字节码数据的一块区域，它存储了每一个类的结构信息，例如运行时的**常量池、字段、方法数据和构造方法**等。

​	*可以看到常量池其实是存放在方法区中的，但在《Java 虚拟机规范》将常量池和方法区放在同一个等级上，这点我们知晓即可。*



**接下来是递归学习补充知识点时间！**

> 补充：什么是字段和属性？类成员(字段)，通常是在类中定义的类成员变量，例如：
>
> public class A{
>
>   private String s = "123";
>
> }
>
> 我们可以说A类中有一个成员变量叫做s，A类有一个字段s 。
>
> 属性只局限于类中方法的声明，并不与类中其他成员相关，属于JavaBean（什么是JavaBean见下面）范畴。例如：
>
> void setA(String s){}
>
> String getA(){}
>
> 当一个类中拥有这样一对方法时，我们可以说，这个类中拥有一个可读写的a属性(注意是小写a)。如果去掉了set的方法，则是可读属性，反之亦然。

> 什么是Javabean？
>
> JavaBean 是特殊的 Java 类，使用 Java 语言书写，并且遵守 JavaBean API 规范。
>
> 接下来给出的是 JavaBean 与其它 Java 类相比而言独一无二的特征：
>
> - 提供一个默认的无参构造函数。
> - 需要被序列化并且实现了 Serializable 接口。
> - 可能有一系列可读写属性。
> - 可能有一系列的 getter 或 setter 方法。

> 补充：什么是常量池
>
> 常量池分三种：Class文件常量池，运行时常量池，字符串常量池
>
> 1. Class文件常量池：
>
>    Class 文件常量池指的是编译生成的 class 字节码文件，其结构中有一项是常量池（Constant Pool Table），用于存放编译期生成的各种字面量（字符串字面量和声明为final的基本数据类型常量值）和符号引用，这部分内容将在类加载后进入方法区的运行时常量池中存放。刚刚了解过JVM编译过程，你应该对这部分很熟悉。
>
> 2. 运行时常量池：
>
>    运行时常量池是方法区的一部分，是一块内存区域。Class 文件常量池将在类加载后进入方法区的运行时常量池中存放。
>
>    一个类加载到 JVM 中后对应一个运行时常量池，运行时常量池相对于 Class 文件常量池来说具备动态性，Class 文件常量只是一个静态存储结构，里面的引用都是符号引用。而运行时常量池可以在运行期间将符号引用解析为直接引用。
>
>    运行时常量池就是用来索引和查找字段和方法名称和描述符的。给定任意一个方法或字段的索引，通过这个索引最终可得到该方法或字段所属的类型信息和名称及描述符信息。
>
>    这部分看不懂没关系，后面的类加载机制会讲到这部分
>
> 3. 字符串常量池：
>
>    Java7之后，在堆内存中，储存的是字符串对象的引用，字符串实例是在堆中



### 2.2 深入了解Java堆

​	Java堆根据对象存活的时间不同，将其分为了**年轻代**、**老年代**两个区域，年轻代还进一步被划分为了Eden区，From Survivor 0、To Surviver 1区。

​	*在Java8之前，堆中还有永久代，但是在Java8中，把存放元数据的永久内存从堆内存移到了本地内存中，叫做元空间（Metaspace）*

​	*再补充一点：永久代在Java7之前就是方法区，后来在1.7中为了“去永久代”，将原本放在方法区中的静态变量，字符串常量池等移到了堆内存。后来在1.8又移到了本地内存*

​	![](https://img2020.cnblogs.com/blog/1828752/202004/1828752-20200403203500814-314521655.png)

​	![](https://img2018.cnblogs.com/blog/595137/201901/595137-20190103103329413-247778313.png)

 	当有对象需要进行内存分配时，它将永远被优先分配到年轻代的Eden区，当Eden区内存空间不足时，JVM会启动垃圾回收，此时Eden区中没有被引用的对象的内存就会被回收，而一些存活时间较长的对象则会进入老年代。

​	**那如何判断这个对象要不要进入老年代呢？**

​	在 JVM 中有一个名为 -XX:MaxTenuringThreshold 的参数专门用来设置晋升到老年代所需要经历的 GC 次数，即在年轻代的对象经过了指定次数的 GC 后，将在下次 GC 时进入老年代。

​	**Java堆为什么要分年轻代和老年代呢？**

​	根据我们的经验，虚拟机中的对象必然有存活时间长的，也有存活时间短的。如果我们不将其进行区分，那么因为存活短的对象有很多，导致我们要频繁的进行GC。而每次GC我们都要堆所有内存都进行扫描，但事实上，有一部分存活时间长的对象是没必要每次都进行扫描的，所以我们为了提升GC效率，就进行分区了。

​	**Eden，From，To三个区域的内存大小分配比例是怎么样的？为什么这么设计？**

​	Eden：From：To = 8：1：1

​	这是 IBM 公司根据大量统计得出的结果。根据 IBM 公司对对象存活时间的统计，他们发现 80% 的对象存活时间都很短。于是他们将 Eden 区设置为年轻代的 80%，这样可以减少内存空间的浪费，提高内存空间利用率。

​	**为什么要有From、To这两个区域？**



###  2.3 私有部分：PC寄存器、Java虚拟机栈、本地方法栈

​	**PC 寄存器** 顾名思义 Program Counter 寄存器，指的是保存线程当前正在执行的方法。如果这个方法不是 native 方法，那么 PC 寄存器就保存 Java 虚拟机正在执行的字节码指令地址。如果是 native 方法，那么 PC 寄存器保存的值是 undefined。任意时刻，一条 Java 虚拟机线程只会执行一个方法的代码，而这个被线程执行的方法称为该线程的当前方法，其地址被存在 PC 寄存器中。

​	**Java 虚拟机栈** 这个栈与线程同时创建，用来存储栈帧，即存储局部变量与一些过程结果的地方。栈帧存储的数据包括：局部变量表、操作数栈。

​	**本地方法栈** 当 Java 虚拟机使用其他语言（例如 C 语言）来实现指令集解释器时，也会使用到本地方法栈。如果 Java 虚拟机不支持 natvie 方法，并且自己也不依赖传统栈的话，可以无需支持本地方法栈。

## 3. Java类加载机制

### 3.1  类加载的七个阶段

JVM 虚拟机执行 class 字节码的过程可以分为七个阶段：**加载、验证、准备、解析、初始化、使用、卸载。**

**加载：**

加载阶段是类加载过程的第一个阶段。在这个阶段，JVM 的主要目的是将字节码从各个位置（网络、磁盘等）转化为二进制字节流加载到内存中，接着会为这个类在 JVM 的方法区创建一个对应的 Class 对象，这个 Class 对象就是这个类各种数据的访问入口。

**验证：**

主要是进行两部分校验：

- JVM规范校验：JVM 会对字节流进行文件格式校验，判断其是否符合 JVM 规范，是否能被当前版本的虚拟机处理。例如：文件是否是以 `0x cafe bene`开头，主次版本号是否在当前虚拟机处理范围之内等。
- 代码逻辑校验：比如传参类型是否错误，是否使用了未创建的类等等

**⭐准备：**

当完成校验之后，JVM便会开始为类变量分配内存并进行初始化。

这里需要注意以下两点：

- **内存分配的对象。**Java 中的变量有「类变量」和「类成员变量」两种类型，「类变量」指的是被 static 修饰的变量，而其他所有类型的变量都属于「类成员变量」。在准备阶段，JVM 只会为「类变量」分配内存，而不会为「类成员变量」分配内存。「类成员变量」的内存分配需要等到初始化阶段才开始。

例如下面的代码在准备阶段，只会为 factor 属性分配内存，而不会为 website 属性分配内存。

```java
public static int factor = 3;
public String website = "www.baidu.com";
```

- **初始化的类型。**在准备阶段，JVM 会为类变量分配内存，并为其初始化。但是这里的初始化指的是为变量赋予 Java 语言中该数据类型的**零值**，而不是用户代码里初始化的值。

例如下面的代码在准备阶段之后，sector 的值将是 0，而不是 3。

```
public static int sector = 3;
```

但如果一个**变量是常量（被 static final 修饰）**的话，那么在准备阶段，属性便会被赋予用户希望的值。例如下面的代码在准备阶段之后，number 的值将是 3，而不是 0。

```java
public static final int number = 3;
```

**解析**：

还记得我们之前说过的，在编译时，并不知道类引用的具体地址，所以我们用符号来替代类引用关系吗？

在解析阶段，主要任务正是将其在常量池中的符号引用替换成其在内存中的直接引用。

**⭐初始化**

到了初始化阶段，用户定义的 Java 程序代码才真正开始执行。在这个阶段，JVM 会根据语句执行顺序对类对象进行初始化，一般来说当 JVM 遇到下面 5 种情况的时候会触发初始化：

- 遇到 new、getstatic、putstatic、invokestatic 这四条字节码指令时，如果类没有进行过初始化，则需要先触发其初始化。生成这4条指令的最常见的Java代码场景是：使用new关键字实例化对象的时候、读取或设置一个类的静态字段（被final修饰、已在编译器把结果放入常量池的静态字段除外）的时候，以及调用一个类的静态方法的时候。
- 使用 java.lang.reflect 包的方法对类进行反射调用的时候，如果类没有进行过初始化，则需要先触发其初始化。
- 当初始化一个类的时候，如果发现其父类还没有进行过初始化，则需要先触发其父类的初始化。
- 当虚拟机启动时，用户需要指定一个要执行的主类（包含main()方法的那个类），虚拟机会先初始化这个主类。
- 当使用 JDK1.7 动态语言支持时，如果一个 java.lang.invoke.MethodHandle实例最后的解析结果 REF_getstatic,REF_putstatic,REF_invokeStatic 的方法句柄，并且这个方法句柄所对应的类没有进行初始化，则需要先出触发其初始化。

看到上面几个条件你可能会晕了，但是不要紧，不需要背，知道一下就好，后面用到的时候回到找一下就可以了。

**使用**

JVM从入口方法开始执行程序代码

**卸载**

代码执行完毕，JVM开始销毁创建的Class对象，最后负责运行的JVM也退出内存。



### 3.2 深入了解类初始化

下面我们将结合实际例子，去深入了解类的初始化过程

首先给出结论，Java代码编译成字节码之后，是没有构造方法之类的概念的，只有**类初始化方法**和**对象初始化方法**

- 类初始化方法。编译器会按照其出现顺序，收集static变量的赋值语句、静态代码块，最终组成类初始化方法。**类初始化方法一般在类初始化的时候执行。**
- 对象初始化方法。编译器会按照其出现顺序，收集成员变量的赋值语句、普通代码块，最后收集构造函数的代码，最终组成对象初始化方法。**对象初始化方法一般在实例化类对象的时候执行。**

给个实例来理解这两个概念

```java
public class Book {
    public static void main(String[] args)
    {
        System.out.println("Hello ShuYi.");
    }

    Book()
    {
        System.out.println("书的构造方法");
        System.out.println("price=" + price +",amount=" + amount);
    }

    {
        System.out.println("书的普通代码块");
    }

    int price = 110;

    static
    {
        System.out.println("书的静态代码块");
    }

    static int amount = 112;
}
```

```java
书的静态代码块
Hello ShuYi.
```

上面这个例子中

类初始化方法：

```java
	static
   	{
        System.out.println("书的静态代码块");
    }

    static int amount = 112;
```

对象初始化方法：

```java
 	Book()
    {
        System.out.println("书的构造方法");
        System.out.println("price=" + price +",amount=" + amount);
    }

    {
        System.out.println("书的普通代码块");
    }

    int price = 110;
```

这下就很容易看透这个问题了，首先由于我们之前在“初始化”部分提到的“当虚拟机启动时，用户需要指定一个要执行的主类（包含main()方法的那个类），虚拟机会先初始化这个主类。”，我们要先初始化main所在的类，所以执行类的初始化方法，然后由于book并没有被实例化，所以不用执行对象初始化方法。

我们下面看一个稍微难一点的例子：

```java
public class Book {
    public static void main(String[] args)
    {
        staticFunction();
    }
	int price = 110;
    static int amount = 112;
   
    static Book book = new Book();
	
    static
    {
        System.out.println("书的静态代码块");
    }

    {
        System.out.println("书的普通代码块");
    }

    Book()
    {
        System.out.println("书的构造方法");
        System.out.println("price=" + price +",amount=" + amount);
    }

    public static void staticFunction(){
        System.out.println("书的静态方法");
    }
```

这个最终的输出是什么呢？我们来一步一步分析

首先，像上一题一样，我们要先初始化main函数所在的类，所以执行Book类的类初始化方法

```java
 	static Book book = new Book();

    static
    {
        System.out.println("书的静态代码块");
    }
	 public static void staticFunction(){
        System.out.println("书的静态方法");
    }
	static int amount = 112;
```

执行第一步我们发现，他竟要先把Book实例化了！那我们就去执行Book类的对象初始化方法

```java
 	int price = 110;
	{
        System.out.println("书的普通代码块");
    }
	Book()
    {
        System.out.println("书的构造方法");
        System.out.println("price=" + price +",amount=" + amount);
    }
```

执行过程中我们发现，我们需要知道price和amount的大小，price的大小很明显是110，那amount呢？我们还没执行到static int amount = 112呢，amount的大小是多少？

在**“准备”**那部分我们学到了，JVM会先给static变量分配内存，如果不是final static则会先赋**零值**，那很明显这里amount=0。

随后我们再去执行剩下的类初始化方法和main函数中调用的staticFunction()

所以会输出

```java
书的普通代码块
书的构造方法
price=110,amount=0
书的静态代码块
书的静态方法   
```



接下来我们再看一个调用父类初始化的例子：

```java
class Grandpa
{
    static
    {
        System.out.println("爷爷在静态代码块");
    }
}    
class Father extends Grandpa
{
    static
    {
        System.out.println("爸爸在静态代码块");
    }

    public static int factor = 25;

    public Father()
    {
        System.out.println("我是爸爸~");
    }
}
class Son extends Father
{
    static 
    {
        System.out.println("儿子在静态代码块");
    }

    public Son()
    {
        System.out.println("我是儿子~");
    }
}
public class InitializationDemo
{
    public static void main(String[] args)
    {
        System.out.println("爸爸的岁数:" + Son.factor);	//入口
    }
}
```

我们来一步步分析：

首先程序进入main方法，发现需要调用Son.factor，但是factor并不在son这个类中，于是我们去到父类找，找到了factor，触发了父类的初始化

这里要提到一个知识点：**对于静态字段，只有直接定义这个字段的类才会被初始化（执行静态代码块）。**因此通过其子类来引用父类中定义的静态字段，只会触发父类的初始化而不会触发子类的初始化。

但根据我们上面说到的“当初始化一个类的时候，如果发现其父类还没有进行过初始化，则需要先触发其父类的初始化”，我们需要先初始化Grandpa类，再初始化Father类。

最后所有的父类都初始化好了，Son类才能调用父类的静态变量，从而输出“爸爸的岁数：25”

最终输出

```java
爷爷在静态代码块
爸爸在静态代码块
爸爸的岁数:25
```



### 3.3总结

从上面几个例子可以看出，分析一个类的执行顺序大概可以按照如下步骤：

- **确定类变量的初始值。**在类加载的准备阶段，JVM 会为类变量初始化零值，这时候类变量会有一个初始的零值。如果是被 final 修饰的类变量，则直接会被初始成用户想要的值。
- **初始化入口方法。**当进入类加载的初始化阶段后，JVM 会寻找整个 main 方法入口，从而初始化 main 方法所在的整个类。当需要对一个类进行初始化时，会首先初始化类构造器（），之后初始化对象构造器（）。
- **初始化类构造器。**JVM 会按顺序收集类变量的赋值语句、静态代码块，最终组成类构造器由 JVM 执行。
- **初始化对象构造器。**JVM 会按照收集成员变量的赋值语句、普通代码块，最后收集构造方法，将它们组成对象构造器，最终由 JVM 执行。

如果在初始化 main 方法所在类的时候遇到了其他类的初始化，那么就先加载对应的类，加载完成之后返回。如此反复循环，最终返回 main 方法所在类。

## 4. 垃圾回收机制

### 4.1 谁是垃圾？

> 当然我是辣鸡了！ 

开个玩笑

面对垃圾回收问题，我们首先要知道谁是垃圾，在Java中如果一个对象不可能再被引用，那么这个对象就是垃圾。

在C++中我们采用只能指针来防止内存泄漏，其中shared_ptr就是采用【引用计数法】来进行垃圾判断的，但是熟悉C++的同学一定知道，由于循环引用问题，C++中的shared_ptr会出现死锁的问题，在C++中我们是采用weak_ptr来解决这个问题的，那么在Java中呢？

### 4.2 可达性分析算法

在Java中我们采用可达性分析算法来判断谁是垃圾

可达性分析算法（Reachability Analysis）的基本思路是，通过一些被称为引用链（GC Roots）的对象作为起点，从这些节点开始向下搜索，搜索走过的路径被称为（Reference Chain)，当一个对象到 GC Roots 没有任何引用链相连时（即从 GC Roots 节点到该节点不可达），则证明该对象是不可用的。

![](https://picb.zhimg.com/v2-43ddfa6f70d6c3fde381454105af6472_b.jpg)

那问题又来了，谁是GC ROOT呢？GC Root 就是一组活跃引用的集合

这里我们分为四种

- 所有当前被加载的 Java 类的引用
- Java 类的引用类型静态变量
- Java类的运行时常量池里的引用类型常量
- 本地方法栈中 JNI（即一般说的 Native 方法）引用的对象

我们来依次看看这四种GC ROOT

1. 所有当前被加载的 Java 类的引用

   ```java
   public class StackLocalParameter {
   	public StackLocalParameter(String name){}
   }
   
   public static void testGC(){
   	StackLocalParameter s = new StackLocalParameter("localParameter");
   	s = null;
   }
   ```

   此时的 s，即为 GC Root，当s置空时，localParameter 对象也断掉了与 GC Root 的引用链，将被回收。

2. Java 类的引用类型静态变量

   ```java
   public class MethodAreaStaicProperties {
   	public static MethodAreaStaicProperties m;
   	public MethodAreaStaicProperties(String name){}
   }
   
   public static void testGC(){
   	MethodAreaStaicProperties s = new MethodAreaStaicProperties("properties");
   	s.m = new MethodAreaStaicProperties("parameter"); 
       //用实例调用静态变量时被允许的，但是不推荐这么用，还是应该用类名去调。在这里是为了凸显s为null后，m未被回收
   	s = null;
   }
   ```

   s 为 GC Root，s 置为 null，经过 GC 后，s 所指向的 properties 对象由于无法与 GC Root 建立关系被回收。

   而 m 作为类的静态属性，也属于 GC Root，parameter 对象依然与 GC root 建立着连接，所以此时 parameter 对象并不会被回收。

3. Java类的运行时常量池里的引用类型常量

   ```java
   public class MethodAreaStaicProperties {
   	public static final MethodAreaStaicProperties m = MethodAreaStaicProperties("final");
   	public MethodAreaStaicProperties(String name){}
   }
   
   public static void testGC(){
   	MethodAreaStaicProperties s = new MethodAreaStaicProperties("staticProperties");
   	s = null;
   }
   ```

   m 即为方法区中的常量引用，也为 GC Root，s 置为 null 后，final 对象也不会因没有与 GC Root 建立联系而被回收。

4. 本地方法栈中 JNI（即一般说的 Native 方法）引用的对象

   这个一般不常用



**但这里有个需要注意的点：即使在可达性分析算法中不可达的对象,也并非是“非回收不可”的,这时候它们暂时处于“等待”阶段,要真正宣告一个对象回收,至少要经历两次标记过程:如果对象在进行可达性分析后发现没有与GC Roots相连接的引用链,那它将会被第一次标记并且进行一次筛选,筛选的条件是此对象是否有必要执行finalize()方法。当对象没有覆盖finalize()方法,或者finalize()方法已经被虚拟机调用，这个时候虚拟机将进行回收**

> 如果这个对象被判定为有必要执行finalize()方法,那么这个对象将会放置在一个叫做F-Queue的队列之中,并在稍后由一个由虚拟机自动建立的、低优先级的Finalizer线程去执行它。这里所谓的“执行”是指虚拟机会触发这个方法,但并不承诺会等待它运行结束,这样做的原因是,如果一个对象在finalize()方法中执行缓慢,或者发生了死循环(更极端的情况),将很可能会导致F-Queue队列中其他对象永久处于等待,甚至导致整个内存回收系统崩溃。
>
> finalize()方法是对象逃脱回收的最后一次机会,稍后GC将对F-Queue中的对象进行第二次小规模的标记,如果对象要在finalize()中跳出回收——只要重新与引用链上的任何一个对象建立关联即可,譬如把自己(this关键字)赋值给某个类变量或者对象的成员变量,那在第二次标记时它将被移除出“即将回收”的集合;如果对象这时候还没有逃脱,那基本上它就真的被回收了。

### 4.3 垃圾回收算法

#### 标记---清除算法

![](https://pic1.zhimg.com/v2-20129b9cd43d0f9b3bd37537d639464c_b.jpg)

标记---清除算法分两步

- 先把内存区域中的对象进行标记，分清哪些是可回收的
- 回收被标记的垃圾

缺点：会导致内存碎片

#### 复制算法

![](https://pic1.zhimg.com/v2-bb9ee29feb22355eed2c1d3ac45606b7_b.jpg)

​	复制算法（Copying）是在标记清除算法上演化而来，解决标记清除算法的内存碎片问题。它将可用内存按容量划分为大小相等的两块，每次只使用其中的一块。当这一块的内存用完了，就将还存活着的对象复制到另外一块上面，然后再把已使用过的内存空间一次清理掉。保证了内存的连续可用，内存分配时也就不用考虑内存碎片等复杂情况，逻辑清晰，运行高效。

缺点：内存利用率低

#### 标记整理算法

![](https://pic2.zhimg.com/v2-49f3e16de8d552ef585862062abe3f18_b.jpg)



​	标记整理算法（Mark-Compact）标记过程仍然与标记 --- 清除算法一样，但后续步骤不是直接对可回收对象进行清理，而是让所有存活的对象都向一端移动，再清理掉端边界以外的内存区域。

​	标记整理算法一方面在标记-清除算法上做了升级，解决了内存碎片的问题，也规避了复制算法只能利用一半内存区域的弊端。看起来很美好，但从上图可以看到，它对内存变动更频繁，需要整理所有存活对象的引用地址，在效率上比复制算法要差很多。

#### 分代思想

​	当我们学完上面三种垃圾回收算法，出现一个很明显的问题，没有一种方法能够完美解决垃圾回收的问题，所以单独使用一种方法回收效率不会很好。对此JVM的设计者提出了分代算法。

​	所谓分代算法，就是根据 JVM 内存的不同内存区域，采用不同的垃圾回收算法。例如对于存活对象少的新生代区域，比较适合采用复制算法。这样只需要复制少量对象，便可完成垃圾回收，并且还不会有内存碎片。而对于老年代这种存活对象多的区域，比较适合采用标记压缩算法或标记清除算法，这样不需要移动太多的内存对象。

**Eden区**

​	在我们之前的学习中，我们知道Eden区中的对象98%都是朝生夕死的，所以针对这一现状，大多数情况下，对象会在新生代 Eden 区中进行分配，当 Eden 区没有足够空间进行分配时，虚拟机会发起一次 Minor GC，Minor GC 相比 Major GC 更频繁，回收速度也更快。

​	通过 Minor GC 之后，Eden 会被清空，Eden 区中绝大部分对象会被回收，而那些无需回收的存活对象，将会进到 Survivor 的 From 区（若 From 区不够，则直接进入 Old 区）。

**Survivor区**

​	Survivor区事实上相当于Eden区和Old区的一个缓冲区，这里我们需要面对两个问题：

​	**1.为什么需要Survivor区？**

​	这里我们假设没有Survivor区，那么就会出现这样的情况，一旦我们的Eden执行了一次Minor GC，剩下存活的对象就会被放到Old区，但事实上，很多对象也就能活过这一次Minor GC，把他们放入Old区纯属浪费资源，到最后Old区内存不足，就会出现问题

​	**2.为什么需要俩？**

​	设置两个 Survivor 区最大的好处就是解决内存碎片化。

​	我们先假设一下，Survivor 如果只有一个区域会怎样。Minor GC 执行后，Eden 区被清空了，存活的对象放到了 Survivor 区，而之前 Survivor 区中的对象，可能也有一些是需要被清除的。问题来了，这时候我们怎么清除它们？在这种场景下，我们只能标记清除，而我们知道标记清除最大的问题就是内存碎片，在新生代这种经常会消亡的区域，采用标记清除必然会让内存产生严重的碎片化。因为 Survivor 有2个区域，所以每次 Minor GC，会将之前 Eden 区和 From 区中的存活对象复制到 To 区域。第二次 Minor GC 时，From 与 To 职责兑换，这时候会将 Eden 区和 To 区中的存活对象再复制到 From 区域，以此反复。

​	这种机制最大的好处就是，整个过程中，永远有一个 Survivor space 是空的，另一个非空的 Survivor space 是无碎片的。那么，Survivor 为什么不分更多块呢？比方说分成三个、四个、五个?显然，如果 Survivor 区再细分下去，每一块的空间就会比较小，容易导致 Survivor 区满，两块 Survivor 区可能是经过权衡之后的最佳方案。

**Old区**

​	老年代占据着2/3的堆内存空间，只有在 Major GC 的时候才会进行清理，每次 GC 都会触发“Stop-The-World”。内存越大，STW 的时间也越长，所以内存也不仅仅是越大就越好。由于复制算法在对象存活率较高的老年代会进行很多次的复制操作，效率很低，所以老年代这里采用的是标记 --- 整理算法。

> 什么是“Stop-The-World”？   线程挂起，清理垃圾，清理完了再重新跑线程

​	除了上述所说，在内存担保机制下，无法安置的对象会直接进到老年代，以下几种情况也会进入老年代。

1、大对象

​	大对象指需要大量连续内存空间的对象，这部分对象不管是不是“朝生夕死”，都会直接进到老年代。这样做主要是为了避免在 Eden 区及2个 Survivor 区之间发生大量的内存复制。当你的系统有非常多“朝生夕死”的大对象时，得注意了。

2、长期存活对象

​	虚拟机给每个对象定义了一个对象年龄（Age）计数器。正常情况下对象会不断的在 Survivor 的 From 区与 To 区之间移动，对象在 Survivor 区中没经历一次 Minor GC，年龄就增加1岁。当年龄增加到15岁时，这时候就会被转移到老年代。当然，这里的15，JVM 也支持进行特殊设置。

3、动态对象年龄

​	虚拟机并不是永远地要求对象的年龄必须达到了MaxTenuringThreshold才能晋升老年代，如果在Survivor空间中相同年龄所有对象大小的总和大于Survivor空间的一半，年龄大于或等于该年龄的对象就可以直接进入老年代，无须等到MaxTenuringThreshold中要求的年龄。

#### 分区思想

​	JVM 中其实还有一个分区思想，即将整个堆空间划分成连续的不同小区间。每一个小区间都独立使用，独立回收，这种算法的好处是可以控制一次回收多少个区间，可以较好地控制 GC 时间。