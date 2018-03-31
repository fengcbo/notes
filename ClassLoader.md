# 类加载器
## ClassLoader

### class doc

类加载器(class loader)是负责加载类的一个对象。ClassLoader是一个抽象类。给定一个类的二进制名，类加载器试图去定位和生成组成class定义的数据。一个典型的策略是把这个名字转换成一个文件名，然后从文件系统中读取这个名字对应的class文件。

每一个Class对象都包含一个定义它的ClassLoader的引用。

数组类型的Class对象并不是用过类加载器创建的，而是java运行时作为必须的类型而自动创建的。数组类型的类加载器，作为Class.getClassLoader()返回值，同他的元素类型的类加载器相同。如果元素是原生类型，则数组类型就没有类加载器。

应用程序实现ClassLoader的子类是为了扩展JVM动态加载类的方式。

安全管理器通常使用类加载器来标识安全的域。

类加载器使用委托模式来查找类和资源文件。类加载器的每个实例都有一个相关的父加载器。当被请求查找一个类或者资源时，类加载器实例会在自己查找之前，先委托父加载器去查找类和资源。java虚拟机内置的类加载器，称作 "bootstrap class loader"，并没有父加载器，但是可以作为其他类加载器实例的父类。

支持并发加载类的类加载器被称作并发类加载器，并且必须在他们的class初始化的时候通过执行 ClassLoader.registerAsParallelCapable方法来注册他们自己。注意ClassLoader默认被注册为支持并发的。但是，如果它的子类支持并发，那么他们仍然需要去注册他们自己。
在委托模型不是严格分层的环境中，类加载器需要支持并发，否则类加载将导致死锁，因为加载器锁在类加载的过程中被持有(查看loadClass方法)。

通常，java虚拟机是以平台依赖的方式从本地文件系统中加载类。比如，在Unix系统上，虚拟机从环境变量 CLASSPATH 定义的文件夹中加载类。

但是，有些类并不是来源于类；它们可能来自其他源，比如网络，或者它们可以被应用创建。defineClass 方法将一个字节数组转换成一个Class实例。这个新定义的Class的实例可以通过 Class.newInstance 创建。

被类加载器初始化的对象的方法和构造方法可能引用其他类。为了确定涉及到的类，java虚拟机将会执行最初创建这个类的类加载器的loadClass方法。

比如，应用程序可以创建一个网络类加载器来从服务器下载class文件。示例代码如下：

```
ClassLoader loader = new NetworkClassLoader(host, port);
Object main = loader.loadClass("Main", true).newInstance();
```

网络类加载器子类必须定义findClass和loadClassData方法来从网络加载class。一旦它下载了组成class的二进制，它就需要使用defineClass 方法来创建一个class实例。一个简单的实现如下：

```
class NetworkClassLoader extends ClassLoader {
    String host;
    int port;

    public Class findClass(String name) {
        byte[] b = loadClassData(name);
        return defineClass(name, b, 0, b.length);
    }

    private byte[] loadClassData(String name) {
        // load the class data from the connection
        . . .
    }
}
```

### 二进制名字

任何作为String参数传给类加载器方法的类名必须是一个java语言规范定义的二进制名。

有效类名的实例包括：
```
   "java.lang.String"
   "javax.swing.JSpinner$DefaultEditor"
   "java.security.KeyStore$Builder$FileBuilder$1"
   "java.net.URLClassLoader$3$1"
```

### 自定义类加载器

自定义自己的类加载器需要三个步骤：

1. 定义一个类继承java.lang.ClassLoader
2. 通过二进制名字加载二进制文件
3. 重写findClass方法，通过二进制初始化Class对象

下面是示例

```
public class LoaderTest {

    public static void main(String[] args) throws ClassNotFoundException {
        String[] strs = new String[2];
        System.out.println(strs.getClass().getClassLoader());

        System.out.println("===============================");

        InterfaceLoader[] loaders = new InterfaceLoader[2];
        System.out.println(loaders.getClass().getClassLoader());

        System.out.println("===============================");

        CustomClassLoader customClassLoader = new CustomClassLoader(CustomClassLoader.class.getName());
        Class<?> clazz = customClassLoader.loadClass("com.shiluns.arithmetic.Arithmetic");

        System.out.println(clazz);
    }
}

// 1. 继承ClassLoader
class CustomClassLoader extends ClassLoader {

    private String classLoaderName;

    private final String fileExtension = ".class";

    public CustomClassLoader(String classLoaderName){
        super();
        this.classLoaderName = classLoaderName;
    }

    public  CustomClassLoader(ClassLoader parent){
        super(parent);
    }

    // 3. 通过二进制初始化Class对象
    @Override
    protected Class<?> findClass(String name) throws ClassNotFoundException {
        byte[] bytes = loadClassData(name);
        return defineClass(name, bytes, 0, bytes.length);
    }

    // 2. 通过二进制名字加载字节码
    private byte[] loadClassData(String name) {
        // load the class data from the connection
        byte[] data = null;
        String className = name.replace('.', '/') + fileExtension;
        FileInputStream fis = null;
        ByteArrayOutputStream bas = null;
        try {
            fis = new FileInputStream(new File(className));
            bas = new ByteArrayOutputStream();
            int ch = 0;
            while ((ch = fis.read()) != -1){
                bas.write(ch);
            }
            data = bas.toByteArray();
        } catch (Exception e) {
            e.printStackTrace();
        } finally {
            try {
                fis.close();
            } catch (IOException e) {
                e.printStackTrace();
            }
        }
        return data;
    }
}
```
