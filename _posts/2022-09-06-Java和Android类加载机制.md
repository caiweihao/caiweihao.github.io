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
| 类加载器名称 | 功能 |
| :----:| :----:  | 
| 启动类加载器 BootStrapClassLoader | 启动类加载器主要加载的是JVM自身需要的类，这个类加载使用C++语言实现的，负责加载<JAVA_HOME>/lib目录下的类，是虚拟机自身的一部分。 |
| 扩展类加载器 ExtensionClassLoader | 扩展类加载器是由Java语言实现的，是Launcher的静态内部类，它负责加载<JAVA_HOME>/lib/ext目录下或者由系统变量-Djava.ext.dir指定位路径中的类库。 |
| 系统类加载器 SystemClassLoader | 它负责加载系统类路径java -classpath或-D java.class.path 指定路径下的类库，也就是我们经常用到的classpath路径，开发者可以直接使用系统类加载器，一般情况下该类加载是程序中默认的类加载器，通过ClassLoader#getSystemClassLoader()方法可以获取到该类加载器 |
| 自定义类加载器 | 负责记载自己定义位置的类 |

上面表格中下边的加载器持有上面加载器的引用，不是继承关系。
双亲委派机制，其工作原理的是，如果一个类加载器收到了类加载请求，先去查找缓存的加载类中查找，如果没有找到，它并不会自己先去加载，而是把这个请求委托给父类的加载器去执行，如果父类加载器还存在其父类加载器，则进一步向上委托，依次递归，请求最终将到达顶层的启动类加载器，如果父类加载器可以完成类加载任务，就成功返回，倘若父类加载器无法完成此加载任务，子加载器才会尝试自己去加载，

双亲委派机制的优势：采用双亲委派模式的是好处是Java类随着它的类加载器一起具备了一种带有优先级的层次关系，通过这种层级关可以避免类的重复加载，当父亲已经加载了该类时，就没有必要子ClassLoader再加载一次。其次是考虑到安全因素，java核心api中定义类型不会被随意替换，假设通过网络传递一个名为java.lang.Integer的类，通过双亲委托模式传递到启动类加载器，而启动类加载器在核心Java API发现这个名字的类，发现该类已被加载，并不会重新加载网络传递的过来的java.lang.Integer，而直接返回已加载过的Integer.class，这样便可以防止核心API库被随意篡改。
向上委托保证了核心Java API不会被恶意替换（安全），向下委托保证类能够被加载。

Java加载的是class， Android加载的是dex（相同常量会缩减为一份），一个dex可以有多个class。

Android热修复实现原理
-----------------
1. 经过对PathClassLoader、DexClassLoader、BaseDexClassLoader、DexPathList的分析，我们知道，安卓的类加载器在加载一个类时会先从自身DexPathList对象中的Element数组中获取（Element[] dexElements）到对应的类，之后再加载。采用的是数组遍历的方式，不过注意，遍历出来的是一个个的dex文件。
2. 在for循环中，首先遍历出来的是dex文件，然后再是从dex文件中获取class，所以，我们只要让修复好的class打包成一个dex文件，放于Element数组的第一个元素，这样就能保证获取到的class是最新修复好的class了（当然，有bug的class也是存在的，不过是放在了Element数组中比修复bug的dex靠后的位置，所以没有机会被加载，类不会重复加载）。
3. 当ClassLoader加载到正确的类之后就不会去加载错误的类了 ，所以可以在dexElements中将正确的类放在错误类的前面就可以了。找到修复bug的类之后，将修复bug的类打包程dex文件，将其放在dexElements中的最前方。

修复代码如下：

```java
try {
            String fileName = "hotfix-debug.dex";
            File apk = new File(getCacheDir() + fileName);
            ClassLoader classLoader = getClassLoader();
            Class loaderClass = BaseDexClassLoader.class;
            Field pathListField = loaderClass.getDeclaredField("pathList");
            pathListField.setAccessible(true);
            Object pathListObject = pathListField.get(classLoader);
            Class pathListClass = pathListObject.getClass();
            Field dexElementsField = pathListClass.getDeclaredField("dexElements");
            dexElementsField.setAccessible(true);
            Object dexElementsObject = dexElementsField.get(pathListObject);

            PathClassLoader newClassLoader = new PathClassLoader(apk.getPath(), null);
            Object newPathListObject = pathListField.get(newClassLoader);
            Object newDexElementsObject = dexElementsField.get(newPathListObject);
            int oldLength = Array.getLength(dexElementsObject);
            int newLength = Array.getLength(newDexElementsObject);
            Object concateObject = Array.newInstance(dexElementsObject.getClass().getComponentType(), oldLength + newLength);
            for (int i = 0; i < newLength; i++) {
                Array.set(concateObject, i, Array.get(newDexElementsObject, i));
            }
            for (int i = 0; i < oldLength; i++) {
                Array.set(concateObject, newLength + i, Array.get(dexElementsObject, i));
            }
            dexElementsField.set(pathListObject, concateObject);
        } catch (NoSuchFieldException e) {
            e.printStackTrace();
        } catch (IllegalAccessException e) {
            e.printStackTrace();
        }
```
