# JVM、JDK、JRE关系

![](../../picture/javase%E7%9A%84%E7%BB%84%E4%BB%B6%E5%85%B3%E7%B3%BB)

- **JDK：** Java开发包，包含开发Java程序所必须的编译、运行等开发工具以及JRE (JRE包含JVM)
- **JRE：** Java运行环境，提供Java程序运行所需要的软件环境，包含JVM和丰富的系统类库(类库即Java提前封装好的功能类,可以直接拿来用) 。如果只是运行Java程序，只需安装JRE即可。
- **JVM：** Java虚拟机，执行`.class`字节码文件

## Java程序运行过程

1. 使用编辑器或IDE编写Java源代码，即`.java`文件
2. 通过`javac`（Java编译器）编译`.java`文件为`.class`字节码文件
3. 字节码文件可以在任何平台由JVM运行
4. JVM将字节码文件翻译为机器可以理解的二进制机器码

```
Java文件=> 编译器 => JVM 可执行的 Java 字节码(即虚拟指令)
=> JVM => JVM 中解释器 => 机器可执行的二进制机器码 => 程序运行
```

# 什么是JVM

1. **规范：**JVM就是一种规范，提供可执行`.class`字节码文件的运行环境。不同的供应商提供不同的实现，最受欢迎的实现是`Hotspot`。

![1624792743170](../../picture/1624792743170.png)

2. **风格：**JVM有两种不同的风格-client和server。虽然Server和Client VMs相似，但是Server VM经过了特殊的调优，可以最大限度地提高峰值运行速度。它用于执行长时间运行的服务器应用程序，这些应用程序需要尽可能快的运行速度，而不是快速启动时间或更小的运行时内存占用。开发人员可以通过指定-client或-server来选择他们想要的系统。
3. **JVM生成周期：**main方法执行JVM同时启动（周期起点）-》JVM实例所有非守护线程结束（JVM结束）
4. **JVM线程**：只要有一个非守护线程没结束，JVM就无法退出
   - 守护线程：只有一个GC线程是守护线程
   - 非守护线程（普通线程）：main方法就是一个非守护线程，main方法中启动的匿名线程也是非守护线程
5. **JVM和系统：**JVM之所以称为虚拟机，是因为它提供了一个不依赖于底层操作系统和机器硬件体系结构的机器接口。JVM内存分布在操作系统的堆中。

![](../../picture/JVM和计算机.jpg)

# JVM 结构

![](../../picture/JVM内存模型.png)

## 类加载器Class Load

类加载器就是用来加载`.class`文件的子系统， 其主要功能有三个：`loading`(加载），`linking`（链接）,`initialization`（初始化）。 

### Loading

加载类，JVM有三种类加载方式：Bootstrap,extension,application

![](../../picture/JVM 类加载器.png)

```java
Test test1 = new Test();
 Class<? extends Test> testClass = test1.getClass();   //获取对象的类

ClassLoader classLoader = testClass.getClassLoader(); //获取类加载器

System.out.println(classLoader);             //AppClassLoader 
System.out.println(classLoader.getParent()); //ExtClassLoader 
System.out.println(classLoader.getParent().getParent()); //null C\C++编写无法获取
```

#### 双亲委派机制

java.lang包下的`ClassLoader`类，`loadClass`方法如下：

```java
public Class<?> loadClass(String name) throws ClassNotFoundException {
        return loadClass(name, false);
    }
    //              -----??-----
    protected Class<?> loadClass(String name, boolean resolve)
        throws ClassNotFoundException
    {
            // 首先，检查是否已经被类加载器加载过
            Class<?> c = findLoadedClass(name);
            if (c == null) {
                try {
                    // 存在父加载器，递归的交由父加载器
                    if (parent != null) {
                        c = parent.loadClass(name, false);
                    } else {
                        // 直到最上面的Bootstrap类加载器
                        c = findBootstrapClassOrNull(name);
                    }
                } catch (ClassNotFoundException e) {
                    // ClassNotFoundException thrown if class not found
                    // from the non-null parent class loader
                }
 
                if (c == null) {
                    // If still not found, then invoke findClass in order
                    // to find the class.
                    c = findClass(name);
                }
            }
            return c;
    }
```

![](../../picture/%E5%8F%8C%E4%BA%B2%E5%A7%94%E6%89%98%E6%9C%BA%E5%88%B6.png)

 在类加载的过程中，存在着双亲委派机制

1. 当要加载一个类时，先在应用程序加载类中**检查是否被加载过**，如果加载过就无需加载了
2. 如果没有就由父类加载器执行`loadClass`方法，父类也会先**检查是否被加载过**，一直向上委托，直到启动类加载器
3. 启动类加载器**检查是否加载过**，有的话直接加载结束，没有才由下面的加载器加载 ，一直都最底层，如果都无法加载就会抛出`ClassNotFoundException`

##### 作用

保护了Java基础类无法被恶意修改，防止危险代码植入。

比如：当替换`String`类（java.lang包中），由于Bootstrap ClassLoader（启动类加载器）已经加载过了，就会直接执行java.lang包中的`String`类，而这个类并无`main()`方法，所有报错

```java
package java.lang;   // 自定义的包

public class String {
    public static void main(String[] args) {
        System.out.println("这是自定义的java.lang.String类");
    }
}
```

**报错：**

```java
错误: 在类 java.lang.String 中找不到 main 方法, 请将 main 方法定义为:
   public static void main(String[] args)
否则 JavaFX 应用程序类必须扩展javafx.application.Application
```

### Linking

类加载完成后，执行Linking（链接），一个**字节码验证器**将验证生成的字节码是否正确，如果验证失败，得到一个验证错误。此时，还将内存分配给类中的静态变量和静态方法。

### Initialization

这是类加载的，最后一个阶段，所有的静态变量都被赋以原始初值，并执行静态代码块。

## 运行时数据区（Run-Time Data Area）

### Java对象实例化过程



### 堆（Heap）

1. 所有线程共享
2.  存储程序运行时被创建的所有对象实例（new 出来的对象）、数组
3.  JVM中管理内存最大的一个区域
4. 可以通过-Xms和-Xmx控制堆的内存大小
5. 如果堆中没有内存存储新的对象实例堆也无法扩展，就会抛出`OutOfMemoryErroy`异常
6. GC垃圾回收主要作用区（清理垃圾对象），一般做JVM调优也是在此区

#### 年轻代

#### 老年代

#### 永久代

#### 元空间

### 栈（Stack）

1. 线程私有，生命周期与线程相同，每个线程一个

2. 每个线程创建的同时就会创建一个JVM栈，JVM栈存放当前线程中的局部变量（Java8大基本数类型,非基本类型的对象只在栈存方指向堆的地址值）、部分返回结果、对象的引用

   ![](../../picture/栈.png)

3. JVM最小单位可以理解为一个栈帧，一个栈帧对应一个方法，main()方法是第一个启动的在最底下；当Method1()被调用就会被压栈，Method1()又调用Method2()，Method2()也会被压栈；依次执行完毕，从栈顶开始出栈。

​       **注意：**

JVM规定了栈的最大深度，当方法调用栈太深，会抛出`StackOverflowError`错误

```java
public class MyTest {
    public static void main(String[] args) {
        MyTest myTest = new MyTest();
        myTest.a();
    }

    public void a(){
        b();
    }

    public void b(){
        a();
    }
}
```

代码中出现了循环调用

### 方法区（Method Area）

1. 所有线程共享
2. 存储static（静态变量）、final（常量）、类模板信息（不是对象实例）、常量池



### 程序计数器（PC Register）

 每个线程都有一个程序计数器，是线程私有的，就是一个指针，指向方法区中的方法字节码，存储当前正在执行的语句的物理内存地址 。占用的内存空间非常小，几乎忽略不计。

### 本地方法栈（Native Method Stack）

1. 线程私有，生命周期与线程相同，每个线程一个。

2. 作用和JVM栈类似，区别是JVM栈执行的是Java方法，本地方法栈是执行`native`修饰的本地方法的，这是为了扩展一些Java无法实现的方法功能。

#### Native关键字 本地方法

* 凡是被`native`关键字修饰的方法，说明Java无法实现，就会调用底层C\C++语言库，属于本地方法
* 执行的时候会进入本地方法栈
* 调用本地方法接口  JNI
* 本地方法库实现本地方法接口

## 执行引擎（Execution Engine）

所有分配给JVM的代码都是由执行引擎（Execution Engine） 执行，执行引擎读取字节码文件逐行执行。使用内置的解释器和即时编译器（JIT）将字节码转换为机器码并执行。

* **解释器（Interpreter）：**
* **即时编译器（JIT Compiler）：**

## 本地方法接口（Native Method Interface）

`native`调用的接口，扩展不同语言供Java使用

## 本地方法库（Navtive Method Library）

本地方法接口的具体实现

## 沙箱机制

* **字节码校验器：**确保Java类文件遵循Java语言规范，帮助Java实现内存保护；不是所有类都会经过字节码检验，比如核心类
* **类加载器：**
  * 防止恶意代码干涉
  * 守护被信任的类库边界
  * 将代码归入保护域，确定了代码可以进行哪些操作

# 面试点

## 如何排查JVM问题

**对于正在运行的系统：**

1. 可以使用jmap来查看JVM各个区域的使用情况
2. 可以通过jstack来查看线程的运行情况，比如哪些线程阻塞了是否出现死锁
3. 可以通过jstat命令来查看垃圾回收情况，特别注意fullgc，如果fullgc比较频繁那么就要进行调优了（一般younggc就会清理掉对象，不会频繁使用fullgc）
   * 如果频繁使用fullgc但是没有出现内存溢出，那么表示fullgc回收了大部分对象，最好让这些对象能在younggc就能被回收，避免这些对象进入老年代。出现这种情况，就要排查这些存活时间不长的对象是否很大导致年轻代无法存放才直接进入老年代，可以加大年轻代的大小。
   * 找到占用内存最多的线程，定位到具体方法，优化方法，避免一些对象的创建从而减少内存占用

**对于已经出现OOM错误的系统：**

1. 一般我们都会在系统启动时加上==-XX:+HeapDumpOnOutOfMemoryError -XX:HeapDumpPath=/usr/local/base==命令，当系统出现OOM错误时会生成dump文件
2. 使用sisualvm等工具分析dump文件，找到异常实例或异常的线程（占用CPU高），进而优化代码