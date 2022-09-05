[SimpleButterKnife]([https://www.runoob.com](https://github.com/caiweihao/SimpleButterKnife))
========

`SimpleButterKnife` 用两种方式来实现findViewById功能。
1. 使用反射（运行时获取）
2. Annotation Processor

上面两种方式都会使用注解，先来简单介绍一下注解。
```java
package com.example.simplebutterknife;

import java.lang.annotation.ElementType;
import java.lang.annotation.Retention;
import java.lang.annotation.RetentionPolicy;
import java.lang.annotation.Target;

@Retention(RetentionPolicy.RUNTIME) //注解保留范围，SOURCE代表代码级别（Annotation Processor可以使用），CLASS代表编译器级别 ， RUNTIME代码VM级别（反射可以使用）
@Target(ElementType.FIELD) //注解使用范围
public @interface ReflectBindView {
    int value();
}
```
   
因为运行时注解须要在Activity初始化中进行绑定操做，调用了大量反射相关代码，在界面复杂的状况下，使用这种方法就会严重影响Activity初始化效率。而ButterKnife使用了更高效的方式——Annotation Processor来完成这一工作，那么什么是Annotation Processor呢？
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



