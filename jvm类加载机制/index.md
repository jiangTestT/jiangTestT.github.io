# JVM类加载机制


## JVM类加载机制

### 类加载步骤

 JVM 采用的是懒加载机制，即只有类在被使用时才加载。类的加载主要可分为以下几步：

**加载：**即把类字节码文件加载到内存，在一步会在内存中生成一个代表这个类的`java.lang.Class`对象，作为方法区这个类的各种数据的访问入口。

**验证：**校验字节码文件的准确性，每个字节码文件都有固定的格式，以此来校验字节码文件是否被损坏。

**准备：**给静态变量分配内存空间并赋与默认值。

**解析：**将符号引用替换为直接引用，该阶段会把一些静态方法(符号引用，比如 main() 方法)替换为指向数据所在内存的指针或句柄等（直接引用），这是所谓的静态链接过程（类加载期间完成），动态链接是在程序运行期间完成的将符号引用替换为直接引用。

**初始化：**对类变量初始化赋值，并执行静态代码块。

### 类加载器

 JVM 的类加载主要有四类加载器来配合完成。

**引导类加载器（BootstrapClassLoad）：**负责加载支撑JVM运行的位于JRE的lib目录下的核心类库，比如 rt.jar、charsets.jar 等。为避免核心类库被篡改，这类核心类库只能有引导类加载器加载，即使自定义类加载指定去加载该路径下的类也会报错无法加载成功。引导类加载器由 C++ 代码实现，因此，在 JAVA 程序中对它的引用值为 null。

**扩展类加载器（ExtClassLoad）：**负责加载支撑 JVM 运行的位于JRE的lib目录下的 ext 扩展目录中的 JAR 类包。

**应用程序类加载器（AppClassLoad）：**负责加载 ClassPath 路径下的类包，主要就是加载你自己写的那些类。

**自定义加载器：**负责加载用户自定义路径下的类包。自定义加载器需要继承 ClassLoad 类，同时，可根据自身业务需要是否要重新其加载类的 loadClass 和 findClass 方法。

**注意：类加载器从上往下构成"父子关系",注意，是在加载器类中有一个 parent 属性的值为上一层类加载器对象，并非 JAVA 层面的类继承关系。**

### 类加载机制

JVM 的类加载机制采用的是双亲委派机制：

1. 当加载一个类时，如果使用自定义类加载器，则从自定义加载器查看该加载器是否加载过此类，如果没有则委托上层的 **AppClassLoad** 去加载。
2. 在 **AppClassLoad** 加载时，同样的检查当前类加载器是否加载过此类，如果没有则委托上层的 **ExtClassLoad** 去加载。
3. 在 **ExtClassLoad** 加载时，同样的检查当前类加载器是否加载过此类，没有则委托上层的 **BootstrapClassLoad** 去加载。
4. 在 **BootstrapClassLoad** 加载时，同样的检查当前类加载器是否加载过此类，如果没有，则由 **BootstrapClassLoad** 去加载此类。
5. 如果在 **BootstrapClassLoad** 管辖的路径下没加载到此类，则有下层的 **ExtClassLoad** 去加载。
6. 如果在 **ExtClassLoad** 管辖的路径下没加载到此类，则有下层的 **AppClassLoad** 去加载。
7. 如果在 **AppClassLoad** 管辖的路径下没加载到此类，则有下层的自定义类加载器去加载。
8. 再加载不到则抛出异常。可以总结成一句话:**从下往上检查类是否被加载过，从上往下去加载类。**

![](https://crry-assist.oss-cn-chengdu.aliyuncs.com/Screenshot/%E5%8F%8C%E4%BA%B2%E5%A7%94%E6%B4%BE%E6%9C%BA%E5%88%B6.png)

查看自己代码的类加载器

```java
public class TestClass {

    public static void main(String[] args) {
        ClassLoader currentClassLoader = TestClass.class.getClassLoader();
        System.out.println(currentClassLoader);
        System.out.println(currentClassLoader.getParent());
        System.out.println(currentClassLoader.getParent().getParent());
    }
}
```

返回结果如下：

```tex
sun.misc.Launcher$AppClassLoader@18b4aac2
sun.misc.Launcher$ExtClassLoader@6f539caf
null
```

我们可以发现，当前类的类加载器为 AppClassLoader，其父加载器为 ExtClassLoader。而 ExtClassLoader 的父加载器输出为 **null**。这其实是 BootstrapClassLoad。

### 双亲委派机制的意义

 **沙箱安全机制：**各个类加载器管辖自己路径下类的加载，比如核心类库只能有引导类加载器加载，可以防止核心类库被篡改。自己写一个跟核心类库一模一样类名、包名的 Object、String 类，是不会被加载成功的。

**避免类的重复加载：**当父亲已经加载了该类时，就没有必要子 ClassLoader 再加载一 次，保证被加载类的唯一性。

**提升类加载效率：**因为在应用程序中，90% 以上的类都是程序员自己写的类，所以从下往上开始检查能提升类加载的效率。

### 打破双亲委派机制

打破双亲委派的两种方式：

1.通过 spi 机制，使用 ServiceLoader.load 去加载

2.通过自定义类加载器，继承 classloader，重写 loadclass 方法

#### SPI机制

spi 机制是一种服务发现机制。它通过在 ClassPath 路径下的 META-INF/services 文件夹查找文件，自动加载文件里所定义的类。这一机制为很多框架扩展提供了可能，比如在 JDBC 中就使用到了 SPI 机制。

**为什么通过 spi 机制就能打破双亲委托？**

因为在某些情况下父类加载器需要委托子类加载器去加载 class 文件。受到加载范围的限制，父类加载器无法加载到需要的文件。

以 Driver 接口为例，DriverManager 通过 Bootstrap ClassLoader 加载进来的，而 com.mysql.cj.jdbc.Driver 是通过 Application ClassLoader 加载进来的。由于双亲委派模型，父加载器是拿不到通过子加载器加载的类的。这个时候就需要启动类加载器来委托子类来加载 Driver 实现，从而破坏了双亲委派。

文件名为：**java.sql.Driver**，这是 JDK 包，Driver 接口的路径。

文件内容为：**com.mysql.cj.jdbc.Driver**，这是 mysql 继承 JDK Driver，实现接口功能的类路径。

![](https://crry-assist.oss-cn-chengdu.aliyuncs.com/Screenshot/Driver%20ClassLoader.png)

SPI 机制 Demo

```java
public interface SPIService {
    String getName();
}

public class SPIServiceImplA implements SPIService{
    @Override
    public String getName() {
        return "SPIServiceImplA";
    }
}

public class SPIServiceImplB implements SPIService{

    @Override
    public String getName() {
        return "SPIServiceImplB";
    }
}

public class TestClass {

    public static void main(String[] args) {
        ServiceLoader<SPIService> serviceLoader = ServiceLoader.load(SPIService.class);
        for (SPIService spiService : serviceLoader) {
            System.out.println(spiService.getName());
        }
    }
}
```

配置文件，文件名为接口的全路径，文件内容为实现类的全路径。

文件路径及文件名：src/main/resources/META-INF/services/com.jerry.spring.server.type.SPIService

文件内容：com.jerry.spring.server.type.SPIServiceImplA

此时输出结果为：SPIServiceImplA

#### 自定义类加载器

实现逻辑：自定义类继承 classLoader，作为自定义类加载器，重写 loadClass 方法，不让它执行双亲委派逻辑，从而打破双亲委派。

```java
import org.apache.commons.io.IOUtils;

import java.io.File;
import java.io.FileInputStream;
import java.io.IOException;

public class MyClassLoader extends ClassLoader {
    private String basePath;
    private final static String FILE_EXT = ".class";

    public void setBasePath(String basePath) {
        // 确保路径以分隔符结尾
        if (!basePath.endsWith(File.separator)) {
            this.basePath = basePath + File.separator;
        } else {
            this.basePath = basePath;
        }
    }

    private byte[] loadClassData(String name) throws IOException {
        // 将包名转换为文件路径：将点号替换为文件分隔符
        String path = name.replace('.', File.separatorChar) + FILE_EXT;
        File file = new File(basePath + path);

        if (!file.exists()) {
            System.out.println("类文件不存在: " + file.getAbsolutePath());
            return null;
        }

        FileInputStream fis = null;
        try {
            fis = new FileInputStream(file);
            return IOUtils.toByteArray(fis);
        } finally {
            IOUtils.closeQuietly(fis);
        }
    }

    @Override
    protected Class<?> findClass(String name) throws ClassNotFoundException {
        try {
            byte[] data = loadClassData(name);
            if (data == null) {
                throw new ClassNotFoundException("Class not found: " + name);
            }
            return defineClass(name, data, 0, data.length);
        } catch (IOException e) {
            throw new ClassNotFoundException("Failed to load class: " + name, e);
        }
    }

    @Override
    public Class<?> loadClass(String name) throws ClassNotFoundException {
        // 首先检查是否已经加载
        Class<?> loadedClass = findLoadedClass(name);
        if (loadedClass != null) {
            return loadedClass;
        }

        // 遵循双亲委派：优先让父加载器加载
        if (name.startsWith("java.")) {
            return super.loadClass(name);
        }

        try {
            // 尝试使用自定义逻辑加载
            return findClass(name);
        } catch (ClassNotFoundException e) {
            // 如果自定义加载失败，委托给父类加载器
            return super.loadClass(name);
        }
    }

    public static void main(String[] args) throws Exception {
        MyClassLoader classLoader = new MyClassLoader();

        // 直接使用当前类的目录作为基准路径
        String currentClassPath = MyClassLoader.class.getProtectionDomain()
                .getCodeSource().getLocation().getPath();

        // 如果是Windows，处理路径问题
        if (System.getProperty("os.name").toLowerCase().contains("win")) {
            currentClassPath = currentClassPath.substring(1); // 移除开头的/
        }

        System.out.println("当前类路径: " + currentClassPath);
        classLoader.setBasePath(currentClassPath);

        // 加载当前包下的类
        Class<?> loadedClass = classLoader.loadClass("com.jerry.spring.server.type.SPIService");
        System.out.println("加载的类: " + loadedClass.getName());
        System.out.println("类加载器: " + loadedClass.getClassLoader());
    }
}
```

输出结果如下：

```te
当前类路径: E:/Workspace/Spring-Freamwork/spring-server/target/classes/
加载的类: com.jerry.spring.server.type.SPIService
类加载器: com.jerry.spring.server.type.MyClassLoader@48140564
```


