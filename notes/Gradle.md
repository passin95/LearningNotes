
<!-- TOC -->

- [一、Gradle 简介](#%E4%B8%80gradle-%E7%AE%80%E4%BB%8B)
- [二、Gradle 的工作流程](#%E4%BA%8Cgradle-%E7%9A%84%E5%B7%A5%E4%BD%9C%E6%B5%81%E7%A8%8B)
- [三、Task](#%E4%B8%89task)
- [四、Extension](#%E5%9B%9Bextension)
- [五、自定义 Plugin](#%E4%BA%94%E8%87%AA%E5%AE%9A%E4%B9%89-plugin)
  - [5.1 模块内 build.gradle](#51-%E6%A8%A1%E5%9D%97%E5%86%85-buildgradle)
  - [5.2 buildSrc](#52-buildsrc)
  - [5.3 library](#53-library)
- [六、Transform](#%E5%85%ADtransform)

<!-- /TOC -->
# 一、Gradle 简介

Gradle 是基于依赖关系的项目自动化构建开源工具。它可以管理项目中的差异、依赖、编译、打包、部署等等。

Gradle 不单单是一个配置脚本，它的背后是几门语言，大体上可为分：

- [Groovy Language](http://www.groovy-lang.org/api.html)
- [Gradle DSL](https://docs.gradle.org/current/javadoc/org/gradle/api/Project.html)
- [Android DSL](http://google.github.io/android-gradle-dsl/current/index.html)

DSL 的全称是 Domain Specific Language，领域特定语言，即只能用于特定的某个领域。

对于 Android 开发者来说，Gradle 的架构可以分为三层：

- 最下层是底层 Gradle 框架，它主要提供一些基础服务，如 Task 的依赖以及有向无环图的构建等；
- 再上面则是 Google 编译工具团队的 Android Gradle Plugin（即 com.android.tools.build:gradle:+），它在 Gradle 框架的基础上，创建了很多与 Android 项目打包有关的 Task、artifacts。
- 最上面则是开发者自定义的 Plugin，一般是在 Android Gradle plugin 提供的 Task 的基础上，插入一些自定义的 Task，或者是使用 Transform 在编译期进行字节码修改。

# 二、Gradle 的工作流程

Gradle 的工作流程可以分为 3 个阶段：

- 初始化阶段：setting.gradle 执行，确定主 project 和子 project。 
- 配置阶段：执行所有在 setting.gradle 中配置的 project 下的 build.gradle（一般会包含很多 plugin），根据每一个的 Task 的依赖关系构建出一个有向无环图。
- 执行阶段：根据确定的有向无环图，按顺序执行 Task（被依赖的 Task 会先于依赖它的 Task 执行，且每一个 Task 只会执行一次）。

<div align="center">  <img src="../pictures//Gradle%20执行时序.webp"/> </div>

因此:

- 若要在初始化阶段和配置阶段间插入执行代码，可写在 setting.gradle 文件的 **include()** 之后。
- 若要在配置阶段和执行阶段间插入执行代码，则写在 Project.afterEvaluate() 方法的闭包回调中。

# 三、Task

定义 Task 的方式很多，例如我们可以直接在 gradle 文件下添加一个 Task：

```groovy

task task1{
    println "configuration task1"
}

task task2{
    println "configuration task2"
    // 发生在 configuration 阶段，在 task 创建过程中就会被执行。（例如 Sync Project with Gradle Files）
    // 可以在配置阶段调用 dependsOn() 方法设置该 Task 执行所依赖的其它 Task。
    doLast{
        println "execution task2 doLast"
        // 发生在 execution 阶段。
        // 在当前 Task 的执行队列的后面插入代码。
    }
    doFirst{
        println "execution task2 doFirst"
        // 发生在 execution 阶段。
        // 在当前 Task 的执行队列的头部插入代码。
    }
}

// 定义一个名为 task3 的 Task，属于 taskGroup 分组，并且依赖 task1 和 task2 两个 Task。
project.task('task3', group: "taskGroup", description: "我是Task3", dependsOn: ["task1", "task2"]).doLast {
    println "execution task3"
}
```

在命令界面 Terminal 输入 gradlew task3，则相关的输出结果为：

```
> Configure project :app
configuration task1
configuration task2

> Task :app:task2
execution task2 doFirst
execution task2 doLast

> Task :app:task3
execution task3
```

也可以自定义一个 Task 类：

```groovy
class MyTask extends DefaultTask {

    // 被 @TaskAction 注解的方法在该 Task 被调用时执行。
    @TaskAction
    def hello(){
        println "Hello task"
    }
}


// 注册 Task。
project.getTasks().create("taskName", MyTask);
```

# 四、Extension

Extension 用于 gradle 文件的数据配置。

（1）定义一个固定数量的配置项：

```groovy
class MyPlugin implements Plugin<Project> {

    @Override
    void apply(Project project) {
        // fruit 是配置名，project 是 Fruit 构造函数传参。
        def extension = project.extensions.create("fruit", Fruit, project)
    }

    static class Fruit {
        String versionName

        Fruit(Project project) {
            project.extensions.create("apple", Apple)
            project.extensions.create("banana", Banana)
        }
    }

    static class Apple {
        String name
        float weight
    }

    static class Banana {
        String name
        int size
    }
}
```

之后可在 gradle 文件进行配置：

```groovy
fruit{
    versionName "fruit"
    apple{
        name "good apple"
        weight 10f
    }

    banana{
        name 'good banana'
        size 10
    }
}
```

（2）定义一个包含不定数量的配置项的 Extension：

```groovy
class MyPlugin implements Plugin<Project> {

    @Override
    void apply(Project project) {
        def instantiator = ((DefaultGradle) project.getGradle()).getServices().get(Instantiator.class)
        NamedDomainObjectContainer<Student> studentContainer = project.container(Student, new StudentExtFactory(instantiator))
        project.extensions.add('students', studentContainer)

        project.afterEvaluate {
            // 拿到 students 的配置信息并输出。
            NamedDomainObjectContainer<Student> students = (NamedDomainObjectContainer<Student>) project.extensions.getByName("students");

            students.forEach{
                String format = String.format("name: %s , age: %s", it.getName(), it.getAge());
                println("student : ${format}")
            };
        }
    }

    static class Student {
        // 必须有一个 String name 属性，以及一个以 name 作为参数的构造函数。
        String name
        int age
        boolean isMale

        Student(String name) {
            this.name = name
        }
    }

    static class StudentExtFactory implements NamedDomainObjectFactory<Student> {
        private Instantiator instantiator

        StudentExtFactory(Instantiator instantiator) {
            this.instantiator = instantiator
        }

        @Override
        Student create(String name) {
            // NamedDomainObjectFactory 是命名对象容器，创建的对象必须要有 name 属性作为容器内元素的标识。
            return instantiator.newInstance(Student.class, name)
        }
    }
}
```

之后便可在 gradle 文件进行配置：

```groovy
students{
    passin{
        age = 24
        isMale = true
    }

    xiaohong{
        age = 24
        isMale = false
    }
}
```

# 五、自定义 Plugin

自定义 Gradle Plugin 有三种途径：

- 模块内直接编写；
- buildSrc 中编写；
- library 模块中编写再发布到仓库进行依赖。

## 5.1 模块内 build.gradle

可以在 build.gradle 中直接编写并直接应用；

```groovy
apply plugin: 'com.android.application'

class MyPlugin implements Plugin<Project>{
    @Override
    void apply(Project target) {
        // 定义一个名为 Hello 的 Task。
        target.task("Hello"){
            println('Hello World')
        }
    }
}

apply plugin: MyPlugin
```

接着在 Terminal 输入 gradlew -q Hello，其中 -q 是为了只输出关键信息，不输出日志信息。便可只执行指定的 Task **Hello**，最后的输出结果如下：

```
Hello World
```

缺点：只能在当前模块使用。

## 5.2 buildSrc 

buildSrc 的原理是：运行 Gradle 时，它会检查项目根目录是否存在一个名为 buildSrc 的目录。然后 Gradle 会自动编译目录下的代码，并将其放入构建脚本的类路径中。因此还可以利用它去 [管理项目的依赖](https://juejin.im/post/5b0fb0e56fb9a00a012b7fda)，让依赖支持自动补全和单击跳转。

buildSrc 和 library 配置和使用 gradle 插件的方式几乎一致，区别在于 library 需要发布到 maven，因此具体的配置统一在下一小节说明。

缺点：只能在当前工程中使用。但是可以用于 plugin 开发阶段的调试。

## 5.3 library

1. 新建一个模块，并对模块下的的 build.gradle 添加依赖：

```groovy
apply plugin: 'groovy'
apply plugin: 'maven'
apply plugin: 'com.jfrog.bintray'

sourceCompatibility = JavaVersion.VERSION_1_8
targetCompatibility = JavaVersion.VERSION_1_8

dependencies {
    implementation gradleApi()
    implementation localGroovy()
}
repositories {
    jcenter()
}


Properties properties = new Properties()
properties.load(project.rootProject.file('local.properties').newDataInputStream())
def artifactory_user = properties.getProperty("artifactory_user")
def artifactory_password = properties.getProperty("artifactory_password")
def artifactory_contextUrl = properties.getProperty("artifactory_contextUrl")
def artifactory_snapshot_repoKey = properties.getProperty("artifactory_snapshot_repoKey") // 快照 key
def artifactory_release_repoKey = properties.getProperty("artifactory_release_repoKey") // 正式版 key

def artifact_group = 'me.passin' 
def artifact_id = 'plugin' // 默认为项目名
def artifact_version='0.0.1'

group = 'me.passin'
version = artifact_version

def maven_type_snapshot = true // 是否是快照版本。
def debug_flag = true // true: 部署到本地 maven 仓库，false：部署到 maven 私服。

uploadArchives {
    repositories {
        mavenDeployer {
            if (debug_flag) {
                // 部署到本地
                repository(url: uri('../repo-local')) 
            } else {
                //部署到远程 maven 仓库。
                def repoKey = maven_type_snapshot ? artifactory_snapshot_repoKey : artifactory_release_repoKey
                repository(url: "${artifactory_contextUrl}/${repoKey}") {
                    authentication(userName: artifactory_user, password: artifactory_password)
                }
            }

            pom.groupId = artifact_group
            pom.artifactId = artifact_id
            pom.version = artifact_version + (maven_type_snapshot ? '-SNAPSHOT' : '')

            pom.project {
                licenses {
                    license {
                        name 'The Apache Software License, Version 2.0'
                        url 'http://www.apache.org/licenses/LICENSE-2.0.txt'
                    }
                }
            }
        }
    }
}


def siteUrl = '' // 项目主页。
def gitUrl = '' // Git 仓库的 url。

install {
    repositories.mavenInstaller {
        // 生成 pom.xml 和参数
        pom {
            project {
                packaging 'jar' // 打包成 jar 包。
                name 'Plugin'// 可选，项目名称。
                description 'This is a plugin demo'// 可选，项目描述。
                url siteUrl // 项目主页，这里是引用上面定义好。

                // 开源协议
                licenses {
                    license {
                        name 'The Apache Software License, Version 2.0'
                        url 'http://www.apache.org/licenses/LICENSE-2.0.txt'
                    }
                }

                developers {
                    developer {
                        id 'passin95' // 填写 Bintray 的用户名。
                        name 'passin' // 开发者名字。
                        email 'zengbinbinpassin@foxmail.com' // 开发者邮箱。
                    }
                }

                scm {
                    connection gitUrl // Git 仓库地址。
                    developerConnection gitUrl // Git 仓库地址。
                    url siteUrl // 项目主页。
                }
            }
        }
    }
}

//Properties properties = new Properties()
//properties.load(project.rootProject.file('local.properties').newDataInputStream())
bintray {
    user = properties.getProperty("bintray.user") // Bintray 的用户名。
    key = properties.getProperty("bintray.apikey") // Bintray 的 ApiKey。

    pkg {
        repo = "maven"  // 在 Bintray 创建的 maven 仓库名。
        name = "Plugin"  // 发布到 Bintray 上的项目名。
        licenses = ["Apache-2.0"] // 协议。
        websiteUrl = siteUrl // 项目主页。
        vcsUrl = gitUrl // Git 仓库的 url。
        publish = true // 是否是公开项目。
//        userOrg = properties.getProperty("bintray.userOrg") // Bintray 的用户名。
    }
    configurations = ['archives']
}

task sourcesJar(type: Jar) {
    from project.file('src/main/groovy')
    classifier = 'sources'
}

artifacts {
    archives sourcesJar
}
```

2. 若使用 kotlin、java 则在 src/main/java 目录下编写代码；若使用 groovy 则在 src/main/groovy 目录下编写代码。

```groovy
package me.passin.plugin

class MyPlugin implements Plugin<Project> {

    @Override
    void apply(Project target) {
        def extension = target.extensions.create("passin", Extension)
        target.afterEvaluate {
            println "Hello ${extension.name}"
        }
    }

    class Extension {
        // groovy 会默认自动实现 set/get 方法。
        def name = "Default"
    }
}
```

3. 新建文件 src/resources/META-INF/gradle-plugins/plugin.demo.properties，文件名（plugin.demo）即对应着使用时的插件名:

```
apply plugin: 'plugin-demo'
```

文件内容：

```
// 指向实现接口 Plugin 的类。
implementation-class=me.passin.plugin.MyPlugin
```

4. 输入命令 gradlew -p plugin（library name） clean build uploadArchives -info 发布到本地 maven 仓库。输入命令 gradle install + gradle bintrayUpload 上传到 bintray 远程仓库。

5. 引入发布的 plugin 并使用。

Project 下的 build.gradle:

```groovy
buildscript {
    repositories {
        google()
        jcenter()
        maven{
            //本地 maven 仓库。
            url uri('./repo-local')
        }
    }
    dependencies {
        classpath 'com.android.tools.build:gradle:3.5.3'
        // 上传到远程仓库 bintray 的插件 
        classpath 'com.jfrog.bintray.gradle:gradle-bintray-plugin:1.8.4'
        // 依赖自定义的插件
        classpath 'me.passin:plugin:0.0.1-SNAPSHOT'
    }
}
```

使用自定义插件的模块下的 build.gradle:

```groovy
apply plugin: 'com.android.application'
apply plugin: 'plugin-demo'

passin {
    // 和配置 android.compileSdkVersion 是一样的原理。本质上是调用了 setName('world')。
    name 'world'
}
android {
    compileSdkVersion 29
    ……
}
```

最后在命令界面 Terminal 输入 gradlew 模块名:assembleDebug 执行整个打 debug 包过程，其中就会输出自定义的插件结果:

```
Hello world
```

# 六、Transform

Transform 是 Android Gradle plugin 团队提供给开发者使用的一个抽象类，它提供接口让开发者可以在项目构建阶段，即在源文件编译成为 .class 文件之后，dex 之前进行字节码层面的修改。Transform 本质上也是执行在 Task 中。

借助 javaassist，ASM 这样的字节码处理工具，可在自定义的 Transform 中进行代码的插入、修改、替换、新建类等。

```groovy
class MyPlugin implements Plugin<Project> {

    @Override
    void apply(Project project) {
        def transform = new TransformDeme()
        // 拿到所有 Android 插件的基本扩展，并注册新的 Transform。
        def baseExtension = project.extensions.getByType(BaseExtension)
        baseExtension.registerTransform(transform)
    }

    static class TransformDeme extends Transform {
        @Override
        String getName() {
            // 返回转换的唯一名称，也是 Task 名称的组成部分。
            return "passinTransform"
        }

        @Override
        Set<QualifiedContent.ContentType> getInputTypes() {
            // 筛选修改的文件类型,例如 class 文件，jar 包等。
            // 这里修改 class 文件。
            return TransformManager.CONTENT_CLASS
        }

        @Override
        Set<? super QualifiedContent.Scope> getScopes() {
            // 作用范围。
            // 这里选择了整个打包项目的代码。
            return TransformManager.SCOPE_FULL_PROJECT
        }

        @Override
        boolean isIncremental() {
            // 是否支持增量编译。
            // 如果支持的话在 input 的数据中根据场景可能会包含 changed/removed/added 的文件。通过 jarInputs.state 判断该 jar 包文件在该次打包的状态。
            // 一般情况下处理可以分为 2 类：
            // 第一类是 removed 状态文件，应该在输出流进行手动删除。
            // 第二类是 notchanged、added、changed，在此基础上进行需求开发即可。
            return false
        }

        @Override
        void transform(TransformInvocation transformInvocation) throws TransformException, InterruptedException, IOException {
            // 具体的转换代码，默认情况下拿到输入后，不会自动原样输出，需要手动开发者实现。

            // 转换前的输入。
            def inputs = transformInvocation.inputs
            // 转换后的输出。下一个 Task 的 inputs 则是上一个 Task 的 outputs。
            def outputProvider = transformInvocation.outputProvider

            inputs.each {
                // inputs 有 2 种：
                // 1.directoryInput 集合：以源码方式参与项目编译的所有目录结构及其目录下的源码文件。
                // 2.JarInput 集合：参与项目编译的所有本地 jar 包和远程 jar 包。

                it.jarInputs.each {
                    println("start：${it.file}")
                    File dest = outputProvider.getContentLocation(it.name, it.contentTypes, it.scopes, Format.JAR)
                    println("dest：${dest}")

                    // 复制 gradle 的缓存 jar 包中到模块下的 build 文件夹下。例如：
                    // start：C:\Users\passin\.gradle\caches\modules-2\files-2.1\org.jetbrains.kotlin\kotlin-stdlib-jdk8\1.3.50\bf65725d4ae2cf00010d84e945fcbc201f590e11\kotlin-stdlib-jdk8-1.3.50.jar
                    // dest：D:\MyApplication\app\build\intermediates\transforms\passinTransform\debug\0.jar
                    FileUtils.copyFile(it.file, dest)
                }

                it.directoryInputs.each {
                    println(it.file)
                    File dest = outputProvider.getContentLocation(it.name, it.contentTypes, it.scopes, Format.DIRECTORY)
                    FileUtils.copyDirectory(it.file, dest)
                }
            }
        }
    }
}

```