
ASM 是一个广泛使用的字节码操作库，前面之所以 ASM 需要一点操作，是因为你需要熟悉字节码，还有一些 ASM 中各种 API 之间的关系。其次，既然是操作字节码的，那我们肯定需要先拿到字节码，这就涉及到了 Gradle Transform API 了。但是使用 Gradle Transform API 是依赖于 Gradle Plugin 的，所以可以看出想学 ASM，还是需要很多前置知识的。

还不是因为可以结合 Transform API 做一些很神奇的事？我们可以在自定义 Plugin 中去注册我们的 Transform，以此来干预打包流程。

Transform 在 Android 中都会被封装成 Task，Task 就是 Gradle 的灵魂。Android 中常见的 Task 有 ProGuard 混淆、MultiDex 重分包等等，我们写的 Transform 是会首先执行的，执行完之后再执行 Android 自带的 Transfrom。因此我们可以利用 TinyPng 在打包的时候批量压缩 res 下的所有 png 图片，甚至可以修改字节码，把项目中的 new Thread 全部替换成 CustomThread 等，Transform API 还是很强大的，本节先讲如何去自定义一个 Gradle Plugin，下一篇会仔细讲 Transform API 的玩法。


