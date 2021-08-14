


<!-- TOC -->

- [一、元注解](#一元注解)
    - [1.1 @Retention](#11-retention)
    - [1.2 @Target](#12-target)
- [二、APT](#二apt)
    - [2.1 Module 结构](#21-module-结构)
    - [2.2 AbstractProcessor](#22-abstractprocessor)
    - [2.3 Element](#23-element)
        - [2.3.1 基础接口 Element](#231-基础接口-element)
        - [2.3.2 VariableElement](#232-variableelement)
        - [2.3.3 TypeParameterElement](#233-typeparameterelement)
        - [2.3.4 ExecutableElement](#234-executableelement)
        - [2.3.5 TypeElement](#235-typeelement)
        - [2.3.6 PackageElement](#236-packageelement)
    - [2.4 JavaPoet 的使用](#24-javapoet-的使用)
- [三、手写 ButterKnife](#三手写-butterknife)
    - [3.1 butterknife-annotations](#31-butterknife-annotations)
    - [3.2 butterknife-api](#32-butterknife-api)
    - [3.3 butterknife-compiler](#33-butterknife-compiler)

<!-- /TOC -->

# 一、元注解

为了方便查阅，此处简述常用元注解的含义和取值。

元注解就是用来定义注解的注解，java.lang.annotation 提供了四种元注解：

- @Target：注解用于什么地方；
- @Retention：注解的生命周期；
- @Documented：是否将注解信息添加在 JavaDoc 文档中；
- @Inherited：是否允许子类继承该注解。

## 1.1 @Retention

- RetentionPolicy.SOURCE：在 javac 编译之后丢弃，因此它们不会写入字节码。@Override, @SuppressWarnings 都属于这类注解，APT 也可使用该注解；
- RetentionPolicy.CLASS：注释将由编译器记录在字节码文件中，但不会加载到 JVM 中。注解默认使用这种方式。与 SOURCE 的作用区别在于，CLASS 修饰的注解可以作为字节码修改或插桩的依据；
- RetentionPolicy.RUNTIME：注释将由编译器记录在字节码文件中，并在运行时也加载到 JVM 中，因此使用在需要反射机制读取该注解信息的时候。

## 1.2 @Target

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

# 二、APT

APT（Annotation Processing Tool） 即为编译时注解处理器，可以用来在编译时扫描和处理注解。通过获取到注解和被注解对象的相关信息后，再根据需求自动生成类文件以及类中的代码，省去了手动编写。

## 2.1 Module 结构

- x-annotation：存放自定义注解；
- x-compiler：用于编写注解处理器，里面的代码不会被打包进 apk；
- x-api：对用户提供的 API，就是正常编写的代码以及 APT 生成的代码。

一般情况下 x-compiler 和 x-api 都会依赖 x-annotation。

除此之外在 x-compiler 模块，还会依赖 2 个库以方便开发。

```java
implementation 'com.google.auto.service:auto-service:1.0-rc6'
implementation 'com.squareup:javapoet:1.11.1'
```

- AutoService：会自动在 META-INF 文件夹下生成 Processor 配置信息文件，该文件里就是实现该服务接口的具体实现类。而当外部程序装配这个模块的时候，就能通过该 jar 包 META-INF/services/ 里的配置文件找到具体的实现类名，并装载实例化完成模块的注入。

注：AutoService 会在 Gradle 5.X 版本失效，因此建议还是手动指定注解处理器：在 x-compiler 模块下新建 src/main/resources/META-INF/services/javax.annotation.processing.Processor 文件，文件内容为自定义注解处理器的路径，若有多个注解处理器则换行添加说明。

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

## 2.3 Element

Element 接口族与 Type 接口族很相似，都是代表着元素的类型以及所包含的一些其它信息（例如参数、注解），区别在于：Element 所代表的元素只在编译期可见，而 Type 所代表的元素是运行期可见。

- VariableElement：成员变量、枚举、局部变量、方法参数；
- TypeParameterElement：泛型；
- ExecutableElement：方法；
- TypeElement：类、接口、注解；
- PackageElement：包。

API 文档：http://www.matools.com/api/java8，包名 javax.lang.model.element。

### 2.3.1 基础接口 Element

```java
public interface Element extends AnnotatedConstruct {
    
    /**
     * 返回元素的类型信息。
     * 通过 TypeName typeName = ClassName.get(element.asType()) 即可拿到类型名。
     */
    TypeMirror asType();

    /**
     * 返回元素的种类。例如 PACKAGE（包）、ENUM（枚举）、CLASS（类）。
     */
    ElementKind getKind();
    
    /**
     * 返回元素的修饰符，不包括注释。例如 FINAL、PUBLIC。
     */
    Set<Modifier> getModifiers();
    
    /**
     * 返回此元素的简单（不包含路径）名称。
     * 例如：
     * 1、java.util.Set<E> ，则返回 “Set”；
     * 2、constructor ，则返回 “<init>”；
     * 3、static initializer（静态代码块），则返回“ <clinit> ”。
     */
    Name getSimpleName();

    /**
     * 返回该元素的父元素（封闭元素）。
     * 例如：
     * 1、对于 TypeElement 而言，它的父元素就是 PackageElement。
     * 2、对于 PackageElement，它的父元素则为 null。
     */
    Element getEnclosingElement();
    /**
     * 返回该元素直接包含的子元素。
     * 例如：
     * 1、对于 PackageElement（包名），它可以包含 TypeElement（例如类或接口）；
     * 2、对于 TypeElement ，可以包含 VariableElement（例如成员变量）或 ExecutableElement（方法）。
     */
    List<? extends Element> getEnclosedElements();

    /**
     * 返回该元素上的所有注解的类型信息。
     */
    List<? extends AnnotationMirror> getAnnotationMirrors();

    /**
     * 返回元素上指定类型的注解对象，如果不存在则返回 null。
     */
    <A extends Annotation> A getAnnotation(Class<A> annotationType);

    /**
     * 将访问者应用于此元素。 
     */
    <R, P> R accept(ElementVisitor<R, P> v, P p);
}
```

### 2.3.2 VariableElement

VariableElement：成员变量、枚举、局部变量、方法参数。

```java
public interface VariableElement extends Element {
    
    /**
     * 如果是编译时常量，则返回此变量的值（基本类型会被装箱），否则返回 null。
     */
    Object getConstantValue();

    /**
     * 此变量元素的简单名称。
     */
    Name getSimpleName();

    /**
     * 返回该元素的父元素（封闭元素）。
     */
    Element getEnclosingElement();
}
```

### 2.3.3 TypeParameterElement

TypeParameterElement：泛型。

```java
public interface TypeParameterElement extends Element {

    /**
     * 返回泛型作用的元素（例如具体的方法、类）。
     */
    Element getGenericElement();

    /**
     * 返回泛型继承的类型。
     */
    List<? extends TypeMirror> getBounds();

    /**
     * 返回泛型的父元素（封闭元素），一般和 getGenericElement() 返回的元素一致。
     */
    Element getEnclosingElement();
}
```

### 2.3.4 ExecutableElement

ExecutableElement：类中的方法、类的构造方法、注解方法。

```java
public interface ExecutableElement extends Element, Parameterizable {

    /**
     * 按照声明顺序返回方法上申明的所有泛型。
     */
    List<? extends TypeParameterElement> getTypeParameters();

    /**
     * 返回方法的返回值类型。
     */
    TypeMirror getReturnType();

    /**
     * 按照声明顺序返回方法中的所有参数元素。
     */
    List<? extends VariableElement> getParameters();

    /**
     * 返回方法所对应的接收器类型。
     */
    TypeMirror getReceiverType();

    /**
     * 如果此方法接受可变数量的参数，则返回 true。
     */
    boolean isVarArgs();
    
    /**
     * 如果此方法是默认的方法（接口默认实现），则返回 true。
     */
    boolean isDefault();

    /**
     * 以声明顺序返回此方法 throws 子句中列出的异常。 
     */
    List<? extends TypeMirror> getThrownTypes();

    /**
     * 如果该方法是注解的方法，则返回默认值。反之返回 null。
     */
    AnnotationValue getDefaultValue();
}
```

### 2.3.5 TypeElement

TypeElement：类、接口、注解。

```java
public interface TypeElement extends Element, Parameterizable, QualifiedNameable {

    /**
     * 返回该元素直接包含的子元素。
     */
    List<? extends Element> getEnclosedElements();
    /**
     * 返回此类型元素的嵌套种类。
     * 类型元素的种类有四种：top-level（顶层）、member（内部类、静态内部类）、local（局部）和 anonymous（匿名）。
     */
    NestingKind getNestingKind();

    /**
     * 返回此类型元素的完全限定名称。对于没有规范名称的局部类和匿名类，返回一个空名称。
     * 一般类型的名称不包括对其形式类型参数的任何引用。例如，接口 java.util.Set<E> 的完全限定名称是 "java.util.Set"。
     */
    Name getQualifiedName();

    /**
     * 返回此类型元素的简单名称。 对于匿名类，返回一个空的名称。 
     */
    Name getSimpleName();

    /**
     * 返回此类型元素的直接超类。
     * 如果此类型元素表示一个接口或者类 java.lang.Object，则返回一个种类为 NONE 的 NoType。
     */
    TypeMirror getSuperclass();

    /**
     * 返回直接由此类实现或直接由此接口扩展的接口类型。
     */
    List<? extends TypeMirror> getInterfaces();

    /**
     * 按照声明顺序返回此类型元素上申明的所有泛型。
     */
    List<? extends TypeParameterElement> getTypeParameters();

    /**
     * 返回该元素的父元素（封闭元素）。
     * 对于 top-level 类型的类，则是包名，对于 member，则是包含他的元素（例如类）。
     */
    Element getEnclosingElement();
}

```

### 2.3.6 PackageElement

PackageElement：包。

```java
public interface PackageElement extends Element, QualifiedNameable {

    /**
     * 返回此包的完全限定名称。
     */
    Name getQualifiedName();
    
    /**
     * 返回此包的简单名称。
     */
    Name getSimpleName();

    /**
     * 返回此包中的 top-level 类和接口。
     */
    List<? extends Element> getEnclosedElements();

    /**
     * 包是否命名。true：这是一个未命名的包。
     */
    boolean isUnnamed();

     /**
     * 恒定返回 null，因为一个包不被另一个元素包含。
     */
    Element getEnclosingElement();
}
```

## 2.4 JavaPoet 的使用

- MethodSpec：代表一个构造函数或方法声明。
- TypeSpec：代表一个类，接口，或者枚举。
- FieldSpec：代表一个成员变量，一个字段。
- JavaFile：包含一个顶级类的 Java 文件。

以下实例为了方便展示，将注解和响应的类文件都在一个 Module。

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
                .addFileComment("这是文件顶部注释")
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
// 这是文件顶部注释
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


# 三、手写 ButterKnife

基本的配置不在赘述。主要从围绕具体的代码去看代码结构和编写思路。

技巧：

- 在实际的 APT 开发过程中，我们可以先手写生成类的具体实现，然后再对照着具体的代码从上往下一步一步写生成代码。
- 对于不确定的方法或类型时可以通过查询 [Java 文档](http://www.matools.com/api/java8) 或调试解决。

本小节将实现 ButterKnife 的 @BindView。除了示例模块，整个实现分为 3 个模块：[butterknife-annotations](#31-butterknife-annotations)、butterknife-api、butterknife-compiler。项目地址：https://github.com/passin95/APT_butterknife 。

DemoActivity：开发者使用示例。
DemoActivity_ViewBinding：通过 APT 生成的代码文件。

```java
public class DemoActivity extends AppCompatActivity {

    private Unbinder unbinder;

    @BindView(R.id.fl_root)
    FrameLayout mFlRoot;
    @BindView(R.id.tv)
    TextView mTv;
    

    @Override
    protected void onCreate(@Nullable Bundle savedInstanceState) {
        super.onCreate(savedInstanceState);
        setContentView(R.layout.activity_demo);
        unbinder = ButterKnife.bind(this);
        mFlRoot.setBackgroundColor(getResources().getColor(R.color.colorAccent));
        mTv.setText("Hello World");
    }

    @Override
    protected void onDestroy() {
        super.onDestroy();
        unbinder.unbind();
    }
}

public class DemoActivity_ViewBinding implements Unbinder {
  private DemoActivity target;

  @UiThread
  public DemoActivity_ViewBinding(DemoActivity target) {
    this(target, target.getWindow().getDecorView());
  }

  @UiThread
  public DemoActivity_ViewBinding(DemoActivity target, View source) {
    this.target = target;

    target.mFlRoot = (FrameLayout) source.findViewById(2131165267);
    target.mTv = (TextView) source.findViewById(2131165344);
  }

  @Override
  @CallSuper
  public void unbind() {
    DemoActivity target = this.target;
    if (target == null) throw new IllegalStateException("Bindings already cleared.");
    this.target = null;

    target.mFlRoot = null;
    target.mTv = null;
  }
}
```

## 3.1 butterknife-annotations

该模块下只有一个注解类。

```java
@Retention(CLASS)
@Target({FIELD})
public @interface BindView {
    @IdRes int value();
}
```

## 3.2 butterknife-api

Unbinder：所有生成的绑定类应当实现该接口，用于解绑视图。

```java
public interface Unbinder {
    @UiThread
    void unbind();

    Unbinder EMPTY = new Unbinder() {
        @Override
        public void unbind() {
        }
    };
}
```

ButterKnife：通过反射实例化 APT 生成的绑定类，并在绑定类的构造函数中进行绑定。

```java
public final class ButterKnife {
    
    private ButterKnife() {
        throw new AssertionError("No instances.");
    }

    private static final String TAG = "ButterKnife";

    private static boolean debug = false;

    public static void setDebug(boolean debug) {
        ButterKnife.debug = debug;
    }

    static final Map<Class<?>, Constructor<? extends Unbinder>> BINDINGS = new LinkedHashMap<>();

    @NonNull
    @UiThread
    public static Unbinder bind(@NonNull Activity target) {
        View sourceView = target.getWindow().getDecorView();
        return bind(target, sourceView);
    }

    @NonNull
    @UiThread
    public static Unbinder bind(@NonNull View target) {
        return bind(target, target);
    }

    @NonNull
    @UiThread
    public static Unbinder bind(@NonNull Dialog target) {
        View sourceView = target.getWindow().getDecorView();
        return bind(target, sourceView);
    }

    @NonNull
    @UiThread
    public static Unbinder bind(@NonNull Object target, @NonNull Activity source) {
        View sourceView = source.getWindow().getDecorView();
        return bind(target, sourceView);
    }

    @NonNull
    @UiThread
    public static Unbinder bind(@NonNull Object target, @NonNull Dialog source) {
        View sourceView = source.getWindow().getDecorView();
        return bind(target, sourceView);
    }

    /**
     * 绑定 source 中的视图到目标类 target 中。
     */
    @NonNull
    @UiThread
    public static Unbinder bind(@NonNull Object target, @NonNull View source) {
        Class<?> targetClass = target.getClass();
        if (debug) {
            Log.d(TAG, "查找绑定" + targetClass.getName());
        }
        Constructor<? extends Unbinder> constructor = findBindingConstructorForClass(targetClass);

        if (constructor == null) {
            return Unbinder.EMPTY;
        }

        try {
            return constructor.newInstance(target, source);
        } catch (IllegalAccessException e) {
            throw new RuntimeException("Unable to invoke " + constructor, e);
        } catch (InstantiationException e) {
            throw new RuntimeException("Unable to invoke " + constructor, e);
        } catch (InvocationTargetException e) {
            Throwable cause = e.getCause();
            if (cause instanceof RuntimeException) {
                throw (RuntimeException) cause;
            }
            if (cause instanceof Error) {
                throw (Error) cause;
            }
            throw new RuntimeException("Unable to create binding instance.", cause);
        }
    }

    @Nullable
    @CheckResult
    @UiThread
    private static Constructor<? extends Unbinder> findBindingConstructorForClass(Class<?> cls) {
        // 对绑定类的构造器进行缓存，因此绑定类会存在于整个应用的生命周期。
        Constructor<? extends Unbinder> bindingCtor = BINDINGS.get(cls);
        if (bindingCtor != null || BINDINGS.containsKey(cls)) {
            if (debug) {
                Log.d(TAG, cls.getName() + "缓存已存在绑定映射中");
            }
            return bindingCtor;
        }
        String clsName = cls.getName();
        // 不应该对 framework 层的类进行绑定。
        if (clsName.startsWith("android.") || clsName.startsWith("java.")
                || clsName.startsWith("androidx.")) {
            if (debug) Log.d(TAG, "到达 framework 类。放弃搜索");
            return null;
        }
        try {
            // 该类会在编译阶段自动生成。API 和 APT 需要协议好反射类的格式。
            Class<?> bindingClass = cls.getClassLoader().loadClass(clsName + "_ViewBinding");
            bindingCtor = (Constructor<? extends Unbinder>) bindingClass.getConstructor(cls, View.class);
            if (debug) {
                Log.d(TAG, "查找到" + cls.getName() + "的绑定类");
            }
        } catch (ClassNotFoundException e) {
            if (debug) Log.d(TAG, "未发现，尝试从父类查找：" + cls.getSuperclass().getName());
            bindingCtor = findBindingConstructorForClass(cls.getSuperclass());
        } catch (NoSuchMethodException e) {
            throw new RuntimeException("Unable to find binding constructor for " + clsName, e);
        }
        BINDINGS.put(cls, bindingCtor);
        return bindingCtor;
    }
}
```

## 3.3 butterknife-compiler

ButterKnifeProcessor：声明支持的选项、支持的注解类型、支持的 Java 版本号，以及获取生成类信息来源和代码的整体执行导向。

BindingSet：储存生成类的所需信息以及执行具体的生成代码。

ViewBinding：实体类，保存着绑定视图的基本信息以及对应的 Id。

```java
public final class ButterKnifeProcessor extends AbstractProcessor {
    protected Filer mFiler;
    protected Messager mMessager;
    protected Types mTypes;
    protected Elements mElements;
    static final String VIEW_TYPE = "android.view.View";
    static final String ACTIVITY_TYPE = "android.app.Activity";
    static final String DIALOG_TYPE = "android.app.Dialog";

    @Override
    public synchronized void init(ProcessingEnvironment processingEnvironment) {
        super.init(processingEnvironment);
        mFiler = processingEnv.getFiler();
        mMessager = processingEnv.getMessager();
        mTypes = processingEnv.getTypeUtils();
        mElements = processingEnv.getElementUtils();
    }

    @Override
    public Set<String> getSupportedOptions() {
        // 支持的选项（key）。
//        ImmutableSet.Builder<String> builder = ImmutableSet.builder();
//        builder.add("HOST");
//        return builder.build();
        // 可在模块内的 build.gradle 配置，并在 init() 中通过 processingEnvironment.getOptions().get("HOST") 拿到具体的值。
        // android {
        //     defaultConfig {
        //        javaCompileOptions {
        //            annotationProcessorOptions {
        //                arguments = ["HOST": "component"]
        //            }
        //        }
        //    }
        //
        return super.getSupportedOptions();
    }

    @Override
    public Set<String> getSupportedAnnotationTypes() {
        // 支持的注解。
        Set<String> types = new LinkedHashSet<>();
        for (Class<? extends Annotation> annotation : getSupportedAnnotations()) {
            types.add(annotation.getCanonicalName());
        }
        return types;
    }

    private Set<Class<? extends Annotation>> getSupportedAnnotations() {
        Set<Class<? extends Annotation>> annotations = new LinkedHashSet<>();
        annotations.add(BindView.class);
        return annotations;
    }

    @Override
    public SourceVersion getSupportedSourceVersion() {
        // 支持的 Java 版本号。
        return SourceVersion.latestSupported();
    }

    @Override
    public boolean process(Set<? extends TypeElement> set, RoundEnvironment env) {
        Map<TypeElement, BindingSet> bindingMap = findAndParseTargets(env);

        for (Map.Entry<TypeElement, BindingSet> entry : bindingMap.entrySet()) {
            TypeElement typeElement = entry.getKey();
            BindingSet binding = entry.getValue();

            JavaFile javaFile = binding.brewJava();
            try {
                javaFile.writeTo(mFiler);
            } catch (IOException e) {
                error(typeElement, "Unable to write binding for type %s: %s", typeElement, e.getMessage());
            }
        }

        // 如果返回 true，则不会传递给后续处理器进行处理; 如果返回 false，则注释类型是无人认领的，后续的处理器可能会继续处理它们。
        return false;
    }

    private Map<TypeElement, BindingSet> findAndParseTargets(RoundEnvironment env) {
        // 一个类会有多次 BindView 的使用，因此需要有一个对应关系。
        // key 为被注解元素所在的类。
        // value 之所以是构造类，是为了处理父类有绑定的情况。
        Map<TypeElement, BindingSet.Builder> builderMap = new LinkedHashMap();
        // 所有被注解元素所在的类。
        Set<TypeElement> bindingTargetElements = new LinkedHashSet<>();

        // getElementsAnnotatedWith 可以拿到所有添加 @BindView 注解的元素。
        for (Element element : env.getElementsAnnotatedWith(BindView.class)) {
            try {
                // 解析元素信息并配置绑定类生成需要的信息。
                parseBindView(element, builderMap, bindingTargetElements);
            } catch (Exception e) {
                logParsingError(element, BindView.class, e);
            }
        }

        // 从 bindingTargetElements 中筛选出父类也是被注解元素所在的类。
        // key 为被注解元素所在的类且需要继承父类。
        // value 为父类的要求（是否需要传递参数 view）以及父类的类名。
        Map<TypeElement, ClasspathBindingSet> classpathBindings = findAllSupertypeBindings(builderMap, bindingTargetElements);

        Deque<Map.Entry<TypeElement, BindingSet.Builder>> entries = new ArrayDeque<>(builderMap.entrySet());
        Map<TypeElement, BindingSet> bindingMap = new LinkedHashMap<>();
        while (!entries.isEmpty()) {
            Map.Entry<TypeElement, BindingSet.Builder> entry = entries.removeFirst();

            TypeElement type = entry.getKey();
            BindingSet.Builder builder = entry.getValue();

            TypeElement parentType = findParentType(type, bindingTargetElements, classpathBindings.keySet());
            if (parentType == null) {
                bindingMap.put(type, builder.build());
            } else {
                BindingInformationProvider parentBinding = bindingMap.get(parentType);
                if (parentBinding == null) {
                    parentBinding = classpathBindings.get(parentType);
                }
                if (parentBinding != null) {
                    builder.setParent(parentBinding);
                    bindingMap.put(type, builder.build());
                } else {
                    // 具有超类绑定，但我们尚未构建它。重新排队以便稍后使用。
                    entries.addLast(entry);
                }
            }
        }

        return bindingMap;
    }


    private void parseBindView(Element element, Map<TypeElement, BindingSet.Builder> builderMap,
                               Set<TypeElement> bindingTargetElements) {
        // element：被注解元素。
        // enclosingElement：被注解元素的父元素，它是一个类，例如 Activity。
        TypeElement enclosingElement = (TypeElement) element.getEnclosingElement();

        // 首先验证 element 是否满足框架所定义的基本限制。
        boolean hasError = isInaccessibleViaGeneratedCode(BindView.class, "fields", element)
                || isBindingInWrongPackage(BindView.class, element);

        TypeMirror elementType = element.asType();
        if (elementType.getKind() == TypeKind.TYPEVAR) {
            TypeVariable typeVariable = (TypeVariable) elementType;
            elementType = typeVariable.getUpperBound();
        }
        Name qualifiedName = enclosingElement.getQualifiedName();
        Name simpleName = element.getSimpleName();
        // 验证 element 是否是 View 的子类。
        if (!isSubtypeOfType(elementType, VIEW_TYPE) && !isInterface(elementType)) {
            if (elementType.getKind() == TypeKind.ERROR) {
                note(element, "@%s field with unresolved type (%s) "
                                + "must elsewhere be generated as a View or interface. (%s.%s)",
                        BindView.class.getSimpleName(), elementType, qualifiedName, simpleName);
            } else {
                error(element, "@%s fields must extend from View or be an interface. (%s.%s)",
                        BindView.class.getSimpleName(), qualifiedName, simpleName);
                hasError = true;
            }
        }

        if (hasError) {
            return;
        }

        // 获取元素上注解的值。
        int id = element.getAnnotation(BindView.class).value();
        BindingSet.Builder builder = builderMap.get(enclosingElement);
        if (builder != null) {
            String existingBindingName = builder.findExistingBindingName(id);
            // 出现同一个 id 绑定了多次视图，则打印错误且不会尝试多次绑定。
            if (existingBindingName != null) {
                error(element, "Attempt to use @%s for an already bound ID %d on '%s'. (%s.%s)",
                        BindView.class.getSimpleName(), id, existingBindingName,
                        enclosingElement.getQualifiedName(), element.getSimpleName());
                return;
            }
        } else {
            builder = getOrCreateBindingBuilder(builderMap, enclosingElement);
        }

        String sampleName = simpleName.toString();
        TypeName typeName = TypeName.get(elementType);
        boolean required = Utils.isFieldRequired(element);
        builder.addField(id, new ViewBinding(id, sampleName, typeName, required));

        bindingTargetElements.add(enclosingElement);
    }


    private boolean isInaccessibleViaGeneratedCode(Class<? extends Annotation> annotationClass,
                                                   String targetThing, Element element) {
        boolean hasError = false;
        TypeElement enclosingElement = (TypeElement) element.getEnclosingElement();

        // 验证字段或方法的修饰符。
        Set<Modifier> modifiers = element.getModifiers();
        // 不应该被 private 和 static 关键字修饰。
        if (modifiers.contains(PRIVATE) || modifiers.contains(STATIC)) {
            error(element, "@%s %s must not be private or static. (%s.%s)",
                    annotationClass.getSimpleName(), targetThing, enclosingElement.getQualifiedName(),
                    element.getSimpleName());
            hasError = true;
        }

        // 验证注解元素的父元素类型，它应该是一个类。
        if (enclosingElement.getKind() != CLASS) {
            error(enclosingElement, "@%s %s may only be contained in classes. (%s.%s)",
                    annotationClass.getSimpleName(), targetThing, enclosingElement.getQualifiedName(),
                    element.getSimpleName());
            hasError = true;
        }

        // 不应该是私有的。
        if (enclosingElement.getModifiers().contains(PRIVATE)) {
            error(enclosingElement, "@%s %s may not be contained in private classes. (%s.%s)",
                    annotationClass.getSimpleName(), targetThing, enclosingElement.getQualifiedName(),
                    element.getSimpleName());
            hasError = true;
        }

        return hasError;
    }


    private boolean isBindingInWrongPackage(Class<? extends Annotation> annotationClass,
                                            Element element) {
        TypeElement enclosingElement = (TypeElement) element.getEnclosingElement();
        String qualifiedName = enclosingElement.getQualifiedName().toString();

        if (qualifiedName.startsWith("android.")) {
            error(element, "@%s-annotated class incorrectly in Android framework package. (%s)",
                    annotationClass.getSimpleName(), qualifiedName);
            return true;
        }
        if (qualifiedName.startsWith("java.")) {
            error(element, "@%s-annotated class incorrectly in Java framework package. (%s)",
                    annotationClass.getSimpleName(), qualifiedName);
            return true;
        }

        return false;
    }

    static boolean isSubtypeOfType(TypeMirror typeMirror, String otherType) {
        if (otherType.equals(typeMirror.toString())) {
            return true;
        }
        if (typeMirror.getKind() != TypeKind.DECLARED) {
            return false;
        }
        DeclaredType declaredType = (DeclaredType) typeMirror;
        List<? extends TypeMirror> typeArguments = declaredType.getTypeArguments();
        if (typeArguments.size() > 0) {
            StringBuilder typeString = new StringBuilder(declaredType.asElement().toString());
            typeString.append('<');
            for (int i = 0; i < typeArguments.size(); i++) {
                if (i > 0) {
                    typeString.append(',');
                }
                typeString.append('?');
            }
            typeString.append('>');
            if (typeString.toString().equals(otherType)) {
                return true;
            }
        }
        Element element = declaredType.asElement();
        if (!(element instanceof TypeElement)) {
            return false;
        }
        TypeElement typeElement = (TypeElement) element;
        TypeMirror superType = typeElement.getSuperclass();
        if (isSubtypeOfType(superType, otherType)) {
            return true;
        }
        for (TypeMirror interfaceType : typeElement.getInterfaces()) {
            if (isSubtypeOfType(interfaceType, otherType)) {
                return true;
            }
        }
        return false;
    }


    private Map<TypeElement, ClasspathBindingSet> findAllSupertypeBindings(
            Map<TypeElement, BindingSet.Builder> builderMap, Set<TypeElement> processedInThisRound) {
        Map<TypeElement, ClasspathBindingSet> classpathBindings = new HashMap<>();
        // 获取所有支持的注解。
        Set<Class<? extends Annotation>> supportedAnnotations = getSupportedAnnotations();
        // 所有需要在构造函数传参 View 的注解。
        Set<Class<? extends Annotation>> requireViewInConstructor =
                ImmutableSet.<Class<? extends Annotation>>builder().add(BindView.class).build();
        // 剩下的注解只需要向父类传参 context。
        supportedAnnotations.removeAll(requireViewInConstructor);

        // typeElement 为被注解元素所在的类。
        for (TypeElement typeElement : builderMap.keySet()) {
            // 确保在子类之前处理超类，因为父类的构造函数参数需要子类提供。
            // superClasses 存储着需要继承父类的类元素。
            Deque<TypeElement> superClasses = new ArrayDeque<>();
            TypeElement superClass = getSuperClass(typeElement);
            while (superClass != null && !processedInThisRound.contains(superClass)
                    && !classpathBindings.containsKey(superClass)) {
                // 逐级添加存在绑定的父类到队列的头部，直至最终的父类。
                // 这样能保证在有多重继承关系的类之间，父类一定在子类的前面。
                superClasses.addFirst(superClass);
                superClass = getSuperClass(superClass);
            }

            // 标记着父类的构造函数是否需要参数 View。
            boolean parentHasConstructorWithView = false;
            while (!superClasses.isEmpty()) {
                TypeElement superclass = superClasses.removeFirst();
                // 查找 superclass 的父类的要求以及类名。
                ClasspathBindingSet classpathBinding =
                        findBindingInfoForType(superclass, requireViewInConstructor, supportedAnnotations,
                                parentHasConstructorWithView);
                if (classpathBinding != null) {
                    // superclass 父类的要求，它的所有子类也必须要提供。
                    parentHasConstructorWithView |= classpathBinding.constructorNeedsView();
                    classpathBindings.put(superclass, classpathBinding);
                }
            }
        }
        return ImmutableMap.copyOf(classpathBindings);
    }

    private ClasspathBindingSet findBindingInfoForType(
            TypeElement typeElement, Set<Class<? extends Annotation>> requireConstructorWithView,
            Set<Class<? extends Annotation>> otherAnnotations, boolean needsConstructorWithView) {
        boolean foundSupportedAnnotation = false;
        for (Element enclosedElement : typeElement.getEnclosedElements()) {
            for (Class<? extends Annotation> bindViewAnnotation : requireConstructorWithView) {
                if (enclosedElement.getAnnotation(bindViewAnnotation) != null) {
                    return new ClasspathBindingSet(true, typeElement);
                }
            }
            for (Class<? extends Annotation> supportedAnnotation : otherAnnotations) {
                if (enclosedElement.getAnnotation(supportedAnnotation) != null) {
                    if (needsConstructorWithView) {
                        return new ClasspathBindingSet(true, typeElement);
                    }
                    foundSupportedAnnotation = true;
                }
            }
        }
        if (foundSupportedAnnotation) {
            return new ClasspathBindingSet(false, typeElement);
        } else {
            return null;
        }
    }

    private BindingSet.Builder getOrCreateBindingBuilder(
            Map<TypeElement, BindingSet.Builder> builderMap, TypeElement enclosingElement) {
        BindingSet.Builder builder = builderMap.get(enclosingElement);
        if (builder == null) {
            builder = BindingSet.newBuilder(enclosingElement);
            builderMap.put(enclosingElement, builder);
        }
        return builder;
    }

    private void logParsingError(Element element, Class<? extends Annotation> annotation,
                                 Exception e) {
        StringWriter stackTrace = new StringWriter();
        e.printStackTrace(new PrintWriter(stackTrace));
        error(element, "Unable to parse @%s binding.\n\n%s", annotation.getSimpleName(), stackTrace);
    }

    @Nullable
    private TypeElement findParentType(TypeElement typeElement, Set<TypeElement> parents, Set<TypeElement> classpathParents) {
        while (true) {
            typeElement = getSuperClass(typeElement);
            if (typeElement == null || parents.contains(typeElement)
                    || classpathParents.contains(typeElement)) {
                return typeElement;
            }
        }
    }

    private boolean isInterface(TypeMirror typeMirror) {
        return typeMirror instanceof DeclaredType
                && ((DeclaredType) typeMirror).asElement().getKind() == INTERFACE;
    }

    @Nullable
    private TypeElement getSuperClass(TypeElement typeElement) {
        TypeMirror type = typeElement.getSuperclass();
        if (type.getKind() == TypeKind.NONE) {
            return null;
        }
        return (TypeElement) ((DeclaredType) type).asElement();
    }

    private void note(Element element, String message, Object... args) {
        printMessage(Diagnostic.Kind.NOTE, element, message, args);
    }

    private void error(Element element, String message, Object... args) {
        printMessage(Diagnostic.Kind.ERROR, element, message, args);
    }

    private void printMessage(Diagnostic.Kind kind, Element element, String message, Object[] args) {
        if (args.length > 0) {
            message = String.format(message, args);
        }

        processingEnv.getMessager().printMessage(kind, message, element);
    }
}
```



```java
final class BindingSet implements BindingInformationProvider {
    private static final ClassName VIEW = ClassName.get("android.view", "View");
    private static final ClassName CONTEXT = ClassName.get("android.content", "Context");
    private static final ClassName UI_THREAD =
            ClassName.get("androidx.annotation", "UiThread");
    private static final ClassName CALL_SUPER =
            ClassName.get("androidx.annotation", "CallSuper");
    private static final ClassName UNBINDER = ClassName.get("me.passin.butterknife.api", "Unbinder");

    private final TypeName targetTypeName;
    private final ClassName bindingClassName;
    private final TypeElement enclosingElement;
    private final boolean isFinal;
    private final boolean isView;
    private final boolean isActivity;
    private final boolean isDialog;
    private final ImmutableList<ViewBinding> viewBindings;
    private final @Nullable BindingInformationProvider parentBinding;

    private BindingSet(
            TypeName targetTypeName, ClassName bindingClassName, TypeElement enclosingElement,
            boolean isFinal, boolean isView, boolean isActivity, boolean isDialog,
            ImmutableList<ViewBinding> viewBindings,
            @Nullable BindingInformationProvider parentBinding) {
        this.isFinal = isFinal;
        this.targetTypeName = targetTypeName;
        this.bindingClassName = bindingClassName;
        this.enclosingElement = enclosingElement;
        this.isView = isView;
        this.isActivity = isActivity;
        this.isDialog = isDialog;
        this.viewBindings = viewBindings;
        this.parentBinding = parentBinding;
    }

    @Override
    public ClassName getBindingClassName() {
        return bindingClassName;
    }

    JavaFile brewJava() {
        // 创建类
        TypeSpec bindingConfiguration = createClass();
        // 创建文件
        return JavaFile.builder(bindingClassName.packageName(), bindingConfiguration)
                // 添加文件顶部注释
                .addFileComment("Generated code from Butter Knife. Do not modify!")
                .build();
    }


    /**
     * public class DemoActivity_ViewBinding implements Unbinder {
     *   private DemoActivity target;
     *
     *   @UiThread
     *   public DemoActivity_ViewBinding(DemoActivity target) {
     *     this(target, target.getWindow().getDecorView());
     *   }
     *
     *   @UiThread
     *   public DemoActivity_ViewBinding(DemoActivity target, View source) {
     *     this.target = target;
     *
     *     target.mFlRoot = (FrameLayout) source.findViewById(2131165267);
     *     target.mTv = (TextView) source.findViewById(2131165344);
     *   }
     *
     *   @Override
     *   @CallSuper
     *   public void unbind() {
     *     DemoActivity target = this.target;
     *     if (target == null) throw new IllegalStateException("Bindings already cleared.");
     *     this.target = null;
     *
     *     target.mFlRoot = null;
     *     target.mTv = null;
     *   }
     * }
     * 先手写一个生成类的具体，然后从上往下一步一步写生成代码。
     */
    private TypeSpec createClass() {
        TypeSpec.Builder result = TypeSpec.classBuilder(bindingClassName.simpleName())
                .addModifiers(PUBLIC)
                // 申明与此文件创建有关的来源元素，在创建文件时使用。
                .addOriginatingElement(enclosingElement);
        if (isFinal) {
            result.addModifiers(FINAL);
        }

        // target 的父类有使用 ButterKnife 绑定，则直接继承父类，父类会实现 Unbinder 接口。
        if (parentBinding != null) {
            result.superclass(parentBinding.getBindingClassName());
        } else {
            // 否则实现 Unbinder 接口，用于解绑视图。
            result.addSuperinterface(UNBINDER);
        }

        // 接着是添加 target 变量。
        result.addField(targetTypeName, "target", PRIVATE);

        // 添加针对 target 对象的构造方法。
        if (isView) {
            result.addMethod(createBindingConstructorForView());
        } else if (isActivity) {
            result.addMethod(createBindingConstructorForActivity());
        } else if (isDialog) {
            result.addMethod(createBindingConstructorForDialog());
        }
        // 最后都会调用该构造函数，并在构造函数中对视图进行绑定。
        result.addMethod(createBindingConstructor());

        // 添加解绑方法，就是对 Unbinder 接口的实现。
        result.addMethod(createBindingUnbindMethod(result));

        return result.build();
    }

    private MethodSpec createBindingConstructorForView() {
        MethodSpec.Builder builder = MethodSpec.constructorBuilder()
                .addAnnotation(UI_THREAD)
                .addModifiers(PUBLIC)
                .addParameter(targetTypeName, "target");
        if (constructorNeedsView()) {
            builder.addStatement("this(target, target)");
        } else {
            builder.addStatement("this(target, target.getContext())");
        }
        return builder.build();
    }

    private MethodSpec createBindingConstructorForActivity() {
        MethodSpec.Builder builder = MethodSpec.constructorBuilder()
                .addAnnotation(UI_THREAD)
                .addModifiers(PUBLIC)
                .addParameter(targetTypeName, "target");
        if (constructorNeedsView()) {
            builder.addStatement("this(target, target.getWindow().getDecorView())");
        } else {
            builder.addStatement("this(target, target)");
        }
        return builder.build();
    }

    private MethodSpec createBindingConstructorForDialog() {
        MethodSpec.Builder builder = MethodSpec.constructorBuilder()
                .addAnnotation(UI_THREAD)
                .addModifiers(PUBLIC)
                .addParameter(targetTypeName, "target");
        if (constructorNeedsView()) {
            builder.addStatement("this(target, target.getWindow().getDecorView())");
        } else {
            builder.addStatement("this(target, target.getContext())");
        }
        return builder.build();
    }

    private MethodSpec createBindingViewDelegateConstructor() {
        return MethodSpec.constructorBuilder()
                .addJavadoc("@deprecated Use {@link #$T($T, $T)} for direct creation.\n    "
                                + "Only present for runtime invocation through {@code ButterKnife.bind()}.\n",
                        bindingClassName, targetTypeName, CONTEXT)
                .addAnnotation(Deprecated.class)
                .addAnnotation(UI_THREAD)
                .addModifiers(PUBLIC)
                .addParameter(targetTypeName, "target")
                .addParameter(VIEW, "source")
                .addStatement(("this(target, source.getContext())"))
                .build();
    }

    private MethodSpec createBindingConstructor() {
        MethodSpec.Builder constructor = MethodSpec.constructorBuilder()
                .addAnnotation(UI_THREAD)
                .addModifiers(PUBLIC);

        constructor.addParameter(targetTypeName, "target");

        // 由于没有添加绑定资源的功能，因此该方法的返回值都会走 true。
        // 保留判断主要是为了说明编写的思路。
        if (constructorNeedsView()) {
            constructor.addParameter(VIEW, "source");
        } else {
            constructor.addParameter(CONTEXT, "context");
        }

        if (parentBinding != null) {
            if (parentBinding.constructorNeedsView()) {
                constructor.addStatement("super(target, source)");
            } else if (constructorNeedsView()) {
                // 走到这里说明父类绑定了资源，而子类需要绑定视图，因此需要传 source.getContext() 给父视图（和子类生成的构造函数有关）。
                constructor.addStatement("super(target, source.getContext())");
            } else {
                // 走到这里说明父类绑定了资源，而子类也需要绑定资源，因此需要传 context 给父视图（和子类生成的构造函数有关）。
                constructor.addStatement("super(target, context)");
            }
            constructor.addCode("\n");
        }
        // 对 target 赋值。
        constructor.addStatement("this.target = target");
        constructor.addCode("\n");

        if (hasViewBindings()) {
            for (ViewBinding binding : viewBindings) {
                // 添加视图绑定代码。
                addViewBinding(constructor, binding);
            }
        }

        return constructor.build();
    }

    private MethodSpec createBindingUnbindMethod(TypeSpec.Builder bindingClass) {
        MethodSpec.Builder result = MethodSpec.methodBuilder("unbind")
                .addAnnotation(Override.class)
                .addModifiers(PUBLIC);
        if (!isFinal && parentBinding == null) {
            result.addAnnotation(CALL_SUPER);
        }

        result.addStatement("$T target = this.target", targetTypeName);
        result.addStatement("if (target == null) throw new $T($S)", IllegalStateException.class,
                "Bindings already cleared.");
        result.addStatement("this.target = null");
        result.addCode("\n");
        for (ViewBinding binding : viewBindings) {
            result.addStatement("target.$L = null", binding.getSampleName());
        }

        if (parentBinding != null) {
            result.addCode("\n");
            result.addStatement("super.unbind()");
        }
        return result.build();
    }

    private void addViewBinding(MethodSpec.Builder result, ViewBinding binding) {
        // 添加代码块。
        CodeBlock.Builder builder = CodeBlock.builder()
                .add("target.$L = ", binding.getSampleName());

        boolean requiresCast = requiresCast(binding.getTypeName());
        if (requiresCast) {
            builder.add("($T) ", binding.getTypeName());
        }
        builder.add("source.findViewById($L)", binding.getId());
        result.addStatement("$L", builder.build());
    }

    /**
     * 当前类需要绑定视图时返回 true。
     */
    private boolean hasViewBindings() {
        return !viewBindings.isEmpty();
    }

    /**
     * 如果此绑定需要视图，则为 true。否则，仅需要上下文 context。
     */
    @Override
    public boolean constructorNeedsView() {
        return hasViewBindings()
                || (parentBinding != null && parentBinding.constructorNeedsView());
    }

    static boolean requiresCast(TypeName type) {
        return !VIEW_TYPE.equals(type.toString());
    }

    @Override
    public String toString() {
        return bindingClassName.toString();
    }

    static Builder newBuilder(TypeElement enclosingElement) {
        return new Builder(enclosingElement);
    }

    static ClassName getBindingClassName(TypeElement typeElement) {
        String packageName = getPackage(typeElement).getQualifiedName().toString();
        String className = typeElement.getQualifiedName().toString().substring(
                packageName.length() + 1).replace('.', '$');
        return ClassName.get(packageName, className + "_ViewBinding");
    }

    static final class Builder {
        private final TypeName targetTypeName;
        private final ClassName bindingClassName;
        private final TypeElement enclosingElement;
        private final boolean isFinal;
        private final boolean isView;
        private final boolean isActivity;
        private final boolean isDialog;

        private @Nullable BindingInformationProvider parentBinding;

        private final Map<Integer, ViewBinding> viewIdMap = new LinkedHashMap<>();

        private Builder(TypeElement element) {
            this.enclosingElement = element;

            // 根据绑定的元素，解析生成文件所需要的信息。
            TypeMirror typeMirror = enclosingElement.asType();

            isView = isSubtypeOfType(typeMirror, VIEW_TYPE);
            isActivity = isSubtypeOfType(typeMirror, ACTIVITY_TYPE);
            isDialog = isSubtypeOfType(typeMirror, DIALOG_TYPE);

            TypeName targetTypeName = TypeName.get(typeMirror);
            if (targetTypeName instanceof ParameterizedTypeName) {
                targetTypeName = ((ParameterizedTypeName) targetTypeName).rawType;
            }
            this.targetTypeName = targetTypeName;

            bindingClassName = getBindingClassName(enclosingElement);
            isFinal = enclosingElement.getModifiers().contains(Modifier.FINAL);
        }


        void addField(int id, ViewBinding binding) {
            viewIdMap.put(id, binding);
        }

        void setParent(BindingInformationProvider parent) {
            this.parentBinding = parent;
        }

        @Nullable
        String findExistingBindingName(int id) {
            ViewBinding viewBinding = viewIdMap.get(id);
            if (viewBinding == null) {
                return null;
            }
            return viewBinding.getSampleName();
        }


        BindingSet build() {
            ImmutableList.Builder<ViewBinding> viewBindings = ImmutableList.builder();
            for (ViewBinding viewBinding : viewIdMap.values()) {
                viewBindings.add(viewBinding);
            }
            return new BindingSet(targetTypeName, bindingClassName, enclosingElement, isFinal, isView,
                    isActivity, isDialog, viewBindings.build(), parentBinding);
        }
    }
}

interface BindingInformationProvider {
    boolean constructorNeedsView();

    ClassName getBindingClassName();
}

final class ClasspathBindingSet implements BindingInformationProvider {
    private boolean constructorNeedsView;
    private ClassName className;

    ClasspathBindingSet(boolean constructorNeedsView, TypeElement classElement) {
        this.constructorNeedsView = constructorNeedsView;
        this.className = BindingSet.getBindingClassName(classElement);
    }

    @Override
    public ClassName getBindingClassName() {
        return className;
    }

    @Override
    public boolean constructorNeedsView() {
        return constructorNeedsView;
    }
}
```

```java
final class ViewBinding {
    private final int id;
    private final String sampleName;
    private final TypeName typeName;
    private final boolean required;

    public ViewBinding(int id, String sampleName, TypeName typeName, boolean required) {
        this.id = id;
        this.sampleName = sampleName;
        this.typeName = typeName;
        this.required = required;
    }

    public int getId() {
        return id;
    }

    public String getSampleName() {
        return sampleName;
    }

    public TypeName getTypeName() {
        return typeName;
    }

    public boolean isRequired() {
        return required;
    }

}
```