title: 类加载原理与getResourceAsStream
date: 2019-1-05
tags: [java]
categories: Java
toc: true
---

&emsp;&emsp;在项目启动时加载数据库脚本初始化数据库，采用getResourceAsStream去加载脚本，结果报错找不到对应文件。
{% codeblock lang:java %}
    ...
    InputStream sqlFileIn = null;

    sqlFileIn = SqliteUtil.class.getResourceAsStream( "/init.sql" );
    StringBuffer sqlSb = new StringBuffer();
    byte[] buff = new byte[ 1024 ];
    int byteRead = 0;

   while ( ( byteRead = sqlFileIn.read( buff ) ) != -1 ) {
        sqlSb.append( new String( buff, 0, byteRead, "utf-8" ) );
   }
    sqlFileIn.close();
    ...
    {% endcodeblock %}
&emsp;&emsp;没有办法，只能从getResourceAsStream的源码看起，一点一点找出结果。
    
### getResourceAsStream
{% codeblock lang:java %}
public InputStream getResourceAsStream(String name) {
     //先对资源路径解析
    name = resolveName(name);
    ClassLoader cl = getClassLoader0();
    if (cl==null) {
        // A system class.
        return ClassLoader.getSystemResourceAsStream(name);
    }
    //调用的getResource方法
    return cl.getResourceAsStream(name);
}
{% endcodeblock %}
&emsp;&emsp;Class类的getResourceAsStream方法优先委托当前Class的classLoader去加载资源文件，如果当前类的classLoader为null，则调用ClassLoader的getSystemResourceAsStream加载

{% codeblock lang:java %}
/**
 * Add a package name prefix if the name is not absolute,Remove leading "/" if name is absolute
 */
private String resolveName(String name) {
    if (name == null) {
        return name;
    }
    if (!name.startsWith("/")) {
        Class<?> c = this;
        while (c.isArray()) {
            c = c.getComponentType();
        }
        String baseName = c.getName();
        int index = baseName.lastIndexOf('.');
        if (index != -1) {
            name = baseName.substring(0, index).replace('.', '/')
                +"/"+name;
        }
    } else {
        name = name.substring(1);
    }
    return name;
}
{% endcodeblock %}
&emsp;&emsp;这里getResourceAsStream需要调用resolveName()解析资源路径：如果文件名以/开头，则移除前面的/，如果不以/开头，说明是相对路径，则在前面补充上其所在包名。这里是关键，很明显我的路径"/init.sql"是有问题的，系统会在当前类同级目录下寻找init.sql文件，但我的文件没放在这，所以报错。

&emsp;&emsp;再看看ClassLoader的getSystemResourceAsStream，这个方法优先通过系统指定类加载器加载资源，又引出了getSystemClassLoader方法
{% codeblock lang:java %}
public static InputStream getSystemResourceAsStream(String name) {
    URL url = getSystemResource(name);
    try {
        return url != null ? url.openStream() : null;
    } catch (IOException e) {
        return null;
    }
}

public static URL getSystemResource(String name) {
    //获取system classLoader ，也即 AppClassloader
    ClassLoader system = getSystemClassLoader();
    if (system == null) {
        //获取不到则通过BootStrapClassLoader加载资源
        return getBootstrapResource(name);
    }
    return system.getResource(name);
}

public URL getResource(String name) {
    URL url;
    if (parent != null) {
        url = parent.getResource(name);
    } else {
        url = getBootstrapResource(name);
    }
    if (url == null) {
        url = findResource(name);
    }
    return url;
}
{% endcodeblock %}

### getSystemClassLoader
在ClassLoader类中getSystemClassLoader方法调用私有的initSystemClassLoader方法获得AppClassloader实例：

{% codeblock lang:java %}
public static ClassLoader getSystemClassLoader() {
    initSystemClassLoader();
    if (scl == null) {
        return null;
    }
    SecurityManager sm = System.getSecurityManager();
    if (sm != null) {
        checkClassLoaderPermission(scl, Reflection.getCallerClass());
    }
    return scl;
}

private static synchronized void initSystemClassLoader() {
    if (!sclSet) {
        if (scl != null)
            throw new IllegalStateException("recursive invocation");
        sun.misc.Launcher l = sun.misc.Launcher.getLauncher();
        if (l != null) {
            Throwable oops = null;
           //
            scl = l.getClassLoader();
            try {
                scl = AccessController.doPrivileged(
                    new SystemClassLoaderAction(scl));
            } catch (PrivilegedActionException pae) {
                oops = pae.getCause();
                if (oops instanceof InvocationTargetException) {
                    oops = oops.getCause();
                }
            }
            if (oops != null) {
                if (oops instanceof Error) {
                    throw (Error) oops;
                } else {
                    // wrap the exception
                    throw new Error(oops);
                }
            }
        }
        sclSet = true;
    }
}
{% endcodeblock %}
&emsp;&emsp;sun.misc.Launcher是jre中用于启动程序入口main()的类。Launcher类在new自己的时候生成AppClassloader实例并且放在自己的私有变量loader里。

### Launcher
{% codeblock lang:java %}
public class Launcher {
    ...
    //私有classLoader
    private ClassLoader loader;
    
    public static Launcher getLauncher() {
        return launcher;
    }
    //公有构造
    public Launcher() {
        //创建扩展类加载器：ExtClassLoader是Launcher的内部类，继承自URLClassLoader
        Launcher.ExtClassLoader var1;
        try {
            var1 = Launcher.ExtClassLoader.getExtClassLoader();
        } catch (IOException var10) {
            throw new InternalError("Could not create extension class loader", var10);
        }
    
        try {
            //创建用于启动应用程序的类加载器
            this.loader = Launcher.AppClassLoader.getAppClassLoader(var1);
        } catch (IOException var9) {
            throw new InternalError("Could not create application class loader", var9);
        }
        //每个线程持有一个ContextClassLoader，可以用get,set方法获取或定义。如果不加指定，就是启动线程那个类自己的类加载器。如果不是main线程，new出来的线程的话，就是父线程的类加载器
        Thread.currentThread().setContextClassLoader(this.loader);
        String var2 = System.getProperty("java.security.manager");
        if (var2 != null) {
            //根据需求创建安全管理器：SecurityManager实例
            SecurityManager var3 = null;
            if (!"".equals(var2) && !"default".equals(var2)) {
                try {
                    var3 = (SecurityManager)this.loader.loadClass(var2).newInstance();
                } catch (IllegalAccessException var5) {
                } catch (InstantiationException var6) {
                } catch (ClassNotFoundException var7) {
                } catch (ClassCastException var8) {
                }
            } else {
                var3 = new SecurityManager();
            }
    
            if (var3 == null) {
                throw new InternalError("Could not create SecurityManager: " + var2);
            }
    
            System.setSecurityManager(var3);
        }
    }
    
    ...
}
{% endcodeblock %}
也就是说launcher初始化时做了以下工作：
{% blockquote %}
1.创建扩展类加载器ExtClassLoader
2.创建用于启动应用程序的类加载器AppClassLoader
3.设置当前线程的上下文类加载器为前一步创建的AppClassLoader实例
4.创建安全管理器SecurityManager
{% endblockquote %}
&emsp;&emsp;Launcher类使用了一种类似单例模式的方法，既提供了单例模式的接口getLauncher()又把构造函数设成了public的。但是在ClassLoader中是通过单例模式取得的Launcher 实例的，所以我们写的每个类加载器得到的AppClassloader都是同一个AppClassloader类实例，也就是说所有通过正常双亲委派模式的类加载器加载的classpath下的和ext下的所有类在方法区都是同一个类，堆中的Class实例也是同一个。


#### ExtClassLoader
{% codeblock lang:java %}
static class ExtClassLoader extends URLClassLoader {
    public static Launcher.ExtClassLoader getExtClassLoader() throws IOException {
        final File[] var0 = getExtDirs();
        try {
            return (Launcher.ExtClassLoader)AccessController.doPrivileged(new PrivilegedExceptionAction<Launcher.ExtClassLoader>() {
                public Launcher.ExtClassLoader run() throws IOException {
                    int var1 = var0.length;

                    for(int var2 = 0; var2 < var1; ++var2) {
                        MetaIndex.registerDirectory(var0[var2]);
                    }
                    return new Launcher.ExtClassLoader(var0);
                }
            });
        } catch (PrivilegedActionException var2) {
            throw (IOException)var2.getException();
        }
    }

    public ExtClassLoader(File[] var1) throws IOException {
        //显示指定java.ext.dirs目录，可以通过参数Djava.ext.dirs=… 修改
        super(getExtURLs(var1), (ClassLoader)null, Launcher.factory);
        SharedSecrets.getJavaNetAccess().getURLClassPath(this).initLookupCache(this);
    }

    private static File[] getExtDirs() {
        //ExtClassLoader加载目录
        String var0 = System.getProperty("java.ext.dirs");
        ...
    }
}
{% endcodeblock %}
&emsp;&emsp;ExtClassLoader加载java.ext.dirs目录下的文件，默认是jre安装目录/lib/ext，是一些JDK或JRE的可选择功能扩展包。

#### AppClassLoader
 {% codeblock lang:java %}
 static class AppClassLoader extends URLClassLoader {
     ...
     public static ClassLoader getAppClassLoader(final ClassLoader var0) throws IOException {
         final String var1 = System.getProperty("java.class.path");
         final File[] var2 = var1 == null ? new File[0] : Launcher.getClassPath(var1);
         return (ClassLoader)AccessController.doPrivileged(new PrivilegedAction<Launcher.AppClassLoader>() {
             public Launcher.AppClassLoader run() {
                 URL[] var1x = var1 == null ? new URL[0] : Launcher.pathToURLs(var2);
                 return new Launcher.AppClassLoader(var1x, var0);
             }
         });
     }
     ...
 }
 {% endcodeblock %}
&emsp;&emsp;可以看到这里取的是环境变量java.class.path中设定的路径作为类加载的搜索路径。可以通过对该变量的设定来修改默认配置。 
{% blockquote %}
classpath是指 WEB-INF文件夹下的classes目录 ，所有src目录下面的java、xml、properties等文件编译后都会在此，所以在开发时常将相应的xml配置文件放于src或其子目录下。 
{% endblockquote %} 

### 类加载器总结
&emsp;&emsp;Java 中的类加载器大致可以分成两类，一类是系统提供的，另外一类则是由 Java 应用开发人员编写的。系统提供的类加载器主要有下面三个：
{% blockquote %}
 　　引导类加载器（bootstrap class loader）：它用来加载 Java 的核心库，是用原生代码来实现的，并不继承自java.lang.ClassLoader。
 　　扩展类加载器（extensions class loader）：它用来加载 Java 的扩展库。Java 虚拟机的实现会提供一个扩展库目录。该类加载器在此目录里面查找并加载 Java 类。
 　　系统类加载器（system class loader）：它根据 Java 应用的类路径（CLASSPATH）来加载 Java 类。一般来说，Java 应用的类都是由它来完成加载的。可以通过 ClassLoader.getSystemClassLoader() 来获取它。
 {% endblockquote %}
 　　&emsp;&emsp;除了系统提供的类加载器以外，开发人员可以通过继承 java.lang.ClassLoader 类的方式实现自己的类加载器，以满足一些特殊的需求。
 　　&emsp;&emsp;除了引导类加载器之外，所有的类加载器都有一个父类加载器。类加载采用委托模式，先一层一层交给父类加载，父加载不成功再一层一层转给子加载。
 {% blockquote %}
 　　&emsp;&emsp;要点1：为什么采用这种委托方式，是为了安全，比如用户自定义了个java.lang.String,那么如果不交给引导类加载器去加载的话，内存中就会有不止一个String的类实例。而且一个限定包内访问权限的内容，黑客也可以用这种方式获取（要点2再继续说明）。采用了这种方式的话，引导类加载器只会加载一次类，看见用户自定义的String来了，就去看自己有没有加载，结果是系统一启动就加载了java.lang.String类，就不会再去加载了。
 　　&emsp;&emsp;要点2：判断一个类是否相等不仅要看类是否名字一样，而且要看是否有同一个类初始化加载器。所以如果黑客要自己搞一个java.lang.Hack类来加载，由委托模式开始，引导类加载器加载这个类失败，那就只能交给用户自定义的类加载起来加载。所以这个类和系统的那个lang包里的类不在一个初始化加载器里，就算包名都一样，还是不能访问那些包内可见的内容的。
 {% endblockquote %}
 
&emsp;&emsp;进一步说明：有两个术语，一个叫“定义类加载器”，一个叫“初始类加载器”。比如有如下的类加载器结构：

     bootstrap
       ExtClassloader
         AppClassloader
         -自定义clsloadr1
         -自定义clsloadr2 
&emsp;&emsp;如果用“自定义clsloadr1”加载java.lang.String类，那么根据双亲委派最终bootstrap会加载此类，那么bootstrap类就叫做该类的“定义类加载器”，而包括bootstrap的所有得到该类class实例的类加载器都叫做“初始类加载器”。
 
&emsp;&emsp;所说的“命名空间”，是指jvm为每个类加载器维护的一个“表”,这个表记录了所有以此类加载器为“初始类加载器”（而不是定义类加载器，所以一个类可以存在于很多的命名空间中）加载的类的列表。所以，对于String类来说，bootstrap是“定义类加载器”，AppClassloader是“初始类加载器”。根据刚才所说，String类在AppClassloader的命名空间中（同时也在bootstrap，ExtClassloader的命名空间中，因为bootstrap，ExtClassloader也是String的初始类加载器），所以在AppClassloader命名空间中的类可以随便访问String类。这样就可以解释“处在不同命名空间的类，不能直接互相访问”这句话了。
 
&emsp;&emsp;一个类，由不同的类加载器实例加载的话，会在方法区产生两个不同的类，彼此不可见，并且在堆中生成不同Class实例。
 
&emsp;&emsp;由不同类加载器实例（比如-自定义clsloadr1，-自定义clsloadr2）所加载的classpath下和ext下的类，也就是由我们自定义的类加载器委派给AppClassloader和ExtClassloader加载的类，在内存中是同一个类吗？
&emsp;&emsp;所有继承ClassLoader并且没有重写getSystemClassLoader方法的类加载器，通过getSystemClassLoader方法得到的AppClassloader都是同一个AppClassloader实例，类似单例模式。也就是所有通过正常双亲委派模式的类加载器加载的classpath下的和ext下的所有类在方法区都是同一个类，堆中的Class实例也是同一个。
 
     ContextClassLoader
     每个线程持有一个ContextClassLoader，可以用get,set方法获取或定义。如果不加指定，就是启动线程那么类自己的类加载器。如果不是main线程，new出来的线程的话，就是父线程的类加载器。
     为什么要有这么一个东西呢，因为为了安全ClassLoader的委托机制不能满足一些特定需要,这个时候就要用这种方式走后门。比如jdbc,jndi,tomcat等:
     Java 提供了很多服务提供者接口（Service Provider Interface，SPI），允许第三方为这些接口提供实现。常见的 SPI 有 JDBC、JCE、JNDI、JAXP 和 JBI 等。这些 SPI 的接口由 Java 核心库来提供，如 JAXP 的 SPI 接口定义包含在 javax.xml.parsers 包中。这些 SPI 的实现代码很可能是作为 Java 应用所依赖的 jar 包被包含进来，可以通过类路径（CLASSPATH）来找到。而问题在于，SPI 的接口是 Java 核心库的一部分，是由引导类加载器来加载的；SPI 实现的 Java 类一般是由系统类加载器来加载的。引导类加载器是无法找到 SPI 的实现类的，因为它只加载 Java 的核心库。它也不能代理给系统类加载器，因为它是系统类加载器的祖先类加载器。也就是说，类加载器的代理模式无法解决这个问题。
     线程上下文类加载器正好解决了这个问题。在 SPI 接口的代码中使用线程上下文类加载器，就可以成功的加载到 SPI 实现的类。线程上下文类加载器在很多 SPI 的实现中都会用到。
 
### 总结
 了解了类加载机制后，那么前面getResourceAsStream的问题就好解决了，总结一下getResourceAsStream的用法：
 {% blockquote %}
 1. Class.getResourceAsStream(String path) ：  path 不以’/'开头时默认是从此类所在的包下取资源，以’/'开头则是从 ClassPath根下获取。其只是通过path构造一个绝对路径，最终还是由ClassLoader获取资源。      
 2. Class.getClassLoader.getResourceAsStream(String path) ： 默认则是从ClassPath根下获取，path不能以’/'开头，最终是由   ClassLoader获取资源。
 3. ServletContext. getResourceAsStream(String path)： 默认从WebAPP根目录下取资源，Tomcat下path是否以’/'开头无所谓，当然这和具体的容器实现有关。    
 4. Jsp下的application内置对象就是上面的ServletContext的一种实现
  {% endblockquote %}