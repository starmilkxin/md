Spring事务分为编程式事务和声明式事务。<br/>
编程式事务就是手写，手动事务的开启、提交和回滚。<br/>
声明式事务就是注解@Transactional。<br/>
<br/>
那么声明式事务的实现原理又是什么呢？

# JDK动态代理
JDK动态代理是利用反射机制生成一个实现代理接口的匿名类，在调用具体方法前调用InvokeHandler来处理。<br/>
JDK动态代理只能用于带接口的类。

```Java
//JDK动态代理实现InvocationHandler接口
public class JdkProxy implements InvocationHandler {
    private Object target ;//需要代理的目标对象
    
    @Override
    public Object invoke(Object proxy, Method method, Object[] args) throws Throwable {
        System.out.println("JDK动态代理，监听开始！");
        Object result = method.invoke(target, args);
        System.out.println("JDK动态代理，监听结束！");
        return result;
    }
    //定义获取代理对象方法
    private Object getJDKProxy(Object targetObject){
        //为目标对象target赋值
        this.target = targetObject;
        //JDK动态代理只能针对实现了接口的类进行代理，newProxyInstance 函数所需参数就可看出
        return Proxy.newProxyInstance(targetObject.getClass().getClassLoader(), targetObject.getClass().getInterfaces(), this);
    }
    
    public static void main(String[] args) {
        JdkProxy jdkProxy = new JdkProxy();//实例化JDKProxy对象
        UserManager user = (UserManager) jdkProxy.getJDKProxy(new UserManagerImpl());//获取代理对象
        user.addUser("admin", "123123");//执行新增方法
    }
    
}
```

<br/>
简单点的形象代码就是这样

```Java
//接口
interface Service {
    void doNeedTx();

    void doNotneedTx();
}

//目标类，实现接口
class ServiceImpl implements Service {

    @Transactional
    @Override
    public void doNeedTx() {
        System.out.println("execute doNeedTx in ServiceImpl");
    }

    //no annotation here
    @Override
    public void doNotneedTx() {
        this.doNeedTx();
    }
}

//代理类，也要实现相同的接口
class ProxyByJdkDynamic implements Service {

    //包含目标对象
    private Service target;

    public ProxyByJdkDynamic(Service target) {
        this.target = target;
    }

    //目标类中此方法带注解，进行特殊处理
    @Override
    public void doNeedTx() {
        //开启事务
        System.out.println("-> create Tx here in Proxy");
        //调用目标对象的方法，该方法已在事务中了
        target.doNeedTx();
        //提交事务
        System.out.println("<- commit Tx here in Proxy");
    }

    //目标类中此方法没有注解，只做简单的调用
    @Override
    public void doNotneedTx() {
        //直接调用目标对象方法
        target.doNotneedTx();
    }
}
```

此时有几个问题：
1. 问：假如有个接口，它包含两个方法a和b，然后有一个类实现了该接口。在该实现类里在a上标上事务注解、b上不标，此时事务是怎样的？<br/>
   答：a标注解了，肯定有事务，b没有注解，所以没有事务。

2. 问：在方法b里调用方法a，其它保持不变，此时再调用方法b，会有事务吗？<br/>
   答：b没有事务。<br/>
   目标类是我们自己写的，肯定是没有事务的。代理类是系统生成的，对带注解的方法进行事务增强，没有注解的方法原样调用，所以事务是代理类加上去的。那回到这个问题，我们调用的方法不带注解，因此代理类不开事务，而是直接调用目标对象的方法。当进入目标对象的方法后，执行的上下文已经变成目标对象本身了，因为目标对象的代码是我们自己写的，和事务没有半毛钱关系，此时你再调用带注解的方法，照样没有事务，只是一个普通的方法调用而已。

# CGLIB
CGLIB动态代理是利用asm开源包，对代理对象类的class文件加载进来，通过修改其字节码生成子类来处理。<br/>
CGLIB动态代理既能用于带接口的类又能用于不带接口的类。<br/>
如何强制使用CGLIB实现AOP？
1. 添加CGLIB库，SPRING_HOME/cglib/*.jar
2. 在spring配置文件中加入<aop:aspectj-autoproxy proxy-target-class="true"/>

```Java
//Cglib动态代理，实现MethodInterceptor接口
public class CglibProxy implements MethodInterceptor {
    private Object target;//需要代理的目标对象
    
    //重写拦截方法
    @Override
    public Object intercept(Object obj, Method method, Object[] arr, MethodProxy proxy) throws Throwable {
        System.out.println("Cglib动态代理，监听开始！");
        Object invoke = method.invoke(target, arr);//方法执行，参数：target 目标对象 arr参数数组
        System.out.println("Cglib动态代理，监听结束！");
        return invoke;
    }
    //定义获取代理对象方法
    public Object getCglibProxy(Object objectTarget){
        //为目标对象target赋值
        this.target = objectTarget;
        Enhancer enhancer = new Enhancer();
        //设置父类,因为Cglib是针对指定的类生成一个子类，所以需要指定父类
        enhancer.setSuperclass(objectTarget.getClass());
        enhancer.setCallback(this);// 设置回调 
        Object result = enhancer.create();//创建并返回代理对象
        return result;
    }
    
    public static void main(String[] args) {
        CglibProxy cglib = new CglibProxy();//实例化CglibProxy对象
        UserManager user =  (UserManager) cglib.getCglibProxy(new UserManagerImpl());//获取代理对象
        user.delUser("admin");//执行删除方法
    }
    
}
```

<br/>
简单点的形象代码就是这样

```Java
class Target {

    @Transactional
    public void doNeedTx() {
        System.out.println("execute doNeedTx in Target");
    }

    //no annotation here
    public void doNotneedTx() {
        this.doNeedTx();
    }
}

class ProxyByCGLIB extends Target {

    private Target target;

    public ProxyByCGLIB(Target target) {
        this.target = target;
    }

    @Override
    public void doNeedTx() {
        System.out.println("-> create Tx in Proxy");
        target.doNeedTx();
        System.out.println("<- commit Tx in Proxy");
    }

    @Override
    public void doNotneedTx() {
        target.doNotneedTx();
    }
}
```

因为是继承，final方法不能被重写所以不行，static方法和继承无关，private方法不能被继承所以不行，protected方法和package方法理论上要看具体环境但是Spring选择让protected和package方法不支持事务，所以只有public方法支持事务。


# 总结
1. 如果目标对象实现了接口，默认情况下会采用JDK的动态代理实现AOP <br/>
2. 如果目标对象实现了接口，可以强制使用CGLIB实现AOP <br/>
3. 如果目标对象没有实现了接口，必须采用CGLIB库，spring会自动在JDK动态代理和CGLIB之间转换<br/>
4. 只要是以代理方式实现的声明式事务，无论是JDK动态代理，还是CGLIB直接写字节码生成代理，都只有public方法上的事务注解才起作用。而且必须在代理类外部调用才行，如果直接在目标类里面调用，事务照样不起作用。<br/>

JDK动态代理和CGLIB字节码生成的区别？
1. JDK动态代理只能对实现了接口的类生成代理，而不能针对类
2. CGLIB是针对类实现代理，主要是对指定的类生成一个子类，覆盖其中的方法。<br/>
   因为是继承，final方法不能被重写所以不行，static方法和继承无关，private方法不能被继承所以不行，protected方法和package方法理论上要看具体环境但是Spring选择让protected和package方法不支持事务，所以只有public方法支持事务。<br/>
   <br/>
