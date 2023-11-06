Kotlin序列化是一个跨平台的支持多种格式的序列化方法，支持ProtocolBuffer，JSON等格式。Kotlin序列化不仅仅是一个框架，框架提供的插件生成可访问的访问者式的代码。

`Kotlin serialization ` Kotlin DSL配置如下
1. 增加Kotlin serialization的plugin
2. 增加serialization的json依赖
配置如下：
```java
plugins {
    kotlin("jvm") version "1.9.0"
    kotlin("plugin.serialization") version "1.9.0"
}

repositories {
    mavenCentral()
}

dependencies {
    implementation("org.jetbrains.kotlinx:kotlinx-serialization-json:1.6.0")
    implementation("com.google.code.gson:gson:2.10.1")
}

```
   
Java的JSON解析和Kotlin有很大区别：
| JSON编解码异同| Java | Kotlin|
| :----:| :----:  | :----:  | 
| 属性是否为空 | Java的属性是可以为空的，所以从字符串解码时候，相应的属性对（名称和值）可以缺失 |没有默认值的属性不可以缺失 |
| 类名是否需要标记 | 不需要 |需要用@Serializable注解进行标识 |
|倾向于使用什么技术 | 关键字如transient |需要用 @Transient注解进行标识 |
|更改属性名称 | 需要用@SerializedName("title")注解进行标识，字符串中只有用title才能进行，用原来的属性名不行 |需要用 @JsonNames("title")注解进行标识，用title和原来的属性名都可以 |


