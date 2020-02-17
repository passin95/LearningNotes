<!-- TOC -->

- [一、基础类型](#%E4%B8%80%E5%9F%BA%E7%A1%80%E7%B1%BB%E5%9E%8B)
  - [1.1 缓存池](#11-%E7%BC%93%E5%AD%98%E6%B1%A0)
  - [1.2 BigDecimal](#12-bigdecimal)
- [二、String](#%E4%BA%8Cstring)
  - [2.1 String 不可变的好处](#21-string-%E4%B8%8D%E5%8F%AF%E5%8F%98%E7%9A%84%E5%A5%BD%E5%A4%84)
  - [2.2 String, StringBuffer and StringBuilder](#22-string-stringbuffer-and-stringbuilder)
  - [2.3 new String("abc")](#23-new-string%22abc%22)
  - [2.4 String.intern()](#24-stringintern)
- [三、参数传递](#%E4%B8%89%E5%8F%82%E6%95%B0%E4%BC%A0%E9%80%92)
- [四、继承](#%E5%9B%9B%E7%BB%A7%E6%89%BF)
  - [4.1 访问权限](#41-%E8%AE%BF%E9%97%AE%E6%9D%83%E9%99%90)
  - [4.2 抽象类与接口](#42-%E6%8A%BD%E8%B1%A1%E7%B1%BB%E4%B8%8E%E6%8E%A5%E5%8F%A3)
    - [4.2.1 抽象类](#421-%E6%8A%BD%E8%B1%A1%E7%B1%BB)
    - [4.2.2 接口](#422-%E6%8E%A5%E5%8F%A3)
    - [4.2.3 比较](#423-%E6%AF%94%E8%BE%83)
  - [4.3 super](#43-super)
  - [4.4 重写与重载](#44-%E9%87%8D%E5%86%99%E4%B8%8E%E9%87%8D%E8%BD%BD)
- [五、Object 通用方法](#%E4%BA%94object-%E9%80%9A%E7%94%A8%E6%96%B9%E6%B3%95)
  - [5.1 equals](#51-equals)
    - [5.1.1 equals() 与 == 的区别](#511-equals-%E4%B8%8E--%E7%9A%84%E5%8C%BA%E5%88%AB)
    - [5.1.2 重写 equals()](#512-%E9%87%8D%E5%86%99-equals)
  - [5.2 hashCode()](#52-hashcode)
    - [5.2.1 hashCode 与 equals 的关系](#521-hashcode-%E4%B8%8E-equals-%E7%9A%84%E5%85%B3%E7%B3%BB)
    - [5.2.2 如何重写 hashCode](#522-%E5%A6%82%E4%BD%95%E9%87%8D%E5%86%99-hashcode)
  - [5.3 toString()](#53-tostring)
  - [5.4 clone()](#54-clone)
- [六、关键字](#%E5%85%AD%E5%85%B3%E9%94%AE%E5%AD%97)
  - [6.1 final](#61-final)
  - [6.2 static](#62-static)
- [七、反射](#%E4%B8%83%E5%8F%8D%E5%B0%84)
  - [7.1 反射的基本运用](#71-%E5%8F%8D%E5%B0%84%E7%9A%84%E5%9F%BA%E6%9C%AC%E8%BF%90%E7%94%A8)
    - [7.1.1 Class 对象的获取](#711-class-%E5%AF%B9%E8%B1%A1%E7%9A%84%E8%8E%B7%E5%8F%96)
    - [7.2.2 反射 API](#722-%E5%8F%8D%E5%B0%84-api)
  - [7.2 反射的优缺点](#72-%E5%8F%8D%E5%B0%84%E7%9A%84%E4%BC%98%E7%BC%BA%E7%82%B9)
- [八、异常](#%E5%85%AB%E5%BC%82%E5%B8%B8)
- [九、泛型](#%E4%B9%9D%E6%B3%9B%E5%9E%8B)
- [十、注解](#%E5%8D%81%E6%B3%A8%E8%A7%A3)
  - [10.1 元注解](#101-%E5%85%83%E6%B3%A8%E8%A7%A3)
    - [10.1.1 @Retention](#1011-retention)
    - [10.1.2 @Target](#1012-target)
    - [10.1.3 @Inherited](#1013-inherited)
- [十一、Type](#%E5%8D%81%E4%B8%80type)
  - [11.1 Class](#111-class)
  - [11.2 ParameterizedType](#112-parameterizedtype)
  - [11.3 GenericArrayType](#113-genericarraytype)
  - [11.4 TypeVariable](#114-typevariable)
- [十二、JDK 和 JRE](#%E5%8D%81%E4%BA%8Cjdk-%E5%92%8C-jre)
- [参考资料](#%E5%8F%82%E8%80%83%E8%B5%84%E6%96%99)

<!-- /TOC -->

# 一、基础类型

基础类型包括（类型/字节数）：

- byte/8
- char/16
- short/16
- int/32
- float/32
- long/64
- double/64
- boolean/~

基本类型都有对应的包装类型，基本类型与其对应的包装类型之间的赋值使用自动装箱与拆箱完成。

```java
Integer x = 2;     // 装箱
int y = x;         // 拆箱
```

## 1.1 缓存池

new Integer(123) 与 Integer.valueOf(123) 的区别在于：

new Integer(123) 每次都会新建一个对象，而 Integer.valueOf(123) 会优先使用缓存池中的对象，不存在再新建一个对象。
```java
public static Integer valueOf(int i) {
    if (i >= IntegerCache.low && i <= IntegerCache.high)
        return IntegerCache.cache[i + (-IntegerCache.low)];
    return new Integer(i);
}
```

编译器会在自动装箱过程调用 valueOf() 方法，因此多个值相同且值在缓存池范围内的 Integer 实例使用自动装箱来创建，那么就会引用相同的对象。

```java
Integer m = 123;
Integer n = 123;
System.out.println(m == n); // true
```

基本类型对应的缓冲池范围如下：

- int：-128 ~ 127
- short：-128 ~ 127
- byte：all
- boolean：true or false
- char：\u0000 ~ \u007F

## 1.2 BigDecimal

由于浮点数存在精度损失，因此在浮点数之间的等值判断中，基本数据类型不能用==来比较，包装数据类型不能用 equals 来判断。

因此需要使用 BigDecimal 来定义浮点数的值，再进行浮点数的运算操作。

```java
BigDecimal a = new BigDecimal("1.0");
BigDecimal b = new BigDecimal("0.9");
BigDecimal x = a.subtract(b); // 0.1
```

通过 **setScale()** 设置保留几位小数以及保留规则。

```java
BigDecimal m = new BigDecimal("1.23543");
BigDecimal n = m.setScale(3,BigDecimal.ROUND_HALF_DOWN);
System.out.println(n);// 1.235
```

注：实例化 BigDecimal 对象时，优先使用入参为 String 的构造方法或 BigDecimal.valueOf()，否则可能造成精度丢失（例如 double 会精度丢失）。

# 二、String

String 被声明为 final，因此它不可被继承。

在 Java 8 中，内部使用 char 数组存储数据，该数组被申明为 final，也就是说 String 不可变。

```java
public final class String
    implements java.io.Serializable, Comparable<String>, CharSequence {
    /** The value is used for character storage. */
    private final char value[];
}
```

在 Java 9 之后，String 类的实现改用 byte 数组存储字符串，同时使用 coder 来标识使用了哪种编码， byte[] 和 coder 被申明为 final，也就是说 String 同样不可变。

```java
public final class String
    implements java.io.Serializable, Comparable<String>, CharSequence {
    /** The value is used for character storage. */
    private final byte[] value;

    /** The identifier of the encoding used to encode the bytes in {@code value}. */
    private final byte coder;
}
```

## 2.1 String 不可变的好处

**（1）可以缓存 hash 值** 

因为 String 的 hash 值经常被使用，例如 String 用做 HashMap 的 key。不可变的特性可以使得 hash 值也不可变，因此只需要进行一次计算。

**（2）String Pool 的需要** 

如果一个 String 对象已经被创建过了，那么就会从 String Pool 中取得引用。只有 String 是不可变的，才可能使用 String Pool。

<div align ="center"> <img src ="../pictures//String%20缓存池.webp"/> </div><br>

**（3）线程安全** 

String 不可变性天生具备线程安全，可以在多个线程中安全地使用。

## 2.2 String, StringBuffer and StringBuilder

**（1）可变性** 

- String 不可变。
- StringBuffer 和 StringBuilder 可变。

**（2）线程安全** 

- String 不可变，因此是线程安全的。
- StringBuilder 不是线程安全的。
- StringBuffer 是线程安全的，内部使用 synchronized 来同步。

## 2.3 new String("abc")

使用这种方式一共会创建两个字符串对象（前提是 String Pool 中还没有 "abc" 字符串对象）。

1. "abc" 属于字符串字面量，因此编译时期会在 String Pool 中创建一个字符串对象，指向这个 "abc" 字符串字面量；
2. 生成 String 对象，并且 String 对象中的数组指向 "adc" 中的数组。

## 2.4 String.intern()

使用 String.intern() 可以保证相同内容的字符串变量引用相同的内存对象。

下面示例中，s1 和 s2 采用 new String() 的方式新建了两个不同对象，而 s3 是通过 s1.intern() 方法取得一个对象引用，这个方法首先把 s1 引用的对象放到 String Poll（字符串常量池）中，然后返回这个对象引用。因此 s3 和 s1 引用的是同一个字符串常量池的对象。

```java
String s1 = new String("aaa");
String s2 = new String("aaa");
System.out.println(s1 == s2);           // false
String s3 = s1.intern();
System.out.println(s1.intern() == s3);  // true
```

如果是采用 "bbb" 这种使用双引号的形式创建字符串实例，会自动地将新建的对象放入 String Poll 中。

```java
String s4 = "bbb";
String s5 = "bbb";
System.out.println(s4 == s5);  // true
```

在 Java 7 之前，字符串常量池被放在运行时常量池中，它属于永久代。而在 Java 7，字符串常量池被放在堆中。这是因为永久代的空间有限，在大量使用字符串的场景下会导致 OutOfMemoryError 错误。

# 三、参数传递

Java 的参数是以值传递的形式传入方法中，而不是引用传递。

Dog dog 的 dog 是一个指针，存储的是对象的地址。在将一个参数传入一个方法时，本质上是将对象的地址以值的方式传递到形参中。

```java
public class Dog {
    String name;

    Dog(String name) {
        this.name = name;
    }

    String getName() {
        return name;
    }

    String getObjectAddress() {
        return super.toString();
    }
}
```

```java
public class PassByValueExample {
    public static void main(String[] args) {
        Dog dog = new Dog("A");
        System.out.println(dog.getObjectAddress()); // Dog@4554617c
        func(dog);
        System.out.println(dog.getObjectAddress()); // Dog@4554617c
        System.out.println(dog.getName());          // A
    }

    private static void func(Dog dog) {
        System.out.println(dog.getObjectAddress()); // Dog@4554617c
        dog = new Dog("B");
        System.out.println(dog.getObjectAddress()); // Dog@74a14482
        System.out.println(dog.getName());          // B
    }
}
```

# 四、继承

## 4.1 访问权限

Java 中有三个访问权限修饰符：private、protected 以及 public，如果不加访问修饰符，表示包级可见。

可以对类或类中的成员（字段以及方法）加上访问修饰符。

- 成员可见表示其它类可以用这个类的实例访问到该成员。
- 类可见表示其它类可以用这个类创建对象。

protected 用于修饰成员，表示在继承体系中成员对于子类可见，但是这个访问修饰符对于类没有意义。

设计良好的模块会隐藏所有的实现细节，把它的 API 与它的实现清晰地隔离开来。模块之间只通过它们的 API 进行通信，一个模块不需要知道其他模块的内部工作情况，这个概念被称为信息隐藏或封装。因此访问权限应当尽可能地使每个类或者成员不被外界访问。

如果子类的方法覆盖了父类的方法，那么子类中该方法的访问级别不允许低于父类的访问级别。这是为了确保可以使用父类实例的地方都可以使用子类实例，也就是确保满足里氏替换原则。

字段决不能是公有的，因为这么做的话就失去了对这个字段修改行为的控制，客户端可以对其随意修改。可以使用公有的 getter 和 setter 方法来替换公有字段。

```java
public class AccessExample {
    public int x;
}
```

```java
public class AccessExample {
    private int x;

    public int getX() {
        return x;
    }

    public void setX(int x) {
        this.x = x;
    }
}
```

但是也有例外，如果是包级私有的类或者私有的嵌套类，那么直接暴露成员不会有特别大的影响。

```java
public class AccessWithInnerClassExample {
    private class InnerClass {
        int x;
    }

    private InnerClass innerClass;

    public AccessWithInnerClassExample() {
        innerClass = new InnerClass();
    }

    public int getValue() {
        return innerClass.x; // 直接访问
    }
}
```

## 4.2 抽象类与接口

### 4.2.1 抽象类

抽象类和抽象方法都使用 abstract 进行声明。抽象类一般会包含抽象方法，抽象方法一定位于抽象类中。

抽象类和普通类最大的区别是，抽象类不能被实例化，需要继承抽象类才能实例化其子类。

### 4.2.2 接口

接口是抽象类的延伸，在 Java 8 之前，它可以看成是一个完全抽象的类，也就是说它不能有任何的方法实现。

从 Java 8 开始，接口也可以拥有默认的方法实现，这是因为不支持默认方法的接口的维护成本太高了。在 Java 8 之前，如果一个接口想要添加新的方法，那么要修改所有实现了该接口的类。

接口的成员（字段 + 方法）默认都是 public 的，并且不允许定义为 private 或者 protected。

接口的字段默认都是 static 和 final 的。

### 4.2.3 比较

- 从设计层面上看，抽象类提供了一种 IS-A 关系，那么就必须满足里式替换原则，即子类对象必须能够替换掉所有父类对象。而接口更像是一种 LIKE-A 关系，它只是提供一种方法实现契约，并不要求接口和实现接口的类具有 IS-A 关系。
- 从使用上来看，一个类可以实现多个接口，但是不能继承多个抽象类。
- 接口的字段只能是 static 和 final 类型的，而抽象类的字段没有这种限制。
- 接口的方法只能是 public 的，而抽象类的方法可以由多种访问权限。

## 4.3 super

- 访问父类的构造函数：可以使用 super() 函数访问父类的构造函数，从而完成一些初始化的工作。
- 访问父类的成员：如果子类覆盖了父类的中某个方法的实现，可以通过使用 super 关键字来引用父类的方法实现。

```java
public class SuperExample {
    protected int x;
    protected int y;

    public SuperExample(int x, int y) {
        this.x = x;
        this.y = y;
    }

    public void func() {
        System.out.println("SuperExample.func()");
    }
}
```

```java
public class SuperExtendExample extends SuperExample {
    private int z;

    public SuperExtendExample(int x, int y, int z) {
        super(x, y);
        this.z = z;
    }

    @Override
    public void func() {
        super.func();
        System.out.println("SuperExtendExample.func()");
    }
}
```

```java
SuperExample e = new SuperExtendExample(1, 2, 3);
e.func();
```

```html
SuperExample.func()
SuperExtendExample.func()
```

## 4.4 重写与重载

- 重写（Override）存在于继承体系中，方法名、参数列表必须相同，返回值范围小于等于父类，抛出的异常范围小于等于父类，访问修饰符范围大于等于父类；如果父类方法访问修饰符为 private 则子类就不能重写该方法。

- 重载（Overload）存在于同一个类中，指一个方法与已经存在的方法名称上相同，但是参数类型、个数、顺序至少有一个不同。应该注意的是，返回值不同，其它都相同不算是重载。

# 五、Object 通用方法

```java
public final native Class<?> getClass()

public native int hashCode()

public boolean equals(Object obj)

protected native Object clone() throws CloneNotSupportedException

public String toString()

public final native void notify()

public final native void notifyAll()

public final native void wait(long timeout) throws InterruptedException

public final void wait(long timeout, int nanos) throws InterruptedException

public final void wait() throws InterruptedException

protected void finalize() throws Throwable {}
```
 
## 5.1 equals

### 5.1.1 equals() 与 == 的区别

- 对于基本类型，== 判断两个值是否相等，基本类型没有 equals() 方法。
- 对于引用类型，== 判断两个实例是否引用同一个对象，而 equals() 判断引用的对象是否等价。

```java
Integer x = new Integer(1);
Integer y = new Integer(1);
System.out.println(x.equals(y)); // true
System.out.println(x == y);      // false
```

### 5.1.2 重写 equals()

重写 equals() 的必要条件：

（一）自反性

```java
x.equals(x); // true
```

（二）对称性

```java
x.equals(y) == y.equals(x); // true
```

（三）传递性

```java
if (x.equals(y) && y.equals(z))
    x.equals(z); // true;
```

（四）一致性

多次调用 equals() 方法结果不变

```java
x.equals(y) == x.equals(y); // true
```

（五）与 null 的比较

对任何不是 null 的对象 x 调用 x.equals(null) 结果都为 false

```java
x.euqals(null); // false;
```

重写 equals() 的步骤：

- 检查是否为同一个对象的引用，如果是直接返回 true；
- 检查是否是同一个类型，如果不是，直接返回 false；
- 将 Object 实例进行转型；
- 判断每个关键域是否相等。

```java
public class EqualExample {
    private int x;
    private int y;
    private int z;

    public EqualExample(int x, int y, int z) {
        this.x = x;
        this.y = y;
        this.z = z;
    }

    @Override
    public boolean equals(Object o) {
        if (this == o) return true;
        if (o == null || getClass() != o.getClass()) return false;

        EqualExample that = (EqualExample) o;

        if (x != that.x) return false;
        if (y != that.y) return false;
        return z == that.z;
    }
}
```

## 5.2 hashCode()

hashCode() 的作用是获取哈希码，也称为散列码，作用是确定该对象在哈希表中的索引位置，也因此该方法在散列表中才有用，在其它情况下没用。

### 5.2.1 hashCode 与 equals 的关系

- 等价的两个实例散列值一定要相同，但是散列值相同的两个实例不一定等价。

- 在重写 equals() 时应当总是重写 hashCode()，保证等价的两个实例散列值也相等。

下面举例为什么需要同时重写这 2 个方法。

 新建了两个等价的实例，并将它们添加到 HashSet 中。我们希望将这两个实例是一样的，只在集合中添加一个实例，但是因为 EqualExample 没有重写 hashCode()，因此这两个实例的散列值是不同的，最终导致集合添加了两个等价的实例。

```java
EqualExample e1 = new EqualExample(1, 1, 1);
EqualExample e2 = new EqualExample(1, 1, 1);
System.out.println(e1.equals(e2)); // true
HashSet<EqualExample> set = new HashSet<>();
set.add(e1);
set.add(e2);
System.out.println(set.size());   // 2
```
### 5.2.2 如何重写 hashCode

理想的散列函数应当具有均匀性，即不相等的实例应当均匀分布到所有可能的散列值上。这就要求了散列函数要把所有域的值都考虑进来，可以将每个域都当成 R 进制的某一位，然后组成一个 R 进制的整数。R 一般取 31，因为它是一个奇素数，如果是偶数的话，当出现乘法溢出，信息就会丢失，因为与 2 相乘相当于向左移一位。

分别解析自定义类中与 equals 方法相关的域（变量），并以以下方式进行计算散列值。

单个域 a [hashCode] 的取值。

- boolean：true ？ 1 : 0
- byte/short/int/char：(int) a
- long：(int) (a ^ a>>>32)
- float：Float.hashCode(a)
- double：Double.hashCode(a)
- 引用类型：若为 null 则为 0，,否则递归调用该引用类型的 hashCode()。
- 容器类型：若为 null 则为 0，,否则对每个变量进行 result =  31 * result + [hashCode] 计算。
- 
```java
@Override
public int hashCode() {
    int result = 17;

    // 下面一行代码代表一个域
    result = 31 * result + [hashCode];
    result = 31 * result + [hashCode];
    result = 31 * result + [hashCode];

    return result;
}
```

## 5.3 toString()

默认返回 ToStringExample@4554617c 这种形式，其中 @ 后面的数值为散列码的无符号十六进制表示。

```java
public class ToStringExample {
    private int number;

    public ToStringExample(int number) {
        this.number = number;
    }
}
```

```java
ToStringExample example = new ToStringExample(123);
System.out.println(example.toString());
```

```html
ToStringExample@4554617c
```

## 5.4 clone()

clone() 是 Object 的 protect 方法，它不是 public，一个类不显式去重写 clone()，其它类就不能直接去调用该类实例的 clone() 方法。
我们可以通过重写 clone() 去实现浅拷贝或深拷贝。super.clone() 默认情况下实现对象的浅拷贝。

- 浅拷贝：当对象的属性是基本数据类型时，会复制属性及值，当对象的属性有引用类型的时候，会把当前属性引用复制。
- 深拷贝：当对象的属性是基本数据类型时，会复制属性及值，当对象的属性有引用类型的时候，会把当前属性引用的对象再复制一份。

即当需要 clone 的对象的属性都是基本数据类型（不包括数组），深拷贝浅拷贝一样，当需要 clone 的对象的属性有引用类型的时候，浅拷贝直接把引用地址复制过去，深拷贝会把引用的对象再复制一份。

```java
public class ShallowCloneExample implements Cloneable {
    private int[] arr;

    public ShallowCloneExample() {
        arr = new int[10];
        for (int i = 0; i < arr.length; i++) {
            arr[i] = i;
        }
    }

    public void set(int index, int value) {
        arr[index] = value;
    }

    public int get(int index) {
        return arr[index];
    }

    @Override
    protected ShallowCloneExample clone() throws CloneNotSupportedException {
        return (ShallowCloneExample) super.clone();
    }
}
```

```java
ShallowCloneExample e1 = new ShallowCloneExample();
ShallowCloneExample e2 = null;
try {
    e2 = e1.clone();
} catch (CloneNotSupportedException e) {
    e.printStackTrace();
}
e1.set(2, 111);
System.out.println(e2.get(2)); // 111
```

```java
public class DeepCloneExample implements Cloneable {
    private int[] arr;

    public DeepCloneExample() {
        arr = new int[10];
        for (int i = 0; i < arr.length; i++) {
            arr[i] = i;
        }
    }

    public void set(int index, int value) {
        arr[index] = value;
    }

    public int get(int index) {
        return arr[index];
    }

    @Override
    protected DeepCloneExample clone() throws CloneNotSupportedException {
        DeepCloneExample result = (DeepCloneExample) super.clone();
        result.arr = new int[arr.length];
        for (int i = 0; i < arr.length; i++) {
            result.arr[i] = arr[i];
        }
        return result;
    }
}
```

```java
DeepCloneExample e1 = new DeepCloneExample();
DeepCloneExample e2 = null;
try {
    e2 = e1.clone();
} catch (CloneNotSupportedException e) {
    e.printStackTrace();
}
e1.set(2, 111);
System.out.println(e2.get(2)); // 2
```

使用 clone() 方法来拷贝一个对象即复杂又有风险，它会抛出异常，并且还需要类型转换。Effective Java 书上讲到，最好不要去使用 clone()，可以使用拷贝构造函数或者拷贝工厂来拷贝一个对象。

```java
public class CloneConstructorExample {
    private int[] arr;

    public CloneConstructorExample() {
        arr = new int[10];
        for (int i = 0; i < arr.length; i++) {
            arr[i] = i;
        }
    }

    public CloneConstructorExample(CloneConstructorExample original) {
        arr = new int[original.arr.length];
        for (int i = 0; i < original.arr.length; i++) {
            arr[i] = original.arr[i];
        }
    }

    public void set(int index, int value) {
        arr[index] = value;
    }

    public int get(int index) {
        return arr[index];
    }
}
```

```java
CloneConstructorExample e1 = new CloneConstructorExample();
CloneConstructorExample e2 = new CloneConstructorExample(e1);
e1.set(2, 222);
System.out.println(e2.get(2)); // 2
```

# 六、关键字

## 6.1 final

**（1）数据** 

声明数据为常量，可以是编译时常量，也可以是在运行时被初始化后不能被改变的常量。

- 对于基本类型，final 使数值不变；
- 对于引用类型，final 使引用不变，也就不能引用其它对象，但是被引用的对象本身是可以修改的。

```java
final int x = 1;
// x = 2;  // cannot assign value to final variable 'x'
final A y = new A();
y.a = 1;
```

**（2）方法** 

声明方法不能被子类重写。

private 方法隐式地被指定为 final，如果在子类中定义的方法和基类中的一个 private 方法签名相同，此时子类的方法不是重写基类方法，而是在子类中定义了一个新的方法。

**（3）类** 

声明类不允许被继承。

## 6.2 static

**（1）静态变量** 

静态变量在内存中只存在一份，只在类初始化时赋值一次。

- 静态变量：类所有的实例都共享静态变量，可以直接通过类名来访问它；
- 实例变量：每创建一个实例就会产生一个实例变量，它与该实例同生共死。

```java
public class A {
    private int x;         // 实例变量
    private static int y;  // 静态变量

    public static void main(String[] args) {
        // int x = A.x;  // Non-static field 'x' cannot be referenced from a static context
        A a = new A();
        int x = a.x;
        int y = A.y;
    }
}
```

**（2）静态方法** 

静态方法在类加载的时候就存在了，它不依赖于任何实例，所以静态方法必须有实现，也就是说它不能是抽象方法（abstract）。

```java
public abstract class A {
    public static void func1(){
    }
    // public abstract static void func2();  // Illegal combination of modifiers: 'abstract' and 'static'
}
```

只能访问所属类的静态字段和静态方法，方法中不能有 this 和 super 关键字。

```java
public class A {
    
    private static int x;
    private int y;

    public static void func1(){
        int a = x;
        // int b = y;  // Non-static field 'y' cannot be referenced from a static context
        // int b = this.y;     // 'A.this' cannot be referenced from a static context
    }
}
```

**（3）静态代码块** 

静态语句块在类初始化时运行一次。

```java
public class A {
    
    static {
        System.out.println("123");
    }

    public static void main(String[] args) {
        A a1 = new A();
        A a2 = new A();
    }
}

```

```
123
```

**（4）静态内部类** 

非静态内部类依赖于外部类的实例，并持有外部类的引用，而静态内部类则不会。

```java
public class OuterClass {
    
    class InnerClass {
    }

    static class StaticInnerClass {
    }

    public static void main(String[] args) {
        // InnerClass innerClass = new InnerClass(); // 'OuterClass.this' cannot be referenced from a static context
        OuterClass outerClass = new OuterClass();
        InnerClass innerClass = outerClass.new InnerClass();
        StaticInnerClass staticInnerClass = new StaticInnerClass();
    }
}
```

静态内部类不能访问外部类的非静态的变量和方法。

**（5）静态导包**

```java
import static com.xxx.ClassName.*
```

在使用静态变量和方法时不用再指明 ClassName，从而简化代码，但可读性大大降低。

**（6）初始化顺序** 

静态变量和静态语句块优先于实例变量和普通语句块，静态变量和静态语句块的初始化顺序取决于它们在代码中的顺序。

```java
public static String staticField = "静态变量";
```

```java
static {
    System.out.println("静态语句块");
}
```

```java
public String field = "实例变量";
```

```java
{
    System.out.println("普通语句块");
}
```

最后才是构造函数的初始化。

```java
public InitialOrderTest() {
    System.out.println("构造函数");
}
```

存在继承的情况下，初始化顺序为：

- 父类（静态变量、静态语句块）
- 子类（静态变量、静态语句块）
- 父类（实例变量、普通语句块）
- 父类（构造函数）
- 子类（实例变量、普通语句块）
- 子类（构造函数）

# 七、反射

反射机制是指在运行状态中，对于任意一个类，都能够知道这个类的所有属性和方法；对于任意一个对象，都能够调用它的任意一个方法和属性。这种动态获取的信息以及动态调用对象的方法的功能称为 Java 的反射机制。

每个类都有一个 Class 对象，包含了与类有关的信息。当编译一个新类时，会产生一个同名的 .class 文件，该文件内容保存着 Class 对象。

Class 和 java.lang.reflect 一起对反射提供了支持。

## 7.1 反射的基本运用

### 7.1.1 Class 对象的获取

类加载相当于 Class 对象的加载，类在第一次使用时才动态加载到 JVM 中。

（1）使用 Class.forName() 方法

```java
// 这也是在反射需求中，常用的运用中获取 Class 对象的方式。
public static Class<?> forName(String className)
```

（2）某个类的 class

```java
Class<Integer> integerClass = int.class;
Class<Integer> integerClass1 = Integer.class;
```

（3）某个对象的 getClass() 方法

```java
String s = new String("123");
Class<?> aClass = s.getClass();
```

### 7.2.2 反射 API

**（1）判断是否为某个类的实例**

```java
public native boolean isInstance(Object obj);
```

**（2）创建实例**

- 调用 Class 对象的 newInstance()

```java
Class<?> c = String.class;
// 相当于 new 了一个无参对象
String str = (String) c.newInstance();
```

- 获取指定的 Constructor 对象，再调用 Constructor 对象的 newInstance() 方法

```java
Class<?> c = String.class;
// 获取 String 类带一个 String 参数的构造器
Constructor constructor = c.getConstructor(String.class);
// 根据构造器创建实例
Object obj = constructor.newInstance("123");
System.out.println(obj);
```

**（3）获取方法**

```java
// 返回类的所有 public 方法，包括其继承类的公用方法。
public Method[] getMethods() throws SecurityException

// 返回类或接口声明的所有方法，包括公共、保护、默认（包）访问和私有方法，但不包括继承的方法。
public Method[] getDeclaredMethods() throws SecurityException

// 返回一个特定的方法，其中第一个参数为方法名称，后面的参数为方法的参数对应 Class 的对象。方法的区间范围为上面描述的范围。
public Method getMethod(String name, Class<?>... parameterTypes)

public Method getDeclaredMethod(String name, Class<?>... parameterTypes)
```

**（4）获取类的成员变量信息**

```java
// 获取所有 public 的成员变量。
public Field[] getFields() 

// 所有已声明的成员变量，但不能得到其父类的成员变量。
public Field[] getDeclaredFields();

// 获取变量名为 name 的成员变量。
public Field getField(String name)
```

**（5）调用方法**

```java
// 当我们从类中获取了一个方法后，我们就可以用 invoke() 方法来调用这个方法。
// obj 为相应的 Method 对象，args 为具体的参数值。
public Object invoke(Object obj, Object... args)
        throws IllegalAccessException, IllegalArgumentException,
           InvocationTargetException
```

**（6）创建数组**

数组在 Java 里是比较特殊的一种类型，它可以赋值给一个 Object Reference。

```java
Class<?> cls = Class.forName("java.lang.String");
// 第一个参数为数组中的字段类型，第二个参数为容器大小
Object array = Array.newInstance(cls,25);
// 往数组里添加内容
Array.set(array,0,"hello");
Array.set(array,1,"Java");
Array.set(array,2,"fuck");
Array.set(array,3,"Scala");
Array.set(array,4,"Clojure");
// 获取某一项的内容
System.out.println(Array.get(array,3));
```

## 7.2 反射的优缺点

**（1）反射的优点**

1. 提高了程序的灵活性和扩展性，降低耦合性。它允许程序创和控制任何类的对象，无需提前硬编码目标类。
2. 调试器需要能够检查一个类里的私有成员。测试工具可以利用反射来自动地调用类里定义的可被发现的 API，以确保一组测试中有较高的代码覆盖率。

**（2）反射的缺点**

尽管反射非常强大，但也不能滥用。如果一个功能可以不用反射完成，那么最好就不用。在我们使用反射技术时，下面几条内容应该牢记于心。

1. 性能开销：反射涉及了动态类型的解析，所以 JVM 无法对这些代码进行优化。因此，反射操作的效率要比非反射操作低。我们应该避免在经常被执行的代码或对性能要求很高的程序中使用反射。

2. 安全限制：使用反射技术要求程序必须在一个没有安全限制的环境中运行。如果一个程序必须在有安全限制的环境中运行，如 Applet，那么就是个问题了。

3. 安全问题：由于反射可以忽略权限检查，因此可能会破坏封装性而导致安全问题.

# 八、异常

Throwable 是所有的异常的共同的父类，Throwable 分为两种子类： **Error**  和 **Exception**。

<div align ="center"> <img src ="../pictures//Java%20异常结构图.webp" /> </div><br>

其中 Error 表示不希望被程序捕获或者是程序无法处理的错误；Exception 用户程序可能捕捉的异常情况或者说是程序可以处理的异常。

Exception 又分为 **运行时异常（RuntimeException）** 和 **非运行时异常（RuntimeException 之外的异常）**。

Java 异常又可以分为不受检查异常（Unchecked Exception）和检查异常（Checked Exception）。

- 不受检查异常：编译器不要求强制处理的异常，除了 RuntimeException 及其子类以外，其他的 Exception 类和 Error 类都属于这种异常。
- 检查异常：则是编译器要求必须处置的异常。当程序中可能出现这类异常，要么使用 try-catch 语句进行捕获，要么用 throws 抛出，否则无法编译通过。

注意：当 try 语句和 finally 语句中都有 return 语句时，在方法返回之前，finally 语句的内容将被执行，并且 finally 语句的返回值将会覆盖原始的返回值。

# 九、泛型

```java
public class Box<T> {
    // T stands for "Type"
    private T t;
    public void set(T t) { this.t = t; }
    public T get() { return t; }
}
```

- 泛型是通过类型擦除来实现的，编译器在编译时擦除了所有类型相关的信息，然后在运行期拿到泛型元数据进行隐式强转。
- 泛型通过 IDE 的支持，可以尽量保证编译期的类型安全（例如 List 可以而 Array 却不能）。

- [10 道 Java 泛型面试题](https://cloud.tencent.com/developer/article/1033693)

# 十、注解

Java 注解是附加在代码中的一些元信息，用于一些工具在编译、运行时进行解析和使用，起到说明、配置的功能。注解不会也不能影响代码的实际逻辑，仅仅起到辅助性的作用。

- [注解 Annotation 实现原理与自定义注解例子](https://www.cnblogs.com/acm-bingzi/p/javaAnnotation.html)

## 10.1 元注解

元注解就是用来定义注解的注解，java.lang.annotation 提供了四种元注解：

- @Target：注解用于什么地方；
- @Retention：什么时候使用该注解；
- @Documented：是否将注解信息添加在 JavaDoc 文档中；
- @Inherited：是否允许子类继承该注解。

### 10.1.1 @Retention

- RetentionPolicy.SOURCE：在 javac 编译之后丢弃，因此它们不会写入字节码。@Override, @SuppressWarnings 都属于这类注解，APT 也可使用该注解。
- RetentionPolicy.CLASS：注释将由编译器记录在类文件中，但不会加载到 JVM 中。注解默认使用这种方式。与 SOURCE 的作用区别在于，CLASS 修饰的注解可以作为字节码修改或插桩的依据。
- RetentionPolicy.RUNTIME：注释将由编译器记录在类文件中，并在运行时也加载到 JVM 中，因此使用在需要反射机制读取该注解信息的时候。

### 10.1.2 @Target

- ElementType.TYPE：类、接口 (包括注解类型) 或 Enum 声明；
- ElementType.FIELD：成员变量、对象、属性（包括 Enum 实例）；
- ElementType.METHOD：方法；
- ElementType.PARAMETER：方法参数；
- ElementType.CONSTRUCTOR：构造器；
- ElementType.LOCAL_VARIABLE：局部变量；
- ElementType.PACKAGE：用于描述包；
- ElementType.ANNOTATION_TYPE：注解；
- ElementType.TYPE_PARAMETER：类型参数（例如泛型）；
- ElementType.TYPE_USE：类型使用声明。

### 10.1.3 @Inherited 

@Inherited 元注解是一个标记注解，@Inherited 阐述了某个被标注的类型是被继承的。如果一个使用了@Inherited 修饰的 annotation 类型被用于一个 class，则这个 annotation 将被用于该 class 的子类。

# 十一、Type

Type 是 Java 中所有类型的公共超级接口。
Type 体系包括的类型：**原始类型**(raw types) 对应 Class，**参数化类型**(parameterized types) 对应 ParameterizedType，**数组类型**(array types) 对应 GenericArrayType，**类型变量**(type variables) 对应 TypeVariable，**基本类型**(primitive types) 对应 Class。
- 原始类型：不仅仅包含我们平常所指的类，还包括枚举、数组、注解等。
- 参数化类型：List<T>、Map<K,V> 等带有参数化的容器。
- 数组类型：不是 String[] 、byte[] 等数组，而是带有泛型的数组 T[]。
- 类型变量：泛型中的变量，例如 public class Demo<T>{} ，则 T 是类型变量。
- 基本类型：java 的基本类型，即 int,float,double 等。

## 11.1 Class

Class 不是一个接口，而是对 Type 的一个实现类,是 Java 反射的基础，对 Java 类的抽象。

## 11.2 ParameterizedType

参数化类型，即使用了泛型的类，并且没有使用通配符。

```java
public interface ParameterizedType extends Type {
    /**
     * 以 Type[] 形式返回所有泛型的实际类型。
     * 例如 Map<String,User>，调用该方法则返回数组 Type[2]，
     * 其中 Type[0] 为 String，Type[1] 为 User。
     */
    Type[] getActualTypeArguments();

    /**
     * 返回表示类或接口的对象 {@code Type}。
     * 例如 Map<K,V> 调用该方法，则返回接口 Map。
     */
    Type getRawType();

    /**
     * 返回该 Type 的所属者，也可以理解为内部类的所属者即为外部类。
     * 例如 Map.Entry<String,Integer> 调用该方法，则返回接口 Map。
     */
    Type getOwnerType();
}
```

## 11.3 GenericArrayType

泛型数组类型，例如 List<String>[],T[]，而 List<String> 不属于 GenericArrayType（属于 ParameterizedType），String[] 不属于 GenericArrayType（属于原始类型）。

```java
public interface GenericArrayType extends Type {
    /**
     * 返回泛型数组元素的 Type 类型。
     * 例如 List<String>[]，元素为 List<String>，List<String> 属于 ParameterizedType，该方法则返回 ParameterizedType。
     */
    Type getGenericComponentType();
}
```

## 11.4 TypeVariable

TypeVariable 指的是泛型的类型变量，指的是 List<T>、Map<K,V> 的 T、K、V 值，实际的 Java 类型是 TypeVariableImpl（TypeVariable 的子类）。

```java
public interface TypeVariable<D extends GenericDeclaration> extends Type {
    /**
     * 返回一个 Type[]，它的内容是该类型变量的上限集，也就是泛型中 extend 右边的值，如果没有使用 extend，则默认为 {@code Object}，
     * 例如 public class Demo <T extends Number & Serializable>{T t}，则返回数组 Type[2]，其中 Type[0] 为 Number，Type[1] 为 Serializable。
    */
    Type[] getBounds();

    /**
     * 获取声明该类型变量实体。
     * 例如 public class Demo<T> 中的 Demo。
     */
    D getGenericDeclaration();

    /**
     * 获取类型变量在源码中定义的名称。
     * 例如 public class Demo<T> 中的 "T"。
     */
    String getName();

}
```

# 十二、JDK 和 JRE

JDK 全名 Java Development Kit，它是功能齐全的 Java SDK。它拥有 JRE 所拥有的一切，还有编译器（javac）和工具（如 javadoc 和 jdb）。它能够创建和编译程序。

JRE 全名 java runtime environment，是 Java 程序的运行环境。它包含了运行已编译 Java 程序所需的所有内容的集合，包括 Java 虚拟机（JVM），Java 类库，Java 命令和其他的一些基础构件。但是，它不能用于创建新程序。

# 参考资料

- Bloch J. Effective java[M]. Addison-Wesley Professional, 2017.
- [CS-Notes. Java 基础](https://github.com/CyC2018/CS-Notes/blob/master/docs/notes/Java%20%E5%9F%BA%E7%A1%80.md)