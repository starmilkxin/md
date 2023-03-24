![文件](https://xin-xinblog.oss-cn-shanghai.aliyuncs.com/img/20220428091339.png)
在XINApplicationContext中使用Class.getResource和ClassLoader.getResource分别去获取abc文件

首先分别来看看二者的源码

```Java
//Class.getResource
public java.net.URL getResource(String name) {
    name = resolveName(name);
    ClassLoader cl = getClassLoader0();
    if (cl==null) {
        // A system class.
        return ClassLoader.getSystemResource(name);
    }
    return cl.getResource(name);
}

//ClassLoader.getResource
public URL getResource(String name) {
    URL url;
    if (parent != null) {
        url = parent.getResource(name);//交给父加载器递归加载
    } else {
        url = getBootstrapResource(name);
    }
    if (url == null) {
        url = findResource(name);
    }
    return url;
}
```

ClassLoader.getResource是是否递归交给父类加载器递归加载，如果父类加载器为空，则说明是引导类加载器，然后利用该类加载器去获取资源路径。如果url为空，那么就说明父加类载器未获取到，就需要自身类加载器去获取，最后返回url。<br/>
<br/>
可以看到Class.getResource就是先对路径name进行了resolveName，之后还是获取了自己的ClassLoader后调用ClassLoader.getResource。<br/>
<br/>
再看看resolveName

```Java
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
```

其实就是根据你是否在路径前加了"/"，而进行了路径的拼接。<br/>
如果你没有加/，那么就是相对路径，所以就会将之前的路径进行一个拼接，再交给类加载器去获取URL。<br/>
而如果加了"/"，那么就说绝对路径了，此时的绝对路径相当于类加载器的相对路径，所以resloveName会将"/"去除后，交给类加载器去获取URL。

```Java
URL resource = classLoader.getResource("");
URL resource2 = XINApplicationContext.class.getResource("");
System.out.println(resource);
System.out.println(resource2);

file:/D:/IdeaProjects/XINSpring/target/classes/
file:/D:/IdeaProjects/XINSpring/target/classes/com/spring/
//----------------------
URL resource = classLoader.getResource("/");//"/"表示Boot ClassLoader中的加载范围，因为这个类加载器是C++实现的，所以加载范围为null
URL resource2 = XINApplicationContext.class.getResource("/");
System.out.println(resource);
System.out.println(resource2);

null
file:/D:/IdeaProjects/XINSpring/target/classes/
//----------------------
URL resource = classLoader.getResource("abc");
URL resource2 = XINApplicationContext.class.getResource("/abc");
System.out.println(resource);
System.out.println(resource2);

file:/D:/IdeaProjects/XINSpring/target/classes/abc
file:/D:/IdeaProjects/XINSpring/target/classes/abc
```