Java和Android类加载机制以及相关使用场景
=================

Java类加载机制
-----------------
```java
protected Class<?> loadClass(String name, boolean resolve)
        throws ClassNotFoundException
    {
        synchronized (getClassLoadingLock(name)) {
            // First, check if the class has already been loaded
            Class<?> c = findLoadedClass(name);
            if (c == null) {
                long t0 = System.nanoTime();
                try {
                    if (parent != null) {
                        c = parent.loadClass(name, false);
                    } else {
                        c = findBootstrapClassOrNull(name);
                    }
                } catch (ClassNotFoundException e) {
                    // ClassNotFoundException thrown if class not found
                    // from the non-null parent class loader
                }

                if (c == null) {
                    // If still not found, then invoke findClass in order
                    // to find the class.
                    long t1 = System.nanoTime();
                    c = findClass(name);

                    // this is the defining class loader; record the stats
                    sun.misc.PerfCounter.getParentDelegationTime().addTime(t1 - t0);
                    sun.misc.PerfCounter.getFindClassTime().addElapsedTimeFrom(t1);
                    sun.misc.PerfCounter.getFindClasses().increment();
                }
            }
            if (resolve) {
                resolveClass(c);
            }
            return c;
        }
    }

```
| 类加载器名称 | 功能 | 居中对齐 |
| :----:| :----:  | :----: |
| 启动类加载器 BootStrapClassLoader | 启动类加载器主要加载的是JVM自身需要的类，这个类加载使用C++语言实现的，负责加载<JAVA_HOME>/lib目录下的类，是虚拟机自身的一部分。 | 单元格 |
| 扩展类加载器 ExtensionClassLoader | 扩展类加载器是由Java语言实现的，是Launcher的静态内部类，它负责加载<JAVA_HOME>/lib/ext目录下或者由系统变量-Djava.ext.dir指定位路径中的类库。 | 单元格 |
| 系统类加载器 SystemClassLoader | 它负责加载系统类路径java -classpath或-D java.class.path 指定路径下的类库，也就是我们经常用到的classpath路径，开发者可以直接使用系统类加载器，一般情况下该类加载是程序中默认的类加载器，通过ClassLoader#getSystemClassLoader()方法可以获取到该类加载器 | 单元格 |
| 扩展类加载器 ExtensionClassLoade | 负责记载自己定义位置的加载器 | 单元格 |

