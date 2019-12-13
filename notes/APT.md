


<!-- TOC -->

- [一、元注解](#一元注解)
    - [1.1 @Retention](#11-retention)
    - [1.2 @Target](#12-target)
- [二、APT 实现](#二apt-实现)
    - [2.1 Module 结构](#21-module-结构)
    - [2.2 AbstractProcessor](#22-abstractprocessor)
        - [2.2.1 Element](#221-element)
    - [2.3 JavaPoet 的使用](#23-javapoet-的使用)

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
- RetentionPolicy.CLASS：在类加载的时候丢弃。多用于写编译时注解框架，注解默认使用这种方式。
- RetentionPolicy.RUNTIME：始终不会丢弃，运行期也保留该注解，因此使用在需要反射机制读取该注解的信息的时候。

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

# 二、APT 实现

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

- AutoService：会自动在META-INF文件夹下生成 Processor 配置信息文件，该文件里就是实现该服务接口的具体实现类。而当外部程序装配这个模块的时候，就能通过该 jar 包 META-INF/services/ 里的配置文件找到具体的实现类名，并装载实例化完成模块的注入。除吃

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
}
```

- Filer：跟文件相关的辅助类，用于生成 JavaSourceCode；
- Elements：元素相关的辅助类，用于去获取一些元素相关的信息；
- Types：类型辅助类，用于判断类型之间是否存在某种关系；
- Messager：日志辅助类。

### 2.2.1 Element

- VariableElement：成员变量、枚举、局部变量、方法参数；
- ExecutableElement：类中的方法；
- TypeElement：类、接口、注解；
- TypeParameterElement：类的泛型；
- PackageElement：包名。

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