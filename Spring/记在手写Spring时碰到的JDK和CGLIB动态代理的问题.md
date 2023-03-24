在学习手写Spring的途中，想要实现AOP功能，因为AOP是基于代理实现的，所以就要通过JDK或CGLIB动态代理来创建代理类。问题就发生在，被代理的对象中是有属性的，而且该属性在被代理前就会被注入，也就是这里出现了问题。
<br/>
举例假设Person是被代理类，其中有属性people是People类的对象。

```Java
public class Person implements PersonInterface{
    private People people;

    public Person(People people) {
        this.people = people;
    }

    @Override
    public void say() {
        System.out.println(this.people);
    }
}
```

<br/>
一开始我使用JDK进行动态代理，是没有任何问题的。

```Java
Person person = new Person(new People());
PersonInterface personInterface = (PersonInterface) Proxy.newProxyInstance(person.getClass().getClassLoader()
        , person.getClass().getInterfaces(), (Object proxy, Method method, Object[] args)->{
    return method.invoke(person, args);
        });
personInterface.say();

// 结果
// com.People@379619aa
```

<br/>
但是当用CGLIB动态代理时，就会出现问题。

```Java
class MyInvocationHandler implements MethodInterceptor {
    private Object instance;

    public Object getInstance(Object object) {
        this.instance = object;
        //在内存中创建一个动态类的字节码
        Enhancer enhancer=new Enhancer();//此时并没有做继承
        //设置父类,因为Cglib是针对指定的类生成一个子类，所以需要指定父类
        //除了完成继承关系外，还将父类所有的方法名反射过来，并在自己的类中
        enhancer.setSuperclass(this.instance.getClass());

        enhancer.setCallback(this);
        //创建并返回代理对象
        return enhancer.create();
    }

    @Override
    public Object intercept(Object o, Method method, Object[] objects, MethodProxy methodProxy) throws Throwable {
        return methodProxy.invokeSuper(o, objects);
    }
}

MyInvocationHandler myInvocationHandler = new MyInvocationHandler();
Person person = new Person(new People());
Person personproxy = (Person)myInvocationHandler.getInstance(person);
personproxy.say();

// 报错
// Exception in thread "main" java.lang.IllegalArgumentException: Superclass has no null constructors but no arguments were given
```

<br/>
可见CGLIGB动态代理是需要无参构造函数的，所以我们修改下。

```Java
MyInvocationHandler myInvocationHandler = new MyInvocationHandler();
Person person = new Person();
person.setPeople(new People());
Person personproxy = (Person)myInvocationHandler.getInstance(person);
personproxy.say();

// 结果
// null
```

<br/>
为什么经过CGLIB动态代理后，属性就为空了呢？
那是因为CGLIB动态代理是通过ASM生成子类，而生成子类的方式也是通过父类的无参构造函数，所以说父类中的属性值子类都无法直接获得。
<br/>
但是Spring通过CGLIB动态代理却可以获得，这是为什么呢？那就要从CGLIB的底层说起了。

![CGLIB底层](https://cdn.jsdelivr.net/gh/starmilkxin/picturebed/img/20220501195957.png)

我们可以看到CGLIB底层是创建了三个CLASS文件，图中第二个是代理类的class，第一个和第三个分别是代理类和被代理类的FastClass，分别保存各自的函数方法，代理类的FastClass中是增强的方法，被代理类的FastClass中就是原方法。CGLIB正是通过FastClass来代替JDK的反射的使用。
<br/>
在代理类(被代理类是父类，代理类也就是子类)的class文件中，所有的父类的方法都被重写了，都会调用我们一开始重写的intercept方法，回过头来看看我们重写的intercept方法。

```Java
@Override
public Object intercept(Object o, Method method, Object[] objects, MethodProxy methodProxy) throws Throwable {
    return methodProxy.invokeSuper(o, objects);
}
```

<br/>
其中Object o是代理的对象，Method method 是原父类的方法，Object[] objects是方法的参数，MethodProxy methodProxy是代理的方法。
所以我们目前的写法就是，CGLIB通过无参构造了被代理类的子类对象，也就是代理类对象。那么这个代理类对象将父类的所有方法重写了，都会执行intercept方法，在intercept方法中就可以进行上下文操作。<br/>
<br/>
关键点来了<br/>
当执行的函数中，含有另一个自身的其他函数时，就会发生自身调用增强的情况。譬如 proxy.a()方法中有b()，而b()就是proxy的方法，那么就会a()和b()都会进入intercpet方法中被AOP。
但是<br/>
当我们将methodProxy.invokeSuper(o, objects)改写为methodProxy.invoke(o, objects) 或者method.invoke(被代理类对象, objects)后，就不会出现自身调用增强的情况（这也是Spring所使用的方式）。<br/>
<br/>
总体且简洁地来说就是<br/>
假如被代理类中有say()，hi()，hello()方法的话，那么代理类继承了被代理类会重写say()，hi()，hello()这三个方法，使得他们都执行我们重写的intercept方法。在intercpet方法中实现AOP。
如果我们的方法调用是methodProxy.invokeSuper(o, objects)，那么就会调用代理类中的FastClass中的方法

![代理类FastClass](https://cdn.jsdelivr.net/gh/starmilkxin/picturebed/img/20220501203743.png)

可以看到代理类中FastClass方法都是代理类调用自己新创造的代理方法（不要和代理类重写被代理类方法混淆），而其实代理类自己创造的代理方法也就是直接调用原父类的方法，但是由于代理类不含有父类的属性值，因此会出现我们一开始输出null的情况。同时，在调用代理类自己创造的代理方法时，因为this就是自己代理类对象，所以代理方法中出现其他原父类方法的话，依旧会进入到代理类重写的方法中->再进入到intercept中进行AOP->再执行代理类自己创造的代理方法。这也就解释了我们之前的两个问题。<br/>
<br/>
如果我们将methodProxy.invokeSuper(o, objects)改写为methodProxy.invoke(被代理类对象, objects)后，那么调用的就是被代理类的FastClass中的函数，

![被代理类的FastClass](https://cdn.jsdelivr.net/gh/starmilkxin/picturebed/img/20220501204405.png)

可以看到被代理类的FastClass中的方法，就是被代理类对象执行自己原有的方法，因为就是原来的被代理类对象，所以通过这种方式，原属性值不会被丢失！同时因为执行的过程中，this对象是被代理类对象自己，所以再有自己的其他方法的话，就不会再次被AOP，出现自身调用增强的这种情况了！
当然了，如果改为method.invoke(被代理类对象，objects)的话，那就和CGLIB没关系了，也是可以做到和methoProxy.invoke(o, objects)一样的效果的！<br/>
<br/>
在这里还要再纠正几个错误<br/>
首先看看MethodProxy源码分析

```Java
public class MethodProxy {
    //下面的前三个变量在create方法中，都已经得到了初始值了。
    private Signature sig1;
    private Signature sig2;
    private MethodProxy.CreateInfo createInfo;
    //FastClassInfo是在调用methodProxy.invoke或者methodProxy.invokeSuper中，init()会触发，后面再来细看这个。
    private volatile MethodProxy.FastClassInfo fastClassInfo;

    public Object invokeSuper(Object obj, Object[] args) throws Throwable {
        try {
            this.init();
            MethodProxy.FastClassInfo fci = this.fastClassInfo;
            return fci.f2.invoke(fci.i2, obj, args);
        } catch (InvocationTargetException var4) {
            throw var4.getTargetException();
        }
    }
    public Object invoke(Object obj, Object[] args) throws Throwable {
        try {
            this.init();
            MethodProxy.FastClassInfo fci = this.fastClassInfo;
            return fci.f1.invoke(fci.i1, obj, args);
        } catch (InvocationTargetException var4) {
            throw var4.getTargetException();
        } catch (IllegalArgumentException var5) {
            if(this.fastClassInfo.i1 < 0) {
                throw new IllegalArgumentException("Protected method: " + this.sig1);
            } else {
                throw var5;
            }
        }
    }
    //本质就是要生成一个fastClassInfo,fastClassInfo里面存放着两个fastclass，f1,f2。
    //还有两个方法索引的值i1,i2。
    private void init() {
        if(this.fastClassInfo == null) {
            Object var1 = this.initLock;
            synchronized(this.initLock) {
                if(this.fastClassInfo == null) {
                    MethodProxy.CreateInfo ci = this.createInfo;
                    MethodProxy.FastClassInfo fci = new MethodProxy.FastClassInfo();
                    //难道每一个方法，我们都去生成一个fastclass吗？
                    //不是的，每一个方法的fastclass都是一样的，只不过他们的i1,i2不一样。如果缓存中就取出，没有就生成新的FastClass
                    fci.f1 = helper(ci, ci.c1);
                    fci.f2 = helper(ci, ci.c2);
                    fci.i1 = fci.f1.getIndex(this.sig1);
                    fci.i2 = fci.f2.getIndex(this.sig2);
                    this.fastClassInfo = fci;
                    this.createInfo = null;
                }
            }
        }
    }
//根据一个类的信息，返回的该对象的一个Fastclass
    private static FastClass helper(MethodProxy.CreateInfo ci, Class type) {
        Generator g = new Generator();
        g.setType(type);
        g.setClassLoader(ci.c2.getClassLoader());
        g.setNamingPolicy(ci.namingPolicy);
        g.setStrategy(ci.strategy);
        g.setAttemptLoad(ci.attemptLoad);
        return g.create();
    }
}
```

init()的过程就是生成FastClassInfo。<br/>
<br/>
fci存放了两个类的fastClass。<br/>
其中f1是被代理的类的fastclass，f2也就是代理类的fastclass<br/>
这也就是为什么，在classes文件中会生成三个class文件了，一个代理类，两个fastclass.<br/>
invokeSuper会调用f2.invoke，从来调用代理类方法，而invoke会调用f1.invoke，从而调用被代理类方法。<br/>
<br/>
开始纠正几个错误，<br/>
首先method是没有invokeSuper方法的，<br/>
当我们调用methodProxy.invoke(o, objects)和method.invoke(o, objects)时，都会出现无限循环从而栈内存溢出的情况发生（因为他们都是代理类调用被代理类的方法->进入自己重写的方法->进入intercept->反复）。<br/>
当调用methodProxy.invokeSuper(被代理类, objects)时，会因为无法强转而报错，<br/>
当调用methodProxy.invoke(被代理类, objects)时，会成功，因为此时是被代理类调用被代理类的方法。<br/>
<br/>
参考：<br/>
+ https://blog.csdn.net/MakeContral/article/details/79593732
+ https://www.cnblogs.com/lvbinbin2yujie/p/10284316.html
+ https://www.cnblogs.com/lyhero11/p/15557389.html