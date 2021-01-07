# Chapter 8.3 - 函数调用

Created by : Mr Dk.

2020 / 01 / 30 22:58 🧨🧧

Ningbo, Zhejiang, China

---

## 8.3 函数调用

在执行函数中的具体代码之前，需要确定被调用的函数的版本 (哪一个函数被调用)。一切函数的调用在 Class 文件中都只是符号引用，而不是函数在实际运行时内存布局中的入口地址 (直接引用)。

### 8.3.1 解析 (Resolution)

在类加载阶段，有一部分符号引用会被直接转化为直接引用。前提：

* 函数在程序运行前就有一个 **可确定的** 调用版本
* 该函数的调用版本在运行期间不可改变

满足 **编译期可知，运行期不可变** 要求的主要有：

* 静态函数 - 与类型直接关联
* 私有函数 - 在外部不可被访问

这两类函数的特性决定了它们不可能通过继承重写出其它版本，因此它们都在类加载阶段进行解析。**非虚函数** (Non-Virtual Method)：在类加载的时候可以把符号引用解析为直接引用：

* 静态函数
* 私有函数
* 实例构造器 `<init>()`
* 父类函数
* 被 `final` 修饰的函数

解析调用是一个静态的过程，在编译期间就可以确定。在类加载的解析阶段就会把符号引用转换为直接引用，不必延迟到运行期再去完成。

### 8.3.2 分派 (Dispatch)

#### 8.3.2.1 静态分派

例子：

* `class Human`
* `class Man extends Human`
* `class Woman extends Human`

```java
Human man = new Man();
```

将前面的 `Human` 称为静态类型，后面的 `Man` 实际类型或运行时类型。变量 `man` 本身的静态类型不会改变，是编译期可知的，但变量 `man` 的运行时类型在运行期才可确定。

```java
// 运行时类型无法确定
Human human = (new Random()).nextBoolean() ? new Man() : new Woman();

// 静态类型就算变化在编译期也可以确定
sr.sayHello((Man) human);
sr.sayHello((Woman) human);
```

变量的实际类型必须能到程序运行时才能确定。JVM 在进行 **函数重载** 时通过参数的静态类型决定调用哪个重载版本，所有依赖静态类型来决定函数执行版本的分派动作，都称为静态分派。最典型的应用表现就是 **函数重载**

静态分派发生在编译阶段，因此分派不是由 JVM 来执行的。当函数没有合适的重载版本时，会发生安全的类型转换从而适配合适的重载版本。

#### 8.3.2.2 动态分派

动态分派与 **重写** (Override) 有密切关联。在选择函数的执行版本时，不再根据静态类型来选择，而是根据运行时类型。在字节码的角度看，调用了 `invokevirtual` 指令。该指令的运行时解析过程分为以下几步：

1. 找到操作数栈顶元素指向对象的运行时类型
2. 如果找到该类型中与常量描述符和简单名称都相符的函数，则进行访问权校验，如果通过则返回这个函数的直接引用；否则返回异常
3. 否则，按照继承关系依次对各个父类重复上一步的搜索和验证过程
4. 若始终没有找到合适的函数，则异常

由于第一步就要确定元素的运行时类型，因此需要根据函数调用者的运行时类型来选择函数版本。在运行期根据实际类型确定函数执行版本的分派称为 **动态分派**。字段不使用 `invokevirtual` 指令，所以字段永远不参与多态：

* 子类声明与父类同名的字段时，子类内存中两个字段都会存在
* 子类字段会遮蔽父类同名的字段

```java
static class Father {
    public int money = 1;

    public Father() {
        money = 2;
        showMeTheMoney();
    }

    public void showMeTheMoney() {
        System.out.println("I am Father, I have $" + money);
    }
}

static class Son extends Father {
    public int money = 3;

    public Son() {
        money = 4;
        showMeTheMoney();
    }

    public void showMeTheMoney() {
        System.out.println("I am Son, I have $" + money);
    }
}

public static void main(String[] args) {
    Father gay = new Son();
    System.out.println("This gay has $" + gay.money);
}
```

运行结果：

```
I am Son, I have $0
I am Son, I have $4
This gay has $2
```

> 为什么是 gay......

`Son` 创建时，隐式调用了 `Father` 类的构造函数。`Father` 类中调用的 `showMeTheMoney()` 是一个虚方法调用。

* 因为对象的运行时类型是 `Son` 类
* 所以调用的是 `Son` 的 `showMeTheMoney()`

此时 `Son` 中的 `money` 经过了类初始化，但还没有调用构造函数，所以是 `0`。下面调用 `Son` 的构造函数，`money` 被赋值为 `4`，因此输出中是 `4`。最后显式输出 `Father` 的 `money`，经过赋值，`Father` 的 `money` 的值为 `2`。`Father` 和 `Son` 的 `money` 是两个独立的变量。

#### 8.3.2.3 单分派与多分派

Java 的静态分派属于多分派：符号引用转换为直接引用时，有多个分派目标可供选择；Java 的动态分派属于单分派：符号引用转换为直接引用时，只有一个实际类型作为选择依据。Java 是一门静态多分派、动态单分派的语言。

#### 8.3.2.4 虚拟机动态分派的实现

每次运行时进行符号引用到直接引用的转换。JVM 真正运行时不会如此频繁地访问类的 metadata，常见的优化手段是在方法区建立 **虚方法表**，代替元数据查找以提高性能：

* 虚方法表存放各个方法入口的实际地址
* 若子类没有重写父类方法，则入口指向父类相同方法的入口
* 若子类重写了方法，子类虚方法表的入口被替换为子类实现版本的入口地址

虚方法表在类加载的链接阶段进行初始化，准备了类变量的初始值后，虚方法表也会被初始化完毕。

---

## 8.4 动态类型语言支持

`var` 是在编译时根据赋值符号右边的表达式静态推断数据类型，本质上是一种语法糖，不是动态类型。`dynamic` 在编译时不关心类型，等到运行时再进行类型判断。JVM 诞生之后，只增加过一条字节码指令 - `invokedynamic`。

---

