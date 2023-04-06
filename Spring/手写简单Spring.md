# 手写简单 Spring

## 自定义注解

```java
@Retention(RetentionPolicy.RUNTIME)
@Target(ElementType.TYPE)
public @interface ComponentScan {
    String value() default "";
}

@Retention(RetentionPolicy.RUNTIME)
@Target(ElementType.TYPE)
public @interface Component {
    String value() default "";
}

@Retention(RetentionPolicy.RUNTIME)
@Target({ElementType.FIELD, ElementType.METHOD})
public @interface Autowired {
}

@Retention(RetentionPolicy.RUNTIME)
@Target(ElementType.TYPE)
public @interface Scope {
    String value() default "singleton";
}
```

## ApplicationContext 初始化

根据传入的配置类，解析配置类，实例并初始化每个bean:

```java
public XINApplicationContext(Class<?> configClass) {
    this.configClass = configClass;
    // 解析配置类
    scan(configClass);
    // 实例并初始化每个bean
    for (String beanName : beanDefinitionMap.keySet()) {
        Object bean = getBean(beanName);
    }
}
```

解析配置类的具体步骤为:

- 获取该配置类 ComponentScan 上的值（扫描路径），默认为配置类的路径。
- 将该路径的资源加入队列中。
- 循环迭代，取出队列中的文件:
  - 如果 file 是文件夹，则将其中所有的文件加入队列中。
  - 如果 file 是 class 文件，则解析该 class 文件。

解析 class 文件:

- 获取到类名，通过类加载器得到类，如果当前类没有 Component 注解直接跳过。
- 如果当前类有 Component 注解，如果实现了 BeanPostProcessor 接口，则构建实例加入到 beanPostProcessorList 中。
- 构造 BeanDefinition，记录 scope，根据 beanName 将 beanDefinition 存入 beanDefinitionMap(ConcurrentHashMap) 中。

```java
    /**
     * 将所有bean的beanDefinition放入beanDefinitionMap中
     * @param configClass 配置文件类
     */
    public void scan (Class<?> configClass) {
        //解析配置类
        //解析ComponentScan
        ComponentScan componentScan = configClass.getDeclaredAnnotation(ComponentScan.class);
        //扫描的路径
        String path = componentScan.value();
        //默认为配置类的路径
        if (path.equals("")) {
            String _path = configClass.getResource("").getPath();
            path = _path.substring(_path.indexOf("classes/")+8);
        }
        path = path.replace(".", "/");
        ClassLoader classLoader = XINApplicationContext.class.getClassLoader();//app
        //转化后的扫描的路径
        URL resource = classLoader.getResource(path);
        assert resource != null;
        //得到文件
        File file = new File(resource.getFile());
        //用于循环迭代
        Queue<File> fileQueue = new LinkedList<>();
        fileQueue.add(file);
        while (!fileQueue.isEmpty()) {
            file = fileQueue.poll();
            //如果file是文件夹，则将其中所有的文件加入队列中
            if (file.isDirectory()) {
                File[] files = file.listFiles();
                if (files != null) {
                    fileQueue.addAll(Arrays.asList(files));
                }
            }else {
                String fileName = file.getAbsolutePath();
                //说明是class文件
                if (fileName.endsWith(".class")) {
                    String className = fileName.substring(fileName.indexOf("classes")+8, fileName.indexOf(".class"));
                    //获取到类名
                    className = className.replace("\\", ".");
                    try {
                        //通过类加载器得到类
                        Class<?> clazz = classLoader.loadClass(className);
                        //当前类有Component注解
                        if (clazz.isAnnotationPresent(Component.class)) {
                            // BeanPostProcessor
                            if (BeanPostProcessor.class.isAssignableFrom(clazz)) {
                                BeanPostProcessor instance = (BeanPostProcessor) clazz.getDeclaredConstructor().newInstance();
                                beanPostProcessorList.add(instance);
                            }

                            //Bean解析类
                            BeanDefinition beanDefinition = new BeanDefinition();
                            beanDefinition.setClazz(clazz);
                            Component componentAnnotation = clazz.getAnnotation(Component.class);
                            String beanName = componentAnnotation.value().equals("") ?
                                    //类名首字母小写作为beanName
                                    clazz.getSimpleName().substring(0, 1).toLowerCase() + clazz.getSimpleName().substring(1)
                                    : componentAnnotation.value();

                            //当前类有Scope注解
                            if (clazz.isAnnotationPresent(Scope.class)) {
                                //判断是单例还是原型
                                Scope scopeAnnotation = clazz.getDeclaredAnnotation(Scope.class);
                                if (!scopeAnnotation.value().equals("singleton") && !scopeAnnotation.value().equals("prototype")) {
                                    throw new RuntimeException("Scope value wrong");
                                }else {
                                    beanDefinition.setScope(scopeAnnotation.value());
                                }
                            }
                            beanDefinitionMap.put(beanName, beanDefinition);
                        }
                    } catch (ClassNotFoundException | NoSuchMethodException | InvocationTargetException | InstantiationException | IllegalAccessException e) {
                        e.printStackTrace();
                    }
                }
            }
        }
    }
```

实例并初始化每个 bean:

```java
// 实例并初始化每个bean
for (String beanName : beanDefinitionMap.keySet()) {
    Object bean = getBean(beanName);
}
```

其中:

- 如果 bean 为单例，则分别从一级、二级、三级缓存中获取，如果获取不到，则 createBean 新建实例。
- 如果 bean 为原型，则直接 createBean 新建实例。

createBean:

- 反射创建实例。
- 单例情况下考虑循环依赖（原型情况下是无法解决循环依赖问题的）。
- 依赖注入。
- Aware回调。
- bean 前置处理。
- bean 初始化。
- bean 后置处理。
- 单例情况下考虑缓存升级并获取缓存中的实例。

```java
    /**
     * 根据bean解析类创建bean
     * @param beanDefinition bean解析类
     * @return bean
     */
    public Object createBean(String beanName, BeanDefinition beanDefinition) {
        //先根据Clazz获取其Constructor
        Constructor<?> constructor = null;
        try {
            constructor = beanDefinition.getClazz().getConstructor();
            // 使得可以调用私有方法
            constructor.setAccessible(true);
            // 创建实例
            Object instance = constructor.newInstance();

            // 单例情况下考虑循环依赖
            if (beanDefinition.getScope().equals("singleton")) {
                // 提前暴露到三级缓存
                singletonFactories.put(beanName, new SingletonFactory(beanName, beanDefinition, instance));
            }

            for (Field declaredField : beanDefinition.getClazz().getDeclaredFields()) {
                //依赖注入
                //根据属性名
                if (declaredField.isAnnotationPresent(Autowired.class)) {
                    Object bean = getBean(declaredField.getName());
                    if (bean == null) {
                        throw new RuntimeException("Autowired 属性不存在");
                    }
                    declaredField.setAccessible(true);
                    declaredField.set(instance, bean);
                }
            }

            // Aware回调
            if (instance instanceof BeanNameAware) {
                ((BeanNameAware)instance).setBeanName(beanName);
            }

            // beanPostProcessor#postProcessBeforeInitialization
            for (BeanPostProcessor beanPostProcessor : beanPostProcessorList) {
                instance = beanPostProcessor.postProcessBeforeInitialization(instance, beanName);
            }

            // 初始化
            if (instance instanceof InitializingBean) {
                try {
                    ((InitializingBean)instance).afterPropertiesSet();
                }catch (Exception e) {
                    e.printStackTrace();
                }
            }

            // beanPostProcessor#postProcessAfterInitialization
            for (BeanPostProcessor beanPostProcessor : beanPostProcessorList) {
                instance = beanPostProcessor.postProcessAfterInitialization(instance, beanName);
            }

            // 单例情况下考虑缓存升级并获取缓存中的实例
            if (beanDefinition.getScope().equals("singleton")) {
                instance = getSingletonObject(beanName);
                // 将二级缓存删除并升级到一级缓存
                singletonObjects.put(beanName, instance);
                earlySingletonObjects.remove(beanName);
            }

            return instance;
        } catch (NoSuchMethodException | InvocationTargetException | InstantiationException | IllegalAccessException e) {
            e.printStackTrace();
        }
        return null;
    }
```

## 使用

配置类

```java
@ComponentScan
public class MyConfig {
}
```

服务

```java
@Component
public class PersonService {
    @Autowired
    private UserService userService;

    public void ok() {
        System.out.println(this.getClass());
        System.out.println(userService.getClass());
    }
}

@Component
public class UserService{
    @Autowired
    private PersonService personService;

    public void ok() {
        System.out.println(this.getClass());
        System.out.println(personService.getClass());
    }
}
```

运行

```java
public class Test {
    public static void main(String[] args) {
        XINApplicationContext applicationContext = new XINApplicationContext(MyConfig.class);
        UserService userService = (UserService) applicationContext.getBean("userService");
        userService.ok();
        PersonService personService = (PersonService) applicationContext.getBean("personService");
        personService.ok();
    }
}
```

