<!-- TOC -->

- [一、前言](#一前言)
- [二、应用安全活动周期](#二应用安全活动周期)
    - [2.1 开发阶段](#21-开发阶段)
    - [2.2 测试阶段](#22-测试阶段)
    - [2.3 上线阶段](#23-上线阶段)
    - [2.4 运营阶段](#24-运营阶段)
- [三、应用安全案例](#三应用安全案例)
    - [3.1 漏洞修复](#31-漏洞修复)
        - [3.1.1 业务漏洞](#311-业务漏洞)
        - [3.1.2 WebView 漏洞](#312-webview-漏洞)
        - [3.1.3 Intent Scheme URL](#313-intent-scheme-url)
    - [3.2 功能泄漏](#32-功能泄漏)
        - [3.2.1 PendingIntent](#321-pendingintent)
        - [3.2.2 外部应用调组件](#322-外部应用调组件)
        - [3.2.3 广播](#323-广播)
    - [3.3 逆向对抗](#33-逆向对抗)
        - [3.3.1 二次打包](#331-二次打包)
        - [3.3.2 debuggable](#332-debuggable)
        - [3.3.3 检查环境](#333-检查环境)
        - [3.3.4 明文字符串](#334-明文字符串)
        - [3.3.5 代码混淆](#335-代码混淆)
        - [3.3.6 核心逻辑保护](#336-核心逻辑保护)
    - [3.4 数据安全](#34-数据安全)
        - [3.4.1 数据泄露](#341-数据泄露)
        - [3.4.2 密码输入](#342-密码输入)
        - [3.4.3 密码学](#343-密码学)
            - [3.4.3.1 固定初始化向量](#3431-固定初始化向量)
            - [3.4.3.2 密钥硬编码](#3432-密钥硬编码)
            - [3.4.3.3 不安全的密码学实现](#3433-不安全的密码学实现)
            - [3.4.3.4 不安全的 Hash 算法](#3434-不安全的-hash-算法)
    - [3.5 网络安全](#35-网络安全)
        - [3.5.1 校验服务器主机](#351-校验服务器主机)
        - [3.5.2 SSL 证书校验](#352-ssl-证书校验)
- [参考资料](#参考资料)

<!-- /TOC -->
 
# 一、前言

从本质上讲，客户端没有什么是彻底牢不可破的，只能从各方面出发去提高逆向的成本。

Android 安全防护，从开发者角度来说主要分为三个层面去防护：

1. 增大应用逆向的难度。
2. 解决应用存在的漏洞。
3. 若应用被逆向成功后，增大源码的阅读难度。

而本文的核心主要围绕第 2、3 个层面，也就是普通开发者的角度去做安全防护，若公司没有专门的安全部门，第 1 个层面建议直接使用第三方安全厂商提供的 **付费** 安全方案。

# 二、应用安全活动周期

应用安全活动周期可分为 **开发阶段**、**测试阶段**、**预发布阶段**、**运营阶段** 四个阶段去逐步推进。

## 2.1 开发阶段

开发者应当在开发前去审阅整个开发流程中可能出现的安全问题或注意事项。下一章会以案例的方式去简述常见的安全点：

## 2.2 测试阶段

测试阶段是对应用进行安全扫描以及渗透测试。

安全扫描一般使用第三方安全厂商（360）提供的安全扫描即可，除此之外，现在部分应用商店应用审核阶段也包含了安全扫描。

渗透测试基本上都是人工渗透，是针对目标场景进行入侵尝试，比如一个支付应用，支付密码、协议保护这些都是一个关键的覆盖场景。

## 2.3 上线阶段

将安全扫描的存在的漏洞进行修补。这些漏洞一部分可以通过第三方安全厂商加固（加壳、vmp、java2c、h5 保护）或提供安全组件 SDK（安全键盘、传输协议加密、设备指纹）解决，另一部分需要我们在开发阶段处理好。

## 2.4 运营阶段

应用发布之后，需要监控应用到底有没有被破解，开发者需要根据监控数据做分析，需要做相应的安全策略升级和对抗。

# 三、应用安全案例

## 3.1 漏洞修复

### 3.1.1 业务漏洞

示例一：

验证码验证和账号生成两个接口调用，客户端在调验证码验证成功后，再调账号生成接口生成账号，攻击者可在逆向成功后直接调账号生成接口，创建账号。

示例二：

某 App 核心业务只依赖于客户端实现，服务器接口主要起到付费拦截的功能，攻击者可在逆向成功后修改控制流逻辑直接使用本地的服务。

开发建议：

1. 验证和结果应当作为一个接口供客户端去调用。
2. 核心业务应当交由服务器确定结果后再下发内容给客户端执行。
3. 将业务逻辑放置服务器相对来说是最安全的做法。

示例三：zip 中的文件不允许使用../../file 这样的路径，因为可能被篡改目录结构，造成攻击。攻击者可以利用多个"../"在解压时改变 zip 文件存放的位置，当文件已经存在是就会进行覆盖，如果覆盖掉的文件是 so、dex 或者 odex 文件，就有可能造成严重的安全问题。

开发建议：对文件路径进行判断，存在".."时抛出异常，重要的 zip 文件需要验签。


### 3.1.2 WebView 漏洞

[WebView 漏洞](./hybrid%20开发#221-webviewaddjavascriptinterface-%E5%8F%8A%E6%BC%8F%E6%B4%9E%E5%A4%84%E7%90%86)

### 3.1.3 Intent Scheme URL

示例：如果浏览器支持 Intent Scheme Uri 语法，如果过滤不当，那么恶意用户可能通过浏览器 js 代码进行一些恶意行为，比如盗取 cookie 等。

开发建议：

```java
// 如果使用了 parseUri()，intent 至少得包含以下 3 个策略。
Intent intent = Intent.parseUri(uri);   
intent.addCategory("android.intent.category.BROWSABLE");
intent.setComponent(null);
intent.setSelector(null);
context.startActivityIfNeeded(intent,-1);
```

## 3.2 功能泄漏

### 3.2.1 PendingIntent

示例：PendingIntent 可以让其他 App 中的代码像是运行自己 App 中，因此在使用 PendingIntent 时，PendingIntent 中的 intent 如果是隐式的 Intent，容易遭到劫持，导致信息泄露。

开发建议：禁止使用一个空 Intent 去构造 PendingIntent，构造 PendingIntent 的 Intent 一定要设置 ComponentName 或者 Action。

### 3.2.2 外部应用调组件

示例：恶意应用可通过向受害者应用通过 Intent 向组件发送此空数据、异常或者畸形数据，若没有进行异常容错处理，则可能发生 Crash 从而导致应用不可使用（本地拒绝服务）。

开发建议：

1. 仅内部使用的组件设置 "android:exported" 属性为 false。包含 intent-filter 时 exported 默认值为 true，反之为 false。
2. Receiver/Provider 不能在毫无权限控制的情况下，将 android:export 设置为 true。
3. 对外对外服务的组件需对接收的数据（Intent）进行容错处理（例如 try catch）。

### 3.2.3 广播

示例：仅应用内使用的广播，通过 Activity.sendBroadcast() 的形式发送被外部应用接收。

开发建议：

1. 应用内的广播使用 LocalBroadcastManager.getInstance(this).sendBroadcast(intent) 进行发送，或调 Intent.setPackage() 做包名限制。
2. 需求对外使用的广播，注意不要广播敏感信息，同时发送方和接受方要指定好相应的访问权限。

## 3.3 逆向对抗

### 3.3.1 二次打包

示例：App 可被修改，被套壳加入攻击者代码，重新打包发布。

开发建议：在 native 层启动签名校验，非原签名直接崩溃或无法正常运行逻辑。

### 3.3.2 debuggable

示例：生产环境的应用 android:debuggable 属性为 true 导致应用可以 debug。

开发建议：针对环境做控制，Release 包 android:debuggable 设置为 false。
 
### 3.3.3 检查环境

示例：攻击者通常需要 root、模拟器 或者 xposed 框架环境。

开发建议：可按需针对手机是否 root，是否是模拟器，是否处于 xposed 框架环境直接将应用挂起。

### 3.3.4 明文字符串

示例：应用被逆向后，关键字符串直接被全局搜索到线索，例如密钥。

开发建议：代码中的的关键字符串需要加密处理。

### 3.3.5 代码混淆

示例：apk 被反编译后，直接明文看到代码源代码。

开发建议：java、资源、c/c++ 均可混淆，增加逆向难度。

- [Android 混淆](./Android%20混淆.md)
- c/c++混淆工具：[LLVM](https://github.com/obfuscator-llvm/obfuscator/wiki) 是逆向对抗利器，它的处理过程是一般的编译器都有的三个过程，包括前端、优化器（中间件）、后端。通过这个优化器我们可以实现各种效果，例如代码控制流扁平化，虚假控制流，字符串加密，符号混淆，指令替换等。

### 3.3.6 核心逻辑保护

示例：Java 层代码攻击门槛低，易于逆向。

开发建议：核心逻辑保护在 native 层实现，结合逻辑混淆，提高逆向门槛。

## 3.4 数据安全

### 3.4.1 数据泄露

示例一：为了方便对应用数据的备份和恢复，在 Android 2.2 以后增加了 android:allowBackup 属性值（默认为 true），可通过 adb backup 和 adb restore 来备份（导出）和恢复应用程序数据。

开发建议：将 android:allowbackup 属性设置为 false，防止 adb backup 导出数据。

示例二：手机 Root 后，可导出数据库数据，直接查看 SharedPreferences 的 xml 文件，泄漏敏感信息。

开发建议：存储在数据库、SharedPreferences 等轻量级存储的数据需要对数据进行加密，取出来的时候进行解密。

示例三：在 App 的开发过程中，为了方便调试，通常会使用 Log 函数输出一些关键流程的信息，这些信息中通常会包含敏感内容，如执行流程、明文的用户名密码等。

开发建议：针对不同环境做日志控制，生产环境关闭日志，开发环境打开日志。

密钥加密存储或者经过变形处理后用于加解密运算，切勿硬编码到代码中。说明：应用程序在加解密时，使用硬编码在程序中的密钥，攻击者通过反编译拿到密钥可以轻易解密 App 通信数据。

示例四：所需的动态文件下载到共有目录被恶意修改。

开发建议：将所需要动态加载的文件放置在 apk 内部或应用私有目录中，如果应用必须要把所加载的文件放置在可被其他应用读写的目录中 (比如 sdcard)，建议对不可信的加载源进行完整性校验和白名单处理，以保证不被恶意代码注入。

### 3.4.2 密码输入

示例：/dev/input/event 可以读取到按键和触屏，并可进行录屏从而观察到输入过程。

开发建议：使用系统厂商提供的安全键盘，会在使用期间禁止截屏和录屏；对于金融级别的安全密码可选择自定义安全键盘（包括随机按键），并且代码层面上禁止输入界面的截屏和录屏。

### 3.4.3 密码学

[分组加密的四种模式(ECB、CBC、CFB、OFB)](https://www.cnblogs.com/happyhippy/archive/2006/12/23/601353.html)：

#### 3.4.3.1 固定初始化向量

示例：使用固定初始化向量，结果密码文本可预测性会高得多，容易受到字典式攻击。

开发建议：禁止使用常量初始化矢量参数构建 IvParameterSpec，建议 IV 通过随机方式产生。IV 的作用主要是用于产生密文的第一个 block，以使最终生成的密文产生差异（明文相同的情况下），使密码攻击变得更为困难，除此之外 IV 并无其它用途。在使用随机 IV 与服务器交互时，发送方需要将随机生成的 IV 通过请求头或其他安全的方式发送给对方。

#### 3.4.3.2 密钥硬编码

示例：某 App 与服务器通信的接口采用 Http 传输数据，但有对传输的部分参数进行了加密，加密算法采用 AES，但是密钥硬编码在 Java 代码中。App 被逆向后可分析请求参数，从而伪造请求引起越权访问的风险，如越权查看其它用户的订单等。

开发建议：

1. 禁止直接硬编码密钥，可将密钥拆分加密存放，或使用白盒密钥，增加逆向难度。
2. 加解密在 native 层进行，可将密钥加密后拆分存在在多个普通文件中，文件则存放在 assets 目录下。
   
#### 3.4.3.3 不安全的密码学实现

示例：使用 Android 的 AES/DES/DESede 加密算法时，其中 ECB 模式的安全性较弱，由于所有分组的加密方式一致，明文中的重复内容会在密文中有所体现，因此难以抵抗统计分析攻击。

开发建议：建议使用 CBC 或 CFB 加密模式，且 IV 随机生成。默认的加密模式 ECB 仅一般只适用于小数据量的字符信息加密，例如秘钥。

#### 3.4.3.4 不安全的 Hash 算法

示例：不安全的 Hash 算法 (MD5/SHA-1) 容易被差分攻击破译，它的基本原理是通过比较分析有特定区别的明文在通过加密后的变化传播情况来产生相同的输出。

加密算法：使用 SHA-256 等安全性更高的 Hash 算法。

## 3.5 网络安全

### 3.5.1 校验服务器主机

示例：在握手期间，如果 URL 的主机名和服务器的标识主机名不匹配，则有安全风险，因为默认接受所有域名。

开发建议：实现的 HostnameVerifier 子类，对服务器主机名进行校验；或使用 OkHttp 提供的默认实现 OkHostnameVerifier。

```java
public class TrustSpecifyHostnameVerifier implements HostnameVerifier {

    private final Set<String> hostNameSet = new HashSet<>();

    public TrustSpecifyHostnameVerifier() {
        hostNameSet.add("hostname1");
        hostNameSet.add("hostname2");
        hostNameSet.add("hostname3");
        // ……
    }

    @Override
    public boolean verify(String hostname, SSLSession session) {
        if (hostList.contains(hostname)) {
            return true;
        } else {
            final HostnameVerifier hv = HttpsURLConnection.getDefaultHostnameVerifier();
            return hv.verify(hostname, session);
        }
    }
}
```

### 3.5.2 SSL 证书校验

示例：没有对证书进行校验，具体代码如下所示：

```java
TrustManager tm = new X509TrustManager() {

    public java.security.cert.X509Certificate[] getAcceptedIssuers() {
        // 返回受信任的 X509 证书数组。
        return null;
    }

    @Override
    public void checkClientTrusted(java.security.cert.X509Certificate[] chain, String authType)
            throws java.security.cert.CertificateException {
        // 检查客户端证书，检查不通过则手动抛 CertificateException 异常。
    }

    @Override
    public void checkServerTrusted(java.security.cert.X509Certificate[] chain, String authType)
            throws java.security.cert.CertificateException {
        // 检查服务器证书，检查不通过则手动抛 CertificateException 异常。
    }
};
sslContext.init(null, new TrustManager[]{tm}, null);
```

开发建议：使用 TrustManagerFactory.getDefaultAlgorithm() 构建默认的 TrustManager，或自己实现 TrustManager。

# 参考资料 

- [Android 应用安全提升攻略](https://mp.weixin.qq.com/s/QEAYGLlR2qQQSfp4ObetZw)
- [阿里巴巴 Android 开发手册](https://edu.aliyun.com/course/813?utm_content=g_1000029585)
