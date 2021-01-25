[TOC]

# Java虚拟机学习笔记（二）——类文件结构

## 1.Class类文件结构

任何一个Class文件对应着唯一一个类或者是接口定义信息。

Class文件是一组以8字节为基础单位的二进制流，各个数据紧凑严格的按照顺序排列在文件中，中间没有任何添加分割的分隔符，使得Class文件中全是程序运行的必要数据。当遇到需要占用8字节以上的数据时，会按照高位在前的方式分割成若干个8个字节进行存储。

Class 文件格式一般采用一种伪结构来存储数据，这种伪结构有两种数据类型

* 无符号数 u1、u2、u4、u8分别代表1字节、2字节、4字节、8字节的无符号数

* 表 表是由多个无符号数或者其他表作为数据结构构成的复杂数据类型，表的命名习惯以“_info”结尾。整个Class文件可以视为一张表，这张表由下图所示的数据项严格构成。

  ![5E80B19FE6AB23ADCB2B8EB239C8F2FA](C:\Users\wlk\Desktop\JVM学习笔记\图片\5E80B19FE6AB23ADCB2B8EB239C8F2FA.jpg)

![code](C:\Users\wlk\Desktop\JVM学习笔记\图片\code.png)

### 1.1 魔数与版本

前四个字节是一个固定值，我们通过字节码可以看到 “ca fe ba be”，它用来确定此文件是否是一个能被虚拟机接受的Class文件。

使用魔数而不是扩展名来进行识别主要是因为扩展名可以被随意地修改。

接下来两个字节是次版本号（Minor Version），从JDK1.2到JDK12版本号均未使用，都为零，JDK12后，设计者重新启用次版本号。

第七和第八字节是主版本号（Major Version），对应规则如下：

![F9C9EC6C02EF820460472789FF9F9F8A](C:\Users\wlk\Desktop\JVM学习笔记\图片\F9C9EC6C02EF820460472789FF9F9F8A.jpg)

可以看到，我的第七第八字节是“00 34”，这是十六进制的数，我们换算成十进制为52，对应着JDK版本号为JDK8.

### 1.2 常量池

接下来两字节是常量池容量计数器，“00 22”，对应着应该有34个常量，但是，这里要注意，我们的常量池的容量计数不是从零开始的，而是从一开始的，表示着我们的常量索引是从“1~33”，实际只有33项。

我们使用javap -v 来验证一下：

![javap](C:\Users\wlk\Desktop\JVM学习笔记\图片\javap.png)

可以看到，常量池这有33项。同时，使用javap -v 已经将我们Class文件中的字节码转换成我们可以看懂的命令了。

我们可以看到，常量池中主要存放两大类的常量：字面量和符号引用。

### 1.3 其他：

访问标志、类索引、父类索引、接口索引集合、字段表集合、方法表集合、属性表集合。

## 2.字节码指令

### 2.1 加载和存储指令

将一个局部变量加载到操作栈：iload、lload、fload、dload、aload

将一个数值从操作数栈存储到局部变量表中：istore、lstore、fstore、dstore、astore

将一个常量加载到操作数栈：bipush、sipush、ldc、iconst、lconst、fconst、dconst

### 2.2 运算指令

加法指令：iadd、ladd、fadd、dadd

减法指令：isub、lsub、fsub、dsub

乘法指令：imul、lmul、fmul、dmul

除法指令：idiv、ldiv、fdiv、ddiv

求余指令：irem、lrem、frem、drem

取反指令：ineg、lneg、fneg、dneg

位移指令：ishl、ishr、iushl、lshl、lshr、lushr

按位或指令：ior、lor

按位与指令：iand、land

按位异或指令：ixor、lxor

局部变量自增指令：iinc

比较指令：dcmpg、dcmpl、fcmpg、fcmpl、lcmp

### 2.3 对象创建与访问指令

创建类实例的指令：new

创建数组的指令：newarray、anewarray、multianewarray

访问类字段（static，或者称为类变量）和实例字段（实例变量）指令：getfield、putfield、getstatic、putstatic

取数组长度指令：array length

检查类实例类型的指令：instanceof、checkcast

### 2.4 操作数栈管理指令

将操作数栈顶的一个或两个元素出栈：pop、pop2

复制栈顶一个或两个数值并将复制值或双份复制值重新压入栈顶：dup、dup2...

将栈最顶端的两个数值互换：swap

### 2.5 方法调用和返回指令

invokevirtual：用于调用对象的实例方法，根据对象的实际类型进行分派。

invokespecial：用于调用一些需要特殊处理的实例方法，包括实例初始化方法、私有方法、父类方法。

invokeinterface：用于调用接口方法，会在运行时搜索一个实现了这个接口的方法对象，找出合适的方法进行调用。

invokestatic：调用类的静态方法。

### 2.6 其他

异常处理指令、同步指令、控制转移指令。