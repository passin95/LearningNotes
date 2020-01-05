


<!-- TOC -->

- [一、元注解](#%E4%B8%80%E5%85%83%E6%B3%A8%E8%A7%A3)
  - [1.1 @Retention](#11-retention)
  - [1.2 @Target](#12-target)
- [二、APT](#%E4%BA%8Capt)
  - [2.1 Module 结构](#21-module-%E7%BB%93%E6%9E%84)
  - [2.2 AbstractProcessor](#22-abstractprocessor)
    - [2.2.1 Element](#221-element)
    - [2.2.1 RoundEnvironment](#221-roundenvironment)
  - [2.3 JavaPoet 的使用](#23-javapoet-%E7%9A%84%E4%BD%BF%E7%94%A8)

<!-- /TOC -->

# 一、元注解

为了方便查阅，此处简述常用元注解的含义和取值。

元注解就是用来定义注解的注解，java.lang.annotation 提供了四种元注解：

- @Target：注解用于什么地方 
- @Retention：什么时候使用该注解
- @Documented：是否将注解信息添加在 JavaDoc 文档中
- @Inherited：是否允许子类继承该注解

## 1.1 @Retention

- RetentionPolicy.SOURCE：在编译阶段丢弃。这些注解在开始编译阶段就不再有任何意义，所以它们不会写入字节码。@Override, @SuppressWarnings 都属于这类注解。
- RetentionPolicy.CLASS：注释将由编译器记录在类文件中，但不会加载到 JVM 中。多用于写编译时注解框架，注解默认使用这种方式。
- RetentionPolicy.RUNTIME：注释将由编译器记录在类文件中，并在运行时也加载到 JVM 中，因此使用在需要反射机制读取该注解的信息的时候。

## 1.2 @Target

- ElementType.TYPE：类、接口 (包括注解类型) 或 Enum 声明
- ElementType.FIELD：成员变量、对象、属性（包括 Enum 实例）
- ElementType.METHOD：方法
- ElementType.PARAMETER：方法参数
- ElementType.CONSTRUCTOR：构造器
- ElementType.LOCAL_VARIABLE：局部变量
- ElementType.PACKAGE：用于描述包
- ElementType.ANNOTATION_TYPE：注解
- ElementType.TYPE_PARAMETER：类型参数（例如泛型）
- ElementType.TYPE_USE：类型使用声明

# 二、APT

APT（Annotation Processing Tool） 即为编译时注解处理器，可以用来在编译时扫描和处理注解。通过获取到注解和被注解对象的相关信息后，再根据需求自动生成类文件以及类中的代码，省去了手动编写。

## 2.1 Module 结构

- x-annotation：存放自定义注解。
- x-compiler：用于编写注解处理器，里面的代码不会被打包进 apk。
- x-api：对用户提供的 API，就是正常编写的代码以及 APT 生成的代码。

一般情况下 x-compiler 和 x-api 都会依赖 x-annotation。

除此之外在 x-compiler 模块，还会依赖 2 个库以方便开发。

```java
implementation 'com.google.auto.service:auto-service:1.0-rc6'
implementation 'com.squareup:javapoet:1.11.1'
```

- AutoService：会自动在 META-INF 文件夹下生成 Processor 配置信息文件，该文件里就是实现该服务接口的具体实现类。而当外部程序装配这个模块的时候，就能通过该 jar 包 META-INF/services/ 里的配置文件找到具体的实现类名，并装载实例化完成模块的注入。

- JavaPoet：用于方便生成 .java 源文件的 Java API。它的优势在于通过简单易懂的 Java API 在编译器生成代码。

## 2.2 AbstractProcessor 

```java
public abstract class BaseProcessor extends AbstractProcessor {

    protected Filer mFiler;
    protected Messager mMessager;
    protected Types mTypes;
    protected Elements mElements;

    @Override
    public synchronized void init(ProcessingEnvironment processingEnvironment) {
        super.init(processingEnvironment);

        mFiler = processingEnv.getFiler();
        mMessager = processingEnv.getMessager();
        mTypes = processingEnv.getTypeUtils();
        mElements = processingEnv.getElementUtils();
    }
    
    public boolean process(Set<? extends TypeElement> set, RoundEnvironment roundEnvironment) {

    }
}
```

- Filer：跟文件相关的辅助类，用于生成 JavaSourceCode；
- Elements：元素相关的辅助类，用于去获取一些元素相关的信息；
- Types：类型辅助类，用于判断类型之间是否存在某种关系；
- Messager：日志辅助类。

### 2.2.1 Element

Element接口族与Type接口族区别
Element所代表的元素只在编译期可见，用于保存元素在编译期的各种状态，而Type所代表的元素是运行期可见，用于保存元素在运行期的各种状态。

Element 有以下几种类型：

- VariableElement：成员变量、枚举、局部变量、方法参数；
- ExecutableElement：类中的方法；
- TypeElement：类、接口、注解；
- TypeParameterElement：类的泛型；
- PackageElement：包名。

Element 常用方法说明：

- Element.getKind()：获取元素类型；
- ExecutableElement.getReturnType().getKind()：获得方法返回类型；
- VariableElement.asType().getKind()：获得参数类型；

### 2.2.1 RoundEnvironment

```java
public interface RoundEnvironment {
    /**
     * 此 round 生成的类型不是以注释处理的后续 round 为准，则返回 true；否则返回 false。
     */
    boolean processingOver();
    /**
     * 在之前的其它 round 中发生错误，则返回 true；否则返回 false。
     */
    boolean errorRaised();
    /**
     * 返回 round 生成的注释处理根元素，例如 Activity 类对象，它的一级子元素则是变量、方法、内部类等，它的上级元素是包名。
     */
    Set<? extends Element> getRootElements();
    /**
     * 返回使用给定注释类型注释的元素。
     */
    Set<? extends Element> getElementsAnnotatedWith(TypeElement var1);
    /**
     * 返回给定注释类型注释的元素。
     */
    Set<? extends Element> getElementsAnnotatedWith(Class<? extends Annotation> var1);
}

```
## 2.3 JavaPoet 的使用

- MethodSpec：代表一个构造函数或方法声明。
- TypeSpec：代表一个类，接口，或者枚举。
- FieldSpec：代表一个成员变量，一个字段。
- JavaFile：包含一个顶级类的 Java 文件。

以下实例为了方便展示，将注解和响应的类文件都在一个 Module。

在实际开发中，可以使用 mElements.getTypeElement(类文件路径) 拿到相应的类信息，再通过 ClassName.get(Element) 拿到 ClassName。

```java
@AutoService(Processor.class)
@SupportedSourceVersion(SourceVersion.RELEASE_7)
@SupportedAnnotationTypes({"com.example.test.Inject"})
public class TestProcessor extends BaseProcessor {

    @Override
    public boolean process(Set<? extends TypeElement> set, RoundEnvironment roundEnvironment) {
        // 类名
        TypeSpec typeSpec = TypeSpec.classBuilder("HelloWorld")
                .addAnnotation(ClassAnnotation.class)
                .addModifiers(Modifier.PUBLIC)
                // 添加类的泛型申明
                .addTypeVariable(TypeVariableName.get("K", ClassName.get(String.class)))
                // 继承类
                .superclass(Test.class)
                // 实现接口
                .addSuperinterface(TestInterface.class)
                .addField(FieldSpec.builder(String.class, "a", Modifier.FINAL, Modifier.PUBLIC, Modifier.STATIC)
                        .initializer("$S", "123").build())
                // 普通语句块
                .addInitializerBlock(CodeBlock.builder().addStatement("//普通语句块").build())
                .addStaticBlock(CodeBlock.builder().addStatement("//静态语句块").build())
                // 添加一个方法
                .addMethod(MethodSpecExample())
                // 添加一个内部类
                .addType(TypeSpec.classBuilder("InnerClass").build())
                // 其它方法
                // 枚举类使用
//                .addEnumConstant("123")
                .build();

        try {
            JavaFile.builder("com.example.myapplication.impl", typeSpec)
                    .indent("    ")
                    .build()
                    .writeTo(mFiler);
        } catch (IOException e) {
        }

        return true;
    }

    private MethodSpec MethodSpecExample() {
        return MethodSpec.methodBuilder("methodName")
                .addJavadoc("这是注释\n")
                .addAnnotation(Nonnull.class)
                // 添加修饰符
                .addModifiers(Modifier.PUBLIC, Modifier.FINAL)
                // 添加方法的泛型申明
                .addTypeVariable(TypeVariableName.get("T", ClassName.get("com.example.test", "Test")))
                .addTypeVariable(TypeVariableName.get("B"))
                // 添加参数
                .addParameter(TypeVariableName.get("T", ClassName.get("com.example.test", "Test")), "T")
                .addParameter(TypeVariableName.get("B"), "B")
                .addParameter(TypeName.BOOLEAN, "booleanValue", Modifier.FINAL)
                .addParameter(TypeName.get(String[].class), "string")
                // 长度可变的参数，最后一个参数必须为数组
                .varargs()
                // 添加方法要抛出的异常
                .addException(TypeName.get(Exception.class))
                .addStatement("int result = 0")
                // 占位符 %S -> String  字符串
                //        $T -> Types 类、接口等
                //        %N -> Names（方便）
                //        %L -> double int long 类型变量
                .beginControlFlow("for (int i = $L; i < $L; i++)", 0, 10)
                .beginControlFlow("if (i > 5 && booleanValue)")
                .addStatement("result = result + i")
                .nextControlFlow("else if (i > 8)")
                .addStatement("result = result * i")
                .endControlFlow()
                .endControlFlow()
                .addStatement("return result")
                // 返回类型
                .returns(TypeName.INT)
                .build();
    }

}

```

生成的 java 代码：

```java
package com.example.myapplication.impl;

import com.example.test.ClassAnnotation;
import com.example.test.Test;
import com.example.test.TestInterface;
import java.lang.Exception;
import java.lang.String;
import javax.annotation.Nonnull;

@ClassAnnotation
public class HelloWorld<K extends String> extends Test implements TestInterface {
    public static final String a = "123";

    static {
        //静态语句块;
    }

    {
        //普通语句块;
    }

    /**
     * 这是注释
     */
    @Nonnull
    public final <T extends Test, B> int methodName(T T, B B, final boolean booleanValue,
            String... string) throws Exception {
        int result = 0;
        for (int i = 0; i < 10; i++) {
            if (i > 5 && booleanValue) {
                result = result + i;
            } else if (i > 8) {
                result = result * i;
            }
        }
        return result;
    }

    class InnerClass {
    }
}

```