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
        // 将系统类加载器当做本类加载器的父加载器
        super();
        this.classLoaderName = classLoaderName;
    }

    public  CustomClassLoader(String classLoaderName,ClassLoader parent){
        // 将 parent 加载器当做当前类加载器的父加载器
        super(parent);
        this.classLoaderName = classLoaderName;
    }

    // 3. 通过二进制初始化Class对象
    @Override
    protected Class<?> findClass(String name) throws ClassNotFoundException {
        System.out.println("find class invoked");
        System.out.println("class loader:" + classLoaderName);
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

### method introduction

#### protected Class<?> findClass(String name) throws ClassNotFoundException

通过指定的二进制名字查找class。这个方法需要遵行委托模式的类加载器实现重写以加载class，并且由 loadClass 让父加载器检查需要的class之后 调用。默认的实现抛出ClassNotFoundException异常。

我们自定义类加载器时需要重写这个方法，但这个方法并不会被我们手动调用，而是通过 loadClass方法来调用，而程序员只需要调用loadClass来获取Class对象即可。

####  protected final Class<?> defineClass(String name, byte[] b, int off, int len) throws ClassFormatError

将一个字节数组转化成Class对象。在这个Class使用之前，这个方法必须被解析。

这个方法为新定义的Class分配了一个默认的 ProtectionDomain。调用 Policy.getPolicy().getPermissions(new CodeSource(null, null)) 时，ProtectionDomain 被有效授予所返回的相同权限集。默认域在第一次调用 defineClass 时创建，并在后续调用时被重用。

要将特定的 ProtectionDomain 分配给类，需要使用 defineClass 方法，该方法将 ProtectionDomain 用作其参数之一。

参数
    name 所需的类的二进制名字，或者 如果不知道名字可以是null
    b 组成类数据的字节数组。在off 和  off+len-1 之间的字节需要是如JVM规范中定义的合法的class文件格式。
    off 类数据的开始偏移量
    len 类数据的长度

返回值
    通过制定的类数据创建的Class对象

异常
    ClassFormatError 这个是个受检异常，应该说是错误。如果数据不是一个合法的class
    IndexOutOfBoundsException 如果off或者len是负数，或者off+len大于b.length
    SecurityException 如果试图将此类添加到包含由不同证书集签名的类（而不是此类，此类未签名）的包中，或者 name 以 " java." 开头。说明java不予许包名以java开头。

#### public Class<?> loadClass(String name) throws ClassNotFoundException

通过给定二进制名字加载类。 这个方法和loadClass(String, boolean)方法以相同的方式查找类。JVM执行这个方法来解析类的引用。执行这个方法与执行loadClass(String, false)效果相同

#### protected Class<?> loadClass(String name, boolean resolve) throws ClassNotFoundException

通过给定二进制名字加载类。这个方法的默认实现通过如下顺序查找类：

1. 执行findLoadedClass(String)检查这个类是否已经被加载
2. 执行父加载器的loadClass 方法。如果父类加载器为null，则使用内置的类加载器
3. 执行 findClass(String) 方法查找类

如果通过以上步骤找到了类，并且resolve为true，这个方法将会在结果Class对象上执行resolveClass(Class)方法。鼓励ClassLoader的子类重写 findClass(String)方法，而不知本方法。

除非被重写，在类加载的过程中，这个方法将同步getClassLoadingLock方法的返回结果。

参数
    name 类的二进制名字
    resolve 如果为true则解析类

返回值
    Class对象

异常
    ClassNotFoundException 如果这个类不能被找到


### 双亲委托思考

执行LoaderTest的main方法，看执行结果发现CustomClassLoader的findClass方法并没有执行，这是什么原因呢？
我们修改main方法，打印classLoader

```
System.out.println(clazz.getClassLoader());
```

输出结果是：

```
sun.misc.Launcher$AppClassLoader@18b4aac2
```

所以com.shiluns.arithmetic.Arithmetic并不是我们自定的CustomClassLoader加载的，而是系统类加载器加载的。在loadClass的javadoc中描述了类加载的顺序

1. 执行findLoadedClass(String)检查这个类是否已经被加载
2. 执行父加载器的loadClass 方法。如果父类加载器为null，则使用内置的类加载器
3. 执行 findClass(String) 方法查找类

类加载器加载一个类时会先执行父类的loadClass，而我们定义的CustomClassLoader的父加载器是sun.misc.Launcher$AppClassLoader也就是系统类加载，而我们要加载的类位于工程目录下，所以系统类加载可以加载，结果也就显而易见，系统类加载器是Arithmetic 的***定义类加载器***，而CustomClassLoader是Arithmetic的***初始类加载器***。

```
定义类加载器：若有一个类加载器可以成功加载目标类，那么这个类加载器成为这个目标类的定义类加载器
初始类加载器：所有能成功返回目标类Class对象应用的类加载器(包括定义类加载器)都被成为初始类加载器。
```

如果我们删除classpath下的Arithmetic.class，findClass则会执行，因为CustomClassLoader的父类加载器(系统类加载器)找不到Arithmetic.class，则CustomClassLoader 会去加载这个class。


下面我们修改main方法和CustomClassLoader，如下：

main方法：
```
CustomClassLoader customClassLoader = new CustomClassLoader(CustomClassLoader.class.getName());
customClassLoader.setPath("/Users/duodiankeji-2018/Desktop/");
Class<?> clazz = customClassLoader.loadClass("com.shiluns.arithmetic.Arithmetic");
System.out.println(clazz.hashCode());

System.out.println("================================");

CustomClassLoader customClassLoader2 = new CustomClassLoader(CustomClassLoader.class.getName());
customClassLoader2.setPath("/Users/duodiankeji-2018/Desktop/");
Class<?> clazz2 = customClassLoader2.loadClass("com.shiluns.arithmetic.Arithmetic");
System.out.println(clazz2.hashCode());
```

CustomClassLoader 如下：

```
class CustomClassLoader extends ClassLoader {

    private String classLoaderName;

    private String path;

    private final String fileExtension = ".class";

    public CustomClassLoader(String classLoaderName){
        super();
        this.classLoaderName = classLoaderName;
    }

    public  CustomClassLoader(String classLoaderName,ClassLoader parent){
        super(parent);
        this.classLoaderName = classLoaderName;
    }

    @Override
    protected Class<?> findClass(String name) throws ClassNotFoundException {
        System.out.println("find class invoked");
        System.out.println("class loader:" + classLoaderName);
        byte[] bytes = loadClassData(name);
        return defineClass(name, bytes, 0, bytes.length);
    }

    private byte[] loadClassData(String name) {
        // load the class data from the connection
        byte[] data = null;
        String className = name.replace('.', '/') + fileExtension;

        FileInputStream fis = null;
        ByteArrayOutputStream bas = null;
        try {
            fis = new FileInputStream(new File(this.path + className));
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

    public void setPath(String path) {
        this.path = path.endsWith("/") ? path : path + "/";
    }
}
```

将Arithmetic.class及其包移到桌面目录下，此时classpath下没有Arithmetic.clas文件，说明系统类加载器无法加载。我们运行main方法，发现clazz和clazz2是两个不同的对象(他们的hashCode不一样)。这就导致了同一个类在我们的运行时会有两个对应的Class对象，为什么会这样呢？

这是由于每个类加载都有自己的命名空间，命名空间有该加载器和他所有的父加载器所加载的类组成。

在同一个命名空间下，不回出现类的完整名字想用的类。

在不同的命名空间下，有可能出现类的完整名字相同的两个类。

