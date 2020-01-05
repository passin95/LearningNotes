<!-- TOC -->

- [一、Groovy](#%E4%B8%80groovy)
  - [1.1 数据类型](#11-%E6%95%B0%E6%8D%AE%E7%B1%BB%E5%9E%8B)
    - [1.1.1 String](#111-string)
    - [1.1.2 Closure（闭包）](#112-closure%E9%97%AD%E5%8C%85)
    - [1.1.3 List 和 Map](#113-list-%E5%92%8C-map)
    - [1.1.4 IO](#114-io)
  - [1.2 语法和特性](#12-%E8%AF%AD%E6%B3%95%E5%92%8C%E7%89%B9%E6%80%A7)
    - [1.2.1 常规语法](#121-%E5%B8%B8%E8%A7%84%E8%AF%AD%E6%B3%95)
    - [1.2.2 Class](#122-class)
    - [1.2.3 Getter 和 Setter](#123-getter-%E5%92%8C-setter)
    - [1.2.4 判断是否为真](#124-%E5%88%A4%E6%96%AD%E6%98%AF%E5%90%A6%E4%B8%BA%E7%9C%9F)
    - [1.2.5 简洁的三元表达式](#125-%E7%AE%80%E6%B4%81%E7%9A%84%E4%B8%89%E5%85%83%E8%A1%A8%E8%BE%BE%E5%BC%8F)
    - [1.2.6 简洁的非空判断](#126-%E7%AE%80%E6%B4%81%E7%9A%84%E9%9D%9E%E7%A9%BA%E5%88%A4%E6%96%AD)
    - [1.2.7 使用断言](#127-%E4%BD%BF%E7%94%A8%E6%96%AD%E8%A8%80)
    - [1.2.8 switch 方法](#128-switch-%E6%96%B9%E6%B3%95)
    - [1.2.9 == 和 equals](#129--%E5%92%8C-equals)
- [参考资料](#%E5%8F%82%E8%80%83%E8%B5%84%E6%96%99)

<!-- /TOC -->


# 一、Groovy 

Groovy 是一门 jvm 语言，它最终是要编译成 class 文件然后在 jvm 上执行，所以 Java 语言的特性 Groovy 都支持，我们完全可以混写 Java 和 Groovy。

Groovy 提供了更加灵活简单的语法，大量的语法糖以及闭包特性可以让你用更少的代码来实现和 Java 同样的功能。Groovy 的部分语法和 Kotlin 也极其相似。

## 1.1 数据类型

- Java 中的基本数据类型
- Java 中的对象
- Closure（闭包）
- 加强的 List、Map 等集合类型
- 加强的 File、Stream 等 IO 类型

类型可以显式声明，也可以用 def 来声明，用 def 声明的类型 Groovy 将会进行类型推断。

### 1.1.1 String

```groovy
def a = "hello"
def b = "world"
def c = "${a} ${b}"
println c

outputs:
hello world
```

### 1.1.2 Closure（闭包）

groovy 的闭包类似于 C 语言中函数指针的东西，可以作为方法的参数和返回值，甚至可以作为方法的委托存在。

```groovy
def closure1 = { a, b ->
   a + b
}
// 如果闭包不指定参数，那么它会有一个隐含的参数 it。
def closure2 = {
   println "find ${it}, I am a closure!"
}

println closure1(100,200)
println closure2(300)

outputs:
300
find 300, I am a closure!
```

再例如 Android 项目根目录的 build.gradle 中的代码：

```groovy
allprojects {
    repositories {
        google()
        jcenter()
    }
}
```

等价于

```groovy

allprojects (new Action<Project>() {
    @Override
    void execute(Project project) {
        repositories {
            google()
            jcenter()
        }
    }
})
```

### 1.1.3 List 和 Map

Groovy 加强了 Java 中的集合类，比如 List、Map、Set 等。具体的实例如下：

List:
```groovy
def test = [100, "hello", true]
test[1] = "world"
println test[0]
println test[1]
test << 200 // 列表尾部添加一个元素 200
println test.size

outputs:
100
world
4
```

Map:
```groovy
def test = ["id":1, "name":"passin"]
// value 可直接修改为其它类型。
test.id = "2" 
println test.id

outputs: 
2
```

### 1.1.4 IO 

在 Groovy 中，文件访问要比 Java 简单的多，不管是普通文件还是 xml 文件。

```
text.txt 文本内容：
345
123
234
```

```groovy
def file = new File("test.txt")
println "read file using two parameters"
file.eachLine {line,lineNo ->
    println("${lineNo} ${line}")
}
println "read file using one parameters"
file.eachLine { line ->
    println "${line}"
}

outputs: 
read file using two parameters
1 345
2 123
3 234
read file using one parameters
345
123
234    
```

除了 eachLine，File 还提供了很多 Java 所没有的方法，需要浏览下大概有哪些方法，然后需要用的时候再去查即可。

而 Groovy 访问 xml 有两个类：XmlParser 和 XmlSlurper，二者几乎一样，主要是性别有些差别，XmlParser 主要以 list 的形式来表述 GPath，而 GPathResult 采用 iterators 方式没有了额外空间消耗，但如果要访问最后一个 node 时候相对费时。

```xml
<?xml version ="1.0" encoding ="utf-8"?>
<resources>
    <color name ="colorPrimary">#F44336</color>
    <color name ="colorPrimaryDark">#D32F2F</color>
    <color name ="colorAccent">#536DFE</color>
</resources>
```

```groovy
def xml = new XmlParser().parse(new File("colors.xml"))
println xml['color'].@name[0]
println xml.color[1].@name
println xml.color[2].text()

outputs: 
colorPrimary
colorPrimaryDark
#536DFE
```

## 1.2 语法和特性

### 1.2.1 常规语法

- 语句后面的分号是可以省略的
- 变量的类型和方法的返回值也是可以省略的
- 方法调用时，括号也是可以省略的
- 甚至语句中的 return 都是可以省略的

```groovy
// 省略参数类型。
int hello(msg) {
   // 省略括号。
   println msg 
   // 省略 return。
   1            
}
```

### 1.2.2 Class

所有的 Class 类型对象，都可以省略.class。

```groovy
func(File.class) // 等价于 func(File)

def func(Class clazz) {
}
```

### 1.2.3 Getter 和 Setter

下面两个类完全一致，Groovy 会默认为属性生成 Getter/Setter。

```groovy
class Book {
    private String name;

    String getName() { return name }

    void setName(String name) { this.name = name }
}

class Book {
    String name
}
```

### 1.2.4 判断是否为真

```groovy
if (name != null && name.length > 0) {}

if (name) {}
```

### 1.2.5 简洁的三元表达式

```groovy
def result = name != null ? name : "Unknown"

def result = name ?: "Unknown"
```

### 1.2.6 简洁的非空判断

```groovy
if (order != null) {
    if (order.getCustomer() != null) {
        if (order.getCustomer().getAddress() != null) {
            System.out.println(order.getCustomer().getAddress());
        }
    }
}

println order?.customer?.address
```

### 1.2.7 使用断言

使用 assert 来设置断言，当断言的条件为 false 时，程序将会抛出异常：

```groovy
def check(String name) {
    // name 不能为 null 或者 empty。
    assert name
    // name 不能为 null 或者 empty 并且长度大于 3。
    assert name?.length() > 3
}
```

### 1.2.8 switch 方法

```groovy
def switchTest(def x) {
    def result = ""
    switch (x) {
        case "foo": result = "found foo "
        case "bar": result += "bar"
            break
        case [4, 5, 6, 'inList']: result = "list"
            break
        case 12..30: result = "range"
            break
        case Integer: result = "integer"
            break
        case Number: result = "number"
            break
        case { it > 3 }: result = "number > 3"
            break
        default: result = "default"
    }
    return result
}
```

### 1.2.9 == 和 equals

在 Groovy 中，== 相当于 Java 的 equals，，如果需要比较两个对象是否是同一个，需要使用.is()。

```groovy
class Book {
    private String name;

    Book(String name) {
        this.name = name
    }

    @Override
    protected Object clone() throws CloneNotSupportedException {
        return new Book(name)
    }
}

def a = new Book()
def b = a.clone()
assert a == b
assert a.is(b)
```


# 参考资料

- [Gradle 从入门到实战 - 任玉刚 ](https://blog.csdn.net/singwhatiwanna/article/details/76084580)