
**大纲**


**1\.垃圾回收概述**


**2\.如何判断对象存活**


**3\.各种引用介绍**


**4\.垃圾收集的算法**


**5\.垃圾收集器的设计**


**6\.垃圾回收器列表**


**7\.各种垃圾回收器详情**


**8\.Stop The World现象**


**9\.内存分配与回收策略**


**10\.新生代不同配置演示**


**11\.内存泄漏和内存溢出**


**12\.JDK为提供的工具**


 


**1\.垃圾回收概述**


![](https://p3-sign.toutiaoimg.com/tos-cn-i-6w9my0ksvp/7aaa2b01591446359fe53505c9ab3f88~tplv-obj.image?lk3s=ef143cfe&traceid=20241223202751264D64247DDBA21E5D75&x-expires=2147483647&x-signature=jn%2BAxyrhjBaoSXe1ErYvloN%2B8ZA%3D)
 


**2\.如何判断对象存活**


**(1\)引用计数算法**


**(2\)可达性分析算法**


 


**(1\)引用计数算法**


给对象添加一个引用计数器，每当一个地方引用它时就将计数器加1，当引用失效时就将计数器减1，任何时刻计数器为0的对象都不再被使用。


 


这种算法简单，但是有个致命的缺点，就是不能用于相互引用的情况。


 


优点：快、方便、实现简单


缺点：对象相互引用时，很难判断对象是否应被回收


 


PHP、Python的垃圾回收就是使用了引用计数算法，Java的垃圾回收使用的是可达性分析。



```
//testGC()执行后, objA和objB会不会被GC呢?
public class ReferenceCountingGC {
    public Object instance = null;
    private static final int _1MB = 1024 * 1024;
    //这个成员属性的唯一意义就是占点内存, 以便能在GC日志中看清楚是否有回收过
    private byte[] bigSize =  new byte[2 * _1MB];
    public static void testGC() {
        ReferenceCountingGC objA = new ReferenceCountingGC();
        ReferenceCountingGC objB = new ReferenceCountingGC();
        objA.instance = objB;
        objB.instance = objA;
        
        objA = null;
        objB = null;
        
        //假如这里发生GC, 执行结果表明objA和objB是能被回收, 因而Java虚拟机并不是通过引用计数算法来判断对象是否存活
        System.gc();
    }
}
```

**(2\)可达性分析算法**


通过一系列称为"GC Roots"的根对象作为起始点集，根据引用关系从这些节点往下搜索，搜索走过的路径称为引用链(Reference Chain)。


 


当一个对象到"GC Roots"之间不存在任何引用链的时候，就表示这个对象不可达，不可用了。


![](https://p3-sign.toutiaoimg.com/tos-cn-i-6w9my0ksvp/158259499714404f85dcce943e9e5d29~tplv-obj.image?lk3s=ef143cfe&traceid=20241223202751264D64247DDBA21E5D75&x-expires=2147483647&x-signature=SRg1CDtf8l5xjsnusE8pnZHBxq0%3D)
在Java中，可作为GC Roots的对象包括：


一.在虚拟机栈(栈帧中的本地变量表)中引用的对象：比如各线程栈帧对应的方法中，用到的参数、局部变量、临时变量。


二.方法区中静态属性引用的对象，比如Java类的引用类型的静态变量。


三.方法区中常量引用的对象，比如字符串常量池(StringTable)里的引用。


四.本地方法栈中引用的对象。


五.Java虚拟机内部的引用对象，如基本数据类型对应的Class、系统类加载器、一些常驻的异常对象如NullPointException和OutOfMemoryError等。


六.被同步锁(Synchronized)持有的对象。


七.反映Java虚拟机内部情况的对象，比如JMXBean、JVMTI中的回调、本地代码缓存等。


 


从"GC Roots"出发到达不了的那些对象都是可以被回收的。这里从"GC Roots"出发的引用链中的"引用"当然包括4种引用：强引用、软引用、弱引用、虚引用。


 


**3\.各种引用介绍**


**(1\)强引用**


**(2\)软引用(SoftReference)**


**(3\)弱引用(WeakReference)**


**(4\)虚引用(PhantomReference)**


**(5\)软引用和弱引用的应用场景**


 


**(1\)强引用**


一般Object obj \= new Object()就属于强引用。


 


**(2\)软引用(SoftReference)**


一些有用但并非必需的对象，可以使用软引用进行关联。在系统将要发生OOM之前，这些软引用对象才会被回收。如果这些软引用对象被回收后内存还不够，才会发出OOM异常。



```
//软引用示例
//运行前修改VM Options配置:
//-Xms5m -Xmx5m -XX:+PrintGC
public class TestSoftRef {
    public static class User {
        public int id = 0;
        public String name = "";
      
        public User(int id, String name) {
            super();
            this.id = id;
            this.name = name;
        }
      
        @Override
        public String toString() {
            return "User [id=" + id + ", name=" + name + "]";
        }
    }
    
    //输出结果如下:
    //软引用获取实例: User [id=1, name=Test]
    //AfterGc
    //软引用获取实例: User [id=1, name=Test]
    //实例一直存在: User [id=1, name=Test]
    //实例一直存在: User [id=1, name=Test]
    //实例一直存在: User [id=1, name=Test]
    //实例一直存在: User [id=1, name=Test]
    //实例被回收了: null
    public static void main(String[] args) {
        User u = new User(1,"Test");//u是new User(1,"Test")这个实例的强引用
        SoftReference userSoft = new SoftReference<>(u);//userSoft表示的就是一个软引用
        u = null;//保证new User(1,"Test")这个实例此时只有userSoft在软引用, 它的强引用断开了, 所以此时User实例还不会被回收
      
        System.out.println("软引用获取实例: " + userSoft.get());//打印软引用获取到的实例情况
        System.gc();//强制进行垃圾回收, 由于软引用还在, 此时User实例也还不会被回收;(展示GC的时候, SoftReference不一定会被回收)
        System.out.println("AfterGc");
        System.out.println("软引用获取实例: " + userSoft.get());//new User(1,"Test")没有被回收

        //模拟达到OOM的过程, 同时打印软引用获取的实例情况
        List<byte[]> list = new LinkedList<>();
        try {
            //循环不断往堆里面增加大小为1M的字节数组, 同时我们限制了堆的大小是5M
            for(int i=0;i<100;i++) {
                //User(1,"Test")实例一直存在
                System.out.println("实例一直存在: " + userSoft.get());
                list.add(new byte[1024*1024*1]);
            }
        } catch (Throwable e) {
            //抛出了OOM异常后打印的, 软引用被回收了, User(1,"Test")这个实例被回收了
            System.out.println("实例被回收了: "+ userSoft.get());
        }
    }
}
```

**(3\)弱引用(WeakReference)**


弱引用对象是一些有用(程度比软引用更低)但是并非必需的对象。弱引用关联的对象，只能生存到下一次GC之前。WeakHashMap就用到了弱引用。GC发生时，不管内存够不够，弱引用都会被回收。



```
//弱引用示例
public class TestWeakRef {
    public static class User {
        public int id = 0;
        public String name = "";
      
        public User(int id, String name) {
            super();
            this.id = id;
            this.name = name;
        }
      
        @Override
        public String toString() {
            return "User [id=" + id + ", name=" + name + "]";
        }
    }
    
    //输出结果如下:
    //弱引用获取实例: User [id=1, name=Test]
    //AfterGc
    //弱引用获取实例: null
    public static void main(String[] args) {
        User u = new User(1,"Test");//u是new User(1,"Test")这个实例的强引用
        WeakReference userWeak = new WeakReference<>(u);//userWeak表示的就是一个弱引用
        u = null;//保证new User(1,"Test")这个实例此时只有userWeak在弱引用, 它的强引用断开了, 所以此时User实例还不会被回收

        System.out.println("弱引用获取实例: " + userWeak.get());
        System.gc();//强制进行垃圾回收, 弱引用会被回收;
        System.out.println("AfterGc");
        System.out.println("弱引用获取实例: " + userWeak.get());
    }
}
```

**(4\)虚引用(PhantomReference)**


也叫幽灵引用，最弱的。一个对象是否存在虚引用对它的生存完全不构成任何影响，同时通过虚引用也没法拿到一个对象的实例。虚引用唯一的作用就是被垃圾回收的时候会收到一个通知。


 


**注意：**软引用SoftReference和弱引用WeakReference，可以用在内存资源紧张的情况下以及创建不是很重要的数据缓存。当系统内存不足的时候，缓存中的内容是可以被释放的。另外日常编程中不要手动调用"System.gc();"。


 


**(5\)软引用和弱引用的应用场景**


一.二级缓存可以用弱引用WeakReference


二.构建图片缓存可以用软引用SoftReference


 


一个程序需要处理用户提供的图片。如果将所有图片读入内存，这样虽然能很快打开图片，但内存使用巨大。而且一些使用较少的图片会浪费内存空间，需要手动从内存中移除。如果每次打开图片都从磁盘文件中读取到内存再显示出来，虽然内存占用少，但一些经常使用的图片每次打开都要访问磁盘速度慢。这时就可以用软引用构建缓存。


 


很多系统的缓存功能都符合这样的场景：当内存空间还足够时，能够保留在内存中。如果内存空间在进行GC后仍然非常紧张，那就可以抛弃这些对象。


 


**4\.垃圾收集的算法**


**(1\)标记\-清除算法(Mark\-Sweep)**


**(2\)标记\-复制算法(Copying)**


**(3\)标记\-整理算法(Mark\-Compact)**


**(4\)是否移动回收后的存活对象分析**


**(5\)CMS收集器如何处理空间碎片过多**


 


**(1\)标记\-清除算法(Mark\-Sweep)**


首先标记出所有要回收的对象，标记完成后统一回收所有被标记的对象。也可以先标记存活的对象，标记完成后统一回收未被标记的对象。


 


缺点一：执行效率不稳定


如果Java堆中包含大量对象，而且其中大部分是需要被回收的，这时就必须要进行大量的标记和清除动作。这会导致标记和清除两个过程的执行效率都随对象数量增长而降低。


 


缺点二：内存空间的碎片化问题


标记、清除之后会产生大量不连续的内存碎片。空间碎片太多可能会导致：在程序运行过程中需要分配较大对象时，无法找到足够的连续内存而不得不提前触发一次垃圾回收。


![](https://p3-sign.toutiaoimg.com/tos-cn-i-6w9my0ksvp/d6885f6685c1460797d2caf5e66a9a8c~tplv-obj.image?lk3s=ef143cfe&traceid=20241223202751264D64247DDBA21E5D75&x-expires=2147483647&x-signature=tvPS79PPm1k1suXlOnJJ6OYTSMw%3D)
**(2\)标记\-复制算法(Copying)**


**一.半区复制策略**


**二.更优化的半区复制策略**


**三.标记\-复制算法总结**


 


**一.半区复制策略**


将可用内存按容量划分为大小相等的两块，每次只使用其中的一块。当这一块的内存用完了，就将还存活着的对象复制到另外一块上面，然后再把已使用过的内存空间一次清理掉。如果内存中多数对象都是存活的，该策略会产生大量的内存间复制开销。但对于多数对象都是可回收的情况，需要复制的就是占少数的存活对象。


 


由于每次都是针对整个半区进行内存回收，所以内存分配时就不用考虑有内存碎片的复杂情况，只要移动堆顶指针，按顺序分配即可。


 


这样实现简单，运行高效。只是这种算法的代价是将内存缩小为原来的一半，浪费50%的内存空间。


![](https://p3-sign.toutiaoimg.com/tos-cn-i-6w9my0ksvp/c85295ff767c42c5a275e8c9f599bde1~tplv-obj.image?lk3s=ef143cfe&traceid=20241223202751264D64247DDBA21E5D75&x-expires=2147483647&x-signature=JjOCaU%2FqcMXCRnw7iPPnCbQ5i8I%3D)
**二.更优化的半区复制策略**


HotSpot虚拟机的Serial、ParNew等新生代收集器，就是采用这种策略的。具体就是把新生代分为一块较大的Eden空间和两块较小的Survivor空间，每次分配内存只使用Eden区和其中一块Survivor区。


 


进行垃圾收集时，将Eden和Survivor中存活的对象一次性复制到另外一块Survivor空间上，然后直接清理掉Eden区和已使用过的那块Survivor区的空间。


 


HotSpot虚拟机默认Eden和Survivor的大小比例是8 : 1，也就是每次新生代中可用内存空间为整个新生代空间的90%，只有一个Survivor空间(即10%的新生代空间)是会被浪费掉的。


 


此外，当Survivor空间不足以容纳一次Young GC之后存活的对象，就需要依赖其他内存区域(大多数是老年代)进行分配担保。


 


**三.标记\-复制算法总结**


标记\-复制算法在对象存活率较高时要进行较多的复制操作，效率会降低。更关键的是，如果不想浪费50%的空间，就要额外的空间进行分配担保，以应对被使用的内存中所有对象都100%存活的极端情况。所以在老年代一般不推荐选用这种算法。


 


**(3\)标记\-整理算法(Mark\-Compact)**


首先标记出所有需要回收的对象。在标记完成后，后续步骤不是直接对可回收对象进行清理。而是让所有存活的对象向一端移动，然后直接清理掉端边界外的内存。


 


标记\-清除算法和标记\-整理算法的本质差异在于：前者是一种非移动式的回收算法，而后者是一种移动式的回收算法。


![](https://p3-sign.toutiaoimg.com/tos-cn-i-6w9my0ksvp/e41842a614a44bafbb30b3e4d9897c40~tplv-obj.image?lk3s=ef143cfe&traceid=20241223202751264D64247DDBA21E5D75&x-expires=2147483647&x-signature=PFYA5KNHqZZ5YwjaLZw74wjFUSM%3D)
**(4\)是否移动回收后的存活对象分析**


**一.如果移动存活对象**


尤其是在老年代这种每次回收都有大量对象存活区域，移动存活对象并更新所有引用这些对象的地方将会负担极大，而且这种对象移动操作必须全程暂停用户应用程序才能进行(STW)。


 


**二.如果不移动存活对象**


比如像标记\-清除算法那样不考虑移动和整理存活对象的话，那么空间碎片化的问题就只能依赖更为复杂的内存分配器和内存访问器来解决。内存的访问是用户程序最频繁的操作，这个环节增加额外负担将影响应用程序的吞吐量。


 


可见是否移动回收后的存活对象都存在弊端：移动则内存回收时更复杂，不移动则内存分配时更复杂。


 


从垃圾收集的停顿时间来看，如果不移动对象，那么停顿时间更短甚至不需要停顿。但是从整个应用程序的吞吐量来看，移动对象会更划算。因为不移动对象虽然会使得收集器的效率提升了，但因内存分配和访问相比垃圾收集频率高得多，这部分的耗时增加后，总吞吐量仍然是下降的。


 


HotSpot虚拟机里关注吞吐量的Parallel Old收集器是基于标记\-整理算法，而关注延迟的CMS收集器则是基于标记\-清除算法。


 


**(5\)CMS收集器如何处理空间碎片过多**


和稀泥式不在内存分配和访问上增加太大额外负担，具体做法是：让虚拟机平时采用标记\-清除算法，暂时容忍内存碎片的存在。直到内存空间的碎片化程度已经大到影响对象分配时，再采用标记\-整理算法收集一次，以获得规整的内存空间。


 


**5\.垃圾收集器的设计**


**(1\)分代收集理论的3个假说**


**(2\)垃圾收集器的设计原则**


**(3\)新生代和老年代出现跨代引用的处理**


**(4\)分代收集算法**


 


**(1\)分代收集理论的3个假说**


一.弱分代假说：绝大多数对象都朝生夕灭


二.强分代假说：熬过多次GC过程的对象越难消亡


三.跨代引用假说：跨代引用相对于同代引用来说仅占极少数


 


**(2\)垃圾收集器的设计原则**


假说一和二奠定了垃圾收集器的设计原则。


 


收集器首先应该将Java堆划分出不同的区域，然后将回收对象依据其年龄分配到不同的区域之中存储。其中对象的年龄就是对象熬过垃圾收集过程的次数。


 


如果一个区域中大都是朝生夕灭的对象，则应以最低代价回收大量空间，即每次回收只须关注如何保留少量存活对象而非标记大量被回收的对象。


 


如果一个区域中大都是难以消亡的对象，则应以较低频率回收这个区域，同时兼顾垃圾收集的时间开销和内存利用效率。


 


现在商用Java虚拟机，一般把Java堆划分为新生代和老年代两个区域。新生代中，每次垃圾收集时都发现有大批对象死去，而新生代每次回收后存活的少量对象将会逐步晋升到老年代中存放。


 


**(3\)新生代和老年代出现跨代引用的处理**


分代收集并非简单划分一下内存区域这么容易，至少存在一个明显的问题：对象不是孤立的，对象之间会存在跨代引用。


 


假如要进行一次只局限于新生代区域内的收集(Young GC)，但新生代中的对象是完全有可能被老年代所引用的，为了找出该区域中的存活对象，不得不在固定的GC Roots之外，再额外遍历整个老年代中所有的对象来确保可达性分析结果的正确性。


 


遍历整个老年代所有对象的方案虽然理论可行，但无疑会为内存回收带来了很大的性能负担。


 


根据假说三，不应为少量的跨代引用去扫描整个老年代，也不必浪费空间专门记录每一个对象是否存在哪些跨代引用。只需在新生代上建立一个全局的数据结构(该结构被称为"记忆集")，记忆集会把老年代划分成若干小块，标识出那块内存会存在跨代引用。此后当发生Young GC时，包含了跨代引用的小块内存里的对象会被加入到GC Roots进行扫描。


 


**(4\)分代收集算法**


当前商业虚拟机的垃圾收集都采用分代收集算法，这种算法就是根据对象存活周期的不同将内存划分为几块。一般是把Java堆分为新生代和老年代，这样就可以根据各个年代的特点采用最适当的收集算法。


 


专门研究表明，新生代中的对象98%是朝生夕死的。所以并不需要按照1:1的比例来划分内存空间，而是将内存分为一块较大的Eden空间和两块较小的Survivor空间。每次使用的时候只用Eden和其中一块Survivor空间，回收的时候则将Eden和Survivor中存活的对象一次性复制到另一块Survivor，最后清理掉Eden和刚才用过的Survivor空间。


 


HotSpot虚拟机默认Eden和Survivor的大小比例是8:1，也就是每次新生代中可用内存空间为整个新生代容量的90%，只有10%的内存会被浪费。


 


当然，98%的对象可回收只是一般场景下的数据，我们没有办法保证每次回收都只有不多于10%的对象存活。当Survivor空间不够用时，需要依赖其他内存(老年代)进行分配担保。


 


在新生代中，每次垃圾收集时都发现有大批对象死去，只有少量存活。那就选标记\-复制算法，只需付出少量存活对象的复制成本就能完成收集。而老年代中因为对象存活率高、没有额外空间对它进行分配担保，就必须使用标记\-清除算法或者标记\-整理算法来进行回收。


 


新生代的回收：


Young GC(只发生在新生代的垃圾收集)。


 


老年代的回收：


Full GC(发生在整个Java堆和方法区的垃圾收集)。


 


新生代不断会有少量对象进入老年代，当老年代快填满时会发生Full GC。


 


**6\.垃圾回收器列表**


如何查看机器用的是那款收集器：



```
$ java -XX:+PrintCommandLineFlags -version
-XX:InitialHeapSize=268435456 -XX:MaxHeapSize=4294967296 
-XX:+PrintCommandLineFlags -XX:+UseCompressedClassPointers 
-XX:+UseCompressedOops -XX:+UseParallelGC
java version "1.8.0_202"
Java(TM) SE Runtime Environment (build 1.8.0_202-b08)
Java HotSpot(TM) 64-Bit Server VM (build 25.202-b08, mixed mode)
```

需要记住下图的垃圾收集器和之间的连线关系：


![](https://p3-sign.toutiaoimg.com/tos-cn-i-6w9my0ksvp/d0c46a6c7b624caebc955808a95abd16~tplv-obj.image?lk3s=ef143cfe&traceid=20241223202751264D64247DDBA21E5D75&x-expires=2147483647&x-signature=W6%2BDrA2rwmtu0LPqXQy5hRQ8S%2BY%3D)
并行：垃圾收集的多个线程的同时进行


并发：垃圾收集线程和应用线程同时进行


吞吐量 \= 运行用户代码时间 / (运行用户代码时间 \+ 垃圾收集时间)


垃圾收集时间 \= 垃圾回收频率 \* 单次垃圾回收时间


 


**垃圾回收器新生代列表：**


![](https://p3-sign.toutiaoimg.com/tos-cn-i-6w9my0ksvp/bd087404edf146bd8241064acc65fd3d~tplv-obj.image?lk3s=ef143cfe&traceid=20241223202751264D64247DDBA21E5D75&x-expires=2147483647&x-signature=dPfYc52xG1h73jLT14V8PKzhZe8%3D)
**垃圾回收器老年代列表：**


![](https://p3-sign.toutiaoimg.com/tos-cn-i-6w9my0ksvp/100191159a2c42578cb8381e5b363be2~tplv-obj.image?lk3s=ef143cfe&traceid=20241223202751264D64247DDBA21E5D75&x-expires=2147483647&x-signature=2eRxifzCHfEgllU%2FI%2BVQLhavqjo%3D)
 


**7\.各种垃圾回收器的详情**


**(1\)Serial/Serial Old收集器**


**(2\)ParNew收集器**


**(3\)Parallel Scavenge/Parallel Old收集器**


**(4\)Concurrent Mark Sweep(CMS)收集器**


**(5\)G1收集器**


**(6\)未来的垃圾回收**


 


**(1\)Serial/Serial Old收集器**


单线程，适合单CPU或CPU核心数少的服务器。



```
-XX:+UseSerialGC：表示新生代和老年代都用串行收集器
-XX:+UseParNewGC：表示新生代使用ParNew，老年代使用Serial Old
-XX:+UseParallelGC：表示新生代使用Parallel Scavenge，老年代使用Serial Old 
```

![](https://p3-sign.toutiaoimg.com/tos-cn-i-6w9my0ksvp/fb0ec5f058804c8e8d9bb3d1a755c24e~tplv-obj.image?lk3s=ef143cfe&traceid=20241223202751264D64247DDBA21E5D75&x-expires=2147483647&x-signature=ZliJQKFicswSQDdQOUTmyvI6wlw%3D)
Serial收集器依然是HotSpot运行在客户端模式下的默认新生代收集器。它有优于其他收集器的地方，即简单而高效(与其他收集器的单线程相比)。对于内存资源受限的环境，Serial是所有收集器里内存消耗最小的。对于单核处理器或处理器核心数较少的环境来说，Serial由于没有线程交互开销，专心做GC而获得最高的单线程收集效率。Serial收集器对于运行在客户端模式下的虚拟机来说是一个很好的选择。


 


**(2\)ParNew收集器**


ParNew收集器是Serial收集器的多线程并行版本。


 


和Serial收集器相比，基本没区别(回收策略、算法)，唯一的区别就是：多线程，适合于多CPU的，停顿时间比Serial少。


 


和Parallel Scavenge收集器相比，它关注的是尽可能缩短垃圾收集时用户线程的停顿时间，也就是关注停顿时间。停顿时间短的收集器适合用户交互的程序，以便于提高用户体验。


 


除了Serial收集器外，目前只有ParNew能与CMS收集器配合工作。自JDK9开始，ParNew\+CMS就不再是官方推荐的Server下的解决方案。官方希望被G1取代，甚至取消ParNew\+Serial Old和Serial\+CMS的支持。


 


参数\-XX:\+UseParNewGC


表示新生代使用ParNew，老年代使用Serial Old。


![](https://p3-sign.toutiaoimg.com/tos-cn-i-6w9my0ksvp/08706f045d374468aafc9b0d4851f104~tplv-obj.image?lk3s=ef143cfe&traceid=20241223202751264D64247DDBA21E5D75&x-expires=2147483647&x-signature=DhqBwPLxA2z29GznRG%2FA%2FkxkmUs%3D)
**(3\)Parallel Scavenge/Parallel Old收集器**


Parallel Scavenge收集器的特点是它的关注点与其他收集器不同。CMS等收集器的关注点是尽可能地缩短垃圾收集时用户线程的停顿时间，而Parallel Scavenge收集器的目标是尽可能地达到一个可控制的吞吐量。


 


所谓吞吐量就是CPU用于运行用户代码的时间与CPU总消耗时间的比值，即吞吐量\=运行用户代码时间/(运行用户代码时间\+垃圾收集时间)。虚拟机总共运行100分钟，其中垃圾收集花了1分钟，那吞吐量就是99%。


 


停顿时间越短越适合需要与用户交互或需要保证服务响应质量的程序，高吞吐量则可以高效率地利用CPU时间，尽快完成程序的运算任务，关注高吞吐量主要适合在后台运算而不需要太多交互的任务。


![](https://p3-sign.toutiaoimg.com/tos-cn-i-6w9my0ksvp/6c67dda66ebf49569d09012ad00067ee~tplv-obj.image?lk3s=ef143cfe&traceid=20241223202751264D64247DDBA21E5D75&x-expires=2147483647&x-signature=L%2FEy5Q3tKQfwwCHqI8rfYbkstRs%3D)
参数\-XX:\+UseParallerOldGC


表示新生代使用Parallel Scavenge，老年代使用Parallel Old。


 


参数\-XX:MaxGCPauseMills


参数允许的值是一个大于0的毫秒数，收集器将尽可能地保证内存回收花费的时间不超过设定值。这个参数的值并非设置得越小就能使系统的垃圾收集速度变得越快。GC停顿时间缩短是以牺牲吞吐量和新生代空间来换取的：系统把新生代调小一些，收集300MB新生代肯定比收集500MB快，这也直接导致垃圾收集发生得更频繁一些。原来10秒收集一次、每次停顿100毫秒，一分钟停顿600毫秒。现在5秒收集一次、每次停顿70毫秒，一分钟停顿840毫秒。每次的停顿时间的确在下降，但固定时间下的吞吐量也降下来了。


 


参数\-XX:GCTimeRatio


参数的值应当是一个大于0且小于100的整数，表示非垃圾收集时间与垃圾收集时间的比率。其中默认值为99，就是允许最大1% \= 1 / (1 \+ 99\) 的垃圾收集时间。如果设置为19，那允许的最大GC时间就占总时间的5% \= 1 / (1 \+ 19\)。


 


参数\-XX:\+UseAdaptiveSizePolicy


当这个参数打开后，就不需要手动指定新生代的大小(\-Xmn)、Eden与Survivor区的比例(\-XX:SurvivorRatio)、晋升老年代对象年龄(\-XX:PretenureSizeThreshold)等细节参数了，虚拟机会根据当前系统的运行情况收集性能监控信息，动态调整这些参数以提供最合适的停顿时间或者最大的吞吐量，这种调节方式称为GC自适应的调节策略。


 


如果对于收集器运作原来不太了解，手工优化存在困难的时候，使用Parallel Scavenge收集器配合自适应调节策略，把内存管理的调优任务交给虚拟机去完成将是一个不错的选择。


 


只需要把基本的内存数据设置好(如\-Xmx设置最大堆)，然后使用MaxGCPauseMillis或GCTimeRatio参数给虚拟机设立一个目标，那具体细节参数的调节工作就由虚拟机完成了。


 


其中MaxGCPauseMillis参数更关注最大停顿时间，而GCTimeRatio参数更关注吞吐量。


 


自适应调节策略也是Parallel Scavenge与ParNew的一个重要区别，JDK8中默认使用的是Parallel Scavenge与Parallel Old垃圾回收器。


 


**(4\)Concurrent Mark Sweep(CMS)收集器**


**一.CMS的阶段(初始标记 \+ 并发标记 \+ 重新标记 \+ 并发清除)**


**二.CMS的缺点(资源敏感 \+ 浮动垃圾 \+ 内存碎片)**


 


**一.CMS的阶段(初始标记 \+ 并发标记 \+ 重新标记 \+ 并发清除)**


CMS收集器是一种以获取最短回收停顿时间为目标的收集器。CMS收集器可以让系统停顿时间最短，给用户带来较好的体验。从名字Mark Sweep可看出，CMS收集器是基于标记\-清除算法实现的。它的运作过程相对于前面几种收集器来说更复杂，整个过程分4个阶段。


 


**阶段一：初始标记**


用户程序短暂暂停，仅标记GC Roots能直接关联到的对象，速度很快。


 


**阶段二：并发标记**


和用户程序同时进行，进行GC RootsTracing。


 


**阶段三：重新标记**


用户程序短暂暂停，为修正并发标记期间，因用户程序继续运作而导致标记产生变动的那一部分对象的标记记录，这个阶段的停顿时间一般比初始标记长，但远比并发标记的时间短。


 


**阶段四：并发清除**


在耗时最长的并发标记和并发清除阶段，GC线程都与用户线程一起工作，所以从总体上看，CMS收集器的内存回收过程与用户线程一起并发执行。


![](https://p3-sign.toutiaoimg.com/tos-cn-i-6w9my0ksvp/1321e74c92d642bc8778bc65b7f9969b~tplv-obj.image?lk3s=ef143cfe&traceid=20241223202751264D64247DDBA21E5D75&x-expires=2147483647&x-signature=nDYXB%2BlEHYIl6q2VnrFg8Ts4bgw%3D)
参数\-XX:\+UseConcMarkSweepGC


表示新生代使用ParNew，老年代使用CMS，浮动垃圾和内存碎片的处理需要用Serial Old收集器。


 


CMS收集器的主要优点：并发收集、低停顿。


CMS收集器有三个明显缺点：资源敏感、浮动垃圾、内存碎片。


 


**二.CMS的缺点(资源敏感 \+ 浮动垃圾 \+ 内存碎片)**


**缺点1：资源敏感**


在并发阶段，它虽然不会导致用户线程停顿，但却因为占用一部分线程，也就是处理器的计算能力，而导致应用程序变慢，降低总吞吐。


 


CMS默认启动的回收线程数：(CPU核数 \+ 3\) / 4。


 


如果CPU核心数在4个以上，那么并发回收时，GC线程数只占不超过25%的CPU运算资源，并且会随着CPU核心数量的增加而下降。


 


如果CPU核心数不足4个，CMS对用户程序的影响就可能就变得很大。比如应用原来的CPU负载很高，还要分一半的运算能力去处理GC线程，那么就可能导致用户程序的执行速度忽然大幅降低。


 


**缺点2：浮动垃圾**


由于并发清理时用户线程还在运行，所以还会有新的垃圾不断产生。这一部分垃圾出现在标记过程后，这些垃圾就称为浮动垃圾。CMS无法在当次收集中处理掉它们，只好等下次GC时再清理掉。同时用户的线程还在运行，要给用户线程留下运行的内存空间。


 


参数\-XX:CMSInitiatingOccupancyFraction


由于浮动垃圾和需要留内存给运行着的用户线程，因此CMS不能像其他收集器那样等老年代几乎完全被填满了再进行收集，需要预留一部分空间提供并发收集时的程序运作使用。


 


早期JDK默认下，CMS收集器当老年代使用了68%的空间后就会被激活。如果在应用中老年代增长不是太快，可以适当调高该CMS的激活阈值，以便降低内存回收次数从而获取更好的性能。


 


在JDK1\.6中，CMS收集器的启动阈值已经提升至92%。要是CMS运行期间预留的内存无法满足程序需要，就会出现一次Concurrent Mode Failure，这时虚拟机将启动后备预案，临时启用Serial Old收集器进行老年代的GC，这样停顿时间就很长了。


 


如果\-XX:CMSInitiatingOccupancyFraction设置得太高，就会很容易导致出现大量Concurrent Mode Failure，性能反而降低。


 


**缺点3：内存碎片**


由于CMS使用标记\-清除算法，因此会产生大量空间碎片。空间碎片过多时，分配大对象就会出现问题而提前触发Full GC。


 


参数\-XX:\+UseCMSCompactAtFullCollection


CMS提供这个开关参数(默认开启)便是为了解决内存碎片问题，用于在CMS顶不住要进行Full GC时开启内存碎片的合并整理过程。内存整理的过程是无法并发的，空间碎片问题没有了，但停顿时间更长。


 


这个参数用于设置执行多少次不压缩Full GC后，跟着来一次带压缩的。默认值0，表示每次进入Full GC时都进行碎片整理。


 


**(5\)G1收集器**


**一.G1收集器的特点**


**二.G1收集器的并行与并发**


**三.分代收集与内存布局\-建立停顿时间模型**


**四.G1使用Region划分内存空间要解决的问题**


**五.G1使用的算法(标记整理\+标记复制)**


**六.G1收集垃圾的三个阶段(新生代GC \+ 并发标记 \+ 混合回收)**


**七.G1收集器中重要的参数**


 


**一.G1收集器的特点**


参数\-XX:\+UseG1GC


特点：并行和并发、分代收集、空间整合、没有空间碎片、可预测停顿。整体上看是标记整理算法，局部上看是标记复制算法。


 


经验来说，目前在小内存应用上CMS的表现大概率仍然优于G1，而在大内存应用上G1则能发挥其优势，这个内存平衡点是6～8G之间。


 


**二.G1收集器的并行与并发**


并行与并发：G1能充分利用多CPU、多核环境下的硬件优势。G1可以使用多个CPU(CPU或者CPU核心)来缩短STW停顿的时间。部分其他收集器原本需要停顿Java线程执行的GC动作，G1收集器仍然可以通过并发的方式让Java程序继续执行。


 


**三.分代收集与内存布局\-建立停顿时间模型**


G1之前的垃圾收集器的目标范围要么是整个新生代(Young GC)，要么就是整个老年代(Old GC)，要么就是整个Java堆(Full GC)。


 


G1则可以面向堆内存任何部分来组成回收集进行回收，衡量标准不再是哪个分代，而是哪块内存的垃圾数量最多回收收益最大，这也是G1收集器的Mixed GC模式。


 


G1也遵循分代收集理论，但它不再坚持固定大小和数量的分代区域划分，而是把连续的Java堆划分为多个大小相等的独立区域(Region)。每个Region可根据需要扮演新生代的Eden区、Survivor区或老年代区，然后G1再对不同类型的Region采用不同的策略处理。


 


虽然G1保留了新生代和老年代概念，但是新生代和老年代却不再固定，它们都是一系列区域(不需要连续)的动态集合。


 


G1之所以能建立可预测的停顿时间模型，是因为它将Region作为单次回收的最小单元，每次收集到的内存空间都是Region大小的整数倍，这样可以有计划避免在整个Java堆中进行全区域的垃圾收集。


 


G1收集器会跟踪各个Region区里面的垃圾"价值"大小，价值即回收所获得的空间大小以及回收所需时间的经验值。然后G1收集器会维护一个优先级列表，每次根据用户设定的收集停顿时间，优先处理回收价值大的那些Region。


 


使用Region划分内存空间(化整为零)，通过优先级进行Region区域回收，保证了G1收集器在有限时间内获取尽可能高的收集效率。


 


**四.G1使用Region划分内存空间要解决的问题**


**问题1：跨Region引用对象如何解决?**


使用记忆集避免全堆作为GC Roots扫描，每个Region都有自己的记忆集。这些记忆集会记录下别的Region指向自己的指针，并标记这些指针分别在哪些卡页的范围之内。


 


由于Region数量比传统垃圾收集器的分代数量多得多，因此G1收集器要比其他传统垃圾收集器有着更高的内存负担，G1至少要耗费大约Java堆容量10%至20%的额外内存来维持收集器工作。


 


**问题2：并发标记阶段如何保证收集线程与用户线程互不干扰?**


用户线程改变对象引用关系时，要保证不能打破原本的对象图结构，导致标记结果出现错误。为此，CMS收集器采用增量更新算法来实现，而G1收集器则通过原始快照(SATB)算法来实现。


 


**问题3：怎样建立可靠的停顿预测模型?**


G1收集器的停顿预测模型是以衰减均值为理论基础实现的。在垃圾收集过程中，G1收集器会记录每个Region的回收耗时、每个Region记忆集里的脏卡数量等各个可测量的步骤花费的成本，并分析出平均值、标准偏差、置信度等统计信息。然后由这些信息预测：如果现在开始回收，则哪些Region组成回收集能在不超期望停顿时间的约束下获得最高收益。


 


**五.G1使用的算法(标记整理\+标记复制)**


与CMS的标记\-清理算法不同，G1从整体上来看是基于标记\-整理算法实现的收集器，从局部(两个Region之间)上来看是基于标记\-复制算法实现的。但这两种算法都不产生内存空间碎片，收集后能提供规整的可用内存。从而有利于程序长时间运行，有利于分配大对象。因为分配大对象时不会因无法找到连续内存空间而提前触发下一次GC。


 


**六.G1收集垃圾的三个阶段(新生代GC \+ 并发标记 \+ 混合回收)**


**阶段一：新生代GC**


此时会回收Eden区和Survivor区。回收后所有Eden区被清空，存在一个Survivor区保存了部分数据。老年代区域会增多，因为部分新生代的对象会晋升到老年代。


 


**阶段二：并发标记**


初始标记：短暂停顿，仅仅只是标记一下GC Roots能直接关联到的对象。初始标记的速度很快，产生一个全局停顿，都伴随有一次新生代的GC。


根区域扫描：扫描从Survivor区可以直接到达的老年代区域。


并发标记：扫描和查找整个堆的存活对象，并标记。


重新标记：会产生全局停顿，对并发标记阶段的结果进行修正。


独占清理：会产生全局停顿，对GC回收比例排序，供混合收集阶段使用。


并发清理：识别并清理完全空闲的Region区域，并发进行。


![](https://p3-sign.toutiaoimg.com/tos-cn-i-6w9my0ksvp/d58a3db0e53246cf9aec144a87638987~tplv-obj.image?lk3s=ef143cfe&traceid=20241223202751264D64247DDBA21E5D75&x-expires=2147483647&x-signature=y83tgrIdYFAJIzHNzeDVH8FDoNw%3D)
**阶段3：混合回收**


对含有垃圾比例较高、回收收益较大的Region区域进行回收，G1当出现内存不足的的情况，也可能进行的Full GC回收。


 


**七.G1收集器中重要的参数**


\-XX:MaxGCPauseMillis：指定目标的最大停顿时间，G1会调整新生代和老年代的比例、堆大小、晋升年龄来达到该目标时间。


\-XX:ParallerGCThreads：设置GC的工作线程数。


 


**(6\)未来的垃圾回收**


衡量垃圾收集器的三项指标：内存占用、吞吐量、延迟(停顿时间)。


 


JDK11中的ZGC：一种可扩展的低延迟垃圾收集器，具备以下特点：


一.处理TB量级的堆


二.GC时间不超过10ms


三.与使用G1相比，应用吞吐量的降低不超过15%


 


ZGC通过技术手段把STW的情况控制在仅有一次，就是第一次的初始标记才会发生。这样也就不难理解为什么GC停顿时间不随着堆增大而上升了，因为再大也是通过并发的时间去回收了。关键技术：有色指针(Colored Pointers)、加载屏障(Load Barrier)。


 


**8\.Stop The World现象**


Stop The World是无法避免的。GC收集器和GC调优的目标就是尽可能减少STW的时间和次数。


 


在调优或者在设置堆大小的时候，主要的努力方向就是让这些GC回收尽量发生在新生代，老年代以及永久代都不要进行垃圾回收。


 


下面是验证Stop The World的代码：



```
//一个线程定时打印时间用作模拟用户线程
//一个线程用来不停填充数据用作模拟产生Full GC
//观察对比打印的时间间隔是不是一直处于100ms, 尤其是发生GC的前后两次打印
//运行前配置VM Options:
//-Xms300M -Xms300M -XX:+UseSerialGC -XX:+PrintGCDetails
public class StopWorld {
    //不停往list中填充数据
    public static class FillListThread extends Thread {
        List<byte[]> list = new LinkedList<>();
        @Override
        public void run() {
            try {
                while(true) {
                    //当list到达990M时清空list
                    if (list.size() * 512 / 1024 / 1024 >= 990) {
                        list.clear();
                        System.out.println("list is clear");
                    }
                    byte[] bl;
                    for (int i=0;i<100;i++) {
                        bl = new byte[2 * 1024 * 1024];
                        list.add(bl);
                    }
                    Thread.sleep(1);
                }
            } catch (Exception e) {
            }
        }
    }

    //每100ms定时打印
    public static class TimerThread extends Thread {
        public final static long startTime = System.currentTimeMillis();
        @Override
        public void run() {
            try {
                while(true) {
                    long t =  System.currentTimeMillis() - startTime;
                    System.out.println(t / 1000 + "." + t % 1000);
                    Thread.sleep(100);
                }
            } catch (Exception e) {
            }
        }
    }

    //从输出结果中可见, 在发生GC的前后两次时间打印是大于100ms的, 
    //而其他时候的两次打印之间相差基本是100ms, 从而验证了GC会StopTheWorld
    //部分输出结果如下:
    //...
    //2.264
    //2.367
    //2.467
    //2.571
    //[GC (Allocation Failure) [...] [Times: user=0.24 sys=0.19, real=0.79 secs] (这次gc用了790ms)
    //3.410
    //...
    public static void main(String[] args) {
        FillListThread myThread = new FillListThread();
        TimerThread timerThread = new TimerThread();
        myThread.start();
        timerThread.start();
    }
}
```

 


**9\.内存分配与回收策略**


**(1\)对象优先在Eden区分配**


**(2\)大对象直接进入老年代**


**(3\)长期存活的对象将进入老年代**


**(4\)动态对象年龄判定**


**(5\)空间分配担保机制**


 


**(1\)对象优先在Eden区分配**


如果Eden内存空间不足，就会发生一次Minor GC(Yong GC)。


 


**(2\)大对象直接进入老年代**


大对象就是需要大量连续内存空间的Java对象，比如很长的字符串和大型数组。


 


经常出现大对象的坏处是：


一.虽然内存有空间，但还是要提前进行垃圾回收获取连续空间。


二.在新生代存活的大对象需进行大量的内存复制，影响Young GC时间。


 


参数：\-XX:PretenureSizeThreshold


一个对象如果其大小大于这个参数阈值，则直接在老年代分配。默认为0 ，表示不会直接分配在老年代。


 


**(3\)长期存活的对象将进入老年代**


通过参数\-XX:MaxTenuringThreshold调整，默认超过15岁的对象直接进入老年代。


 


**(4\)动态对象年龄判定**


为了能更好地适应不同程序的内存状况，对象的年龄并非必须达到了MaxTenuringThreshold才能晋升老年代。如果在Survivor中相同年龄所有对象大小的总和大于Survivor空间的一半，那么Survivor中年龄大于或等于该年龄的对象就可以直接进入老年代，无须等到MaxTenuringThreshold中要求的年龄。


 


**(5\)空间分配担保机制**


新生代中有大量的对象存活，但是Survivor空间不够，也就是出现大量对象在Young GC后仍然存活的情况，比如最极端的情况就是内存回收后新生代中所有对象都存活。此时就要老年代进行分配担保，把Survivor无法容纳的对象进入老年代。只要老年代的空闲空间大于新生代对象的总大小或历次晋升的平均大小，就进行Young GC，否则就进行Full GC。


 


**10\.新生代不同配置演示**


新生代大小配置参数的优先级：


高：\-XX:NewSize/MaxNewSize


中：\-Xmn(NewSize\= MaxNewSize)


低：\-XX:NewRatio，新生代和老年代比例


 


参数：\-XX:SurvivorRatio，表示Eden和Survivor的比值。默认为8，表示Eden : FromSurvivor : ToSurvivor \= 8 : 1 : 1。


 


如下同样代码的情况下，不同的新生代配置运行后的GC情况：



```
//存10M的数据
public class NewSize {
    public static void main(String[] args) {
        int cap = 1 * 1024 * 1024;//1M
        byte[] b1 = new byte[cap];
        byte[] b2 = new byte[cap];
        byte[] b3 = new byte[cap];
        byte[] b4 = new byte[cap];
        byte[] b5 = new byte[cap];
        byte[] b6 = new byte[cap];
        byte[] b7 = new byte[cap];
        byte[] b8 = new byte[cap];
        byte[] b9 = new byte[cap];
        byte[] b0 = new byte[cap];
    }
}
```

一.如果堆设为20M，新生代设为2M，Eden : Survivor设为2 : 1



```
$ -Xms20M -Xmx20M -XX:+PrintGCDetails –Xmn2m -XX:SurvivorRatio=2
```

结果：没有垃圾回收，数组都在老年代。


 


二.如果堆设为20M，新生代设为7M，Eden : Survivor设为2 : 1



```
$ -Xms20M -Xmx20M -XX:+PrintGCDetails -Xmn7m -XX:SurvivorRatio=2
```

结果：Survivor放得下一个数组，发生了Young GC垃圾回收。新生代存了部分数组，老年代也保存了部分数组，发生了对象晋升。


 


三.如果堆设为20M，新生代设为15M，Eden : Survivor设为8 : 1



```
$ -Xms20M -Xmx20M -XX:+PrintGCDetails -Xmn15m -XX:SurvivorRatio=8
```

结果：新生代可以放下所有的数组，老年代没放。


 


四.如果堆设为20M，新生代 : 老年代设为1 : 2



```
$ -Xms20M -Xmx20M -XX:+PrintGCDetails -XX:NewRatio=2
```

结果：首先发生了Young GC，然后Eden区5632k，Survivor区512K，放不下一个数组。于是出现了空间分配担保，而且发生了Full GC，因为出现了老年代的连续空间小于新生代对象的总大小。


 


**11\.内存泄漏和内存溢出**


表现都是OOM异常，但两者的本质是不同的。


内存溢出：是实实在在的内存空间不足导致。


内存泄漏：该释放的对象没有释放，多见于使用容器保存元素的场景。


 


内存泄露的代码示例：



```
public class Stack {
    public  Object[] elements;
    private int size = 0;//指示器，指示当前栈顶的位置
    private static final int Cap = 16;

    public Stack() {
        elements = new Object[Cap];
    }

    //入栈
    public void push(Object e) {
        elements[size] = e;
        size++;
    }

    //出栈
    public Object pop() {
        size = size  -1;
        Object o = elements[size];
        //elements[size] = null;
        return o;
    }

    public static void main(String[] args) {
        Stack stack = new Stack();
        Object o = new Object();
        System.out.println("o="+o);
        stack.push(o);
        Object o1 =  stack.pop();
        System.out.println("o1="+o1);
        
        //元素明明出了栈，但实际输出还是存在的, 说明内存还占着，为了避免这种问题，需要把elements[0]在出栈的时候改为null
        System.out.println(stack.elements[0]);
    }
}
```

 


**12\.JDK为提供的工具**


**(1\)jps**


虚拟机进程状况工具，列出当前机器上正在运行的虚拟机进程。


\-m：输出主函数传入的参数；


\-l：输出应用程序主类完整名称；


\-v：列出启动时jvm参数；


 


**(2\)jstat**


虚拟机统计信息监视工具，用于监视JVM运行状态信息的命令行工具。它可显示JVM进程中的类装载、内存、垃圾收集、JIT编译等运行数据，是运行期定位虚拟机性能问题的首选工具。


 


假设需要每250毫秒查询一次进程2764的GC状况，一共查询20次，那么命令应当是：jstat \-gc 2764 250 20


 


假设需要每250毫秒查询一次进程2764的新生代状况，一共查询20次，那么命令应当是：jstat \-gcnew 2764 250 20


 


常用参数：



```
jstat -class (类加载器) 
jstat -compiler (JIT) 
jstat -gc (GC堆状态) 
jstat -gccapacity (各区大小) 
jstat -gccause (最近一次GC统计和原因) 
jstat -gcnew (新生代统计)
jstat -gcnewcapacity (新生代大小)
jstat -gcold (老年代统计)
jstat -gcoldcapacity (老年代大小)
jstat -gcpermcapacity (永久代大小)
jstat -gcutil (GC统计汇总)
jstat -printcompilation (HotSpot编译统计)
```

**(3\)jinfo**


Java配置信息工具，查看和修改虚拟机的参数。



```
jinfo –sysprops 可以查看由System.getProperties()取得的参数
jinfo –flag 未被显式指定的参数的系统默认值
jinfo –flags（注意s）显示虚拟机的参数
jinfo –flag +[参数] 可以增加参数，但是仅限于由java -XX:+PrintFlagsFinal –version查询出来且
```

例如查看打印GC的设置情况：



```
$ jinfo -flag PrintGCDetails 11077
-XX:-PrintGCDetails
```

列出当前Java虚拟机提供的参数：



```
$ java -XX:+PrintFlagsFinal -version
可以进行修改的只能为manageable的参数
jinfo –flag -[参数] 可以去除参数
```

**(4\)jmap**


Java内存映像工具，用于生成堆转储快照(heapdump或dump文件)。jmap的作用并不仅仅是为了获取dump文件，jmap还可以查询finalize执行队列、Java堆和永久代的详细信息，如空间使用率、当前用的是哪种收集器等。


 


和jinfo命令一样，jmap有不少功能在Windows平台下都是受限的，除了生成dump文件的\-dump选项和用于查看每个类的实例、空间占用统计的\-histo选项在所有操作系统都提供之外，其余选项都只能在Linux/Solaris下使用。



```
$ jmap -dump:live,format=b,file=heap.bin 
```

jhat命令与jmap搭配使用，可以分析jmap生成的堆转储快照。


 


使用示例：先在Idea启动StopWorld程序，然后通过jps \-v找到进程号。



```
$ jps -v
11092 StopWorld -Xms300M -Xms300M -XX:+UseSerialGC -XX:+PrintGCDetails -javaagent:/Applications/IntelliJ/IDEA.app/Contents/..
```

获取生成堆转储快照文件，也就是dump文件：



```
$ jmap -dump:live,format=b,file=heap.bin 11092
Dumping heap to /Users/demo/heap.bin ...
Heap dump file created
```

**(5\)jhat(不建议在服务器上分析)**


虚拟机堆转储快照分析工具，jhat dump文件名。对上面jmap得出的堆转储快照文件进行分析：



```
$ jhat /Users/demo/Downloads/heap.bin
Reading from /Users/demo/Downloads/heap.bin...
Dump file created Fri Jan 07 11:30:23 CST 2022
Snapshot read, resolving...
Resolving 5809 objects...
Chasing references, expect 1 dots.
Eliminating duplicate references.
Snapshot resolved.
Started HTTP server on port 7000
Server is ready.
```

屏幕显示"Server is ready."的提示后，在浏览器中键入http://localhost:7000/就可以访问详情。


 


**(6\)jstack**


Java堆栈跟踪工具，该命令用于生成虚拟机当前时刻的线程快照，线程快照就是当前虚拟机内每一条线程正在执行的方法堆栈的集合。


 


生成线程快照的主要目的是定位线程出现长时间停顿的原因，如线程间死锁、死循环、请求外部资源导致长时间等待等，这些都是导致线程长时间停顿的常见原因。


 


在代码中可以用java.lang.Thread类的getAllStackTraces()方法，可以用于获取虚拟机中所有线程的StackTraceElement对象。使用这个方法可以通过简单的几行代码就完成jstack的大部分功能，所以可以用这个方法做个管理页面，随时使用浏览器查看线程堆栈。


 


**(7\)JConsole和VisualVM**


管理远程进程需要在远程程序的启动参数中增加：



```
$ -Djava.rmi.server.hostname=….
$ -Dcom.sun.management.jmxremote
$ -Dcom.sun.management.jmxremote.port=8888
$ -Dcom.sun.management.jmxremote.authenticate=false
$ -Dcom.sun.management.jmxremote.ssl=false
```

**(8\)MAT: 查找内存泄露和溢出**


首先把OOM发生时的文件dump下来：\-XX:\+HeapDumpOnOutOfMemoryError。然后使用MAT需要注意浅堆和深堆。


 


浅堆：是指一个对象所消耗的内存，这个对象本身所占用的内存大小。在32位系统中，一个对象引用会占4个字节，一个int类型会占4个字节。一个long型变量会占8个字节，每个对象头需要占用8个字节。


 


深堆：这个对象被GC回收后，可以真实释放的内存大小，也就是只能通过对象被直接或间接访问到的所有对象的集合。通俗地说，就是指仅被对象所持有的对象的集合，深堆是指对象的保留集中所有的对象的浅堆大小之和。


 


举例：对象A引用了C和D，对象B引用了C和E。那么对象A的浅堆大小只是A本身，不含C和D。而A的实际大小为A、C、D三者之和，A的深堆大小为A与D对象之和。由于对象C还可以通过对象B访问到，因此不在对象A的深堆范围内。


 


 本博客参考[蓝猫加速器配置下载](https://yunbeijia.com)。转载请注明出处！
