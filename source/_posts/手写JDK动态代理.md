---
title: 手写JDK动态代理
date: 2018-12-31 18:29:00
description: 代理模式是很常用的设计模式，各种框架中也随处可见，JDK自身实现了动态代理，首先了解它是如何使用的，原理是什么，然后手写代码完成JDK动态代理的功能。
categories: 
- 设计模式
tags: #文章标签
- 动态代理
- 设计模式
toc: true
comments: true
original:
---
# JDK动态代理

放寒假了，学生小明要回家，但是买不到火车票，只好找黄牛购票，用JDK的动态代理实现。

首先定义一个Student接口

```java
public interface Student {
    void buy();
}
```

定义小明，实现Student接口

```java
public class XiaoMing implements Student {

    @Override
    public void buy() {
        System.out.println("我是小明，我要买票去上海的硬座");
    }
}
```

定义黄牛，实现InvocationHandler接口（JDK的动态代理一定要实现这个接口）

```java
public class HuangNiu implements InvocationHandler {

    private Student target;

     /**
     * 生成代理对象
     * @param target
     * @return
     */
    public Object getInstance(Student target) {
        this.target = target;
        Class clazz = target.getClass();
        return Proxy.newProxyInstance(clazz.getClassLoader(), clazz.getInterfaces(), 				this);
    }

    @Override
    public Object invoke(Object proxy, Method method, Object[] args) throws Throwable {
        System.out.println("我是黄牛，你要什么票?");
        System.out.println("--------------");
        method.invoke(target, args);
        System.out.println("--------------");
        return null;
    }

}
```

测试类：

```java
public class ProxyTest {

    public static void main(String[] args) {
        Student huangniu = (Student) new HuangNiu().getInstance(new XiaoMing());
        huangniu.buy();
    }
}
```

输出结果：

```shell
 我是黄牛，你要什么票?
 --------------------
 我是小明，我要买票去上海的硬座
 --------------------
```

小明要购票，小明不需要自己去售票处，而是找了黄牛，让黄牛代劳，真正购票的人是黄牛

小明 ==>被代理对象

黄牛 ==>代理对象

这时候，运行的是黄牛这个代理类，试着将Proxy.newProxyInstance方法生成的代理类输出

```java
	/**
     * 输出代理对象class
     *
     * @param proxy
     */
    private void output(Object proxy) {
        String className = proxy.getClass().getSimpleName();
        String baseDir = getClass().getResource("").getPath();
        byte[] data = ProxyGenerator.generateProxyClass(className, new Class[]{Student.class});
        FileOutputStream os = null;
        try {
            os = new FileOutputStream(baseDir + "/" + className + ".class");
            os.write(data);
        } catch (FileNotFoundException e) {
            e.printStackTrace();
        } catch (IOException e) {
            e.printStackTrace();
        } finally {
            try {
                os.close();
            } catch (IOException e) {
                e.printStackTrace();
            }
        }

    }
```

得到$Proxy0.class文件（生成的代理类名都是以“$”为前缀，数字为后缀），反编译后的代码：

```java
public final class $Proxy0 extends Proxy implements Student {
    private static Method m1;
    private static Method m2;
    private static Method m3;
    private static Method m0;

    public $Proxy0(InvocationHandler var1) throws  {
        super(var1);
    }

    public final boolean equals(Object var1) throws  {
        try {
            return ((Boolean)super.h.invoke(this, m1, new Object[]{var1})).booleanValue();
        } catch (RuntimeException | Error var3) {
            throw var3;
        } catch (Throwable var4) {
            throw new UndeclaredThrowableException(var4);
        }
    }

    public final String toString() throws  {
        try {
            return (String)super.h.invoke(this, m2, (Object[])null);
        } catch (RuntimeException | Error var2) {
            throw var2;
        } catch (Throwable var3) {
            throw new UndeclaredThrowableException(var3);
        }
    }

    public final void buy() throws  {
        try {
            super.h.invoke(this, m3, (Object[])null);
        } catch (RuntimeException | Error var2) {
            throw var2;
        } catch (Throwable var3) {
            throw new UndeclaredThrowableException(var3);
        }
    }

    public final int hashCode() throws  {
        try {
            return ((Integer)super.h.invoke(this, m0, (Object[])null)).intValue();
        } catch (RuntimeException | Error var2) {
            throw var2;
        } catch (Throwable var3) {
            throw new UndeclaredThrowableException(var3);
        }
    }

    static {
        try {
            m1 = Class.forName("java.lang.Object").getMethod("equals", Class.forName("java.lang.Object"));
            m2 = Class.forName("java.lang.Object").getMethod("toString");
            m3 = Class.forName("com.cjluo.chapter1.proxy.jdk.Student").getMethod("buy");
            m0 = Class.forName("java.lang.Object").getMethod("hashCode");
        } catch (NoSuchMethodException var2) {
            throw new NoSuchMethodError(var2.getMessage());
        } catch (ClassNotFoundException var3) {
            throw new NoClassDefFoundError(var3.getMessage());
        }
    }
}
```

代理类对被代理对象的所有方法都做了代理，通过代理类的InvocationHandler.invoke调用，因此，自己实现动态代理的关键步骤就是：

1. 生成动态代理类；
2. 编译并代理类并加载代理类class文件；
3. 返回代理类给调用者；

以上涉及的JDK动态代理相关类有

```java
InvocationHandler
Proxy
ClassLoader
```

# 自己实现动态代理

首先定义CustomerInvocationHandler接口

```java
public interface CustomerInvocationHandler {

    Object invoke(Object proxy, Method method, Object[] args) throws Throwable;
}
```

理一下JDK动态代理的逻辑，方法的调用者调用的是小明的代理对象 ==>黄牛，因此要定义CustomerProxy并在该类中生成黄牛这个代理类：

```java
public class CustomerProxy {
    //硬编码代理对象名称
    private static final String proxyClassName = "$Proxy0";
    
    public static Object newProxyInstance(CustomerClassLoader loader,
                                          Class<?>[] interfaces,
                                          CustomerInvocationHandler h) {

        try {
            //1、生成代理类java文件
            File f = generatorProxy(interfaces);
            //2、编译代理类java为class
            compilerJava(f);
            //3、自定义加载器加载class
            Class clazz = loader.findClass(proxyClassName);
            //4、生成代理对象并返回
            Constructor c = clazz.getConstructor(CustomerInvocationHandler.class);
            return c.newInstance(h);
        } catch (ClassNotFoundException e) {
            e.printStackTrace();
        } catch (NoSuchMethodException e) {
            e.printStackTrace();
        } catch (InstantiationException e) {
            e.printStackTrace();
        } catch (IllegalAccessException e) {
            e.printStackTrace();
        } catch (InvocationTargetException e) {
            e.printStackTrace();
        }
        return null;
    }

}
```

生成代理类java文件代码：

```java
/**
 * 生成代理类java文件
 *
 * @param interfaces
 * @return
 */
private static File generatorProxy(Class<?>[] interfaces) {
    //接口名
    Class clazz = interfaces[0];
    String interfaceName = clazz.getName();

    //换行
    String newLine = "\r\n";
    StringBuffer sb = new StringBuffer();
    sb.append("package " + clazz.getPackage().getName() + ";").append(newLine);
    sb.append("import java.lang.reflect.Method;").append(newLine);
    sb.append("public class " + proxyClassName + " implements " + interfaceName + "{").append(newLine);
    sb.append("protected CustomerInvocationHandler h;").append(newLine);
    sb.append("public " + proxyClassName + "(CustomerInvocationHandler h){").append(newLine);
    sb.append(" this.h = h; ").append(newLine);
    sb.append("}").append(newLine);
    for (Method m : clazz.getMethods()) {
        sb.append("public " + m.getReturnType().getName() + " " + m.getName() + "(){").append(newLine);
        sb.append("try{").append(newLine);
        sb.append("Method m = " + interfaceName + ".class.getMethod(\"" + m.getName() + "\",new Class[]{});").append(newLine);
        sb.append("this.h.invoke(this,m,null);").append(newLine);
        sb.append("}catch(Throwable e){").append(newLine);
        sb.append("}").append(newLine);
        sb.append("}").append(newLine);

    }
    sb.append("}").append(newLine);
    String src = sb.toString();
    File file = new File(CustomerProxy.class.getResource("").getPath() + "/" + proxyClassName + ".java");
    try {
        FileWriter fw = new FileWriter(file);
        fw.write(src);
        fw.flush();
        fw.close();
    } catch (IOException e) {
        e.printStackTrace();
    }
    return file;
}
```

编译代理类

```java
/**
 * 编译代理类
 *
 * @param f
 */
private static void compilerJava(File f) {
    //java编译器
    JavaCompiler compiler = ToolProvider.getSystemJavaCompiler();
    StandardJavaFileManager manager = compiler.getStandardFileManager(null, null, null);
    Iterable iterable = manager.getJavaFileObjects(f);
    JavaCompiler.CompilationTask task = compiler.getTask(null, manager, null, null, null, iterable);
    task.call();
    try {
        manager.close();
    } catch (IOException e) {
        e.printStackTrace();
    }finally {
        //删除Java文件
        f.delete();
    }
}
```

自定义加载器加载class

```java
public class CustomerClassLoader extends ClassLoader {
    private File baseDir;

    public CustomerClassLoader() {
        String basePath = CustomerClassLoader.class.getResource("").getPath();
        this.baseDir = new File(basePath);
    }

    @Override
    protected Class<?> findClass(String name) throws ClassNotFoundException {
        String className = getClass().getPackage().getName() + "." + name;
        if (baseDir != null) {
            File classFile = new File(baseDir, name.replaceAll("\\.", "/") + ".class");
            if (classFile.exists()) {
                FileInputStream in = null;
                ByteArrayOutputStream out = null;
                try {
                    in = new FileInputStream(classFile);
                    out = new ByteArrayOutputStream();
                    byte[] buff = new byte[1024];
                    int len;
                    while ((len = in.read(buff)) != -1) {
                        out.write(buff, 0, len);
                    }
                    return defineClass(className, out.toByteArray(), 0, out.size());
                } catch (FileNotFoundException e) {
                    e.printStackTrace();
                } catch (IOException e) {
                    e.printStackTrace();
                } finally {
                    //删除class文件
                    classFile.delete();
                }

            }
        }
        return null;
    }
}
```

测试类：

```java
public class CustomerProxyTest {

    public static void main(String[] args) {
        Student huangniu = (Student) new HuangNiu().getInstance(new XiaoMing());
        huangniu.buy();
    }
}
```

运行结果

```shell
 我是黄牛，你要什么票?
 --------------------
 我是小明，我要买票去上海的硬座
 --------------------
```

