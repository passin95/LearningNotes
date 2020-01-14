
<!-- TOC -->

- [一、AspectJ](#%E4%B8%80aspectj)
  - [1.1 语言](#11-%E8%AF%AD%E8%A8%80)
  - [1.2 JoinPoint](#12-joinpoint)
  - [1.3 Pointcuts](#13-pointcuts)
    - [1.3.1 通配符](#131-%E9%80%9A%E9%85%8D%E7%AC%A6)
    - [1.3.2 筛选方式](#132-%E7%AD%9B%E9%80%89%E6%96%B9%E5%BC%8F)
    - [1.3.3 过滤规则](#133-%E8%BF%87%E6%BB%A4%E8%A7%84%E5%88%99)
  - [1.4 Advice](#14-advice)
  - [1.5 实战](#15-%E5%AE%9E%E6%88%98)

<!-- /TOC -->

# 一、AspectJ

[AspectJ](https://www.eclipse.org/aspectj/) 是一个简单易上手的 AOP 工具，它可以通过编写一段通用的代码，然后根据 AspectJ 语法定义一套代码生成规则，AspectJ 就会帮你把这段代码插入到对应的位置。

## 1.1 语言

AspectJ 可支持以下两种语言进行编写：

**（1）** 完全使用 AspectJ 自身的语言，语法和 Java 很类似。

```java

public aspect AspectDemo {

    pointcut XX():
            execution(...);
    before(): XX() {
        //...
    }
    after(): XX() {
        //...
    }
}	
```

**（2）** 纯 Java 语言开发，需要配合注解使用，本文示例都使用该种语言。

```java

@Aspect
public class AspectDemo {

    @Pointcut("execution(...)")
    public void jointPoint() {
    }

    @Before("jointPoint()")
    public void before() {
		//...
    }

    @After("jointPoint()")
    public void after() {
		//...
    }
}

```

## 1.2 JoinPoint

JoinPoint（连接点）表示程序运行时的一些执行点。理论上说，在一个程序中几乎所有代码都可以被看做是 JoinPoint，但是在 AspectJ 1.9.4 中，只有以下情况被认为是 JoinPoint。我们可以通过 JoinPoint.getKind() 获取执行点的类型。

随着版本迭代可查阅 [文档](https://www.eclipse.org/aspectj/doc/released/runtime-api/index.html) 了解可能新增的 Join Point。

- METHOD_EXECUTION：函数执行。以 Log.d() 为例就是 Log.d() 中的代码；
- METHOD_CALL：函数调用。以 Log.d() 为例就是调用 Log.d() 函数；
- CONSTRUCTOR_EXECUTION：构造函数调用；
- CONSTRUCTOR_CALL：构造函数执行；
- FIELD_GET：获取某个成员变量；
- FIELD_SET：设置某个成员变量的值；
- STATICINITIALIZATION：类加载，包含了类加载过程中的所有过程，因此也包含了类中的 static{} 代码块。
- PREINITIALIZATION：对象预实例化。调用 new Object() 时，会先执行 METHOD_CALL Object()，表示先调用 Object() 再到预实例化。这个阶段不包括类中成员变量定义时就赋值的操作，也不包括构造函数中对某些成员变量进行的赋值操作。
- INITIALIZATION：对象实例化。先实例化成员变量，再开始执行构造函数中的方法。
- EXCEPTION_HANDLER：异常处理。比如 try catch(xxx) 中，对应 catch 内的执行。
- SYNCHRONIZATION_LOCK：进入同步锁。
- SYNCHRONIZATION_UNLOCK：退出同步锁。
- Advice execution：

## 1.3 Pointcuts

Pointcuts 可以理解为 JoinPoint 的过滤器，通过编写规则筛选出需要的 JoinPoint。

### 1.3.1 通配符

Pointcuts 规则的通配符很少，主要有以下三种：

- *：匹配除 . 之外的任意数量字符。

- ..：匹配任何数量字符的重复，如在类型模式中匹配任何数量子包；而在方法参数模式中匹配任何数量参数。

- +：匹配指定类型的子类型，仅能作为后缀放在类型模式后边。

### 1.3.2 筛选方式

Pointcuts 的筛选方式可分为 2 类：

**（1）** 与 JoinPoint 的类型密切相关：

<div align="center"> <img src="../pictures//JoinPoint%20与%20Pointcut%20的对应关系.webp"  width="650"/> </div>

即如果想选择类型为 Method execution 的 JoinPoint，则需要在 Pointcut 名称（就是方法）上添加注解 @Pointcut("execution(...)")

```java
@Pointcut("execution(...)")
public void pointcutName() {
}
```

**（2）** 与 JoinPoint 的类型无关：

- within(TypePattern)：表示某个 Package 或者类中的所有 JoinPoint，该种描述用得比较多；
- withincode(Constructor Signature|Method Signature)：表示某个构造函数或方法执行过程中涉及到的 JoinPoint；
- cflow(Pointcuts)：Pointcuts 包含的所有 JoinPoint 以及 Pintcuts 本身也是一个 JoinPoint；
- cflowbelow(Pointcuts)：Pointcuts 包含的所有 JoinPoint；
- this(Type)：Type 是一个类，即该类下所有的 JoinPoint；
- target(Type)：Type 是一个类，所有调用该类的 JoinPoint；
- args(TypeSignature)：例如 args(int,..)，表示第一个参数是 int，后面参数个数和类型不限的 JoinPoint。


### 1.3.3 过滤规则

Pointcuts 的过滤规则格式如下（后面带 ? 的表示可以省略）：

```
execution(修饰符? 返回值的类型 所属的类? 方法名 (参数列表) 抛出的异常?) 
```

其中对于参数列表规则：

- () 匹配一个不接受任何参数的方法；
- (..) 匹配一个接受任意数量参数的方法；
- (*) 匹配了一个接受一个任何类型的参数的方法；
- (*,String) 匹配了一个接受两个参数的方法，其中第一个参数是任意类型，第二个参数必须是 String 类型。

## 1.4 Advice

Advice 的类型有 3 种：before、after 以及 around。

- before(JoinPoint)：执行在具体的 JoinPoint 之前，没有返回值。
- after(JoinPoint)：执行在具体的 JoinPoint 之后，没有返回值。
- around(JoinPoint)：接管 JPoint 的执行，由开发者决定是否调用 joinPoint.proceed() 去决定原 JoinPoint 的逻辑，是否有返回值取决于 JoinPoint 为 METHOD_CALL 时，原方法是否有返回值。

## 1.5 实战

AspectJ 支持三种时期的织入：

1. java 编译时织入：利用 ajc 编译器替代 javac 编译器，直接对 source 文件（aj 文件、或@AspectJ 注解的 Java 文件）直接进行编译成 class 文件并将切面织入进代码。
2. java 编译后织入：利用 ajc 编译器对 javac 编译器编译后的 class 文件或 jar 文件织入切面代码，本质上是在对被选中的 JoinPoint 的前后加一些 hook 函数。
3. 运行时织入：不使用 ajc 编译器，利用 aspectjweaver.jar 工具，使用 java agent 在类加载期将切面织入进代码。

对于 Android 开发者使用的是第二种方式，本小节以实现 **被注解的方法防止 View 被连续点击** 作为需求简述开发步骤：

**（1）** 自定义注解。

```java
package me.passin.aspectj.annotation;

@Retention(RetentionPolicy.RUNTIME)
@Target(ElementType.METHOD)
public @interface SingleClick {

    long DEFAULT_INTERVAL_MILLIS = 1000;

    /**
     * @return 快速点击的间隔（ms）。
     */
    long value() default DEFAULT_INTERVAL_MILLIS;
}
```

**（2）** 编写 Pointcuts 规则，并编写织入的代码。

注：若要以 module 的形式直接进行集成或调试，需要加上第三步内容中对 variants 的配置。

```java
// 标记该类由 AspectJ 处理。
@Aspect
public class SingleClickAspectJ {

    // 被 @SingleClick 修饰的所有方法的 JoinPoint。
    @Pointcut("execution(@com.xuexiang.xaop.annotation.SingleClick * *(..))")
    public void method() {
    }  

    @Around("method() && @annotation(singleClick)")
    public void aroundJoinPoint(ProceedingJoinPoint joinPoint, SingleClick singleClick) throws Throwable {
        View view = null;
        // 获取上的方法参数。
        for (Object arg : joinPoint.getArgs()) {
            if (arg instanceof View) {
                view = (View) arg;
                break;
            }
        }
        if (view != null) {
            if (!ClickUtils.isFastDoubleClick(view, singleClick.value())) {
                // 不是快速点击，执行原方法。
                joinPoint.proceed();
            } else {
                Log.d("SingleClickAspectJ",Utils.getMethodDescribeInfo(joinPoint) + ":发生快速点击，View id:" + view.getId());
            }
        }
    }
}
```

**（3）** 新建一个 gradle plugin 模块，并在发布后 apply。

所需的第三方依赖：

```java
// ajc 编译器。
implementation 'org.aspectj:aspectjtools:1.8.9'
// aspectj 框架。
implementation 'org.aspectj:aspectjrt:1.8.9'
```

```java
class AOPPlugin implements Plugin<Project> {

    @Override
    void apply(Project project) {
        // 区分 application 和 library。
        def hasApp = project.plugins.withType(AppPlugin)
        def hasLib = project.plugins.withType(LibraryPlugin)
        if (!hasApp && !hasLib) {
            throw new IllegalStateException("'android' or 'android-library' plugin required.")
        }

        final def log = project.logger
        final def variants
        if (hasApp) {
            variants = project.android.applicationVariants
        } else {
            variants = project.android.libraryVariants
        }

        project.dependencies {
            // 若使用到了 aspectjrt 中的类，则改为 implementation。
            testImplementation 'org.aspectj:aspectjrt:1.8.9'
            // Implementation '发布的 runtime 代码'
        }
        
        // 若要以 module 的形式直接进行集成或调试，需要在 module 下的 build.gradle 添加以下配置。 
        variants.all { variant ->
            JavaCompile javaCompile = null
            if (variant.hasProperty('javaCompileProvider')) {
                // 针对 gradle 4.10.1 + 的兼容。
                TaskProvider<JavaCompile> provider = variant.javaCompileProvider
                javaCompile = provider.get()
            } else {
                javaCompile = variant.hasProperty('javaCompiler') ? variant.javaCompiler : variant.javaCompile
            }
            javaCompile.doLast {
                // 在 java 文件编译成 class 文件后，扫描所有被 @Aspect 的文件，使用 ajc 编译器进行织入。
                String[] args = ["-showWeaveInfo",
                                 "-1.8",
                                 "-inpath", javaCompile.destinationDir.toString(),
                                 "-aspectpath", javaCompile.classpath.asPath,
                                 "-d", javaCompile.destinationDir.toString(),
                                 "-classpath", javaCompile.classpath.asPath,
                                 "-bootclasspath", project.android.bootClasspath.join(File.pathSeparator)]
                log.debug "ajc args: " + Arrays.toString(args)

                MessageHandler handler = new MessageHandler(true);
                // 执行。
                new Main().run(args, handler);
                for (IMessage message : handler.getMessages(null, true)) {
                    switch (message.getKind()) {
                        case IMessage.ABORT:
                        case IMessage.ERROR:
                        case IMessage.FAIL:
                            log.error message.message, message.thrown
                            break;
                        case IMessage.WARNING:
                            log.warn message.message, message.thrown
                            break;
                        case IMessage.INFO:
                            log.info message.message, message.thrown
                            break;
                        case IMessage.DEBUG:
                            log.debug message.message, message.thrown
                            break;
                    }
                }
            }
        }
    }
}
```
