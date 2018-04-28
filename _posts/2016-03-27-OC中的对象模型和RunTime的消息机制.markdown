---
layout: post
title: OC中的对象模型和RunTime的消息机制
date: 2016-02-16 15:32:24.000000000 +09:00
---

### 1.ISA指针
OC是一门面向对象的编程语言,每一个对象都是一个类的一个实例;在 Objective-C 语言的底层定义中，每一个对象都有一个名为 isa 的指针，指向该对象的类。每一个类描述了一系列它的实例特点，包括成员变量，成员方法等。每一个对象都可以接受消息，而对象能够接收的消息列表是保存在它所对应的类中。
![ISA内部结构.png](http://upload-images.jianshu.io/upload_images/1867963-5c2fdb116ccd9370.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)
### 2.类对象的结构
按照面向对象语言的设计原则，所有事物都应该是对象（严格来说 Objective-C 并没有完全做到这一点，因为它有 int, double 这样的简单变量类型）。在 Objective-C 语言中，每一个类实际上也是一个对象(类对象)。每一个类也有一个名为 isa 的指针。每一个类也可以接受消息.例如[NSObject new]，就是向 NSObject 这个类发送名为new消息;
![类结构体.png](http://upload-images.jianshu.io/upload_images/1867963-6d4e270fc7fb7f03.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)
因为类也是一个对象，那它也必须是另一个类的实现，这个类就是元类 (metaclass)。元类保存了类方法和属性的列表。当一个类方法被调用时，元类会首先查找它本身是否有该类方法的实现，如果没有，则该元类会向它的父类查找该方法，直到一直找到继承链的头。
元类 (metaclass) 也是一个对象，那么元类的 isa 指针又指向哪里呢？为了设计上的完整，所有的元类的 isa 指针都会指向一个根元类 (root metaclass)。根元类 (root metaclass) 本身的 isa 指针指向自己，这样就行成了一个闭环。上面提到，一个对象能够接收的消息列表是保存在它所对应的类中的。在实际编程中，我们几乎不会遇到向元类发消息的情况，那它的 isa 指针在实际上很少用到。不过这么设计保证了面向对象的干净，即所有事物都是对象，都有 isa 指针。
我们再来看看继承关系，由于类方法的定义是保存在元类 (metaclass) 中，而方法调用的规则是，如果该类没有一个方法的实现，则向它的父类继续查找。所以，为了保证父类的类方法可以在子类中可以被调用，所以子类的元类会继承父类的元类，换而言之，类对象和元类对象有着同样的继承关系。


因为对象在内存中的排布可以看成一个结构体，该结构体的大小并不能动态变化。所以无法在运行时动态给对象增加成员变量。
相对的，对象的方法定义都保存在类的可变区域中。Objective-C 2.0 并未在头文件中将实现暴露出来，但在 Objective-C 1.0 中，我们可以看到方法的定义列表是一个名为 methodLists的指针的指针（如下图所示）。通过修改该指针指向的指针的值，就可以实现动态地为某一个类增加成员方法。这也是Category实现的原理。同时也说明了为什么Category只可为对象增加成员方法，却不能增加成员变量。
![类对象结构.png](http://upload-images.jianshu.io/upload_images/1867963-3b992c05b8e13ab7.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

### 3.消息传递和转发机制
消息机制原理：对象根据方法编号SEL去映射表查找对应的方法实现;

SEL又叫选择器，是表示一个方法的selector的指针,映射方法的名字。Objective-C在编译时，会依据每一个方法的名字、参数序列，生成一个唯一的整型标识(Int类型的地址)，这个标识就是SEL。
SEL的作用是作为IMP的KEY，存储在NSSet中，便于hash快速查询方法。SEL不能相同，对应方法可以不同。所以在Objective-C同一个类(及类的继承体系)中，不能存在2个同名的方法，就算参数类型不同。多个方法可以有同一个SEL。
不同的类可以有相同的方法名。不同类的实例对象执行相同的selector时，会在各自的方法列表中去根据selector去寻找自己对应的IMP。
IMP是指向实现函数的指针，通过SEL取得IMP后，我们就获得了最终要找的实现函数的入口。
这个结构体相当于在SEL和IMP之间作了一个绑定。这样有了SEL，我们便可以找到对应的IMP，从而调用方法的实现代码。（在运行时才将SEL和IMP绑定, 动态配置方法）
```
typedef struct objc_method *Method;
struct objc_method {
SEL method_name                 OBJC2_UNAVAILABLE;  // 方法名
char *method_types                  OBJC2_UNAVAILABLE; // 参数类型
IMP method_imp                      OBJC2_UNAVAILABLE;  // 方法实现
}
```
相关概念：类型编码（Type Encoding）
编译器将每个方法的返回值和参数类型编码为一个字符串，并将其与方法的selector关联在一起。可以使用@encode编译器指令来获取它。

消息传递（Messaging）： 在对象之间传递数据并执行任务的过程

Objective－C基于C语言加入了面向对象特性和消息转发机制的动态语言，除编译器外还需要用Runtime系统来动态创建类和对象进行消息发送和转发。

不同语言有不同函数传递方法，C语言 － 函数指针，C++ － 函数调用（引用）类成员函数在编译时候就确定了其所属类别， Objective－C 通过选择器和block。

Objective－C强调消息传递而非方法调用。可以向一个对象传递消息，且不需要再编译期声明这些消息的处理方法。这些方法在运行时才确定。运行时（runtime）具体功能将在下面介绍。
向对象发送消息，实际上是调用objc_msgSend函数，obj_msgSend的实际动作就是：找到这个函数指针，然后调用它。
```
id objc_msgSend(receiver self, selector _cmd, arg1, arg2, ...)
//self和_cmd是隐藏参数，在编译期被插入实现代码。
//self：指向消息的接受者target的对象类型，作为一个占位参数，消息传递成功后self将指向消息的receiver。
//_cmd: 指向方法实现的SEL类型。
```
［receiver message］；
并不会马上执行 receiver 对象的 message方法的代码，而是向receiver发送一条message消息，该句话被编译器转化为：

id obj_msgSend(id self, SEL op, …);
obj_msgSend 发消息流程：
1.首先根据receiver对象的isa指针获取它对应的class
2.优先在class的cache查找message方法，如果找不到，再到methodLists查找
3.如果没有在class找到，再到super_class查找
4.一旦找到message这个方法，就执行它实现的IMP。
5.如果没有找到方法实现,最后触发消息机制进行弥补;

[receiver message]调用方法时，如果在message方法在receiver对象的类继承体系中没有找到方法，那怎么办？一般情况下，程序在运行时就会Crash掉，抛出 unrecognized selector sent to …类似这样的异常信息。但在抛出异常之前，还有三次机会按以下顺序让你拯救程序。

##### Method Resolution
首先Objective-C在运行时调用+ resolveInstanceMethod:或+ resolveClassMethod:方法，让你添加方法的实现。如果你添加方法并返回YES，那系统在运行时就会重新启动一次消息发送的过程。如果resolveInstanceMethod方法返回NO，运行时就跳转到下一步：消息转发(Message Forwarding)
##### Fast Forwarding
如果目标对象实现- forwardingTargetForSelector:方法，系统就会在运行时调用这个方法，只要这个方法返回的不是nil或self，也会重启消息发送的过程，把这消息转发给其他对象来处理。否则，就会继续Normal Fowarding。这里叫Fast，是因为这一步不会创建NSInvocation对象，但Normal Forwarding会创建它，所以相对于更快点。
##### Normal Forwarding
如果没有使用Fast Forwarding来消息转发，最后只有使用Normal Forwarding来进行消息转发。它首先调用methodSignatureForSelector:方法来获取函数的参数和返回值，如果返回为nil，程序会Crash掉，并抛出unrecognized selector sent to instance异常信息。如果返回一个函数签名，系统就会创建一个NSInvocation对象并调用-forwardInvocation:方法。

### 三种方法的区别
Method Resolution：由于Method Resolution不能像消息转发那样可以交给其他对象来处理，所以只适用于在原来的类中代替掉。
Fast Forwarding：它可以将消息处理转发给其他对象，使用范围更广，不只是限于原来的对象。
Normal Forwarding：它跟Fast Forwarding一样可以消息转发，但它能通过NSInvocation对象获取更多消息发送的信息，例如：target、selector、arguments和返回值等信息。


当想使用Category对已存在的类进行扩展时，一般只能添加实例方法或类方法，而不适合添加额外的属性。虽然可以在Category头文件中声明property属性，但在实现文件中编译器是无法synthesize任何实例变量和属性访问方法。这时需要自定义属性访问方法并且使用Associated Objects来给已存在的类Category添加自定义的属性。Associated Objects提供三个API来向对象添加、获取和删除关联值：

void objc_setAssociatedObject (id object, const void *key, id value, objc_AssociationPolicy policy )
id objc_getAssociatedObject (id object, const void *key )
void objc_removeAssociatedObjects (id object )

Associated Objects的key要求是唯一并且是常量，而SEL是满足这个要求的，所以上面的采用隐藏参数_cmd作为key。
隐藏参数self和_cmd

当[receiver message]调用方法时，系统会在运行时偷偷地动态传入两个隐藏参数self和_cmd，之所以称它们为隐藏参数，是因为在源代码中没有声明和定义这两个参数。至于对于self的描述，上面已经解释非常清楚了，下面我们重点讲解_cmd。

_cmd表示当前调用方法，其实它就是一个方法选择器SEL。一般用于判断方法名或在Associated Objects中唯一标识键名，前面在Associated Objects会讲到。
