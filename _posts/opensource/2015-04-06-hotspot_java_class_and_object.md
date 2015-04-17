---
layout: post
title: Hotspot之Java类与Java对象 （二）
category:  源码阅读
tags: [hotspot, openjdk7, java, opensource]
---

本文主要介绍一下Java的类和对象在Hotspot中的表示。

本文目录

* 目录
{:toc}


### Java类与Java对象放在哪？
Hotspot由内存管理子系统、类装载系统、执行引擎和本地方法接口几大部分组成。在内存管理子系统中，内存被划分为方法区、堆、Java栈和本地方法栈几个部分。其中，Java栈和本地方法栈是用于方法调用，每当调用一个Java方法或者本地方法时，Hotspot需要在栈空间划分一个调用栈帧的空间用于函数调用；堆用于放置Java对象，在Java中，所有的对象都是在堆空间中申请，所有的Java对象为垃圾管理系统所管理，自动回收；方法区用于放置Java对象的元数据，即对象所属的类型相关信息，比如Java类的静态变量、成员方法、数据成员等信息。方法区和堆在操作系统角度看，其实都属于堆空间，其区分是依据逻辑进行划分。垃圾收集系统也对这些不同的区域采用不同的收集方式。因此，**Java类相关的信息放置在方法区，而Java对象放置在堆空间**。

在下图中，当需要new一个Java对象前，虚拟机首先要判断此Java类是否被载入内存了？如果不是，那么首先要使用类加载器将此类加载进方法区；载入内存的类是否已经进行`连接`与`初始化`了？如果没有，那么此Java类需要进行了连接和初始化后，此Java类才可以使用。在建立Java对象时，根据在类加载阶段，计算出来的对象的大小，在堆空间申请一段内存；然后，赋初值，即将这里的Java对象和Java对象所属的类联系起来。最后，把这段内存以InstanceOop对象指针的方式返回。（InstanceOop是Hotspot中的类，代表Java对象的是InstanceOop对象，图中的klassOop和instanceKlass组合后代表Java类，下文具体讲）
![Java类与对象在哪？]({{ site.BASE_PATH }}/work_images/hotspot/hotspot_class-object_where.jpg)

### Java对象如何表示?
下图是Hotspot中一些核心类的继承关系。oopDesc是所有类的父类，instanceOopDesc表示Java对象，arrayOopDesc表示Java数组，klassOopDesc表示Java类。

        oopDesc
        |-- instanceOopDesc
        |-- arrayOopDesc
        |   |-- objArrayOopDesc
        |   `-- typeArrayOopDesc
        |-- klassOopDesc


对于Java对象而言，虚拟机是使用instanceOopDesc来表示Java对象的相关信息（在Hotspot中，去除Desc的是相应类的指针类型，比如类型instanceOop是instanceOopDesc的指针类型：` class instanceOopDesc*  instanceOop`）。oopDesc是Java对象和Java数组类对象的父类，也叫**对象头**（oop header），下面是其定义：

        class oopDesc {
            private:
            volatile markOop  _mark;
            union _metadata {
                wideKlassOop    _klass;
                narrowOop       _compressed_klass;
            } _metadata;
        }

**对象头**有两个数据成员。一个是_mark(markOop类型)数据成员，其包含的信息有：与对象相关的锁、供垃圾回收机制所使用的年龄、哈希码等信息。此字段只有一个指针的长度，比如在64位机器上，此字段的长度只有8字节，所以这些信息是根据需要而存储，而非在同一时间包括所有的信息；另一个数据成员是_metadata(此变量为union类型，主要用于是否对指向对象的指针进行压缩处理)。_metadata指向此类对象所属的类。比如，如果此类对象是Java对象或Java数组对象，那么此字段指向的就是此Java对象或Java数组对象所属的Java类或Java数组类。

Hotspot使用instanceOopDesc表示一个Java对象。instanceOopDesc对象包括两部分，一部分是继承至oopDesc类的数据成员(_mark和_metadata)；另一部分是真正的Java类的数据成员，其组成包括：继承至父类的数据成员和自身的数据成员，以及为了对齐而产生的填充数据，具体如下图所示：
![Java对象结构]({{ site.BASE_PATH }}/work_images/hotspot/hotspot_class-object_instanceOop.jpg)

**Java数组对象**。Java数组对象有包含**基本类型的Java数组对象**，比如int数组、boolean数组、char数组等等，Hotspot使用数据类型typeArrayOopDesc表示。当新建一个基本类型数组对象时，虚拟机在内部返回给应用的就是typeArrayOopDesc类型的对象；另一种数组对象是**引用类型的数组对象**，用objArrayOopDesc类表示。当新建一个引用类型的数组时，虚拟机在内部返回给应用的就是objArrayOopDesc类型的对象。Java数组对象除了oopDesc的数据成员外，还有一个字段表示数组的长度。接着的是存放数组元素的空间及可能的填充数据，具体如下图所示：
![Java数组对象结构]({{ site.BASE_PATH }}/work_images/hotspot/hotspot_class-object_objArrayOop.jpg)


### Java类如何表示?
        Klass_vtbl
            Klass
            |-- instanceKlass
            |   |-- instanceMirrorKlass
            |   `-- instanceRefKlass
            |-- arrayKlass
            |   |-- objArrayKlass
            |   `-- typeArrayKlass
            |-- klassKlass
            |   |-- instanceKlassKlass
            |   `-- arrayKlassKlass
            |        |-- objArrayKlassKlass
            |        `-- typeArrayKlassKlass


如上图所示，所有的Klass都继承至Klass_vtbl类，其子类有代表Java类一部分内容的instanceKlass，代表Java数组类的arrayKlass，代表常量池的constantPoolKlass子类，代表方法的methodKlass子类等等。在这些类中拥有virtual函数，故而这些类具备运行时动态分发的功能（这里的动态分发指的是C++的多态。C++类要具备多态，其类至少有一个函数是虚函数，具体可参看候捷翻译的《深入理解C++面向对象机制》一书）。
在上文说到：Java类不仅仅由klassOopDesc表示，那么Java类到底由什么来表示？答案是表示Java类的是一个**组合对象**。这个组合对象由两部分组成：一部分是klassOopDesc的数据成员，另一部分是instanceKlass的数据成员。当一个代表Java类的二进制代码经过解析后，虚拟机首先在方法区中分配足够包括这两部分的内存空间并进行合适的赋初始值工作，并将此对象以klassOopDesc指针的形式返回。换句话说，表示Java类的既是klassOopDesc对象，也是一个instanceKlass对象。在使用过程中，一般是作为klassOopDesc的形式进行使用，但需要使用instanceKlass的数据成员时，就在此指针的基础上加上一个固定的偏移，就可以得到表示instanceKlass的指针。在64位机器上，这个固定的偏移为16字节的长度（即使打开压缩指针存储，但因为内存对齐的需求，也需要跨越16字节的长度，才能得到表示instanceKlass的指针）具体如下图所示，klassOopDesc只有继承至oopDesc的数据成员，即在64位机器上是两个指针的长度，为16字节长度。
![Java数组对象结构]({{ site.BASE_PATH }}/work_images/hotspot/hotspot_class-object_instanceKlass.jpg)

**总结**：当一个新的Java对象需要建立时，首先将代表此类型的二进制数据载入，在方法区建立代表此Java类的组合对象--klassOop和instanceKlass的组合，然后通过运行时的相关函数，从堆空间为此Java类对象分配空间并返还给应用；Java数组对象的建立，首先需要确保数组的元素所代表的类载入，然后需要保证此类所继承的父类、父接口的数组要在方法区存在，如果不存在，首先要在方法区建立之。比如在一个继承体系中，整数类Integer继承至Number类，Number类继承Object类，那么如果建立Integer的Java数组，首先需要建立Number数组类，然后是Object数组类，Object[]的父类为Object类；然后，在方法区为本数组类分配空间并初始化；最后，当相应的Java数组类建立后，Hotspot在堆内存中为Java数组对象分配空间。

**Klass继承体系在运行时的关系图**。在运行时，每一个Java对象或Java数组对象中的Klass分别指向代表Java类的instanceKlass和代表Java数组类的objArrayKlass（引用类型）/typeArrayKlass（基本类型）。而instanceKlass的Klass指向instanceKlassKlass，这个instanceKlassKlass和klassKlass的类对象是在初始化时建立的，属于全局对象；下图有表示Java对象的instanceOop，表示Java类的instanceKlass，表示Java数组对象的objArrayOop/typeArrayOop，表示Java数组类的objArrayKlass/typeArrayKlass，这几个对象都是运行时建立的。具体如下所示：
![Hotspot之运行时核心数据结构图]({{ site.BASE_PATH }}/work_images/hotspot/hotspot_class-object_core_structure_relation.jpg)
上图和[hotspot-docs-fosdem-2007](http://openjdk.java.net/groups/hotspot/docs/FOSDEM-2007-HotSpot.pdf "Klass继承体系在运行时的关系图")稍有不同，本图是根据最新的代码关系所绘制。


### 举例
下面使用实际的事例加以说明。这个例子是一个简单的面向对象的继承结构，父类为Shape，是一个抽象类，三个具体的子类Triangle、Ellipse、Square继承至父类Shape。继承结构如下图所示：
![example_java]({{ site.BASE_PATH }}/work_images/hotspot/hotspot_class-object_example_java.jpg) 
在运行时，当应用需要访问这些类时，Hotspot需要首先解析这些类，并建立instancKlass实例对象表示这个Java类。这个instancKlass实例对象在“对象头”使其指向在初始化时期建立的instanceKlassKlass实例。其它的Java类，比如Integer、String等所有的Java类所建立的instanceKlass实例都指向这个instanceKlassKlass实例，如下图所示。建立的Java对像在其“对象头”处指向所属的Java类，即相应的instanceKlass对象。
![example_java_hotspot]({{ site.BASE_PATH }}/work_images/hotspot/hotspot_class-object_example_java_hotspot.jpg) 

{::comment}
### HSDB
HSDB是一个随着Java安装包进行分发的一个工具。利用此工具可以查看很多虚拟机层的信息。

1. HSDB启动
    +  启动jdb让程序停在想停的位置处，如下所示：
    
                D:\test>jdb -XX:+UseSerialGC 
                Initializing jdb ...
                > stop in Test.fn
                > run Test
    
    +  启动另一个窗口，使用jps查看进程的pid.
    + 启动HSDB： `java -cp .;%JAVA_HOME%/lib/sa-jdi.jar sun.jvm.hotspot.HSDB`
    + 启动HSDB之后，把它连接到目标进程上。从菜单里选择File -> Attach to HotSpot process
 
2. HSDB验证Java对象和Java类
{:/comment}