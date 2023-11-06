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
### 反射相关功能

```java
package com.example.simplebutterknife;

import android.app.Activity;

import java.lang.reflect.Field;
import java.lang.reflect.InvocationTargetException;
import java.lang.reflect.Method;

public class ReflectBindViewUtil {
    public static void bind(Activity activity) {
        Class clazz = activity.getClass();
        Field[] fields = clazz.getDeclaredFields();
        for (Field field : fields) {
            if (field.isAnnotationPresent(ReflectBindView.class)) {
                ReflectBindView annotation = field.getAnnotation(ReflectBindView.class);
                int resId = annotation.value();
                try {
                    Method method = clazz.getMethod("findViewById", int.class);
                    Object invoke = method.invoke(activity, resId);
                    field.setAccessible(true);
                    field.set(activity, invoke);
                } catch (NoSuchMethodException e) {
                    e.printStackTrace();
                } catch (InvocationTargetException e) {
                    e.printStackTrace();
                } catch (IllegalAccessException e) {
                    e.printStackTrace();
                }
            }
        }
    }

    public static void unbind(Activity activity) {
        Class clazz = activity.getClass();
        Field[] fields = clazz.getDeclaredFields();
        for (Field field : fields) {
            if (field.isAnnotationPresent(ReflectBindView.class)) {
                try {
                    field.setAccessible(true);
                    field.set(activity, null);
                } catch (IllegalAccessException e) {
                    e.printStackTrace();
                }
            }
        }
    }

    public static void showMyBindField(Activity activity, String prefix) {
        Class clazz = activity.getClass();
        Field[] fields = clazz.getDeclaredFields();
        for (Field field : fields) {
            if (field.isAnnotationPresent(ReflectBindView.class)) {
                field.setAccessible(true);
                try {
                    System.out.println(prefix + " field name is " + field.getName() + " value is " + field.get(activity));
                } catch (IllegalAccessException e) {
                    e.printStackTrace();
                }
            }
        }
    }
}

```
Annotation Processor使用 [javapoet](https://github.com/square/javapoet)生成Java文件。
Android整个编译过程就是 source(源代码) -> processor（处理器） -> generate （文件生成）-> javacompiler -> .class 文件 -> .dex(只针对安卓)。
![Annotation Processor processing过程](https://github.com/caiweihao/SimpleButterKnife/blob/master/icons/processing.jpg)
### AbstractProcessor实现如下
```java
package com.example.lib;
import java.io.IOException;
import java.util.Collections;
import java.util.Set;

import com.example.lib_annotation.BindView;
import com.google.auto.service.AutoService;
import com.squareup.javapoet.ClassName;
import com.squareup.javapoet.JavaFile;
import com.squareup.javapoet.MethodSpec;
import com.squareup.javapoet.TypeSpec;

import javax.annotation.processing.AbstractProcessor;
import javax.annotation.processing.Filer;
import javax.annotation.processing.ProcessingEnvironment;
import javax.annotation.processing.Processor;
import javax.annotation.processing.RoundEnvironment;
import javax.annotation.processing.SupportedSourceVersion;
import javax.lang.model.SourceVersion;
import javax.lang.model.element.Element;
import javax.lang.model.element.Modifier;
import javax.lang.model.element.TypeElement;
import javax.tools.Diagnostic;

//@SupportedAnnotationTypes("com.example.lib_annotation.BindView")
@SupportedSourceVersion(SourceVersion.RELEASE_8)
@AutoService(Processor.class)
public class BindingProcessor extends AbstractProcessor {
    private Filer filer;
    @Override
    public synchronized void init(ProcessingEnvironment processingEnv) {
        super.init(processingEnv);
        filer = processingEnv.getFiler();
    }

    @Override
    public boolean process(Set<? extends TypeElement> annotations, RoundEnvironment roundEnv) {
        System.out.println("BindingProcessor start");
        for (Element element : roundEnv.getRootElements()) {
            String packageStr = element.getEnclosingElement().toString();
            String classStr = element.getSimpleName().toString();
            ClassName className = ClassName.get(packageStr, classStr + "$Binding");
            MethodSpec.Builder constructorBuilder = MethodSpec.constructorBuilder()
                    .addModifiers(Modifier.PUBLIC)
                    .addParameter(ClassName.get(packageStr, classStr), "activity");
            boolean hasBinding = false;
            for (Element e: element.getEnclosedElements()) {
                BindView bindView = e.getAnnotation(BindView.class);
                if (bindView != null) {
                    hasBinding = true;
                    constructorBuilder.addStatement
                            ("activity.$L = activity.findViewById($L)", e.getSimpleName()
                                    , bindView.value());
                }
            }
            TypeSpec buildClass = TypeSpec.classBuilder(className)
                    .addModifiers(Modifier.PUBLIC)
                    .addMethod(constructorBuilder.build()
                    )
                    .build();
            if (hasBinding) {
                try {
                    JavaFile.builder(packageStr, buildClass)
                            .build()
                            .writeTo(filer);
                } catch (IOException e) {
                    e.printStackTrace();
                }
            }
        }
        processingEnv.getMessager().printMessage(Diagnostic.Kind.NOTE, "BindingProcessor start process annotation");
        return false;
    }

    @Override
    public Set<String> getSupportedAnnotationTypes() {
        System.out.println("BindingProcessor getSupportedAnnotationTypes");
        return Collections.singleton(BindView.class.getCanonicalName());
    }
}
```
### 重新编译
![重新编译](https://github.com/caiweihao/SimpleButterKnife/blob/master/icons/rebuild.jpg)

### 产生Java如下
![产生文件](https://github.com/caiweihao/SimpleButterKnife/blob/master/icons/generatedJava.jpg)

### 通过反射完成映射
```java
package com.example.simplebutterknife;

import android.app.Activity;

import java.lang.reflect.Constructor;
import java.lang.reflect.InvocationTargetException;

public class ProcessBindViewUtil {
    public static void bind(Activity activity) {
        try {
            Class bindClass = Class.forName(activity.getClass().getCanonicalName() + "$Binding");
            Class activityClass = activity.getClass();
            Constructor constructor = bindClass.getConstructor(activityClass);
            constructor.newInstance(activity);
        } catch (ClassNotFoundException e) {
            e.printStackTrace();
        } catch (NoSuchMethodException e) {
            e.printStackTrace();
        } catch (InvocationTargetException e) {
            e.printStackTrace();
        } catch (IllegalAccessException e) {
            e.printStackTrace();
        } catch (InstantiationException e) {
            e.printStackTrace();
        }


    }
}
```
