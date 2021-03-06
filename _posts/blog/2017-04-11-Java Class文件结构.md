---
layout: post
title: Java Class文件结构
description: Java Class文件结构
category: blog
---


### 概述   
1. 越来越多的程序语言选择了与操作系统和机器指令集无关的、平台中立的格式作为程序编译后的存储格式   
2. 字节码（ByteCode）是构成平台无关性的基石   
3. 除Java语言之外，运行在Java虚拟机之上运行的语言还有，如Clojure、Groovy、JRuby、Jython、Scala   
4. 实现语言无关性的基础仍然是虚拟机和字节码存储格式。Java虚拟机不和包括Java在内
的任何语言绑定，它只与“Class文件”这种特定的二进制文件格式所关联   
### Class类文件的结构   
任何一个Class文件都对应着唯一一个类或接口的定义信息，但反过来说，类或接
口并不一定都得定义在文件里（譬如类或接口也可以通过类加载器直接生成）   

        Class文件是一组以8位字节为基础单位的二进制流，排列紧凑，中间无任何分隔符   
        Class文件格式采用一种类似于C语言结构体的伪结构来存储数据，这种伪结构中只有两种数据类型：无符号数和表   
        无符号数可以用来描述数字、索引引用、数量值或者按照UTF-8编码构成字符串值，以u1、u2、u4、u8来分别代表1个字节、2个字节、4个字节和8个字节的无符号数   
        表用于描述有层次关系的复合结构的数据，由多个无符号数或者其他表作为数据项构成的复合数据类型   
        当需要描述同一类型但数量不定的多个数据时，经常会使用该类型的集合来表示，集合的形式是一个前置的容量计数器加若干个连续的数据项的形式   
### 魔数与Class文件的版本   
1. 每个Class文件的头4个字节称为魔数（Magic Number），其值为OxCAFEBABE   
2. 魔数的唯一作用是确定这个文件是否为一个能被虚拟机接受的Class文件   
3. 紧接着魔数的4个字节存储的是Class文件的次版本号和主版本号   
4. 高版本的JDK能向下兼容以前版本的Class文件，但不能运行以后版本的Class文件   

### 常量池   
1. 常量池紧接着版本号之后，是Class文件中最大的表类型数据项   
2. 常量池中常量的数量不固定，在常量池的入口需要放置一项u2类型的数据，代表常量池容量计数值（constant_pool_count），但从1开始计数   
3. 常量池中主要存放两大类常量：字面量（Literal）和符号引用（Symbolic References）   

        字面量存放，文本字符串、声明为final的常量   

        符号引用则存放，类和接口的全限定名、字段的名称和描述符、方法的名称和描述符   

4. 在Class文件中不会保存各个方法、字段的最终内存布局信息，当虚拟机运行时，需要从常量池获得对应的符号引用，再在类创建时或运行时解析、翻译到具体的内存地址之中   
5. 常量池中每一项常量都是一个表，共14种表，均有自己的表结构，但表开始的第一位是一个u1类型的标志位，代表当前这个常量属于哪种常量类型   
   
### 访问标志   
1. 访问标志（access_flags）紧接着常量池结束之后，用两个字节代表
2. 访问标志，用于识别一些类或者接口层次的访问信息，包括：   
        这个Class是类还是接口；   
        是否定义为public类型；   
        是否定义为abstract类型；   
        如果是类的话，是否被声明为final等   
   
### 类索引、父类索引与接口索引集合  
1. Class文件中由这三项数据来确定这个类的继承关系   
> 类索引（this_class）: u2类型的数据,用于确定这个类的全限定名  
> 父类索引（super_class）： u2类型的数据,用于确定这个类的父类的全限定名  
>> 由于Java语言不允许多重继承，所以父类索引只有一个   
>> 除了java.lang.Object外，所有Java类的父类索引都不为0   
> 接口索引集合（interfaces）:一组u2类型的数据的集合, 用来描述这个类实现了哪些接口，这些被实现的接口将按implements语句（如果这个类本身是一个接口，则应当是extends语句）后的接口顺序从左到右排列在接口索引集合中    
2. 类索引、父类索引和接口索引集合都按顺序排列在访问标志之后  
3. 类索引和父类索引用，各自指向一个类型为CONSTANT_Class_info的类描述符常量，通过CONSTANT_Class_info类型的常量中的索引值可以找到定义在CONSTANT_Utf8_info类型的常量中的全限定名字符串   
4. 接口索引集合，入口的第一项——u2类型的数据为接口计数器（interfaces_count），表示索引表的容量。如果该类没有实现任何接口，则该计数器值为0，后面接口的索引表不再占用任何字节  
     
###  字段表集合   
1. 字段表（field_info）用于描述接口或者类中声明的变量。字段（field）包括类级变量以及实例级变量，但不包括在方法内部声明的局部变量  
2. 字段表的结构如下：  
    
|        名称      | 作用 |  
| -----------------| ---- |
| access_flags     | 和类的访问标志类似，标示字段访问属性|
| name_index       | 字段的简单名称
| descriptor_index | 字段和方法的描述符
| attributes_count | 
| attributes       | 

    全限定名是指，把类全名中的“.”替换成“/”，比如：org/fenixsoft/clazz/TestClass；  
    简单名称是指没有类型和参数修饰的方法或者字段名称，这个类中的inc（）方法和m字段的简单名称分别是“inc”和“m”   
    方法和字段的描述符，是用来描述字段的数据类型、方法的参数列表（包括数量、类型以及顺序）和返回值  
    基本数据类型，byte、char、double、float、int、long、short、boolean，分别用大写字符“B”、“C”、“D”、“F”、“I”、“L”、“S”、“Z”表示  
    返回值的void类型都用一个大写字符，“V”来表示   
    对象类型则用字符，“L”加对象的全限定名来表示，如，Ljava/lang/Object   
    对于数组类型，每一维度将使用一个前置的“[”字符来描述，如为“java.lang.String[][]”类型的二维数组，将被记录为：“[[Ljava/lang/String；”，一个整型数组“int[]”将被记录为“[I”   
    用描述符来描述方法时，按照先参数列表，后返回值的顺序描述，参数列表按照参数的
    严格顺序放在一组小括号“（）”之内，如，void add()的描述符为“() V”，int indexOf(char[] c, int s, String str, int m)的描述符为“([CILjava/lang/String;I)I”  
    字段表集合中不会列出从超类或者父接口中继承而来的字段，但有可能列出原本Java代码之中不存在的字段，譬如在内部类中为了保持对外部类的访问性，会自动添加指向外部类实例的this字段  

### 方法表集合  
1. 方法表的描述与字段表很类似  
2. 方法表的结构，包括访问标志（access_flags）、名称索引（name_index）、描述符索引（descriptor_index）、属性表集合（attributes）几项  
3. volatile关键字和transient关键字不能修饰方法，所以方法表的访问标志中没有
ACC_VOLATILE标志和ACC_TRANSIENT标志  
4. synchronized、native、strictfp和abstract关键字可以修饰方法，方法表的访问标志中增加了ACC_SYNCHRONIZED、ACC_NATIVE、ACC_STRICTFP和ACC_ABSTRACT标志  
5. 方法里的Java代码，经过编译器编译成字节码指令后，存放在方法属性表集合中一个名为“Code”的属性里面  
6. 如果父类方法在子类中没有被重写（Override），方法表集合中就不会出现来自父类的方法信息  
7. 有可能会出现由编译器自动添加的方法，最典型的便是类构造器“＜clinit＞”方法和实例构造器“＜init＞”方法  
8. 在Java语言中，要重载（Overload）一个方法，除了要与原方法具有相同的简单名称之
外，还要求必须拥有一个与原方法不同的特征签名  
9. 特征签名就是一个方法中各个参数在常量池中的字段符号引用的集合，也就是因为返回值不会包含在特征签名中  
10. 因此Java语言里面是无法仅仅依靠返回值的不同来对一个已有方法进行重载的  

### 属性表集合  

在Class文件、字段表、方法表都可以携带自己的属性表集合，以用于描述某些场景专有的信息   
属性表集合的顺序没有严格要求，可以自定义自己的属性值
  
1. Code属性  
   1> Java程序方法体中的代码经过Javac编译器处理后，最终变为字节码指令存储在Code属性
   内  
   2> Code属性出现在方法表的属性集合之中，接口或者抽象类中的方法就不存在Code属性  
   3> 但是虚拟机规范中明确限制了一个方法不允许超过65535条字节码指令，即它实际只使用了u2的长度，如果超过这个限制，Javac编译器也会拒绝编译   
   4> 在任何实例方法里面，都可以通过“this”关键字访问到此方法所属的对象,是通过Javac编译器编译的时候把对this关键字的访问转变为对一个普通方法参数的访问，然后在虚拟机调用实例方法时自动传入此参数来实现的  
   5> 对于非static类型的变量（也就是实例变量）的赋值是在实例构造器＜init＞方法中进行的  
   6> 而对于类变量，则有两种方式可以选择：在类构造器＜clinit＞方法中或者使用ConstantValue属性。目前Sun Javac编译器的选择是：如果同时使用final和static来修饰一个变量，并且这个变量的数据类型是基本类型或者java.lang.String的话，就生成ConstantValue属性来进行初始化，如果这个变量没有被final修饰，或者并非基本类型及字符串，则将会选择在＜clinit＞方法中进行初始化   

### 字节码指令简介  
1. Java虚拟机的指令由一个字节长度的、代表着某种特定操作含义的数字（称为操作
码，Opcode）以及跟随其后的零至多个代表此操作所需参数（称为操作数，Operands）而构
成  
2. 由于Java虚拟机采用面向操作数栈而不是寄存器的架构，所以大多数的指令都不包含操作数，只有一个操作码  
3. 虚拟机处理超过一个字节数据的时候，不得不在运行时从字节中重建出具体数据的结构，如果要将一个16位长度的无符号整数使用两个无符号字节存储起来  
4. 有些字节码指令，它们的操作码助记符中都有特殊的字符来表明专门为哪种数据类型服务：i代表对int类型的数据操作，l代表long,s代表short,b代表byte,c代表char,f代表float,d代表double,a代表reference  

### 加载和存储指令  
1. 加载和存储指令用于将数据在栈帧中的局部变量表和操作数栈之间来回传输  
2. 将一个局部变量加载到操作栈：iload、iload_＜n＞、lload、lload_n＞、fload、fload_
＜n＞、dload、dload_＜n＞、aload、aload_＜n＞   
3. 将一个数值从操作数栈存储到局部变量表：istore、istore_＜n＞、lstore、lstore_＜n＞、fstore、fstore_＜n＞、dstore、dstore_＜n＞、astore、astore_＜n＞ 
4. 将一个常量加载到操作数栈：bipush、sipush、ldc、ldc_w、ldc2_w、aconst_null、
iconst_m1、iconst_＜i＞、lconst_＜l＞、fconst_＜f＞、dconst_＜d＞  

### 运算指令  
1. 运算或算术指令用于对两个操作数栈上的值进行某种特定运算，并把结果重新存入到操
作栈顶  

### 类型转换指令  
1. 类型转换指令可以将两种不同的数值类型进行相互转换  
2. Java虚拟机直接支持数值类型的宽化类型转换
（Widening Numeric Conversions，即小范围类型向大范围类型的安全转换）,即：  
    
    int类型到long、float或者double类型  
    long类型到float、double类型  
    float类型到double类型  

3. 处理窄化类型转换（Narrowing Numeric Conversions）时，必须显式地使用转换指令来完成，这些转换指令包括：i2b、i2c、i2s、l2i、f2i、f2l、d2i、d2l和d2f。窄化类型转换可能会导致转换结果产生不同的正负号、不同的数量级的情况，转换过程很可能会导致数值的精度丢失   

### 对象创建与访问指令  
1. 但Java虚拟机对类实例和数组的创建与操作使用了不同的字节码指令  
2. 创建类实例的指令：new  
3. 创建数组的指令：newarray、anewarray、multianewarray  
4. 访问类字段（static字段，或者称为类变量）和实例字段（非static字段，或者称为实例变量）的指令：getstatic、putstatic、getfield、putfield  
5. 操作数栈管理指令  
   
### 操作数栈管理指令  
1. Java虚拟机提供了一些用于直接操作操作数栈的指令  
2. 将操作数栈的栈顶一个或两个元素出栈：pop、pop2  
3. 将栈最顶端的两个数值互换：swap  
  
### 控制转移指令  
1. 控制转移指令可以让Java虚拟机有条件或无条件地从指定的位置指令而不是控制转移指
令的下一条指令继续执行程序，从概念模型上理解，可以认为控制转移指令就是在有条件或无条件地修改PC寄存器的值  
   
### 异常处理指令    
1. 在Java程序中显式抛出异常的操作（throw语句）都由athrow指令来实现  
2. Java虚拟机规范还规定了许多运行时异常会在其他Java虚拟机指令检测到异常状况时自动抛出。例如，在前面介绍的整数运算中，当除数为零时，虚拟机会在idiv或ldiv指令中抛出ArithmeticException异常  
3. 在Java虚拟机中，处理异常（catch语句）不是由字节码指令来实现的（很久之前曾经
使用jsr和ret指令来实现，现在已经不用了），而是采用异常表来完成的  
  
### 同步指令  
1. Java虚拟机可以支持方法级的同步和方法内部一段指令序列的同步，这两种同步结构都
是使用管程（Monitor）来支持的 
2. 方法级的同步是隐式的，即无须通过字节码指令来控制，它实现在方法调用和返回操作
之中。虚拟机可以从方法常量池的方法表结构中的ACC_SYNCHRONIZED访问标志得知一个方法是否声明为同步方法。当方法调用时，调用指令将会检查方法的ACC_SYNCHRONIZED访问标志是否被设置，如果设置了，执行线程就要求先成功持有管程，然后才能执行方法，最后当方法完成（无论是正常完成还是非正常完成）时释放管程。在方法执行期间，执行线
程持有了管程，其他任何线程都无法再获取到同一个管程。如果一个同步方法执行期间抛出了异常，并且在方法内部无法处理此异常，那么这个同步方法所持有的管程将在异常抛到同步方法之外时自动释放    
3. 同步一段指令集序列通常是由Java语言中的synchronized语句块来表示的，Java虚拟机的指令集中有monitorenter和monitorexit两条指令来支持synchronized关键字的语义，正确实现synchronized关键字需要Javac编译器与Java虚拟机两者共同协作支持  
4. 为了保证在方法异常完成时monitorenter和monitorexit指令依然可以正确配对执行，编译器会自动产生一个异常处理器，这个异常处理器声明可处理所有的异常，它的目的就是用来执行monitorexit指令    




