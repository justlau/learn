# JVM

## 一次编译到处运行
当我们想要执行一个java文件时，需要完成以下步骤
````
* 调用 javac 命令编译一个 Demo.java 文件生成一个 Demo.class 文件
* 调用 java 命令时会由ClassLoader将编译后的class文件加载到内存，同时也会将java类库的文件也同时加载到进来
* 加载进来后会通过字节码解释器或者JIT(及时编译器)来进行解释，解释完成后由执行引擎来执行，而执行引擎面对的就是OS硬件

而由ClassLoader加载开始到执行引擎执行的过程就是JVM
````
任何语言，只要能编译成class，并符合class的规范，那么就能够在JVM里运行，而JVM封装了所有操作系统的对应实现，所以能够做到一次编译到处运行。

## Class文件的生命周期
一个class的生命周期为`loading -> linking -> initializing -> gc` 这四个过程

### loading
JVM是按需动态加载，并且采用双亲委派的机制通过ClassLoader对class文件进行加载。
#### 双亲委派
````
 ClassLoader有四种层级，分别是 BootstrapClassLoader -> ExtensionClassLoader -> AppClassLoader -> CustomClassLoader

 BootstrapClassLoader: 是最顶层的ClassLoader，主要加载 /lib/rt.jar,charset.jar 等核心类库，是由C++实现的
 ExtensionClassLoader: 是加载 /jre/lib/ext/*.jar 或者由 -Djava.ext.dirs 指定的jar包
 AppClassLoader: 是加载 classpath 下指定的内容
 CustomClassLoader: 自定义的ClassLoader

 以上ClassLoader的定义以及加载范围均存在与 sun.misc.Launcher 类中
````
需要注意的是，双亲委派机制并不是指这几个ClassLoader是继承关系，而是由一种概念关系，每一个ClassLoader内部都有一个parent属性，通过parent来建立ClassLoader之间的逻辑继承关系，整个双亲委派机制如下：
````
 检查过程
 * 假设我们自定义了ClassLoader，那么在加载一个类时，首先由CustomClassLoader来检查当前需要被加载的class是否在当前classLoader的缓存中已存在，
   如果不存在就向上一级也就是AppClassLoader询问是否被加载，如果AppClassLoader也没有加载过当前class，那么就由AppClassLoader向上一级也就是
   ExtensionClassLoader询问是否已经加载过当前class，如果没有加载，那么就由ExtensionClassLoader继续向上一级也就是BootstrapClassLoader
   询问是否被加载，整个过程如果已经在任意层级的ClassLoader中被加载，那么就不会再加载一次，如果没有就进行下边加载过程。

 加载过程
 * 如果BootstrpClassLoader也没有加载过当前class，那么它就会告诉下一级也就是ExtensionClassLoader让它去加载当前class，
   ExtensionClassLoader会判断当前class是否属于自己加载的范围，如果不属于就委托下一级也就是AppClassLoader进行加载,AppClassLoader也会进行
   范围判断，如果不符合就委托下一级也就是CustomClassLoader进行加载，如果当前class也不符合CustomClassLoader的加载范围，那么就会抛出
   ClassNotFoundException，否则就会被加载到内存。

 整个双亲委派机制的流程便如上所诉，经历了自下而上的检查判断，再到自上而下的加载判断，最终实现class的加载，在代码的实现便是递归的过程，先一层一层往上找，
 如果找到便返回，如果找不到便继续询问上一级，当到达顶层时，判断是否存在，并且尝试加载，如果属于自己便加载，如果不属于便逐层退出。
````
图例如下

![Image text](https://raw.githubusercontent.com/justlau/resources/main/%E5%8F%8C%E4%BA%B2%E5%A7%94%E6%B4%BE.png)

在class加载过程中为什么会使用双亲委派这种机制呢？

    * 从安全角度：如果我们重写了java核心类库的类，并且不通过双亲委派这种机制进行检查再加载，那么就会出现java核心类库被覆盖的风险，从而造成一
      系列的安全风险。比如我们定义String类，并且包名为java.lang.String，通过自定义的ClassLoader将此类加载到内存，覆盖掉java核心类库中
      的String类，并最终提供给用户使用，那么我们就可以通过自定义的String类来对所有字符串操作进行任意处理，比如说密码等重要信息就可以直接获取，
      造成安全风险。
    * 从容量角度：如果需要被加载的class已经被其他ClassLoader加载了，那么就无需继续加载，直接使用即可。
如何打破双亲委派？

    * 由 ClassLoader 的源码可知，双亲委派机制是由 loadClass(String name) 方法定义好了，如果想自定义 ClassLoader 只需要继承 ClassLoader
      并重写 findClass(String name) 方法即可，而如果想要打破双亲委派机制，只需要重写 loadClass(String name) 即可。
何时打破双亲委派机制

    * 在jdk1.2之前，自定义 ClassLoader 都必须要重写 loadClass(String name) 方法，这时就可以打破此机制，这是当时JDK的一个缺陷。
    * 可以在通过 java.lang.Thread#setContextClassLoader 方法来设置某个线程的ClassLoader，比如说项目的启动线程，这时就可以将
      自定义ClassLoader用作项目使用，随便加载任何类到项目中。
    * tomcat的热部署就要求各模块使用指定的ClassLoader，例如：修改了某个jsp文件，tomcat是如何做到不需要重部署便可更新jsp？其实它是将
      修改前加载jsp模块的那个ClassLoader置为空，重新创建一个ClassLoader，通过打破双亲委派机制，重新将此模块下的文件加载了
      一次，便可达到热部署的效果。
### linking
linking 分为三部分组成，分别是`verification -> preparation -> resolution`
#### verification
验证加载进来的文件是否符合JVM规定
#### preparation
给静态成员变量赋`默认值`，比如说int类型的就赋值为0，Integer类型的就赋值为null，然后执行静态代码块
#### resolution
* 将类、方法、属性等符号引用解析为直接引用
* 常量池的各种符号引用解析为指针、偏移量等内存地址的直接引用
### initializing
调用类的初始化方法给静态成员变量赋`初始值`
````
    假设我们有这样一个类
    public class T {
        private int m = 8;
    }


    那么当我们执行 T t = new T()时，通过IDE自带的jclasslib插件可以看到有如下几条指令：
   
    0 new #3 <org/learn/T>                  //申请内存
    3 dup
    4 invokespecial #4 <org/learn/T.<init>> //调用构造方法，并且把m赋值为8
    7 astore_1                              //将new出来T的引用赋值给t
    8 return
````
## 硬件层数据一致性问题
### 缓存一致性问题
[文档](https://cnblogs.com/z00377750/p/9180644.html)
### 缓存行伪共享问题
````
假设我们在主存中有两个值，int a = 10,int b = 20，假设有两个CPU，CPU1和CPU2，CPU1只需要读取a，CPU2只需要读取b

在这样一个假设的前提下，CPU在从主存中读取对应的数据时，会一次读取一个缓存行(cache line)，而这个缓存行多数情况下的大小为64byte，那么假设a和b在
同一个缓存行，CPU1读取a时会将整个缓存行全部读到自己的高速缓存中，而CPU2读取b时也会将整个缓存行全部读取到高速缓存中，那么假设CPU1修改了a的值，根
据缓存一致性协议，需要通知其他CPU这个缓存行已经做出了修改，当CPU2收到缓存行改变的通知时，会重新到主存中加载这个缓存行，这时就出现了一个问题，CPU2
只需要b的值，但是却因为无关数据的改变导致自己不停的从主存中更新数据，同理CPU1也会存在这种问题，这就叫缓存伪共享。
````
### CPU指令乱序问题
当一堆指令到达时，CPU为了提高指令的执行效率，会分析指令之间的依赖关系，若指令之间无依赖关系便可乱序执行这些指令
````
假设有这么三条指令 byte[] a = read(),int a = 10,int b = 20
CPU在收到这三条指令时，先执行read()，也就是从内存中读取数据，而CPU执行指令的时间是比从内存读取时间快的，那么在等待期间，CPU便会分析后续指令是否
与当前指令有依赖关系，若没有依赖关系，便在等待期间同时执行后续指令，若有依赖关系便等待当前指令结束。

依赖关系是指：后一条指令的执行不依赖前一条指令的结果，上述三条指令便没有依赖关系，而如果到达的指令是这样的：int a = 10,int b = a，那么这两条指
令之间是有依赖关系的，这样便只能等待第一条指令执行完毕时，才会执行第二条指令。
````
### 如何在硬件层面保证不乱序
#### 硬件内存屏障，在X86架构下的CPU硬件屏障指令如下：
````
 sfence(写屏障store fence)：在sfence指令前的写操作，必须在sfence指令后的写操作前完成
 lfence(读屏障load fence)：在lfence指令前的读操作当必须在lfence指令后的读操作前完成
 mfence：在mfence指令前的读写操作当必须在mfence指令后的读写操作前完成

````
#### JVM对于内存屏障的规范
JVM的内存屏障是依赖于硬件的实现，它只是定义了一个规范
````
 LoadLoad：
    对于这样的语句 Load1;LoadLoad;Load2
    在Load2及后续读取操作执行前，要保证Load1的指令执行完毕
 StoreStore：
    对于这样的语句 Store1;StoreStore;Store2
    在Store2及后续写入操作执行前，要保证Store1执行完且要对其他CPU可见
 LoadStore：
    对于这样的语句 Load1;LoadStore;Store1
    在Store1及后续操作执行前，要保证Load1要读取的数据被读取完毕
 StoreLoad：
    对于这样的语句 Store1;StoreLoad;Load1
    在Load1及后续操作执行前，保证Store1的指令执行完且要对其他CPU可见
````
#### volatile的实现细节

* 字节码层面
````
public class T {

    volatile int m = 8;
}

通过IDE自带的jclasslib查看字节码文件发现只是在flag上增加了一个ACC_VOLATILE的标识
````
* JVM的实现
````
在volatile的读写都增加屏障

StoreStoreBarrier
volatile写操作
StoreLoadBarrier
----------------------------
LoadLoadBarrier
volatile读操作
LoadStoreBarrier

````
## 对象在内存中的布局
### 对象的创建过程

* 将class文件从磁盘加载到内存
* class linking，也就是校验 -> 静态成员属性赋默认值 -> 解析
* class初始化，给静态成员属性赋初始值，同时执行静态代码块
* 申请对象所需内存
* 普通成员属性赋默认值
* 调用构造方法，字节码层面是<init>
    * 给普通成员属性按顺序赋初始值
    * 执行构造方法代码块
        * 执行super的构造方法，进行super的初始化(如果构造方法里没有显示调用父类的构造方法，那么就会隐式调用父类的无参构造)
        * 执行构造方法代码块
### 对象在内存中的布局
在JVM里，对象在内存中分为三部分
* 对象头，而对象头又分为以下几部分
    * markword 占用8个字节，主要存储对象的hashCode(如果调用了hashCode方法)、GC分代年龄、锁状态标志等信息
    * classpoint 对象指向class的指针，代表当前的实例对象属于哪一个类，在开启-XX:+UseCompressedClassPointers时为4字节，不开启占8字节
    * arraylength 若对象为数组，则还需要记录数组的长度，若为普通对象就没有此项，占用4字节
* 实例数据
    * 主要记录对象成员属性的大小
        * byte      1字节
        * char      2字节
        * short     2字节
        * int       4字节
        * float     4字节
        * long      8字节
        * double    字节
        * boolean   至少1字节
    * 若成员属性为引用类型，开启-XX:+UseCompressedOops时为4字节，不开启为8字节
* 对齐填充
    * 若对象大小所占字节数不满足8的倍数，那么就会自动补齐
## Java Run-Time Data Areas
Java运行时区域分为以下几部分

* 线程私有区
    * Program Counter 程序计数器，由JVM管理的区域
        * 每一个线程都有它自己的程序计数器，而程序计数器里存储的便是当前线程需要执行的所有指令
    * Stack 栈，由JVM管理的区域
        * JVM Stacks
            * 在Java程序中，每一个线程对应一个栈，而栈里存储的都是栈帧，每个方法对应一个栈帧
        * Native Method Stacks 本地方法栈
            * 由c或者c++实现的native方法栈
    * Direct Memory 直接内存，由OS管理的区域
        * 在JDK1.4以后为了增加IO的效率，增加了此块区域，在虚拟机内部是可以直接访问操作系统管理的内存，也就是零拷贝
* 线程共享区
    * Method area 方法区
        * 常量池、class
    * Heap 堆
        * 
### JVM Stack 栈

方法指令初识
````
假设我们有这样一段代码
public class T {

    public void simple() {
        int b = 9;
    }

    public void object(){
        T t = new T();
        simple();
    }

    public static void main(String[] args) {
        System.out.println(args);
    }
}
通过jclasslib可观察到各个方法的字节码文件如下：

simple方法：
0 bipush 9  //将9压栈
2 istore_1  //将9出栈，并且赋值给本地变量表中的第2(因为计数是从0开始的)个声明，也就是b
3 return    //结束

object方法
 0 new #2 <org/learn/T>  //创建一个对象，对象指向常量中的2号数据也就是T.class，并执行一系列初始化动作，比如说赋默认值等，并将对象地址压栈
 3 dup  //复制一份地址并入栈，此时栈里有两个值，都是new T()在内存中的地址
 4 invokespecial #3 <org/learn/T.<init>> //将复制的地址出栈，并执行地址对应对象的构造方法，此时进行成员属性赋初始值、执行构造代码块等
 7 astore_1 //将剩下的地址值出栈并赋值给本地变量表中第2个声明，也就是t
 8 aload_0 //将第0步也就是new T()产生的地址再次入栈
 9 invokevirtual #4 <org/learn/T.simple> //取出栈中的地址，并执行其方法，方法便是指向常量池中4号位置的simple方法
12 return // 结束

main方法
0 getstatic #5 <java/lang/System.out>   //获取常量池中第5位的PrintStream静态类地址并入栈
3 aload_0   //将本地变量表第1位入栈，也就是方法的参数 args
4 invokevirtual #6 <java/io/PrintStream.println> //依次将args PrintStream的地址出栈，并执行PrintStream在常量池中6号位置的方法
7 return    //结束
````
* _push _load 等都属于入栈指令
* _store_x 属于出栈并赋值给本地变量表中`x+1`声明的指令
* invoke 属于执行方法的指令
    * invokeStatic
        * 执行静态方法的指令
    * invokeVirtual
        * 执行非private、final方法的指令，且自带多态，也就是说再执行这条指令前入栈的对象地址是谁，就调用谁的方法
    * invokeInterface
        * 执行Interface的方法，比如说`List<string> list = new ArrayList<>();list.add("hello")`便是使用这个指令
    * invokeSpecial
        * 执行private、final方法的指令，因为这两种修饰符的方法不会被重写，所以不会存在多态情况。
    * invokeDynamic
        * 这条指令是对lambda表达式、反射或者其他动态语言诸如scala、kotlin，或者CGlib、ASM等动态产生的class的支持

## Garbage Collector 垃圾回收
什么是垃圾，没有引用指向的对象，都被称之为垃圾。
### 如何找到垃圾
* reference count 引用计数器
    * 在对象内部会有一个引用计数，若引用数为0，则表明可以被回收
    * rc 的弊端为：不能解决循环引用
* root searching 根可达算法
    * 从根对象开始逐级引用搜索，那些无法被搜索到的对象则为垃圾
    * 根对象实例有哪些？
        * 线程栈变量 当执行一个方法时，方法本身所声明的那些变量对象便是根对象
        * 静态变量 静态变量所引用的对象
        * 常量池 当一个Class的常量引用了其他对象，那么这个被引用的对象也是根对象
        * JNI指针 本地方法栈所引用的对象
### GC Algorithms GC算法
* Mark-Sweep 标记清除算法
    * 一种比较简单的算法，通过根可达算法去扫描，找到那些`存活的对象`并做出标记，然后再一次扫描找到那些`没被标记的对象`进行回收
    * 优缺点
        * 存活对象比较多的情况下效率极高
        * 两次扫描，效率偏低，且容易产生碎片
* Copying 拷贝
    * 将内存划分为大小相等的两块区域，每次只使用其中一块，当触发GC时，通过根可达算法找到存活的对象，并将其拷贝到未使用的那块区域，然后再将已使用的那一块区域全部清理掉
    * 优缺点
        * 适用于存活对象较少的情况，只扫描一次，效率提高没有碎片
        * 空间浪费，移动复制对象，需要调整对象引用
* Mark Compact 标记压缩
    * 简单来说，标记压缩算法就是将所有存活的对象整体向一端移动，将对象间的空隙消除
        * 标记
            * 通过根可达算法，找到那些存活的对象并作出标记
        * 压缩
            * 计算存活对象移动后的地址，并进行保存
            * 更新存活对象的引用
            * 移动存活的对象
    * 优缺点
        * 不会产生碎片，方便对象分配，也不会产生内存减半
        * 扫描两次，需要移动对象，效率偏低
### JVM内存分代模型
* 除Epsilon、ZGC、Shenandoah之外的GC都是使用逻辑分代模型
* G1是逻辑分代，物理不分代，除此之外不仅逻辑分代而且物理也分代

#### 堆内存逻辑分区
逻辑分代是把内存按照`1:2`的比例分为`新生代(new/young)`和`老年代(old)`

年轻代GC：MinorGC/YGC

老年代GC：MajorGC/FullGC
* 新生代，是指那些刚刚new出来的对象，并且这些对象的特点是`大量死去，少量存活，使用于Copying算法`，而新生代又分为三个区域，默认比例为`8:1:1`，这种比例分配的原因是JVM认为eden区有90%的对象会被回收，仅有10%的对象会存活，eden是为了分配对象使用，而两个survivor区则是为了垃圾回收使用。
    * eden(伊甸区)
    * survivor
    * survivor
    * 新生代的垃圾回收逻辑如下
        * 首次YGC时，将eden区活着的对象全部拷贝到survivor1区，然后将eden区全部清除，释放eden区的内存；
        * 再次YGC时，将eden区和survivor1区还存活的对象全部拷贝到survivor2区，同时释放eden和survivor1两块内存；
        * 再次YGC时，将eden区和survivor2区还存活的对象全部拷贝到survivor1区，同时释放eden和survivor2两块内存；
        * 后续GC按照以上逻辑，最后年龄足够的对象进入老年代，同时如果survivor区装不下的对象也会提前进入老年代；
            * 年龄足够是指，从s1到s2区域来回拷贝的次数，这个次数可以通过 -XX:MaxTenuringThreshold进行配置
* 老年代，是指那些多次GC时，都没有被回收的对象，会进入老年代
    * tenured
* 对象何时进入老年代
    * 超过次数
        * 在YGC时，超过从s1到s2区域来回拷贝的次数时就会进入老年代，可通过-XX:MaxTenuringThreshold来设置
        * -XX:MaxTenuringThreshold 的默认值
            * Parallel Scavenge 15
            * CMS 6
            * G1 15
    * 动态年龄
        * 当垃圾回收时，从eden+s1(或s2)拷贝到s2(或s1)区的对象按照年龄从小到大的顺序进行累加，当累加和超过了s1或s2的50%时，会将年龄最大的放入老年代

#### 一个对象从出生到消亡的过程
* 栈上分配
    * 栈上分配是为了JVM为优化对象分配而做的一种优化措施，当创建对象时，JVM会根据逃逸分析判断当前对象是否有逃逸，如果不会逃逸那么就可以再栈上分配内存，这样该对象占用的内存空间就会随着栈帧出栈而销毁，减轻GC的压力。
    * 满足栈上分配的条件
        * 线程私有的小对象
        * 无逃逸 无逃逸是指对象仅在方法中被定义和引用，不会被其他方法或线程所引用
        * 支持标量替换
            * 标量是指不可被进一步分解的量，而java中的基本数据类型就是标量。
            * 替换是指通过逃逸分析确定该对象不会被外部访问，并且对象可以被进一步分解，那么就将这个对象的成员变量分解为方法使用的成员变量来代替，且分解后的成员变量类型为标量类型。
* 线程本地分配TLAB(Thread Local Allocation Buffer)
    * 线程本地分配是指在多线程情况下，每个线程去eden区申请eden区1%大小的内存，为线程独有，然后在这块区域里进行对象分配，不会与其他线程进行竞争，提高效率。

对象分配和消亡的过程
* 当一个对象被创建时，首先会进行判断，是否可以在栈上分配，
    * 可以再栈上分配，那么对象便随着栈帧的弹出而消亡；
    * 如果不能在栈上分配，会判断对象对象的大小是否超过了参数设定的大小
        * 超过参数设定的大小，就直接进入老年代分配，最终等待FullGC而消亡；
        * 没有超过参数设定的大小，就会进入eden区分配，然后通过一次一次的YGC，要么被YGC消亡，要么进入老年代，等待FullGC消亡
