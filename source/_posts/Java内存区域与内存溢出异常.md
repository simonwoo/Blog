title: Java内存区域与内存溢出异常
date: 2016-1-4
tags: [Java, JVM] #文章标签，可空，多标签请用格式，注意:后面有个空格
---
## 运行时数据区域

Java虚拟机在执行Java程序的过程中，会把内存分为不同的数据区域。如下图所示：

![内存分布](http://7xq5i5.com1.z0.glb.clouddn.com/img_jvm_model.png)

### 程序计数器

它是一块较小的内存空间，作用可以当做是当前线程所执行的字节码的行号指示器。在虚拟机的概念模型里，字节码解释器工作时就是通过改变这个计数器的值来选取下一跳需要执行的字节码指令，分支、循环、跳转、异常处理、线程恢复等基础功能都需要依赖这个计数器（字节码行号记录器）。

因为java虚拟机的多线程是通过时间片轮转占用cpu，所以一个处理器只会执行一条线程的指令。为了线程切换后能恢复到正确的位置，每个线程都需要一个独立的程序计数器，这样能使各个线程之间的计数器互不影响，独立存储。这类区域为线程私有内存。

- 如果线程正在执行一个 Java 方法，这个计数器记录的是正在执行的虚拟机字节码指令地址
- 如果线程正在执行的是 Native 方法，这个计数器值为空（Undefined）

此内存区域是唯一一个在 Java 虚拟机规范中没有规定任何 OOM 情况的区域。

### Java虚拟机栈

Java虚拟机栈也是线程私有的，它的生命周期和线程相同。

Java虚拟机栈描述的是Java方法执行的内存模型：每个方法被执行的时候都会创建一个栈帧，用于存储局部变量表、操作栈、动态链接、方法出口等信息。每一个方法被调用直至执行完成的过程就对应着一个栈帧在虚拟机中从入栈到出栈的过程。调用一个方法时创建新的栈帧并压入栈顶部，方法执行完后，这个栈帧就会弹出栈帧的元素作为这个方法的返回值，并清除这个栈帧，Java栈的栈顶就是当前正在执行的活动栈，也就是当前正在执行的方法，PC寄存器也会指向这个地址。

局部变量表存放了基本数据类型、对象引用和returnAddress类型（指向一条字节码指令的地址）。其中64位长度的long和double类型的数据会占用2个局部变量空间（slot），其余数据类型只占用1个。局部变量表所需的内存空间在编译期间完成分配。

在Java虚拟机规范中，对这个区域规定了两种异常情况：

- 如果线程请求的栈深度太深，超出了虚拟机所允许的深度，就会出现StackOverFlowError（比如无限递归。因为每一层栈帧都占用一定空间，而 Xss 规定了栈的最大空间，超出这个值就会报错）
- 虚拟机栈可以动态扩展，如果扩展到无法申请足够的内存空间，会出现OOM

### 本地方法栈

本地方法栈与虚拟机栈的作用是非常类似的，区别是虚拟机栈为虚拟机执行Java方法（也就是字节码）服务，而本地方法栈则是为虚拟机使用到的Native方法服务。因为虚拟机规范没有对这块有太多规定，所以不同的虚拟机可以自由实现它。有的虚拟机（Sun的HotSpot虚拟机）直接就把本地方法栈和虚拟机栈合二为一了。

### Java堆

对于大多数应用来说，Java堆是Java虚拟机所管理的内存中最大的一块，它是所有线程共享的，在虚拟机启动时候创建。Java堆唯一的目的就是存放对象实例（当然还有数组），Java堆是垃圾收集器管理的主要区域。堆可分为老年代和新生代，再细分还可以分为Eden空间、From Survivor空间、To Survivor空间等。主流虚拟机都可扩展（-Xmx和-Xms）

如果堆上没有内存可以完成对象实例的分配，并且堆已经达到了最大容量，无法向OS继续申请的时候，就会抛出OOM异常。

### 方法区

方法区与Java堆一样，是所有线程共享的内存区域，它用于存储已被虚拟机加载的类信息、常量、静态变量、即时编译器编译后的代码等数据。

对于习惯在 HotSpot虚拟机上开发部署程序的开发者来说，很多人倾向于把方法区成为“永久代（Perm Generation）”，但本质上两者并不等价，仅仅是因为 HotSpot 团队选择把 GC 分代收集扩展到方法区，或者说使用永久代来实现方法区而已，目的是为了让 HotSpot 的垃圾回收器可以像管理 Java 堆一样管理这部分内存，不能再编写这部分内存的内存管理代码。对于其他虚拟机（比如 JRockit、IMB J9）来说，是不存在永久代的概念的。

其实 JVM 规范并没有规定如何实现方法区，但是从目前状况来看：使用永久代来实现方法区不是一个好的做法。因为这样更容易遇到内存溢出问题（永久代有-XX:MaxPermSize 的上限，而 J9和 Jrockit 只要没有触碰到进程可用内存的上限，例如32位的4GB，就不会出现问题），同时有极少数方法（比如 String.intern()，这个函数能直接操纵方法区中的常量池）会因为这个原因在不同虚拟机有不同的表现。因此，HotSpot 团队有了放弃永久代并逐步改为采用 Native Memory 来实现方法区的规划，在目前已经发布的 JDK1.7 的 HotSpot 中，已经把放在永久代的字符串常量池移出。

当方法区无法满足分寸分配需求时，就会抛出OOM异常。

### 运行时常量池

方法区的一部分。class文件中除了有类的版本、字段、方法、接口等描述信息外，还有一项信息是常量池(class文件中)，用于存放编译期生成的各种字面量和符号引用，这部分内容将在类加载后存放到方法区的运行时常量池中。除了保存Class文件中描述的符号引用外，还会把翻译出来的直接引用也存储在运行时常量池中。

运行时常量池相对于class文件常量池的另外一个重要特性是具备动态性，Java语言并不要求常量一定是在编译期产生，也就是说，并非是预置入class文件中常量池的内容内能进入方法区的运行时常量池，运行期间也可以将新的常量放入池中，用的比较多是有String.intern()，可以去看下文档。说的很清楚：

### 直接内存

直接内存并不是虚拟机运行时数据区的一部分，也不是Java虚拟机规范中定义的内存区域，但是这部分内存也被频繁地使用，而且也可能导致OutOfMemoryError异常出现。

JDK 1.4中新加入了NIO(NEW Input/Output)类，引入了一种基于通道与缓冲区的I/O方式，可以使用Native函数库直接分配堆外内存，然后通过一个存储在Java堆里面的DirectByteBuffer对象作为这块内存的引用进行操作。这样能在一些场景中显著提高性能，因为避免了在Java堆和Native堆中来回复制数据。

服务器管理员在配置虚拟机参数时，会根据实际内存设置-Xmx等参数信息，但经常忽略直接内存，使得各个内存区域总和大于物理内存限制（包括物理的和操作系统级的限制），从而导致动态扩展时出现OOM。

### 总结

1. 程序计数器：行号指示器；空间小，最快；线程私有；不会有OOM
2. Java虚拟机栈：Java方法执行的内存模型,用于存储局部变量表、操作栈、动态链接、方法出口等信息；线程私有；StackOverFlowError,OOM
3. 本地方法栈：和Java虚拟机栈发挥的作用非常相似，但是市委Native方法服务。
4. Java堆：存放对象实例；线程共享；OOM
5. 方法区：存储已被虚拟机加载的类信息、常量、静态变量、即时编译器编译后的代码等数据；线程共享；OOM
6. 运行时常量池：方法区的一部分；线程共享；存放编译期生成的各种字面量和符号引用；OOM
7. 直接内存：直接内存并不是虚拟机运行时数据区的一部分，但是这部分内存也被频繁地使用，而且也可能导致OutOfMemoryError异常出现；新NIO利用了直接内存，效率高

## HotSpot虚拟机对象探秘

探讨Java堆中对象分配、布局和访问的全过程。

### 对象的创建

1. 虚拟机遇到一条 new 指令时，首先将去检查这个指令的参数是否能在常量池中定位到一个类的符号引用，并且检查这个符号引用代表的类是否已经被加载、解析和初始化过。如果没有，就得执行类的加载过程，这个过程在第七章讲了。TODO 添加第七章链接
2. 类加载检查过之后，虚拟机就为这个新生对象分配内存。目前有两种做法，使用哪种方式是由 GC 回收器是否带有压缩整理功能决定的：

	- 指针碰撞（Bump the Pointer）：假设Java堆中内存是绝对规整的 ，没用过的内存和用过的内存用一个指针划分（需要保证 java 堆中的内存是规整的，一般情况是使用的 GC 回收器有压缩整理功能），分配内存仅仅是将指针向空闲空间那边挪动一段与对象大小相等的距离。假如需要分配8个字节，指针就往后挪8个字节
	- 空闲列表（Free List）：假设Java堆中内存是不规整的，已使用内存和空闲内存交错，虚拟机维护一个列表，记录哪些内存是可用的，分配的时候从列表中遍历，找到合适的内存分配，然后更新列表

3. 分配内存过程中还需要解决线程安全问题。 就刚才的一个修改指针操作，就会带来隐患：对象 A 正分配内存呢，突然对象 B 又同时使用了原来的指针来分配 B 的内存。解决方案也有两种：

	- 同步处理——实际上虚拟机采用 CAS 配上失败重试来保证更新操作的原子性
	- 把内存分配的动作按照线程划分在不同的空间之中进行，即每个线程在 Java 堆中预先分配一小块内存，成为本地线程分配缓存（Thread Local Allocation Buffer，TLAB）。哪个线程要分配内存，就在哪个线程的 TLAB 上分配，用完并分配新的TLAB时，才需要同步锁定（虚拟机是否使用 TLAB，可以通过-XX:+/-UseTLAB 参数来设置）
4. 给内存分配了空间之后，虚拟机需要将分配到的内存空间都初始化为零值（不包括对象头）。如果使用TLAB，这一工作过程也可以提前至TLAB分配时进行。

5. 接下来要对对象进行必要的设置，比如这个对象是哪个类的实例、如何才能找到类的元数据信息、对象的 hashcode 值是多少、对象的 GC 分代年龄等信息，这些信息都放在对象头中。

6. 上面的步骤都完成后，从虚拟机角度来看，一个新的对象已经产生了，但是从 Java 程序的视角来看，对象创建才刚刚开始——<init> 方法还没有执行，所有的字段都还为零。把对象按照程序员的意愿进行初始化，这样一个真正可用的对象才算完全产生出来。

### 对象的内存布局

首先我们要知道的是：在 HotSpot 虚拟机中，对象在内存中存储的布局可以分为3块区域：对象头（Header）、实例数据（Instantce Data）、对齐补充（Padding）。

1. 对象头（Header）：包含两部分信息。第一部分用于存储对象自身的运行时数据，如 hashcode 值、GC 分代的年龄、锁状态标志、线程持有的锁等，官方称为“Mark Word”。第二部分是类型指针，即对象指向它的类元数据的指针，虚拟机通过这个指针来确定这个对象是哪个类的实例
2. 实例数据（Instance Data）：就是程序代码中所定义的各种类型的字段内容。无论是从父类继承下来的，还是在子类中定义的，都需要记录起来。
3. 内存对齐，对象的大小必须是8字节的整数倍

### 对象的访问定位

1. 假如代码出现在方法体中，那么Object obj就会存在在Java虚拟机栈的本地变量表中，作为一个引用类型数据。
2. new Object()则存在在Java堆上。另外，在Java堆上还必须包含能查找到该对象类型数据（如对象类型、父类、实现的接口、方法等）的地址信息，这些类型数据则存储在方法区中。
3. 由于引用类型在Java虚拟机规范里面只规定了一个指向对象的引用，并没有定义这个引用应该通过哪种方式去定位，以及访问到Java堆中的对象的具体位置，因此不同虚拟机实现的对象访问方式可能不同，主流的有：

	- 使用句柄：Java堆中划分一块区域作为句柄池，引用存储的是对象的句柄地址，而句柄中含有对象实例数据和类型数据各自的数据信息
	- 直接指针：引用中直接存储的就是对象的地址，同时还必须包括方法区类型信息的指针
下面是对应的图片：
![reference handler](http://7xq5i5.com1.z0.glb.clouddn.com/img_reference_handler.jpg)

![direct reference](http://7xq5i5.com1.z0.glb.clouddn.com/img_direct_reference.jpg)

对于引用类型的实现，不同的实现方法有不同的特点：

1. 使用句柄：Java堆会划出一块区域作为句柄池，引用中存储的是稳定的句柄地址，而句柄中包含了对象实例数据（也在Java堆）和类型数据（方法区中）各自的地址信息。在对象被移动（垃圾回收时移动对象是非常普遍的行为）时只需要改变句柄中的实例数据指针，而引用本身核方法区的类型数据指针都不需要修改
2. 直接指针：速度更快，因为不需要间接寻址。对于效率而言是更好的，Sun HotSpot就是使用这种方式实现对象访问的。但在其他虚拟机中，使用句柄方式也非常常见。

## 实战

下面我们会演示几个小程序，目的有两个：

1. 通过代码验证Java虚拟机规范中描述的各个运行时区域存储的内容
2. 希望以后遇到类似问题时，能根据异常的信息快速判断是哪个区域的内存溢出，知道怎样的代码可能会导致这些区域的内存溢出，以及出现这些异常后改如何处理

### Java堆溢出

这个顾名思义，是最常见的。因为Java堆上存储的是对象实例，所以只要保证GC roots到该对象有路径可达，就会在不断创建对象的过程中达到Java堆的最大容量而导致溢出。下面是实例代码：

```java
public class HeapOOM{
    static class OOMObject{
    }
    public static void main (String[] args){
        List<OOMObject> list = new ArrayList<OOMObject>();
        while(true){
            list.add(new OOMObject());
        }
    }
}

```

从上面的结果可以看到，发生了OOM异常。要解决这个异常，一般是把内存快照dump（通过-XX:+HeapDumpOnOutOfMemoryError）下来用工具（Eclipse Memmory Analyzer）分析，确认内存中的对象是否是必要的，也就是要先分清到底是出现了内存泄露（Memory Leak）还是内存溢出（Memory overflow）。

- 如果是内存泄露：使用工具查看泄露对象到GC Roots的引用链。于是就可以顺藤摸瓜找到泄漏对象是通过怎样的路径关联GC Roots的，从而准确定位泄露代码的位置
- 如果是内存溢出：就应当检查虚拟机的堆参数（-Xmx和-Xms），与机器物理内存对比看是否可以调大，从代码上检查是否存在某些对象生命周期过长、持有状态时间过长等情况，尝试减少程序运行期间的内存消耗

### 虚拟机栈和本地方法栈溢出

-Xss可以设置栈容量。

- 如果线程请求的栈深度大于虚拟机所允许的最大深度，将抛出StackOverFlowError异常
- 如果虚拟机在扩展栈时无法申请到足够的内存空间，将抛出OutOfMemoryError异常
通过调用无限递归调用，单线程下，无论由于栈帧太大还是虚拟机栈容量太小，当内存无法分配的时候，虚拟机抛出的都是StackOverflowError。通过不断创建线程倒是可以产生内存溢出异常，不过和栈空间是否足够大并不存在任何联系。

### 方法区和运行时常量池溢出

```java
public class RuntimeConstantPoolOOM{
    public static void main（String[]args）{
        String str1=new StringBuilder（"计算机"）.append（"软件"）.toString（）；
        System.out.println（str1.intern（）==str1）；
        String str2=new StringBuilder（"ja"）.append（"va"）.toString（）；
        System.out.println（str2.intern（）==str2）；
    }
}

```

这段代码在JDK 1.6中运行，会得到两个false，而在JDK 1.7中运行，会得到一个true和一个false。产生差异的原因是：在JDK 1.6中，intern（）方法会把首次遇到的字符串实例复制到永久代中，返回的也是永久代中这个字符串实例的引用，而由StringBuilder创建的字符串实例在Java堆上，所以必然不是同一个引用，将返回false。而JDK 1.7（以及部分其他虚拟机，例如JRockit）的intern（）实现不会再复制实例，只是在常量池中记录首次出现的实例引用，因此intern（）返回的引用和由StringBuilder创建的那个字符串实例是同一个。对str2比较返回false是因为“java”这个字符串在执行StringBuilder.toString（）之前已经出现过，字符串常量池中已经有它的引用了，不符合“首次出现”的原则，而“计算机软件”这个字符串则是首次出现的，因此返回true。

方法区溢出也是一种常见的内存溢出异常，一个类要被垃圾收集器回收掉，判定条件是比较苛刻的。在经常动态生成大量Class的应用中，需要特别注意类的回收状况。这类场景除了使用了CGLib字节码增强和动态语言之外，常见的还有：大量JSP或动态产生JSP文件的应用（JSP第一次运行时需要编译为Java类）、基于OSGi的应用（即使是同一个类文件，被不同的加载器加载也会视为不同的类）等。

### 本机直接内存溢出

由DirectMemory导致的内存溢出，一个明显的特征是在Heap Dump文件中不会看见明显的异常，如果读者发现OOM之后Dump文件很小，而程序中又直接或间接使用了NIO，那就可以考虑检查一下是不是这方面的原因。
