![](/assets/trojan_banner.png)

<p align="center">
    <a href="https://gradleupdate.appspot.com/ELELogistics/Trojan/status">
        <img src="https://gradleupdate.appspot.com/ELELogistics/Trojan/status.svg">
    </a>
    <a href="https://www.java.com">
        <img src="https://img.shields.io/badge/language-Java-blue.svg">
    </a>
    <a href="https://raw.githubusercontent.com/EyreFree/EFQRCode/assets/icon/MadeWith%3C3.png">
        <img src="https://img.shields.io/badge/made%20with-%3C3-orange.svg">
    </a>
</p>

[Trojan](https://github.com/ELELogistics/Trojan)是一个稳定高效的移动端轻量级日志SDK，既可以记录通用日志，比如网络请求、电量变化、页面生命周期，也可以记录自定义的日志，从而可以通过用户日志来帮助我们定位分析问题。具有以下特点：

* 简洁的 API，通过几行代码就可以接入，实现日记记录功能；
* 使用 AOP 技术 [Lancet](https://github.com/eleme/lancet) 框架插桩收集通用日志，并且支持增量编译；
* 使用 mmap 技术，保证日记记录的高效性；
* 扩展性高，开发者可以自定义日志文件的上传功能；
* 流量开销小，支持在线配置，远程控制用户日志文件上传与否；
* 稳定性高，目前已稳定运行在饿了么物流团队众包等多个 APP 上。

> [English Introduction](/README.md)

## 综述

在开源的 [Trojan](https://github.com/ELELogistics/Trojan) SDK中，目前采集了 Activity 和 Fragment 生命周期，View Click 事件，网络状态变化，手机电量状态变化等基础日志，还通过 AOP 技术插桩 [KLog](https://github.com/ZhaoKaiQiang/KLog)采集 Log 日志，要是项目中未使用 KLog，也可以根据项目情况具体定制。考虑到每个项目中网络模块的实现框架都不尽相同，有 [OkHttp](https://github.com/square/okhttp)、[Volley](https://github.com/google/volley)、[Android-Async-Http](https://github.com/loopj/android-async-http) 等等，所以采集网络日志这一部分不方便统一处理，需要使用者通过 [Lancet](https://github.com/eleme/lancet) 插桩具体网络框架来收集日志。在 [Demo](https://github.com/ELELogistics/Trojan/blob/master/app/src/main/java/me/ele/trojan/demo/DemoHook.java) 中具体以 OkHttp 为例，实现采集 Http request 和 response 的功能，可作为参考。而与业务相关的日志，需要使用者自己采集。

## 配置

在根目录的 build.gradle 添加：

```java
buildscript {
    dependencies {
        .......
        classpath 'me.ele:lancet-plugin:1.0.2'
    }
}
```

在 app 目录的 'build.gradle' 添加：

```java
apply plugin: 'me.ele.lancet'

dependencies {
    ......
    provided 'me.ele:lancet-base:1.0.2'
    compile 'me.ele:trojan-library:0.1.6'
}
```

## 使用

### 1. 初始化

在自定义的 `Application` 添加：

```java
TrojanConfig config = new TrojanConfig.Builder(this)
    // Set user information
    .userInfo("xxxx")
    // Set device id
    .deviceId("xxxx")
    // Set cipher key if need encry log
    .cipherKey("xxxx")
    // Optional, save log file in sdcard by default
    .logDir("xxxx")
    // Console log switch, the default is open
    .enableLog(true)
    .build();
Trojan.init(config);
```

特别说明：

1. 日志文件默认保存在 sdcard 中，即使应用被卸载，也不会丢失日志；
2. 为兼容多进程，避免文件相互干扰，日志文件保存在各自的目录下，目录以进程名来命名；
3. 默认情况下日志不加密，考虑到加密的高效性，目前仅提供TEA加密方式。

### 2. 记录日志

Trojan 提供两种方式记录日志：

第一种：

```java
Trojan.log("Trojan", "We have a nice day!");
```

第二种：

```java
List<String> msgList = new LinkedList<>();
msgList.add("Hello Trojan!");
msgList.add("We have a nice day!");
msgList.add("Hello world!");
Trojan.log("Trojan", msgList);
```

默认情况下，对于单行日志不加密，如果需要加密，则使用如下：

```java
Trojan.log("Trojan", "We have a nice day!", true);
```

### 3. 用户信息

当用户信息发生改变或者切换用户时，可以调用：

```java
Trojan.refreshUser("new user info");
```

当然了，要是用户登出，可以传空值，即表示登出操作：

```java
Trojan.refreshUser(null);
```

### 4. 上传方案

针对日志上传，在 [Demo](https://github.com/ELELogistics/Trojan/blob/master/app/src/main/java/me/ele/trojan/demo/upload/DemoLeanCloudUploader.java) 中提供了 [LeanCloud](https://leancloud.cn/) 这种免费简单的方式，可以实现上传、浏览、下载等文件服务的基本功能，可供参考。

### 5. 数据解密

当设置了加密秘钥，为保证敏感数据的安全性，可以加密单行日志，在日志分析时就需要对加密数据进行解密。[解密脚本](/decrypt/trojan_decrypt.py)使用如下：

1. 在MAC上编译生成解密SO库，在仓库中已经生成了so库，这一步可以省略：
    
    ```java
    gcc -shared -Wl,-install_name,trojan_decrypt.so -o trojan_decrypt.so -fPIC trojan_decrypt.c
    
    ```
2. 在MAC上调用python脚本解密数据，需要传入解密秘钥和待解密的文件路径，需要注意的是python脚本的路径要对：

    ```java
    python ./trojan_decrypt.py cipher-key cipher-file-path
    
    ```

## 备注

通过集成 [Trojan](https://github.com/ELELogistics/Trojan)，可以轻松实现用户日志的记录功能，是不是很简单呀！要是对 Lancet 的使用有疑问，大家可以参考 [https://github.com/eleme/lancet/blob/dev/README_zh.md](https://github.com/eleme/lancet/blob/dev/README_zh.md)，这里就不赘述。

## 协议

![](/assets/trojan_license.png)

Trojan基于Apache-2.0协议进行分发和使用，更多信息参见[协议](/LICENSE)文件。
