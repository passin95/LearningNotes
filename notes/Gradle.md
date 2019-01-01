

# Gradle

## 简介

Gradle 是基于 Groovy(语言) 的项目自动化构建开源工具。它可以管理项目中的差异(例如资源目录)、依赖、编译、打包、部署等等。

## Groovy

Groovy是一门 jvm语言，它最终是要编译成class文件然后在jvm上执行，所以Java语言的特性Groovy都支持，我们完全可以混写Java和Groovy。
Groovy提供了更加灵活简单的语法，大量的语法糖以及闭包特性可以让你用更少的代码来实现和Java同样的功能。

### Groovy API 文档

学习Groovy最好的方式便是查 API 文档，因此本文的Demo都以实例为主，更详细的用法应当按需查找 API 学习和使用。

[http://www.groovy-lang.org/api.html](http://www.groovy-lang.org/api.html)

[http://docs.groovy-lang.org/latest/html/groovy-jdk/index-all.html](http://docs.groovy-lang.org/latest/html/groovy-jdk/index-all.html)


### 数据类型

- Java中的基本数据类型
- Java中的对象
- Closure（闭包）
- 加强的List、Map等集合类型
- 加强的File、Stream等IO类型

类型可以显示声明，也可以用 def 来声明，用 def 声明的类型Groovy将会进行类型推断。

##### String

```groovy
def a = "hello"
def b = "world"
def c = "${a} ${b}"
println c

outputs:
hello world
```

##### Closure（闭包）

闭包类似于C语言中函数指针的东西，可以作为方法的参数和返回值，也可以作为一个变量而存在。

```groovy
def closure1 = { a, b ->
   a + b
}
// 如果闭包不指定参数，那么它会有一个隐含的参数 it
def closure2 = {
   println "find ${it}, I am a closure!"
}

println closure1(100,200)
println closure2(300)

outputs:
300
find 300, I am a closure!
```

##### List 和 Map

Groovy加强了Java中的集合类，比如List、Map、Set等。具体的实例如下：

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
// value 可直接修改为其它类型
test.id = "2" 
println test.id

outputs: 
2
```

##### IO 

在Groovy中，文件访问要比Java简单的多，不管是普通文件还是xml文件。

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

除了eachLine，File还提供了很多Java所没有的方法，需要浏览下大概有哪些方法，然后需要用的时候再去查就行了，这就是学习Groovy的方式。

而Groovy访问xml有两个类：XmlParser和XmlSlurper，二者几乎一样，主要是性别有些差别，XmlParser主要以 list的形式来表述GPath的，因此Node在可显性有着明显的优势,但需要额外的内存存储。而GPathResult采用iterators方式没有了额外空间消耗，但如果要访问最后一个node时候较费时。

```xml
<?xml version="1.0" encoding="utf-8"?>
<resources>
    <color name="colorPrimary">#F44336</color>
    <color name="colorPrimaryDark">#D32F2F</color>
    <color name="colorAccent">#536DFE</color>
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

### 语法和特性

#### 常规语法

- 语句后面的分号是可以省略的
- 变量的类型和方法的返回值也是可以省略的
- 方法调用时，括号也是可以省略的
- 甚至语句中的return都是可以省略的

在Groovy中很多东西都可以省略，他和kotlin的许多语法也相似，具体的写法选择一个适合自己或公司的写法便好。

```groovy
// 省略参数类型
int hello(msg) {
   // 省略括号
   println msg 
   // 省略 return
   1            
}
```

#### Class是一等公民

所有的Class类型对象，都可以省略.class。

```groovy
func(File.class)
func(File)

def func(Class clazz) {
}
```

#### Getter 和 Setter

下面两个类完全一致，只有有属性就有Getter/Setter；同理，只要有Getter/Setter，那么它就有隐含属性。

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

#### 判断是否为真

```groovy
if (name != null && name.length > 0) {}

if (name) {}
```

#### 简洁的三元表达式

```groovy
def result = name != null ? name : "Unknown"

def result = name ?: "Unknown"
```

#### 简洁的非空判断

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

#### 使用断言

使用assert来设置断言，当断言的条件为false时，程序将会抛出异常：

```groovy
def check(String name) {
    // name 不能为null或者empty
    assert name
    // name 不能为null或者empty 并且长度大于3
    assert name?.length() > 3
}
```

#### switch方法

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

#### ==和equals

在Groovy中，==相当于Java的equals，，如果需要比较两个对象是否是同一个，需要使用.is()。

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

```groovy

allprojects {
    repositories {
        google()
        jcenter()
    }
}

allprojects(new Action<Project>() {
    @Override
    void execute(Project project) {
        project.repositories{
            google()
            jcenter()
        }
    }
})
```

### Gradle 实战

#### 配置构建

```groovy

android {
    
    defaultConfig {
        applicationId "me.passin.pmvp.demo"
        minSdkVersion rootProject.ext.android["minSdkVersion"]
        targetSdkVersion rootProject.ext.android["targetSdkVersion"]
        versionCode rootProject.ext.android["versionCode"]
        versionName rootProject.ext.android["versionName"]
        testInstrumentationRunner rootProject.ext.dependencies["androidJUnitRunner"]

        flavorDimensions "versionCode"
        multiDexEnabled true
    }

    productFlavors {
        pay {
            applicationId "me.passin.pmvp.pay"
        }
        free {
            // 不写则默认使用 defaultConfig 的配置。
        }
    }

}
```

初始化阶段：setting.gradle 执行
定义阶段：所有setting.gradle中的gradle的执行
执行阶段：执行具体的task


# 参考资料

- [Configure your build](https://developer.android.com/studio/build/)
- [Gradle从入门到实战 - 任玉刚](https://blog.csdn.net/singwhatiwanna/article/details/76084580)

