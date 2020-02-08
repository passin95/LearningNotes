


<!-- TOC -->

- [一、预备知识](#%E4%B8%80%E9%A2%84%E5%A4%87%E7%9F%A5%E8%AF%86)
  - [1.1 Android APK 打包流程](#11-android-apk-%E6%89%93%E5%8C%85%E6%B5%81%E7%A8%8B)
  - [1.2 字节码](#12-%E5%AD%97%E8%8A%82%E7%A0%81)
    - [1.2.1 类型对照表](#121-%E7%B1%BB%E5%9E%8B%E5%AF%B9%E7%85%A7%E8%A1%A8)
    - [1.2.2 JVM 指令集](#122-jvm-%E6%8C%87%E4%BB%A4%E9%9B%86)
- [二、ASM 简介](#%E4%BA%8Casm-%E7%AE%80%E4%BB%8B)
- [三、ASM 核心类](#%E4%B8%89asm-%E6%A0%B8%E5%BF%83%E7%B1%BB)
  - [3.1 Visitor](#31-visitor)
    - [3.1.1 ClassVisitor](#311-classvisitor)
    - [3.1.2 FieldVisitor](#312-fieldvisitor)
    - [3.1.3 MethodVisitor](#313-methodvisitor)
    - [3.1.4 AnnotationVisitor](#314-annotationvisitor)
    - [3.1.5 SignatureVisitor](#315-signaturevisitor)
- [四、ASM 实战](#%E5%9B%9Basm-%E5%AE%9E%E6%88%98)
  - [4.1 初探 ASM](#41-%E5%88%9D%E6%8E%A2-asm)
  - [4.2 统计方法耗时（Gradle + ASM）](#42-%E7%BB%9F%E8%AE%A1%E6%96%B9%E6%B3%95%E8%80%97%E6%97%B6gradle--asm)

<!-- /TOC -->

# 一、预备知识

## 1.1 Android APK 打包流程

本文 ASM 主要应用于 Android APK 打包过程中的字节码插桩，因此需要简单了解一下 APK 的打包过程，先看一张 Google 官方的打包流程图：

 <div align="center"> <img src="../pictures//APK%20 打包流程图.webp" /> </div>

**（1）** 打包资源文件

使用 aapt（The Android Asset Packaing Tool）对 res 目录下的资源文件打包生成 **R.java** 文件（资源索引表）以及 **.arsc** 资源文件。该工具位于 android-sdk/platform-tools 目录下。

**（2）** 处理 aidl files

AIDL 用于方便实现进程间通信。如果有 .aidl 文件，会通过 AIDL（Android Interface Definition Language）工具打包成 java 接口类，如果在没有使用到 .aidl 文件，则可以跳过这一步。该工具位于 android-sdk/platform-tools 目录下。

**（3）** 编译项目源代码，生成 .class 文件

将项目中所有的 Java（kotlin） 代码、R.java、aidl.java 通过 Java 编译器（javac）编译成.class 文件。该工具位于 ${JDK_HOME}/javac。

ASM 在就是应用于第（3）步和第（4）步之间。

**（4）** 生成 classes.dex 文件

将源码生成的.class 文件和第三方 jar 包通过 dx 工具打包成 dex 文件。

源码生成的 .class 文件虽然已经可以在 JVM 环境中运行，但是如果要在 Android 运行时环境中执行还需要特殊的处理，那就是 dx 处理，它会将文件进行翻译、重构、解释、压缩等操作，该工具位于 android-sdk/platform-tools 目录下。

**（5）** 打包生成 APK 文件

所有没有编译的资源、编译过的资源（.arsc）和.dex 文件都会通过 apkbuilder 工具打包到一个完成的.apk 文件中。该工具位于 android-sdk/tools 目录下。

**（6）** 对 APK 文件进行签名

一旦 APK 文件生成，它必须被签名才能被安装在设备上。使用 jarsigner 工具对 apk 进行验证签名，得到一个签名后的 apk（signed.apk）。在未指定签名文件时，会使用编译器默认的调试签名文件 debug.keystore。该工具位于 ${JDK_HOME}/jarsigner。

**（7）** 对签名后的 APK 文件进行对齐处理

如果发布的 apk 是正式版的话，就必须使用 zipalign 对 APK 进行对齐处理，对齐的主要过程是将 APK 包中所有的资源文件距离文件起始偏移为 4 字节整数倍，这样通过内存映射访问 apk 文件时的速度会更快。该工具位于 android-sdk/tools 目录下。

## 1.2 字节码

ASM 是对字节码文件进行操作，并且需要操作具体的 **JVM 指令**，因此需要对字节码以及 [Java 虚拟机栈](../notes/Java%20 虚拟机.md#22-java-虚拟机栈) 相关知识有所了解。这里将简述几个关键的概念。

1. 首先编译好的字节码文件本质上是一堆 16 进制的字节，平常用 IDE 打开 .class 文件，看到的是已经被反编译后所熟悉的 Java 代码。 

2. 每一个方法从调用开始到执行完成的过程，就对应着一个栈帧在虚拟机栈里面从入栈到出栈的过程。对于执行引擎来说，活动线程中只有栈顶的栈帧是有效的，称为当前栈帧，这个栈帧所关联的方法称为当前方法。执行引擎所运行的所有字节码指令都只针对当前栈帧进行操作。

3. 虚拟机把操作数栈作为它的工作区——大多数指令都要从这里弹出数据，执行运算，然后把结果压回操作数栈。比如，iadd 指令就要从操作数栈中弹出两个整数，执行加法运算，其结果又压回到操作数栈中，看看下面的示例，它演示了虚拟机是如何把两个 int 类型的局部变量相加，再把结果保存到第三个局部变量的：

```java
    begin  
    iload_0    // push the int in local variable 0 onto the stack  
    iload_1    // push the int in local variable 1 onto the stack  
    iadd       // pop two ints, add them, push result  
    istore_2   // pop int, store into local variable 2  
    end  
```

### 1.2.1 类型对照表

Java 类型和字节码类型描述对照表：

<style>
table th:first-of-type {
    width: 90pt;
}
table th:nth-of-type(2) {
    width: 90pt;
}
table th:nth-of-type(3) {
    width: 110pt;
} 
</style>

| Java Type | Type Descriptor |
| :--------- | :------------------- |
| boolean    | Z                    |
| byte       | B                    |
| short      | S                    |
| int        | I                    |
| float      | F                    |
| long       | J                    |
| double     | D                    |
| Object     | Ljava/lang/Object;   |
| int[]      | [I                   |
| Object[][] | [[Ljava/lang/Object; |


### 1.2.2 JVM 指令集

在初期的学习过程中（不管是字节码还是 ASM），我们常常需要查询 JVM 指令的含义。因此对 JVM 所有指令进行整理，如下表所示，参考自 [Java bytecode instruction listings](https://en.wikipedia.org/wiki/Java_bytecode_instruction_listings)。

注：栈变化一列中的内容越前面的值越是栈底，以 arrayref, index, value -> 为例，栈顶为 value。

| 指令码（hex）| 助记符 | 栈变化（前→后） | 说明 |
| :--------- | :----------------- | :----------- | :--------------------------- |
| 0×00       | nop                | 不改变        | 不执行任何操作 |
| 0×01       | aconst_null        | → null       | 将空引用压入栈顶 |
| 0×02       | iconst_m1          | → -1         | 将 int 类型数值-1 压入栈顶 |
| 0×03       | iconst_0           | → 0          | 将 int 类型数值 0 压入栈顶 |
| 0×04       | iconst_1           | → 1          | 将 int 类型数值 1 压入栈顶 |
| 0×05       | iconst_2           | → 2          | 将 int 类型数值 2 压入栈顶 |
| 0×06       | iconst_3           | → 3          | 将 int 类型数值 3 压入栈顶 |
| 0×07       | iconst_4           | → 4          | 将 int 类型数值 4 压入栈顶 |
| 0×08       | iconst_5           | → 5          | 将 int 类型数值 5 压入栈顶 |
| 0×09       | lconst_0           | → 0L         | 将 long 类型数值 0L 压入栈顶 |
| 0×0a       | lconst_1           | → 1L         | 将 long 类型数值 1L 压入栈顶 |
| 0×0b       | fconst_0           | → 0.0f       | 将 float 类型数值 0.0f 压入栈顶 |
| 0×0c       | fconst_1           | → 1.0f       | 将 float 类型数值 1.0f 压入栈顶 |
| 0×0d       | fconst_2           | → 2.0f       | 将 float 类型数值 2.0f 压入栈顶 |
| 0×0e       | dconst_0           | → 0.0        | 将 double 类型数值 0.0 压入栈顶 |
| 0×0f       | dconst_1           | → 1.0        | 将 double 类型数值 1.0 压入栈顶 |
| 0×10       | bipush             | → value      | 将单字节的某个常量值 (-128~127) 压入栈顶  |
| 0×11       | sipush             | → value      | 将一个短整型的某个常量值 (-32768~32767) 压入栈顶 |
| 0×12       | ldc                | → value      | 将常量索引从常量池（例如 String、int、float、Class）中压入栈顶 |
| 0×13       | ldc_w              | → value      | 将常量索引从常量池（例如 String、int、float、Class）中压入栈顶（宽索引） |
| 0×14       | ldc2_w             | → value      | 将常量索引从常量池（例如 double、long）中压入栈顶（宽索引） |
| 0×15       | iload              | → value      | 将指定索引的 int 类型本地变量压入栈顶 | 
| 0×16       | lload              | → value      | 将指定索引的 long 类型本地变量压入栈顶 |
| 0×17       | fload              | → value      | 将指定索引的 float 类型本地变量压入栈顶 |
| 0×18       | dload              | → value      | 将指定索引的 double 类型本地变量压入栈顶 |
| 0×19       | aload              | → objectref  | 将指定索引的引用类型本地变量压入栈顶 |
| 0×1a       | iload_0            | → value      | 将索引为 0 的 int 类型本地变量压入栈顶 |
| 0×1b       | iload_1            | → value      | 将索引为 1 的 int 类型本地变量压入栈顶 |
| 0×1c       | iload_2            | → value      | 将索引为 2 的 int 类型本地变量压入栈顶 |
| 0×1d       | iload_3            | → value      | 将索引为 3 的 int 类型本地变量压入栈顶 |
| 0×1e       | lload_0            | → value      | 将索引为 0 的 long 类型本地变量压入栈顶 |
| 0×1f       | lload_1            | → value      | 将索引为 1 的 long 类型本地变量压入栈顶 |
| 0×20       | lload_2            | → value      | 将索引为 2 的 long 类型本地变量压入栈顶 |
| 0×21       | lload_3            | → value      | 将索引为 3 的 long 类型本地变量压入栈顶 |
| 0×22       | fload_0            | → value      | 将索引为 0 的 float 类型本地变量压入栈顶 |
| 0×23       | fload_1            | → value      | 将索引为 1 的 float 类型本地变量压入栈顶 |
| 0×24       | fload_2            | → value      | 将索引为 2 的 float 类型本地变量压入栈顶 |
| 0×25       | fload_3            | → value      | 将索引为 3 的 float 类型本地变量压入栈顶 |
| 0×26       | dload_0            | → value      | 将索引为 0 的 double 类型本地变量压入栈顶 |
| 0×27       | dload_1            | → value      | 将索引为 1 的 double 类型本地变量压入栈顶 |
| 0×28       | dload_2            | → value      | 将索引为 2 的 double 类型本地变量压入栈顶 |
| 0×29       | dload_3            | → value      | 将索引为 3 的 double 类型本地变量压入栈顶 |
| 0×2a       | aload_0            | → objectref  | 将索引为 0 的引用类型本地变量压入栈顶 |
| 0×2b       | aload_1            | → objectref  | 将索引为 1 的引用类型本地变量压入栈顶 |
| 0×2c       | aload_2            | → objectref  | 将索引为 2 的引用类型本地变量压入栈顶 |
| 0×2d       | aload_3            | → objectref  | 将索引为 3 的引用类型本地变量压入栈顶 |
| 0×2e       | iaload             | arrayref, index → value | 加载指定索引的 int 类型数值压入栈顶 |
| 0×2f       | laload             | arrayref, index → value | 加载指定索引的 long 类型数值压入栈顶 |
| 0×30       | faload             | arrayref, index → value | 加载指定索引的 float 类型数值压入栈顶 |
| 0×31       | daload             | arrayref, index → value | 加载指定索引的 double 类型数值压入栈顶 |
| 0×32       | aaload             | arrayref, index → value | 加载指定索引的引用类型值压入栈顶 |
| 0×33       | baload             | arrayref, index → value | 加载指定索引的 byte 或 boolean 数值压入栈顶 |
| 0×34       | caload             | arrayref, index → value | 加载指定索引的 char 类型数值压入栈顶 |
| 0×35       | saload             | arrayref, index → value | 加载指定索引的 short 类型数值压入栈顶 |
| 0×36       | istore             | value →      | 将栈顶 int 类型数值存入指定索引的局部变量 |
| 0×37       | lstore             | value →      | 将栈顶 long 类型数值存入指定索引的局部变量 |
| 0×38       | fstore             | value →      | 将栈顶 float 类型数值存入指定索引的局部变量 |
| 0×39       | dstore             | value →      | 将栈顶 double 类型数值存入指定索引的局部变量 |
| 0×3a       | astore             | objectref →  | 将栈顶引用类型值存入指定索引的局部变量 |
| 0×3b       | istore_0           | value →      | 将栈顶 int 类型数值存入索引为 0 的本地变量 |
| 0×3c       | istore_1           | value →      | 将栈顶 int 类型数值存入索引为 1 的本地变量 |
| 0×3d       | istore_2           | value →      | 将栈顶 int 类型数值存入索引为 2 的本地变量 |
| 0×3e       | istore_3           | value →      | 将栈顶 int 类型数值存入索引为 3 的本地变量 |
| 0×3f       | lstore_0           | value →      | 将栈顶 long 类型数值存入索引为 0 的本地变量 |
| 0×40       | lstore_1           | value →      | 将栈顶 long 类型数值存入索引为 1 的本地变量 |
| 0×41       | lstore_2           | value →      | 将栈顶 long 类型数值存入索引为 2 的本地变量 |
| 0×42       | lstore_3           | value →      | 将栈顶 long 类型数值存入索引为 3 的本地变量 |
| 0×43       | fstore_0           | value →      | 将栈顶 float 类型数值存入索引为 0 的本地变量 |
| 0×44       | fstore_1           | value →      | 将栈顶 float 类型数值存入索引为 1 的本地变量 |
| 0×45       | fstore_2           | value →      | 将栈顶 float 类型数值存入索引为 2 的本地变量 |
| 0×46       | fstore_3           | value →      | 将栈顶 float 类型数值存入索引为 3 的本地变量 |
| 0×47       | dstore_0           | value →      | 将栈顶 double 类型数值存入索引为 0 的本地变量 |
| 0×48       | dstore_1           | value →      | 将栈顶 double 类型数值存入索引为 1 的本地变量 |
| 0×49       | dstore_2           | value →      | 将栈顶 double 类型数值存入索引为 2 的本地变量 |
| 0×4a       | dstore_3           | value →      | 将栈顶 double 类型数值存入索引为 3 的本地变量 |
| 0×4b       | astore_0           | objectref →  | 将栈顶引用类型值存入索引为 0 的本地变量 |
| 0×4c       | astore_1           | objectref →  | 将栈顶引用类型值存入索引为 1 的本地变量 |
| 0×4d       | astore_2           | objectref →  | 将栈顶引用类型值存入索引为 2 的本地变量 |
| 0×4e       | astore_3           | objectref →  | 将栈顶引用类型值存入索引为 3 的本地变量 |
| 0×4f       | iastore            | arrayref, index, value → | 将栈顶 int 类型数值存入指定数组的指定索引位置 |
| 0×50       | lastore            | arrayref, index, value → | 将栈顶 long 类型数值存入指定数组的指定索引位置 |
| 0×51       | fastore            | arrayref, index, value → | 将栈顶 float 类型数值存入指定数组的指定索引位置 |
| 0×52       | dastore            | arrayref, index, value → | 将栈顶 double 类型数值存入指定数组的指定索引位置 |
| 0×53       | aastore            | arrayref, index, value → | 将栈顶引用类型数值存入指定数组的指定索引位置 |
| 0×54       | bastore            | arrayref, index, value → | 将栈顶 boolean 或 byte 类型数值存入指定数组的指定索引位置 |
| 0×55       | castore            | arrayref, index, value → | 将栈顶 char 类型数值存入指定数组的指定索引位置 |
| 0×56       | sastore            | arrayref, index, value → | 将栈顶 short 类型数值存入指定数组的指定索引位置 |
| 0×57       | pop                | value →      | 将栈顶数值弹出 (数值不能是 double 或 long 类型的) |
| 0×58       | pop2               | {value2, value1} → | 将栈顶的一个（double 或 long 类型的) 或两个数值（其它类型）弹出 |
| 0×59       | dup                | value → value, value | 复制栈顶数值并将复制值压入栈顶（栈变化：value → value, value） |
| 0×5a       | dup_x1             | value2, value1 → value1, value2, value1 | 复制栈顶数值 value1，并将复制值插入到栈顶向下偏移 2 个位置处（value1，value2 不能是 double 或 long 类型） |
| 0×5b       | dup_x2             | value3, value2, value1 → value1, value3, value2, value1  | 复制栈顶数值 value1，若复制值类型为 double 或 long，则插入到栈顶向下偏移 2 个位置处，若为其它类型，将插入到栈顶向下偏移 3 个位置处 |
| 0×5c       | dup2               | {value2, value1} → {value2, value1}, {value2, value1}  | 同 dup，区别在于支持 double 或 long 类型，可以看做 dup 的两倍字节版本 |
| 0×5d       | dup2_x1            | value3, {value2, value1} → {value2, value1}, value3, {value2, value1}  | 同 dup_x1，区别在于支持 double 或 long 类型 |
| 0×5e       | dup2_x2            | {value4, value3}, {value2, value1} → {value2, value1}, {value4, value3}, {value2, value1}  | 同 dup_x2，区别在于支持 double 或 long 类型 |
| 0×5f       | swap               | value2, value1 → value1, value2 | 将栈顶的两个数值互换 (数值不能是 double 或 long 类型) |
| 0×60       | iadd               | value1, value2 → result | 将栈顶两 int 类型数值相加并将结果压入栈顶 |
| 0×61       | ladd               | value1, value2 → result | 将栈顶两 long 类型数值相加并将结果压入栈顶 |
| 0×62       | fadd               | value1, value2 → result | 将栈顶两 float 类型数值相加并将结果压入栈顶  |
| 0×63       | dadd               | value1, value2 → result | 将栈顶两 double 类型数值相加并将结果压入栈顶 |
| 0×64       | isub               | value1, value2 → result | 将栈顶两 int 类型数值相减并将结果压入栈顶 |
| 0×65       | lsub               | value1, value2 → result | 将栈顶两 long 类型数值相减并将结果压入栈顶 |
| 0×66       | fsub               | value1, value2 → result | 将栈顶两 float 类型数值相减并将结果压入栈顶 |
| 0×67       | dsub               | value1, value2 → result | 将栈顶两 double 类型数值相减并将结果压入栈顶 |
| 0×68       | imul               | value1, value2 → result | 将栈顶两 int 类型数值相乘并将结果压入栈顶 |
| 0×69       | lmul               | value1, value2 → result | 将栈顶两 long 类型数值相乘并将结果压入栈顶 |
| 0×6a       | fmul               | value1, value2 → result | 将栈顶两 float 类型数值相乘并将结果压入栈顶 |
| 0×6b       | dmul               | value1, value2 → result | 将栈顶两 double 类型数值相乘并将结果压入栈顶 |
| 0×6c       | idiv               | value1, value2 → result | 将栈顶两 int 类型数值相除并将结果压入栈顶 |
| 0×6d       | ldiv               | value1, value2 → result | 将栈顶两 long 类型数值相除并将结果压入栈顶 |
| 0×6e       | fdiv               | value1, value2 → result | 将栈顶两 float 类型数值相除并将结果压入栈顶 |
| 0×6f       | ddiv               | value1, value2 → result | 将栈顶两 double 类型数值相除并将结果压入栈顶 |
| 0×70       | irem               | value1, value2 → result | 将栈顶两 int 类型数值作取模运算并将结果压入栈顶 |
| 0×71       | lrem               | value1, value2 → result | 将栈顶两 long 类型数值作取模运算并将结果压入栈顶 |
| 0×72       | frem               | value1, value2 → result | 将栈顶两 float 类型数值作取模运算并将结果压入栈顶 |
| 0×73       | drem               | value1, value2 → result | 将栈顶两 double 类型数值作取模运算并将结果压入栈顶 |
| 0×74       | ineg               | value → result | 将栈顶 int 类型数值取反并将结果压入栈顶 |
| 0×75       | lneg               | value → result | 将栈顶 long 类型数值取反并将结果压入栈顶 |
| 0×76       | fneg               | value → result | 将栈顶 float 类型数值取反并将结果压入栈顶 |
| 0×77       | dneg               | value → result | 将栈顶 double 类型数值取反并将结果压入栈顶 |
| 0×78       | ishl               | value1, value2 → result | 将 int 类型数值（转为二进制数）左移指定位数并将结果压入栈顶 |
| 0×79       | lshl               | value1, value2 → result | 将 long 类型数值左移指定位数并将结果压入栈顶 |
| 0×7a       | ishr               | value1, value2 → result | 将 int 类型数值（符号）右移指定位数并将结果压入栈顶 |
| 0×7b       | lshr               | value1, value2 → result | 将 long 类型数值（符号）右移指定位数并将结果压入栈顶 |
| 0×7c       | iushr              | value1, value2 → result | 将 int 类型数值（无符号）右移指定位数并将结果压入栈顶 |
| 0×7d       | lushr              | value1, value2 → result | 将 long 类型数值（无符号）右移指定位数并将结果压入栈顶 |
| 0×7e       | iand               | value1, value2 → result | 将栈顶两 int 类型数值“按位与”并将结果压入栈顶 |
| 0×7f       | land               | value1, value2 → result | 将栈顶两 long 类型数值“按位与”并将结果压入栈顶 |
| 0×80       | ior                | value1, value2 → result | 将栈顶两 int 类型数值作“按位或”并将结果压入栈顶 |
| 0×81       | lor                | value1, value2 → result | 将栈顶两 long 类型数值作“按位或”并将结果压入栈顶 |
| 0×82       | ixor               | value1, value2 → result | 将栈顶两 int 类型数值作“按位异或”并将结果压入栈顶 |
| 0×83       | lxor               | value1, value2 → result | 将栈顶两 long 类型数值作“按位异或”并将结果压入栈顶 |
| 0×84       | iinc               | 不改变        | 将指定索引的 int 类型变量增加指定的值 const |
| 0×85       | i2l                | value → result | 将栈顶 int 类型数值强制转换成 long 类型数值并将结果压入栈顶 |
| 0×86       | i2f                | value → result | 将栈顶 int 类型数值强制转换成 float 类型数值并将结果压入栈顶 |
| 0×87       | i2d                | value → result | 将栈顶 int 类型数值强制转换成 double 类型数值并将结果压入栈顶 |
| 0×88       | l2i                | value → result | 将栈顶 long 类型数值强制转换成 int 类型数值并将结果压入栈顶 |
| 0×89       | l2f                | value → result | 将栈顶 long 类型数值强制转换成 float 类型数值并将结果压入栈顶 |
| 0×8a       | l2d                | value → result | 将栈顶 long 类型数值强制转换成 double 类型数值并将结果压入栈顶 |
| 0×8b       | f2i                | value → result | 将栈顶 float 类型数值强制转换成 int 类型数值并将结果压入栈顶 |
| 0×8c       | f2l                | value → result | 将栈顶 float 类型数值强制转换成 long 类型数值并将结果压入栈顶 |
| 0×8d       | f2d                | value → result | 将栈顶 float 类型数值强制转换成 double 类型数值并将结果压入栈顶 |
| 0×8e       | d2i                | value → result | 将栈顶 double 类型数值强制转换成 int 类型数值并将结果压入栈顶 |
| 0×8f       | d2l                | value → result | 将栈顶 double 类型数值强制转换成 long 类型数值并将结果压入栈顶 |
| 0×90       | d2f                | value → result | 将栈顶 double 类型数值强制转换成 float 类型数值并将结果压入栈顶 |
| 0×91       | i2b                | value → result | 将栈顶 int 类型数值强制转换成 byte 类型数值并将结果压入栈顶 |
| 0×92       | i2c                | value → result | 将栈顶 int 类型数值强制转换成 char 类型数值并将结果压入栈顶 |
| 0×93       | i2s                | value → result | 将栈顶 int 类型数值强制转换成 short 类型数值并将结果压入栈顶 |
| 0×94       | lcmp               | value1, value2 → result | 比较栈顶两 long 类型数值大小，并将结果（a == b ? 0 : (a < b ? -1 : 1)）压入栈顶 |
| 0×95       | fcmpl              | value1, value2 → result | 比较栈顶两 float 类型数值大小，并将结果压入栈顶；当其中一个数值为 NaN 时，将-1 压入栈顶 |
| 0×96       | fcmpg              | value1, value2 → result | 比较栈顶两 float 类型数值大小，并将结果压入栈顶，当其中一个数值为 NaN 时，将 1 压入栈顶 |
| 0×97       | dcmpl              | value1, value2 → result | 比较栈顶两 double 类型数值大小，并将结果压入栈顶；当其中一个数值为 NaN 时，将-1 压入栈顶 |
| 0×98       | dcmpg              | value1, value2 → result | 比较栈顶两 double 类型数值大小，并将结果压入栈顶，当其中一个数值为 NaN 时，将 1 压入栈顶 |
| 0×99       | ifeq               | value →      | 当栈顶 int 型数值等于 0 时跳转到指定行  |
| 0×9a       | ifne               | value →      | 当栈顶 int 型数值不等于 0 时跳转到指定行 |
| 0×9b       | iflt               | value →      | 当栈顶 int 型数值小于 0 时跳转到指定行 |
| 0×9c       | ifge               | value →      | 当栈顶 int 型数值大于等于 0 时跳转到指定行 |
| 0×9d       | ifgt               | value →      | 当栈顶 int 型数值大于 0 时跳转到指定行 |
| 0×9e       | ifle               | value →      | 当栈顶 int 型数值小于等于 0 时跳转到指定行 |
| 0×9f       | if_icmpeq          | value1, value2 → | 比较栈顶两 int 型数值大小，当结果等于 0 时跳转到指定行 |
| 0×a0       | if_icmpne          | value1, value2 → | 比较栈顶两 int 型数值大小，当结果不等于 0 时跳转到指定行 |
| 0×a1       | if_icmplt          | value1, value2 → | 比较栈顶两 int 型数值大小，当结果小于 0 时跳转到指定行 |
| 0×a2       | if_icmpge          | value1, value2 → | 比较栈顶两 int 型数值大小，当结果大于等于 0 时跳转到指定行 |
| 0×a3       | if_icmpgt          | value1, value2 → | 比较栈顶两 int 型数值大小，当结果大于 0 时跳转到指定行 |
| 0×a4       | if_icmple          | value1, value2 → | 比较栈顶两 int 型数值大小，当结果小于等于 0 时跳转到指定行 |
| 0×a5       | if_acmpeq          | value1, value2 → | 比较栈顶两引用型数值，当结果相等时跳转到指定行 |
| 0×a6       | if_acmpne          | value1, value2 → | 比较栈顶两引用型数值，当结果不相等时跳转到指定行 |
| 0×a7       | goto               | 不改变        | 无条件跳转到指定行 |
| 0×a8       | jsr                | → address    | 无条件跳转到指定行（用于 finally），并将 jsr 下一条指令地址压入栈顶 |
| 0×a9       | ret                | 不改变        | 从取自局部变量索引的地址继续执行 |
| 0×aa       | tableswitch        | index →      | 用于 switch 条件跳转，case 值连续，效率高（可变长度指令） |
| 0×ab       | lookupswitch       | key →        | 用于 switch 条件跳转，case 值不连续，效率相对低（可变长度指令） |
| 0×ac       | ireturn            | value → [empty] | 从当前方法返回 int |
| 0×ad       | lreturn            | value → [empty] | 从当前方法返回 long |
| 0×ae       | freturn            | value → [empty] | 从当前方法返回 float |
| 0×af       | dreturn            | value → [empty] | 从当前方法返回 double |
| 0×b0       | areturn            | objectref → [empty] | 从当前方法返回引用对象 |
| 0×b1       | return             | → [empty]    | 从当前方法返回 void |
| 0×b2       | getstatic          | → value      | 获取指定类的静态域（field），并将其值压入栈顶 |
| 0×b3       | putstatic          | value →      | 为指定的类的静态域赋值 |
| 0×b4       | getfield           | objectref → value | 获取指定类的实例域，并将其值压入栈顶 |
| 0×b5       | putfiel            | objectref, value → | 为指定的类的实例域赋值 |
| 0×b6       | invokevirtual      | objectref, [arg1, arg2, ...] → result | 调用类的实例对象方法 |
| 0×b7       | invokespecial      | objectref, [arg1, arg2, ...] → result | 调用父类构造方法、实例初始化方法（<init>） |
| 0×b8       | invokestatic       | [arg1, arg2, ...] → result | 调用静态方法 |
| 0×b9       | invokeinterface    | objectref, [arg1, arg2, ...] → result | 调用接口方法 |
| 0×ba       | invokedynamic      | [arg1, [arg2 ...]] → result | 通过反射机制调用方法 |
| 0×bb       | new                | → objectref | 创建一个对象，并将其引用值压入栈顶 |
| 0×bc       | newarray           | count → arrayref | 创建一个指定原始类型（如 int, float, char…）的数组，并将其引用值压入栈顶 |
| 0×bd       | anewarray          | count → arrayref | 创建一个引用型（如类，接口，数组）的数组，并将其引用值压入栈顶 |
| 0×be       | arraylength        | arrayref → length | 获得栈顶数组数值的长度并压入栈顶 |
| 0×bf       | athrow             | objectref → [empty], objectref | 引发错误或异常向外层抛出异常（堆栈的其余部分已被清除，只留下对抛出对象的引用） |
| 0×c0       | checkcast          | objectref → objectref | 检验栈顶对象类型转换，检验未通过将抛出 ClassCastException |
| 0×c1       | instanceof         | objectref → result | 检验栈顶对象是否是指定的类，如果是将 1 压入栈顶，否则将 0 压入栈顶 |
| 0×c2       | monitorenter       | objectref →  | 获得对象的锁（synchronized 关键字） |
| 0×c3       | monitorexit        | objectref →  | 释放对象的锁 |
| 0×c4       | wide               |              | 执行其他指令，当本地变量的索引超过 255 时使用该指令扩展索引宽度。 |
| 0×c5       | multianewarray     | count1, [count2,...] → arrayref | 创建一个多维数组 |
| 0×c6       | ifnull             | value →      | 如果值为空，则跳转到指定行 |
| 0×c7       | ifnonnull          | value →      | 如果值不为空，则跳转到指定行 |
| 0×c8       | goto_w             | 不改变        | 无条件跳转到指定行（行标记不够用时） |
| 0×c9       | jsr_w              | → address    | 无条件跳转到指定行（用于 finally），并将 jsr 下一条指令地址压入栈顶（行标记不够用时） |
| 0×ca       | breakpoint         |              | IDE 中的调试断点保留；不应出现在任何类文件中  |
| 0×fe       | impdep1            |              | 为调试程序中依赖于实现的操作保留；不应出现在任何类文件中 |
| 0×ff       | impdep2            |              | 为调试程序中依赖于实现的操作保留；不应出现在任何类文件中 |

# 二、ASM 简介

ASM 是一个 Java 字节码操作与分析框架，它可以对字节码文件以及字节码内容进行增删改查。ASM 通过读取类文件中的元素信息（例如类名称、方法、属性以及指令等）来选择性的改变原有类的行为。

若用于 Java 项目，可通过以下依赖进行导入。

```groovy
// ASM 核心代码。
implementation 'org.ow2.asm:asm:+'
// 提供一些常用的工具和方法适配器。
implementation 'org.ow2.asm:asm-commons:+'
```

若用于 Android 项目，Android Gradle Tool 已包含 ASM 的依赖，可依赖该工具库即可：

```groovy
implementation 'com.android.tools.build:gradle:+'
```

# 三、ASM 核心类

ASM Javadoc：https://asm.ow2.io/javadoc/overview-summary.html

ASM 主要有以下几个核心的类、接口：

- ClassReader：用来读取并解析字节码文件并将相关的事件传递给注册的 ClassVisitor。
- ClassVisitor：定义在读取 Class 字节码时会触发的事件方法，如类头解析完成、注解解析、字段解析、方法解析等，方法皆以 visitXXX() 打头。
- AnnotationVisitor：定义在解析注解时会触发的事件，如解析到一个基本值类型的注解、enum 值类型的注解、Array 值类型的注解、注解值类型的注解等。
- FieldVisitor：定义在解析字段时触发的事件，如解析到字段上的注解、解析到字段相关的属性等。
- MethodVisitor：定义在解析方法时触发的事件，如方法上的注解、属性、代码等。

----------------------------------------------------------------------

- ClassWriter 类：继承于 ClassVisitor，重写了相关 visitXXX() 方法，用于拼接字节码，即开发者若对 ClassWriter 没有任何修改，则调用 toByteArray() 时相当于拷贝了一份 ClassReader 传入的字节码文件，换句话说 ClassWriter 不会影响原字节码文件。
- AnnotationWriter 类：继承于 AnnotationVisitor，用于拼接注解相关字节码。
- FieldWriter 类：继承于 FieldVisitor，用于拼接字段相关字节码。
- MethodWriter 类：继承于 MethodVisitor，用于拼接方法相关字节码。

----------------------------------------------------------------------

- SignatureReader 类：对类定义、字段定义、方法定义、本地变量定义的签名的解析。Signature 因范型引入，用于存储范型定义时的元数据（因为这些元数据在运行时会被擦除）。
- SignatureVisitor 接口：定义在解析泛型时会触发的事件。
- SignatureWriter 类：继承于 SignatureVisitor，用于拼接泛型相关字节码。

----------------------------------------------------------------------

- Opcodes：包括很多常量定义，包括 ASM API 版本号、访问标识（包含了所有修饰符）、JVM 指令、处理标签、数组类型代码等。
- Type 类：类型相关的常量定义。

## 3.1 Visitor

### 3.1.1 ClassVisitor

一个 Java 类中会存在方法、注解、属性等，ClassReader 在解析字节码文件时会回调 ClassVisitor 中对应的 visitXXX() 方法。

```java
public class Demo {

    public static void main(String[] args) throws Exception {
        ClassReader cr = new ClassReader("me/passin/asm/Demo$Passin");
        ClassPrinter printer = new ClassPrinter();
        cr.accept(printer, ClassReader.EXPAND_FRAMES);
    }

    @Deprecated
    final static class Passin implements Serializable {

        public int field1 = 1;
        public final int field2 = 1;

        public Passin() {
        }

        public void method1(int i) throws Exception {
        }

        public float method2(Object o) {
            return 1.0f;
        }

        public Object method3() {
            return new Object();
        }
    }

    static class ClassPrinter extends ClassVisitor {

        public ClassPrinter() {
            super(Opcodes.ASM7);
        }

        @Override
        public void visit(int version, int access, String name, String signature, String superName,
                String[] interfaces) {
            // version：是 JDK 版本号。
            // access：用于添加修饰符，在 ASM 中是以“Opcodes.ACC_”开头的常量。
            // signature：泛型信息，如果类并未定义任何泛型该参数为 null。
            super.visit(version, access, name, signature, superName, interfaces);
            if ((access & Opcodes.ACC_FINAL) == Opcodes.ACC_FINAL) {
                System.out.println("visit() run，类添加了 final 修饰符");
            }
            if (interfaces == null || interfaces.length == 0) {
                System.out.println("visit() run，name：" + name + "，superName：" + superName);
            } else {
                // 此处 demo 只实现了一个接口，因此直接取第一个元素。
                System.out.println(
                        "visit() run，name：" + name + "，superName：" + superName + "interfaces：" + interfaces[0]);
            }
        }

        @Override
        public void visitSource(String source, String debug) {
            System.out.println("visitSource() run，source：" + source + "，debug：" + debug);
            super.visitSource(source, debug);
        }

        @Override
        public AnnotationVisitor visitAnnotation(String descriptor, boolean visible) {
            System.out.println("visitAnnotation() run，descriptor：" + descriptor + "，运行期是否可见：" + visible);
            return super.visitAnnotation(descriptor, visible);
        }

        @Override
        public void visitInnerClass(String name, String outerName, String innerName, int access) {
            System.out.println(
                    "visitInnerClass() run，name：" + name + "，outerName：" + outerName + "，innerName：" + innerName);
            super.visitInnerClass(name, outerName, innerName, access);
        }

        @Override
        public FieldVisitor visitField(int access, String name, String descriptor, String signature, Object value) {
            System.out.println("visitField() run，name：" + name + "，descriptor：" + descriptor + "，value：" + value);
            return super.visitField(access, name, descriptor, signature, value);
        }

        @Override
        public MethodVisitor visitMethod(int access, String name, String desc, String signature, String[] exceptions) {
            if (exceptions != null) {
                System.out.println("visitMethod() run，name：" + name + "，desc：" + desc + "，exceptions：" + exceptions[0]);
            } else {
                System.out.println("visitMethod() run，name：" + name + "，desc：" + desc);
            }
            return super.visitMethod(access, name, desc, signature, exceptions);
        }

        @Override
        public AnnotationVisitor visitTypeAnnotation(int typeRef, TypePath typePath, String descriptor,
                boolean visible) {
            System.out.println("visitTypeAnnotation() run");
            return super.visitTypeAnnotation(typeRef, typePath, descriptor, visible);
        }

        @Override
        public void visitEnd() {
            System.out.println("visitEnd() run");
            super.visitEnd();
        }
    }
}
```

我们可以从输出的结果观察到在 Java 中类、方法、变量在字节码文件中的描述以及解析的顺序。具体输出如下：

```java
visit() run，类添加了 final 修饰符
visit() run，name：me/passin/asm/Demo$Passin，superName：java/lang/Objectinterfaces：java/io/Serializable  131120
visitSource() run，source：Demo.java，debug：null
visitAnnotation() run，descriptor：Ljava/lang/Deprecated;，运行期是否可见：true
visitInnerClass() run，name：me/passin/asm/Demo$Passin，outerName：me/passin/asm/Demo，innerName：Passin
visitField() run，name：field1，descriptor：I，value：null
visitField() run，name：field2，descriptor：I，value：1
visitMethod() run，name：<init>，desc：()V
visitMethod() run，name：method1，desc：(I)V，exceptions：java/lang/Exception
visitMethod() run，name：method2，desc：(Ljava/lang/Object;)F
visitMethod() run，name：method3，desc：()Ljava/lang/Object;
visitEnd() run
```

### 3.1.2 FieldVisitor

- visitAnnotation：访问字段上的注解；
- visitTypeAnnotation：访问字段的类型注解（Java 8 的特性，很少使用）；
- visitEnd：访问结束通知。

### 3.1.3 MethodVisitor

MethodVisitor 的方法比较多，其中以 Insn 结尾的方法，需要配合具体的 JVM 指令，具体的指令的意义可查看 [JVM 指令集](#122-jvm-%E6%8C%87%E4%BB%A4%E9%9B%86)

- visitParameter：访问方法参数；
- visitAnnotationDefualt：访问注解的默认值；
- visitAnnotaion：访问方法上的注解；
- visitTypeAnnotation：访问方法上的类型注解（Java 8 的特性，很少使用）；
- visitAnnotableParameterCount：访问方法参数中拥有注解参数的个数；
- visitParameterAnnotation：访问方法参数上的注解；
- visitCode：开始访问方法的代码；
- visitFrame：方法中局部变量的当前状态以及操作栈成员信息，该方法必须在 visitInsn 方法前调用；
- visitInsn：每一个指令执行都会调用；
- visitIntInsn：访问数值类型（非引用类型）操作数的指令；
- visitVarInsn：访问局部变量指令；
- visitTypeInsn：访问类型指令；
- visitFieldInsn：访问域（Field）指令；
- visitMethodInsn：访问方法操作指令；
- visitJumpInsn：访问跳转指令;
- visitLabel：访问 label，当会在调用该方法后访问该 label 标记的一个指令;
- visitLdcInsn：将 int、float 或 String 型常量值从常量池中推送至栈顶；
- visitIincInsn：访问局部 int 或 long 类型变量自增;
- visitTableSwitchInsn：访问 switch 关键字指令（case 值连续）；
- visitLookupSwitchInsn：访问 switch 关键字指令（case 值不连续）；
- visitMultiANewArrayInsn：访问多维数组指令；
- visitInsnAnnotation：访问指令注解，必须在访问注解之后调用；
- visitTryCatchBlock：访问 try--catch 代码块；
- visitTryCatchAnnotation：访问 try...catch 代码块上异常处理的类型注解，必须在调用 visitTryCatchBlock 之后调用；
- visitLocalVariable：访问局部地变量声明；
- visitLocalVariableAnnotation：访问本地局部地变量的类型注解（Java 8 的特性，很少使用）；
- visitLineNumber：访问行号；
- visitMaxs：访问操作数栈最大值和本地变量表最大值；
- visitEnd：访问结束通知。

### 3.1.4 AnnotationVisitor

- visit：访问注解的原始值；
- visitEnum：访问注解的枚举类型值；
- visitAnnotation：访问嵌套注解类型，也就是一个注解可能被其他注解所注释；
- visitArray: 访问注解的数组值；
- visitEnd：访问结束通知。

### 3.1.5 SignatureVisitor

SignatureVisitor 用于访问泛型。

- visitFormalTypeParameter(final String name)：访问正式类型参数；
- visitClassBound：访问最后一个被访问的正规类型参数的类界限；
- visitInterfaceBound：访问最后一个被访问的正规类型参数的接口界限；
- visitSuperclass：访问该类的超类；
- visitInterface：访问该类所实现的接口；
- visitParameterType：访问方法参数类型；
- visitReturnType：访问方法返回值类型；
- visitExceptionType：访问方法异常类型；
- visitBaseType：访问基本类型的签名；
- visitTypeVariable：访问类型变量的签名；
- visitArrayType：访问一个数组类型的签名；
- visitClassType：开始访问类或者接口类型的签名；
- visitInnerClassType：访问内部类类型；
- visitTypeArgument：访问上次访问的类或内部类类型的无限制类型参数；
- visitTypeArgument(final char wildcard)：访问上次访问的类或内部类类型的类型参数；
- visitEnd：访问结束通知。

# 四、ASM 实战

若对字节码指令不熟悉，可先编写 Java 代码，然后通过 [ASM Bytecode Viewer](https://plugins.jetbrains.com/plugin/10302-asm-bytecode-viewer/) 查看 Java 文件对应的字节码或者使用 ASM 生成该类编译成字节码文件时对应的代码。

所有的练习代码皆在该项目：https://github.com/passin95/ASM_Demo。

## 4.1 初探 ASM 

需求：对一个类插入一个常量、一个方法以及对一个方法插入代码块。

```java
// ASM 字节码插桩前对应的 Java 代码。
class Passin {

    public void insetCodeBlock() {
        System.out.println("insetCodeBlock run");
    }
}

// ASM 字节码插桩后对应的 Java 代。
class Passin {

    @NonNull
    public static final int field = 1;

    public void Method(int i) throws Exception {
        System.out.println("Hello World");
    }

    public void insetCodeBlock() {
        System.out.println("insetCodeBlock run");
    }
}
```

代码：

```java
public class Demo1 {

    static class Passin {

        public void insetCodeBlock() {
            System.out.println("insetCodeBlock run");
        }
    }

    public static void main(String[] args) throws Exception {
        // 读取对应文件或流的字节码。
        ClassReader classReader = new ClassReader("me/passin/asm/Demo1$Passin");
        // 第二个参数为 0 时，表明需要手动计算栈帧大小、局地变量和操作数栈的大小；
        // 为 ClassWriter.COMPUTE_MAXS 时，自动计算操作数栈大小和局部变量的最大个数，但需要自己计算栈帧大小；
        // 为 ClassWriter.COMPUTE_FRAMES 时，从头开始自动计算方法的堆栈映射框架，不需要调用 visitFrame() 和 visitMaxs()，即使调用也会被忽略。
        // 因此从上往下，性能越高也越麻烦。
        ClassWriter classWriter = new ClassWriter(classReader, ClassWriter.COMPUTE_FRAMES);
        CustomClassVisitor classVisitor = new CustomClassVisitor(Opcodes.ASM7, classWriter);
        classReader.accept(classVisitor, ClassReader.EXPAND_FRAMES);

        // 生成一个域（此处是常量），并添加相关关键字以及设置初始化值。
        FieldVisitor fieldVisitor = classWriter.visitField(ACC_PUBLIC | ACC_FINAL | ACC_STATIC, "field", "I", null, 1);
        // 添加域上的注解。
        AnnotationVisitor annotationVisitor = fieldVisitor.visitAnnotation("Landroidx/annotation/NonNull;", false);
        // 通知生成注解结束。
        annotationVisitor.visitEnd();
        // 通知生成变量结束。
        fieldVisitor.visitEnd();

        // 生成方法。
        MethodVisitor methodVisitor = classWriter.visitMethod(ACC_PUBLIC, "method", "(I)V", null, null);
        // 加载类 System 中的静态变量 out 并压入栈顶，该变量的描述是 Ljava/io/PrintStream;。
        methodVisitor.visitFieldInsn(GETSTATIC, "java/lang/System", "out", "Ljava/io/PrintStream;");
        // 将字符串 “Hello World” 常量池中压入栈顶。
        methodVisitor.visitLdcInsn("Hello World");
        // 调用对象 out 的 println 方法。注意顺序不能变，具体的顺序可查看上文的 JVM 指令集。
        methodVisitor
                .visitMethodInsn(INVOKEVIRTUAL, "java/io/PrintStream", "println", "(Ljava/lang/String;)V", false);
        // 当前方法返回 void。
        methodVisitor.visitInsn(Opcodes.RETURN);
        // 通知生成方法结束。
        methodVisitor.visitEnd();

        // 通知生成方法结束。
        classWriter.visitEnd();

        // 获得字节数组结果并输出到文件。
        byte[] newClassBytes = classWriter.toByteArray();
        // 获取项目根目录路径。
        String systemRootUrl = new File("").toURI().toURL().getPath();
        FileOutputStream fos = new FileOutputStream
                (systemRootUrl + "/app/src/main/java/me/passin/asm/Passin.class");
        fos.write(newClassBytes);
        fos.close();
    }

    static class CustomClassVisitor extends ClassVisitor {

        public CustomClassVisitor(int api, ClassVisitor classVisitor) {
            super(api, classVisitor);
        }

        @Override
        public MethodVisitor visitMethod(int access, String name, String descriptor, String signature,
                String[] exceptions) {
            // 如果方法名为 insetCodeBlock，则进行字节码插桩。
            if (name.equals("insetCodeBlock")) {
                // cv 为我们传入的 ClassWriter。
                MethodVisitor methodVisitor = cv.visitMethod(access, name, descriptor, signature, exceptions);
                return new CustomClassMethod(Opcodes.ASM7, methodVisitor);
            }

            return super.visitMethod(access, name, descriptor, signature, exceptions);
        }
    }

    static class CustomClassMethod extends MethodVisitor {

        public CustomClassMethod(int api, MethodVisitor methodVisitor) {
            super(api, methodVisitor);
        }

        @Override
        public void visitCode() {
            // 开始访问方法的代码时调用。
            mv.visitFieldInsn(GETSTATIC, "java/lang/System", "out", "Ljava/io/PrintStream;");
            mv.visitLdcInsn("insetCodeBlock head");
            mv.visitMethodInsn(INVOKEVIRTUAL, "java/io/PrintStream", "println", "(Ljava/lang/String;)V", false);
            super.visitCode();
        }

        @Override
        public void visitInsn(int opcode) {
            // 在 Return 指令前插入代码。
            if (opcode == Opcodes.RETURN) {
                mv.visitFieldInsn(GETSTATIC, "java/lang/System", "out", "Ljava/io/PrintStream;");
                mv.visitLdcInsn("insetCodeBlock foot");
                mv.visitMethodInsn(INVOKEVIRTUAL, "java/io/PrintStream", "println", "(Ljava/lang/String;)V", false);
            }
            super.visitInsn(opcode);
        }
    }
}
```

最后我们可以得到如下的 .class 文件：

```java
class Demo1$Passin {
    @NonNull
    public static final int field = 1;

    Demo1$Passin() {
    }

    public void insetCodeBlock() {
        System.out.println("insetCodeBlock head");
        System.out.println("insetCodeBlock run");
        System.out.println("insetCodeBlock foot");
    }

    public void method(int var1) {
        System.out.println("Hello World");
    }
}

```

## 4.2 统计方法耗时（Gradle + ASM）

需求：对需要统计耗时的方法添家指定注解 @MethodConsumedTime，并输出以下 Java 代码格式的 Log：

```
Log.d("MethodConsumedTime", 方法所处类 -> 方法描述：方法耗时);

```

**(1)** 自定义一个 Plugin

```groovy
class MethodConsumedTimePlugin extends Transform implements Plugin<Project> {

    @Override
    void apply(Project project) {
        def android = project.extensions.getByType(AppExtension)
        // 注册 Transform。
        android.registerTransform(this)
    }

    @Override
    String getName() {
        return "methodConsumedTime"
    }

    @Override
    Set<QualifiedContent.ContentType> getInputTypes() {
        return TransformManager.CONTENT_CLASS
    }

    @Override
    Set<QualifiedContent.Scope> getScopes() {
        return TransformManager.SCOPE_FULL_PROJECT
    }

    @Override
    boolean isIncremental() {
        return false
    }

    @Override
    void transform(TransformInvocation transformInvocation) throws TransformException, InterruptedException, IOException {
        super.transform(transformInvocation)
        println "===============asm visit start==============="

        def startTime = System.currentTimeMillis()

        transformInvocation.inputs.each { input ->
            input.directoryInputs.each { directoryInput ->

                if (directoryInput.file.isDirectory()) {
                    directoryInput.file.eachFileRecurse { File file ->
                        def fileName = file.name
                        if (isNeedInject(fileName)) {
                            ClassReader cr = new ClassReader(file.bytes)
                            ClassWriter cw = new ClassWriter(cr, ClassWriter.COMPUTE_MAXS)
                            // 具体的插桩代码在 MethodConsumedTimeVisitor 中。
                            ClassVisitor cv = new MethodConsumedTimeVisitor(cw)

                            cr.accept(cv, ClassReader.EXPAND_FRAMES)

                            byte[] code = cw.toByteArray()
                            // 直接覆盖掉原来的字节码文件。
                            FileOutputStream fos = new FileOutputStream(file)
                            fos.write(code)
                            fos.close()
                        }
                    }
                }

                def outputDirFile = transformInvocation.outputProvider.getContentLocation(
                        directoryInput.name, directoryInput.contentTypes, directoryInput.scopes, Format.DIRECTORY
                )

                FileUtils.copyDirectory(directoryInput.file, outputDirFile)
            }

            input.jarInputs.each { jarInput ->
                // jar 包不处理。
                def dest = transformInvocation.outputProvider.getContentLocation(
                        jarInput.name, jarInput.contentTypes, jarInput.scopes, Format.JAR
                )
                FileUtils.copyFile(jarInput.file, dest)
            }
        }

        def cost = System.currentTimeMillis() - startTime
        println "MethodConsumedTimePlugin cost $cost millisecond"
        println "===============asm visit end==============="
    }

    static isNeedInject(String name) {
        return name.endsWith(".class") && !name.startsWith("R\$") &&
                "R.class" != name && "BuildConfig.class" != name
    }

}
```

**（2）** 编写插桩代码

```groovy
class MethodConsumedTimeVisitor extends ClassVisitor {

    private String mClassName

    MethodConsumedTimeVisitor(ClassVisitor cv) {
        super(Opcodes.ASM6, cv)
    }

    @Override
    void visit(int version, int access, String name, String signature, String superName, String[] interfaces) {
        super.visit(version, access, name, signature, superName, interfaces)
        mClassName = name
    }

    @Override
    MethodVisitor visitMethod(int access, String methodName, String desc, String signature, String[] exceptions) {
        MethodVisitor mv = cv.visitMethod(access, methodName, desc, signature, exceptions);
        mv = new AdviceAdapter(api, mv, access, methodName, desc) {


            private boolean isNeedInserCode = false;
            private int timeLocalIndex = 0

            @Override
            AnnotationVisitor visitAnnotation(String desc1, boolean visible) {
                if (Type.getDescriptor(MethodConsumedTime.class) == desc1){
                    isNeedInserCode = true;
                }
                return super.visitAnnotation(desc1, visible);
            }

            @Override
            protected void onMethodEnter() {
                super.onMethodEnter()
                if (isNeedInserCode) {
                    // 创建一个 long 类型的局部变量，并返回该变量的索引（通过索引来确定指令的操作对象）。
                    timeLocalIndex = newLocal(Type.LONG_TYPE)
                    // 调用静态方法 System.currentTimeMillis()，拿到刚进入方法时的时间戳，并将返回的结果推到栈顶。
                    mv.visitMethodInsn(INVOKESTATIC, "java/lang/System", "currentTimeMillis", "()J", false)
                    // 将栈顶 long 类型数值存入指定本地变量。
                    mv.visitVarInsn(LSTORE, timeLocalIndex)
                }
            }

            @Override
            protected void onMethodExit(int opcode) {
                super.onMethodExit(opcode)
                if (isNeedInserCode) {
                    // 拿到执行完方法的时间戳。
                    mv.visitMethodInsn(INVOKESTATIC, "java/lang/System", "currentTimeMillis", "()J", false)
                    // 将标识为 timeLocalIndex 的本地变量推到栈顶。
                    mv.visitVarInsn(LLOAD, timeLocalIndex)
                    // 将栈顶两 long 类型出栈后数值相减并将结果压入栈顶，这个差值也就是该方法的耗时。
                    mv.visitInsn(LSUB)
                    // 将差值存入指定本地变量。
                    mv.visitVarInsn(LSTORE, timeLocalIndex)

                    // 将插入的代码：Log.d("MethodConsumedTime", 方法所处类 -> 方法描述：方法耗时);

                    mv.visitLdcInsn("MethodConsumedTime")
                    // 实例化 StringBuilder 对象
                    mv.visitTypeInsn(NEW, "java/lang/StringBuilder")
                    mv.visitInsn(DUP)
                    mv.visitMethodInsn(INVOKESPECIAL, "java/lang/StringBuilder", "<init>", "()V", false)
                    mv.visitLdcInsn(mClassName)
                    mv.visitMethodInsn(INVOKEVIRTUAL, "java/lang/StringBuilder", "append", "(Ljava/lang/String;)Ljava/lang/StringBuilder;", false)
                    mv.visitLdcInsn(" -> ")
                    mv.visitMethodInsn(INVOKEVIRTUAL, "java/lang/StringBuilder", "append", "(Ljava/lang/String;)Ljava/lang/StringBuilder;", false)
                    mv.visitLdcInsn(methodName + methodDesc + "：")
                    mv.visitMethodInsn(INVOKEVIRTUAL, "java/lang/StringBuilder", "append", "(Ljava/lang/String;)Ljava/lang/StringBuilder;", false)
                    mv.visitVarInsn(LLOAD, timeLocalIndex)
                    mv.visitMethodInsn(INVOKEVIRTUAL, "java/lang/StringBuilder", "append", "(J)Ljava/lang/StringBuilder;", false)
                    mv.visitLdcInsn("ms")
                    mv.visitMethodInsn(INVOKEVIRTUAL, "java/lang/StringBuilder", "append", "(Ljava/lang/String;)Ljava/lang/StringBuilder;", false)
                    mv.visitMethodInsn(INVOKEVIRTUAL, "java/lang/StringBuilder", "toString", "()Ljava/lang/String;", false)
                    // 此时栈从底到顶的数值为：MethodConsumedTime，方法所处类 -> 方法描述：方法耗时，再使用栈顶 2 个数值作为参数调用 Log.d()。
                    mv.visitMethodInsn(INVOKESTATIC, "android/util/Log", "d", "(Ljava/lang/String;Ljava/lang/String;)I", false)
                    mv.visitInsn(POP)
                }
            }

        }
        return mv
    }
}
```

**（3）** 应用 plugin。

测试：检测 MainActivity.onCreate() 方法耗时。

```java
public class MainActivity extends AppCompatActivity {

    @Override
    @MethodConsumedTime
    protected void onCreate(Bundle savedInstanceState) {
        super.onCreate(savedInstanceState);
        setContentView(R.layout.activity_main);
    }
}
```

在运行起来后，会在 IDE 中输出以下 Log：

```
2020-02-08 17:10:41.231 24860-24860/me.passin.asm D/MethodConsumedTime: me/passin/asm/MainActivity -> onCreate(Landroid/os/Bundle;)V：114ms
```

最后我们利用 [jadx](https://github.com/skylot/jadx) 反编译 apk ，验证具体插入的代码是否正确：

```java
public class MainActivity extends AppCompatActivity {
    /* access modifiers changed from: protected */
    public void onCreate(Bundle savedInstanceState) {
        long currentTimeMillis = System.currentTimeMillis();
        super.onCreate(savedInstanceState);
        setContentView((int) C0272R.layout.activity_main);
        long currentTimeMillis2 = System.currentTimeMillis() - currentTimeMillis;
        StringBuilder sb = new StringBuilder();
        sb.append("me/passin/asm/MainActivity");
        sb.append(" -> ");
        sb.append("onCreate(Landroid/os/Bundle;)V：");
        sb.append(currentTimeMillis2);
        sb.append("ms");
        Log.d("MethodConsumedTime", sb.toString());
    }
}
```