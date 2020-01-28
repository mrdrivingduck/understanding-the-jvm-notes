# Chapter 6 - 类文件结构

Created by : Mr Dk.

2020 / 01 / 28 16:57 🧨🧧

Ningbo, Zhejiang, China

---

## 6.2 无关性的基石

Java 的口号 - _Write Once, Run Anywhere._

Java 平台无关性的基石：

* 各种不同平台的 JVM
* 所有平台统一支持的程序存储格式 - 字节码 (Byte Code)

在 Java 技术发展之初，设计者们考虑并实现了让其它语言运行在 JVM 之上的可能性

Java 的规范被拆分为：

* Java 语言规范
* Java 虚拟机规范

目前，除 Java 外，发展出了大批运行在 JVM 上的语言

* Kotlin
* Clojure
* Groovy
* JRuby
* JPython
* Scala

任何语言都可以把 JVM 作为语言的运行基础

以 Class 文件作为语言执行的交付媒介

JVM 不关心 Class 文件的来源是什么语言

Java 语言中的各种语法、关键字等语义最终都会由多条字节码指令组合来表达

---

## 6.3 Class 类文件的结构

Class 文件结构相当稳定

基本上只会在原有结构基础上新增内容、扩充功能

* 以 8 个字节为基础单位的二进制流
* 中间没有任何分隔符
* 只有两种数据类型
    * 无符号数 - `u1` `u2` `u4` `u8`
    * 表 - 由多个无符号数或表构成

当需要描述同一类型但数量不定的多个数据时

* 前置一个容量计数器
* 后面紧跟多个数据项

文件结构：

1. magic - 魔数
2. minor_version - 文件次版本
3. major_version - 文件主版本
4. constant_pool_count - 常量池计数值
5. constant_pool - 常量池
6. access_flags - 访问标志
7. this_class - 类索引
8. super_class - 父类索引
9. interfaces_count - 接口索引计数值
10. interfaces - 接口索引
11. fields_count - 字段表计数值
12. fields - 字段表
13. methods_count - 方法表计数值
14. methods - 方法表
15. attributes_count - 属性表计数值
16. attributes - 属性表

### 6.3.1 魔数与 Class 文件的版本

魔数占头四个字节 - `0xCAFEBABE`

紧接的四个字节是版本号

* 前两个字节是次版本号
* 后两个字节是主版本号 (从 `45` (JDK 1.1) 开始)

### 6.3.2 常量池

* Class 文件中的资源仓库
* 与 Class 文件结构中其它项目关联最多的数据
* 数量不固定 - 放置了一个 `u2` 类型的常量池计数值

常量池的计数是从 `1` 开始的 (如 22)

* 那么常量池中有 1 - 21 项常量
* 第 `0` 项代表不引用任何常量池项目

常量池存放两类常量：

* 字面量 (Literal)
* 符号引用 (Symbolic References)
    * 包 (Package)
    * 类和接口的全限定名 (Fully Qualified Name)
    * 字段的名称和描述符
    * 方法的名称和描述符
    * 方法句柄和方法类型
    * 动态调用点和动态常量

常量池中的每一项常量都是一个表

* 表结构的起始第一位是一个 `u1` 类型的标志，代表常量的类型
* 之后是常量的具体内容

不同的常量类型有着不同的结构

* `CONSTANT_Utf8_info`
* `CONSTANT_Interger_info`
* `CONSTANT_Float_info`
* ...

> Oracle 提供了 `javap` 工具用于分析字节码内容

### 6.3.3 访问标志

识别类或接口层次的信息

* 是类还是接口？
* 是否为 `public`
* 是否为 `abstract`
* 是否被声明为 `final`
* ...

### 6.3.4 类索引、父类索引与接口索引集合

Class 文件通过这三项数据确定该类型的继承关系

类索引确定该类的全限定名

父类索引确定该类父类的全限定名 (Java 不允许多继承)

* 除了 `java.lang.Object` 外，父类索引都不为 `0`

接口索引描述该类实现了哪些接口

* 按 `implement` 关键字后的接口顺序从左到右排列在接口索引集合中
* 实现的接口数量不定，所以存在接口索引集合开头有一个计数器

索引指向常量池中的类描述符常量

### 6.3.5 字段表集合

描述接口或类中声明的变量

* 类级变量
* 实例级变量

字段的结构体定义为：

* `access_flags`
    * 作用域 `public` `private` `protected`
    * 是实例变量还是类变量 `static`
    * 可变性 `final`
    * 并发可见性 `volatile` (是否强制从主内存读写)
    * 是否可被序列化 `transient`
    * ...
* `name_index`
    * 指向常量池中的字段简单名称 (没有类型和参数修饰的字段名称)
* `descriptor_index`
    * 指向常量池中的描述符
* `attributes_count` (属性表，可选)
* `attributes`

字段表集合不会列出从父类继承而来的字段

### 6.3.6 方法表集合

描述类或接口中定义的函数

* `access_flags`
    * 作用域 `public` `private` `protected`
    * 是实例变量还是类变量 `static`
    * 可变性 `final`
    * `synchronized` `native` `strictfp` `abstract`
* `name_index`
    * 指向常量池中的函数简单名称 (没有类型和参数修饰的字段名称)
* `descriptor_index`
    * 指向常量池中的描述符
* `attributes_count` (属性表，可选)
* `attributes`

函数中的 Java 代码经过 `javac` 编译为字节码后

存放在方法属性表的 `Code` 属性下

如果父类方法在子类中没有被 override

方法表中就不会出现来自父类的方法信息

另外，在 Java 中，如果要重载一个函数，需要满足：

* 与原方法具有相同的简单名称
* 要与原方法有不同的 __特征签名__
    * 指一个方法中各个参数在常量池中的字段符号引用的集合
    * 返回值不包含在特征签名中

因此 Java 无法只通过返回值来对一个已有的方法进行重载

### 6.3.7 属性表集合

每个属性项的结构

* attribute_name_index
    * 属性名称
    * 从常量池中引用一个 `CONSTANT_Utf8_info` 类型的常量
* attribute_length
    * 属性值的长度
* info
    * 属性值
    * 结构可以完全自定义

使用这些属性项的位置也不同

* 有些在方法表中使用
* 有些在字段表中使用
* ......

每个属性的名称都要从

#### 6.3.7.1 Code 属性

并非所有的方法都有该属性 (比如接口或抽象类)

* `attribute_name_index`
    * 指向 `CONSTANT_Utf8-info` 型常量索引 (固定为 `Code`)
* `attribute_length` - 属性值长度
* `max_stack`
    * 操作数栈深度的最大值
* `max_locals`
    * 局部变量表所需的存储空间
    * 单位是变量槽 (JVM 为局部变量分配内存所使用的最小单位)
* `code_length`
* `code` - 字节码
* `exception_table_length`
* `exception_table` - 显式异常处理表
* `attributes_count`
* `attributes`

其中的异常表用于处理 `try-catch-finally`

异常表的结构为：

* `start_pc`
* `end_pc`
* `handler_pc`
* `catch_type`

含义为，当字节码 `start_pc` 到 `end_pc` 之间出现 `catch_type` 或其子类的异常，则转到 `handler_pc` 行处理

当 `catch_type` 的值为 0 时，任意异常情况都要转到 `handler_pc` 处理

一个例子：

```java
public int inc() {
    int x;
    try {
        x = 1;
        return x;
    } catch (Exception e) {
        x = 2;
        return x;
    } finally {
        x = 3;
    }
}
```

* 如果程序正常运行，执行 `x = 1;` 后保存在返回值的变量槽中
    * 执行 `x = 3;` 后，将返回值变量槽中的 `1` 读入操作栈顶并返回
* 如果程序异常
    * 转入 `x = 2;` 后保存在返回值的变量槽中
    * 执行 `x = 3;` 后，将返回值变量槽中的 `2` 读入操作栈顶并返回

#### 6.3.7.2 Exceptions 属性

与 Code 属性中的异常表不同

作用是列举出函数中可能抛出的受查异常 (Checked Exceptions)

也就是函数描述时 `throws` 关键字后面列举的异常

#### 6.3.7.3 LineNumberTable 属性

描述 Java 源码行号与字节码行号之间的对应关系

用于在抛出异常时，显示在 Java 程序中的出错行号

#### 6.3.7.4 LocalVariableTable 及 LocalVariableTypeTable 属性

描述栈帧中局部变量表中的变量与 Java 源代码中定义变量之间的关系

如果没有这个属性

* 当别人引用这个方法时，所有的参数名称将会丢失
* 比如 IDE 中自动填充的 `arg0` `arg1`

#### 6.3.7.5 SourceFile 及 SourceDebugExtension 属性

记录生成这个 Class 文件的源码文件名称

如果没有这个属性

抛出异常时，堆栈中将不会显示出错代码所属的文件名

#### 6.3.7.6 ConstantValue 属性

通知虚拟机自动为静态变量赋值

只有被 `static` 关键字修饰的变量才能用这种属性

#### 6.3.7.7 InnerClasses 属性

记录内部类与宿主类之前的关联

属性的结构：

* `attribute_name_index` - 对应常量池中的 `InnerClasses`
* `attribute_length`
* `number_of_classes`
* `inner_classes`

每个内部类属性值的结构如下：

* `inner_class_info_index`
    * 指向常量池中的 `CONSTANT_Class_info` 类型常量的索引
    * 代表内部类
* `outer_class_info_index`
    * 同上
    * 代表宿主类
* `inner_name_index`
    * 指向常量池中 `CONSTANT_Utf8_info` 类型常量的索引
    * 如果是匿名内部类，值为 `0`
* `inner_class_access_flags`
    * 内部类的访问标志

#### 6.3.7.8 Deprecated 及 Synthetic 属性

#### 6.3.7.9 StackMapTable 属性

#### 6.3.7.10 Signature 属性

#### 6.3.7.11 BootstrapMethods 属性

#### 6.3.7.12 MethodParameters 属性

记录方法的各个形参名称和信息

之前的 LocalVariableTable 属性是 Code 属性的子属性

这意味着没有方法体就没有局部属性表

而对于接口或抽象方法来说，是可以不存在方法体的

该属性可以将方法中的参数信息保存下来

---

## 6.4 字节码指令简介

JVM 的指令由一个字节长度的操作码

以及跟随其后的 0 或多个操作数构成

大多数指令都不包含操作数，只有一个操作码

JVM 采用 __面向操作数栈__ 而不是面向寄存器的架构

指令参数都放在操作数栈中

由于限制了操作码为 1 字节 - 操作码总数不能超过 256 条

### 6.4.1 字节码与数据类型

大多数指令都包含操作对应的数据类型信息

比如 `iload` `fload` `dload`

但是由于操作码的长度有限

不可能为每条指令都提供不同数据类型的版本

因此 JVM 针对特定的操作只提供了有限数据类型的指令

* 这种特性称为 _Not Orthogonal_

编译器会在编译器或运行期作如下转换：

* `byte` 和 `short` 类型的数据带符号扩展为 `int`
* `boolean` 和 `char` 类型的数据零扩展为 `int`
* 然后统一使用 `int` 版本的指令来操作

### 6.4.2 加载和存储指令

用于将数据在栈帧中的 __局部变量表__ 和 __操作数栈__ 之间来回传输

### 6.4.3 运算指令

对两个操作数栈上的值进行某种特定运算

并将结果重新存入到操作栈顶

### 6.4.4 类型转换指令

用于用户代码中的显式类型转换

JVM 直接支持小范围类型向大范围类型的安全转换

转换指令用于处理窄化类型转换

* 可能产生不同正负号
* 可能导致精度丢失

### 6.4.5 对象创建与访问指令

类实例和数组都是对象

但是创建与操作使用了不同的字节码指令

### 6.4.6 操作数栈管理指令

JVM 提供用于直接操作操作数栈的指令

### 6.4.7 控制转移指令

对于 `boolean` `byte` `char` `short` 类型的条件分支操作

都使用 `int` 类型的分支指令完成

对于 `long` `float` `double` 类型的条件分支比较

则先执行相应类型的分支比较指令

指令返回一个 `int` 型到操作数栈中

随后再执行 `int` 型的条件分支比较完成分支跳转

JVM 提供的 `int` 型的条件分支指令是最为丰富、强大的

### 6.4.8 方法调用和返回指令

* invokevirtual 指令 - 调用对象的实例方法
* invokeinterface 指令 - 调用接口方法
* invokespecial 指令 - 调用需要特殊处理的实例方法 (初始化方法、私有方法、父类方法)
* invokestatic 指令 - 调用类静态方法
* invokedynamic 指令

方法调用指令与数据类型无关

方法返回指令根据返回值的类型区分

### 6.4.9 异常处理指令

用于在 Java 程序中用 `throw` 显式抛出异常

其它异常是自动抛出的

在 JVM 中，处理异常不是使用字节码指令实现的

而是采用异常表来完成

### 6.4.10 同步指令

JVM 支持方法级的同步和方法内部一段指令序列的同步

方法级同步是隐式的，无需通过字节码指令控制，通过管程 (Monitor) 实现

* 函数调用时，调用指令检查函数的 `ACC_SYNCHRONIZED` 访问标志
* 如果设置了该标志，执行线程就需要先成功持有管程，然后才能执行函数
* 函数完成后释放管程 (不管是正常完成还是非正常完成)
* 其它任何线程都无法再获取到同一个管程

同步一段指令集序列由 `synchronized` 语句块表示

JVM 提供 `monitorenter` 和 `monitorexit` 两条指令支持该语义

两条指令必须相互对应

---

## 6.5 公有设计，私有实现

Class 文件格式以及字节码指令集都是 _Java 虚拟机规范_ 中规定好的

满足该规定的约束下，对 JVM 的具体实现做出修改和优化是完全可行的

_Java 虚拟机规范_ 中明确鼓励实现者这样去做

只要优化以后的 JVM 依旧可以正确读取 Class 文件

并且包含在其中的语义能得到完整保持

JVM 在后台如何处理 Class 文件完全是实现者自己的事情

---

