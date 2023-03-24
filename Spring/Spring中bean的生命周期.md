Spring Bean的生命周期是Spring面试热点问题。Spring Bean的生命周期指的是从一个普通的Java类变成Bean的过程，深知Spring源码的人都知道这个给面试官将的话大可讲30分钟以上，如果你不没有学习过Spring的源码，可能就知道Aware接口和调用init方法这样的生命周期，所以这个问题即考察对Spring的微观了解，又考察对Spring的宏观认识，想要答好并不容易！本文希望能够从源码角度入手，帮助面试者彻底搞定Spring Bean的生命周期。
## 总体的创建过程
首先你要明白一点，Spring Bean总体的创建过程如下：<br/>

![Spring Bean总体的创建过程](https://xin-xinblog.oss-cn-shanghai.aliyuncs.com/img/20220430153016.png)

以注解类变成Spring Bean为例，Spring会扫描指定包下面的Java类，然后将其变成beanDefinition对象，然后Spring会根据beanDefinition来创建bean，特别要记住一点，Spring是根据beanDefinition来创建Spring bean的，关于beanDefinition下文会进行分析。

## beanDefinition
![beanDefinition](https://xin-xinblog.oss-cn-shanghai.aliyuncs.com/img/20220430153123.png)

上图就是beanDefinition所包含的内容，看到这些属性如果对Spring有所了解都应该知道几个，如lazyInit懒加载，在Spring中有一个@Lazy注解，用来标识其这个bean是否为懒加载的，scope属性，相应的也有@scope注解来标识这个bean其作用域，是单例还是多例，beanClass属性，在对象实例化时，会根据beanClass来创建对象、又比如autowireMode注入模型这个属性，这个属性用于记录使用怎样的注入模型，注入模型常用的有根据名称和根据类型、不注入三种注入模型。<br/>
<br/>
autowireMode虽然看似我们没有使用到过，但是知道这个对于查看Spring和其他框架整合的时候很有帮助，如果我们在beanDefinition指定其根据类型进行属性注入，那么在创建这个bean时，会将其beanClass中的所有属性都拿到，然后排出调基本属性（如类型String、Double、Boolean、Object），然后对于剩下的进行属性注入，记住一点，在beanDefinition指定其根据类型进行属性注入，即使不在属性上面使用@Autowired注解也会对其进行属性注入。在我们写注解类的时候为什么不使用@Autowired时，其属性就注入不进来呢？那是因为注解类在变成beanDefinition时，其注入类型是不注入，所以此时只有使用@Autowired注解进行标记的属性，才会完成依赖注入。<br/>
<br/>
说了这么多，总之大家要记住Spring会根据beanDefinition来完成bean的创建，为什么不直接使用对象的class对象来创建bean呢？因为在class对象仅仅能描述一个对象的创建，它不足以用来描述一个Spring bean，而对于是否为懒加载、是否是首要的、初始化方法是哪个、销毁方法是哪个，这个Spring中特有的属性在class对象中并没有，所有Spring就定义了beanDefinition来完成bean的创建。

## java类 -> beanDefinition对象
所以说如果要说Bean的生命周期，你不能不说Java类是在那一步变成beanDefinition的，我们先来看一下Spring中AbstractApplicationContext类中的refresh方法，这个方法就可以总结上面的图。

```Java
@Override
	public void refresh() throws BeansException, IllegalStateException {
		synchronized (this.startupShutdownMonitor) {
			// Prepare this context for refreshing.
			准备工作包括设置启动时间，是否激活标识位，
			// 初始化属性源(property source)配置
			prepareRefresh();
 
			// Tell the subclass to refresh the internal bean factory.
			//返回一个factory 为什么需要返回一个工厂
			//因为要对工厂进行初始化
			ConfigurableListableBeanFactory beanFactory = obtainFreshBeanFactory();
 
			// Prepare the bean factory for use in this context.
			//准备工厂
			prepareBeanFactory(beanFactory);
 
			try {
				// Allows post-processing of the bean factory in context subclasses.
				//这个方法在当前版本的spring是没用任何代码的
				//可能spring期待在后面的版本中去扩展吧
				postProcessBeanFactory(beanFactory);
 
				// Invoke factory processors registered as beans in the context.
				//在spring的环境中去执行已经被注册的 factory processors
				//设置执行自定义的ProcessBeanFactory 和spring内部自己定义的
				invokeBeanFactoryPostProcessors(beanFactory);
 
				// Register bean processors that intercept bean creation.
				//注册beanPostProcessor
				registerBeanPostProcessors(beanFactory);
 
				// Initialize message source for this context.
				initMessageSource();
 
				// Initialize event multicaster for this context.
				//初始化应用事件广播器
				initApplicationEventMulticaster();
 
				// Initialize other special beans in specific context subclasses.
				onRefresh();
 
				// Check for listener beans and register them.
				registerListeners();
 
				// Instantiate all remaining (non-lazy-init) singletons.
				finishBeanFactoryInitialization(beanFactory);
 
				// Last step: publish corresponding event.
				finishRefresh();
			}
 
			catch (BeansException ex) {
				if (logger.isWarnEnabled()) {
					logger.warn("Exception encountered during context initialization - " +
							"cancelling refresh attempt: " + ex);
				}
 
				// Destroy already created singletons to avoid dangling resources.
				destroyBeans();
 
				// Reset 'active' flag.
				cancelRefresh(ex);
 
				// Propagate exception to caller.
				throw ex;
			}
 
			finally {
				// Reset common introspection caches in Spring's core, since we
				// might not ever need metadata for singleton beans anymore...
				resetCommonCaches();
			}
		}
	}
```

有关Spring Bean生命周期最主要的方法有三个invokeBeanFactoryPostProcessors、registerBeanPostProcessors和finishBeanFactoryInitialization。<br/>
<br/>
其中invokeBeanFactoryPostProcessors方法会执行BeanFactoryPostProcessors后置处理器及其子接口BeanDefinitionRegistryPostProcessor，执行顺序先是执行BeanDefinitionRegistryPostProcessor接口的postProcessBeanDefinitionRegistry方法，然后执行BeanFactoryPostProcessor接口的postProcessBeanFactory方法。<br/>
<br/>
对于BeanDefinitionRegistryPostProcessor接口的postProcessBeanDefinitionRegistry方法，该步骤会扫描到指定包下面的标有注解的类，然后将其变成BeanDefinition对象，然后放到一个Spring中的Map中，用于下面创建Spring bean的时候使用这个BeanDefinition<br/>
<br/>
其中registerBeanPostProcessors方法根据实现了PriorityOrdered、Ordered接口，排序后注册所有的BeanPostProcessor后置处理器，主要用于Spring Bean创建时，执行这些后置处理器的方法，这也是Spring中提供的扩展点，让我们能够插手Spring bean创建过程。<br/>

![java类 -> beanDefinition对象](https://xin-xinblog.oss-cn-shanghai.aliyuncs.com/img/20220430153343.png)

## beanDefinition对象 -> Spring中的bean
finishBeanFactoryInitialization是完成非懒加载的Spring bean的创建的工作，你要想说Spring的生命周期，不要整其他没用的，直接告诉他在该步骤中会有8个后置处理的方法4个后置处理器的类贯穿在对象的实例化、赋值、初始化、和销毁的过程中，这4个后置处理器出现的地方分别为：

![后置处理器流程](https://xin-xinblog.oss-cn-shanghai.aliyuncs.com/img/20220430153446.png)

关于每个后置处理的作用如下：

![后置处理器关系](https://xin-xinblog.oss-cn-shanghai.aliyuncs.com/img/20220430153645.png)

首先看一下继承关系，InstantiationAwareBeanPostProcessor、SmartInstantiationAwareBeanPostProcessor、MergedBeanDefinitionPostProcessor直接或间接的继承BeanPostProcessor，而SmartInitializingSingleton则是单独的一个接口。<br/>
<br/>
一、InstantiationAwareBeanPostProcessor
InstantiationAwareBeanPostProcessor接口继承BeanPostProcessor接口，它内部提供了3个方法，再加上BeanPostProcessor接口内部的2个方法，所以实现这个接口需要实现5个方法。InstantiationAwareBeanPostProcessor接口的主要作用在于目标对象的实例化过程中需要处理的事情，包括实例化对象的前后过程以及实例的属性设置<br/>
<br/>
在org.springframework.beans.factory.support.AbstractAutowireCapableBeanFactory#createBean()方法的Object bean = resolveBeforeInstantiation(beanName, mbdToUse);方法里面执行了这个后置处理器<br/>
<br/>
二、SmartInstantiationAwareBeanPostProcessor
智能实例化Bean后置处理器（继承InstantiationAwareBeanPostProcessor）<br/>
<br/>
三、MergedBeanDefinitionPostProcessor<br/>
<br/>
四、SmartInitializingSingleton
智能初始化Singleton，该接口只有一个方法，是在所有的非懒加载单实例bean都成功创建并且放到Spring IOC容器之后进行执行的。<br/>
<br/>
下面按照图中的顺序进行说明每个后置处理的作用：<br/>

1、InstantiationAwareBeanPostProcessor# postProcessBeforeInstantiation<br/>

在目标对象实例化之前调用，方法的返回值类型是Object，我们可以返回任何类型的值。由于这个时候目标对象还未实例化，所以这个返回值可以用来代替原本该生成的目标对象的实例(一般都是代理对象)。如果该方法的返回值代替原本该生成的目标对象，后续只有postProcessAfterInitialization方法会调用，其它方法不再调用；否则按照正常的流程走<br/>

2、SmartInstantiationAwareBeanPostProcessor# determineCandidateConstructors<br/>

检测Bean的构造器，可以检测出多个候选构造器<br/>

3、MergedBeanDefinitionPostProcessor# postProcessMergedBeanDefinition<br/>

缓存bean的注入信息的后置处理器，仅仅是缓存或者干脆叫做查找更加合适，没有完成注入，注入是另外一个后置处理器的作用<br/>

4、SmartInstantiationAwareBeanPostProcessor# getEarlyBeanReference<br/>

循环引用的后置处理器，这个东西比较复杂， 获得提前暴露的bean引用。主要用于解决循环引用的问题，只有单例对象才会调用此方法，以后如果有时间，我会写一篇关于Spring是怎么处理循环依赖的，会将这个，现在你就只需要知道他能提前暴露bean就行了。<br/>

5、InstantiationAwareBeanPostProcessor# postProcessAfterInstantiation<br/>

方法在目标对象实例化之后调用，这个时候对象已经被实例化，但是该实例的属性还未被设置，都是null。如果该方法返回false，会忽略属性值的设置；如果返回true，会按照正常流程设置属性值。方法不管postProcessBeforeInstantiation方法的返回值是什么都会执行<br/>

6、InstantiationAwareBeanPostProcessor# postProcessPropertie<br/>

方法对属性值进行修改(这个时候属性值还未被设置，但是我们可以修改原本该设置进去的属性值)。如果postProcessAfterInstantiation方法返回false，该方法不会被调用。可以在该方法内完成对属性的自动注入（注意：在Spring5.1之前，完成属性自动注入的方法是postProcessPropertyValues方法，在5.1就开始使用postProcessPropertie方法了，如果发现不同也不必惊慌，就是改了一下方法名，内部逻辑一样的）（在这一步，会完成标注了@Autowired、@Resource注解的自动装配，如果有兴趣的同学可以看一下：Spring源码分析@Autowired、@Resource注解的区别，是从源码角度分析他们之间装配的不同，不过需要一点Spring的功底）<br/>
<br/>
调用invokeInitMethods方法 <br/>

先看代码invokeInitMethods方法的代码

```Java
private void invokeAwareMethods(final String beanName, final Object bean) {
		if (bean instanceof Aware) {
			if (bean instanceof BeanNameAware) {
				((BeanNameAware) bean).setBeanName(beanName);
			}
			if (bean instanceof BeanClassLoaderAware) {
				ClassLoader bcl = getBeanClassLoader();
				if (bcl != null) {
					((BeanClassLoaderAware) bean).setBeanClassLoader(bcl);
				}
			}
			if (bean instanceof BeanFactoryAware) {
				((BeanFactoryAware) bean).setBeanFactory(AbstractAutowireCapableBeanFactory.this);
			}
		}
	}
```

该方法就是查看这个bean是否实现应该的Aware接口，如果实现了，则调用相应的接口。 <br/>

7.BeanPostProcessor# postProcessBeforeInitialization<br/>

该方法会在初始化之前进行执行，其中有一个实现类比较重要ApplicationContextAwareProcessor，该后置处理的一个作用就是当应用程序定义的Bean实现ApplicationContextAware接口时注入ApplicationContext对象：

```Java
class ApplicationContextAwareProcessor implements BeanPostProcessor {
 
	private final ConfigurableApplicationContext applicationContext;
 
	private final StringValueResolver embeddedValueResolver;
 
 
	/**
	 * Create a new ApplicationContextAwareProcessor for the given context.
	 */
	public ApplicationContextAwareProcessor(ConfigurableApplicationContext applicationContext) {
		this.applicationContext = applicationContext;
		this.embeddedValueResolver = new EmbeddedValueResolver(applicationContext.getBeanFactory());
	}
 
 
	@Override
	@Nullable
	public Object postProcessBeforeInitialization(final Object bean, String beanName) throws BeansException {
		AccessControlContext acc = null;
 
		if (System.getSecurityManager() != null &&
				(bean instanceof EnvironmentAware || bean instanceof EmbeddedValueResolverAware ||
						bean instanceof ResourceLoaderAware || bean instanceof ApplicationEventPublisherAware ||
						bean instanceof MessageSourceAware || bean instanceof ApplicationContextAware)) {
			acc = this.applicationContext.getBeanFactory().getAccessControlContext();
		}
 
		if (acc != null) {
			AccessController.doPrivileged((PrivilegedAction<Object>) () -> {
				invokeAwareInterfaces(bean);
				return null;
			}, acc);
		}
		else {
			invokeAwareInterfaces(bean);
		}
 
		return bean;
	}
 
	private void invokeAwareInterfaces(Object bean) {
		if (bean instanceof Aware) {  //判断该类是否是Aware的子类
			if (bean instanceof EnvironmentAware) {
 
				((EnvironmentAware) bean).setEnvironment(this.applicationContext.getEnvironment());
			}
			if (bean instanceof EmbeddedValueResolverAware) {
				((EmbeddedValueResolverAware) bean).setEmbeddedValueResolver(this.embeddedValueResolver);
			}
			if (bean instanceof ResourceLoaderAware) {
				((ResourceLoaderAware) bean).setResourceLoader(this.applicationContext);
			}
			if (bean instanceof ApplicationEventPublisherAware) {
				((ApplicationEventPublisherAware) bean).setApplicationEventPublisher(this.applicationContext);
			}
			if (bean instanceof MessageSourceAware) {
				((MessageSourceAware) bean).setMessageSource(this.applicationContext);
			}
			//spring帮你set一个applicationContext对象
			//所以当我们自己的一个对象实现了ApplicationContextAware对象只需要提供setter就能得到applicationContext对象
			//此处应该有点赞加收藏
			if (bean instanceof ApplicationContextAware) {
				if (!bean.getClass().getSimpleName().equals("IndexDao"))
				((ApplicationContextAware) bean).setApplicationContext(this.applicationContext);
			}
		}
	}
 
	@Override
	public Object postProcessAfterInitialization(Object bean, String beanName) {
		return bean;
	}
 
}
```

看到这里你可能恍然大悟，原来调用BeanFactoryAware和ApplicationContextAware时，相应的处理是不同的，这个你可能看别人写的博客时，完全就没有介绍，，所以，你如果把这个不同点说出来，那么面试官就会认为你是看过源码的人。ApplicationContextAware是在一个BeanPostProcessor后置处理器的postProcessBeforeInitialization方法中进行执行的。<br/>

回调执行bean的声明周期回调中的init方法，该步骤不做介绍，就是执行bean的init方法<br/>

8.BeanPostProcessor# postProcessAfterInitialization<br/>

该后置处理器的执行是在调用init方法后面进行执行，主要是判断该bean是否需要被AOP代理增强，如果需要的话，则会在该步骤返回一个代理对象。<br/>

9.SmartInitializingSingleton# afterSingletonsInstantiated<br/>

该方法会在所有的非懒加载单实例bean都成功创建并且放到Spring IOC容器之后，依次遍历所有的bean，如果当前这个bean是SmartInitializingSingleton的子类，那么就强转成SmartInitializingSingleton类，然后调用SmartInitializingSingleton的afterSingletonsInstantiated方法。<br/>

查看一下DefaultListableBeanFactory#preInstantiateSingletons方法<br/>

```Java
public void preInstantiateSingletons() throws BeansException {
		// Iterate over a copy to allow for init methods which in turn register new bean definitions.
		// While this may not be part of the regular factory bootstrap, it does otherwise work fine.
		List<String> beanNames = new ArrayList<>(this.beanDefinitionNames);
 
		// Trigger initialization of all non-lazy singleton beans...
        //1.遍历所有的beanDefinition对象，然后根据beanDefinition去创建对象，此步骤会完成当前
        //Spring容器中所有的bean，getBean方法会查看容器中是否有该bean，如果没有则去创建
		for (String beanName : beanNames) {
			RootBeanDefinition bd = getMergedLocalBeanDefinition(beanName);
			if (!bd.isAbstract() && bd.isSingleton() && !bd.isLazyInit()) {
				if (isFactoryBean(beanName)) {
					Object bean = getBean(FACTORY_BEAN_PREFIX + beanName);
					if (bean instanceof FactoryBean) {
						final FactoryBean<?> factory = (FactoryBean<?>) bean;
						boolean isEagerInit;
						if (System.getSecurityManager() != null && factory instanceof SmartFactoryBean) {
							isEagerInit = AccessController.doPrivileged((PrivilegedAction<Boolean>)
											((SmartFactoryBean<?>) factory)::isEagerInit,
									getAccessControlContext());
						}
						else {
							isEagerInit = (factory instanceof SmartFactoryBean &&
									((SmartFactoryBean<?>) factory).isEagerInit());
						}
						if (isEagerInit) {
							getBean(beanName);
						}
					}
				}
				else {
					getBean(beanName);
				}
			}
		}
 
		// Trigger post-initialization callback for all applicable beans...
                //2.走到这里的时候，说明所有的bean都已经被成功创建，依次遍历所有的bean
		for (String beanName : beanNames) {
			Object singletonInstance = getSingleton(beanName);
                      //3.判断该bean是否为SmartInitializingSingleton接口的子类，如果是，则执行afterSingletonsInstantiated方法
			if (singletonInstance instanceof SmartInitializingSingleton) {
				final SmartInitializingSingleton smartSingleton = (SmartInitializingSingleton) singletonInstance;
				if (System.getSecurityManager() != null) {
					AccessController.doPrivileged((PrivilegedAction<Object>) () -> {
						smartSingleton.afterSingletonsInstantiated();
						return null;
					}, getAccessControlContext());
				}
				else {
					smartSingleton.afterSingletonsInstantiated();
				}
			}
		}
	}
```

对于SmartInitializingSingleton# afterSingletonsInstantiated方法，是在所有的bean完成之后进行调用的，这个扩展点可以让我们自己很轻松的自定义注解，完成我们想要实现的逻辑，比如使用@EventListener注解实现事件监听，其底层就是童年各国SmartInitializingSingleton# afterSingletonsInstantiated来实现的，关于@EventListener可以自行百度，以后有时间再进行分析，而且我在看公司内部基于Spring开发的框架时，也用到了SmartInitializingSingleton进行定义注解，实现自己的逻辑，对于Spring提供的这么多扩展点，感觉就这个我能用到一点（还是太辣鸡o(╥﹏╥)o），所以对于这个重点说一下。<br/>

 此时，Spring Bean的生命周期就算说完了，如果你能把上面的步骤将给面试官听，那么保证是没有问题的，在回答这个问题的时候可以从下面几个点思考：<br/>

 1.普通Java类是在哪一步变成beanDefinition的<br/>

 2.有8个后置处理器方法4个后置处理器，贯穿了对象的创建->属性赋值->初始化->销毁<br/>

 3.然后再说处理器方法执行的时机和作用<br/>

 4.invokeInitMethods执行的Aware和BeanPostProcessor# postProcessBeforeInitialization方法执行的aware<br/>

总结<br/>
Spring Bean的生命周期分为四个阶段和多个扩展点。扩展点又可以分为影响多个Bean和影响单个Bean。整理如下：<br/>
<br/>
四个阶段
+ 实例化 Instantiation
+ 属性赋值 Populate
+ 初始化 Initialization
+ 销毁 Destruction

多个扩展点
+ 影响多个Bean
+ + BeanPostProcessor 
+ + 1. postProcessBeforeInitialization
+ + 2. postProcessAfterInitialization
+ + InstantiationAwareBeanPostProcessor
+ + 1. postProcessBeforeInstantiation
+ + 2. postProcessAfterInstantiation
+ + 3. postProcessPropertyValues
+ + MergedBeanDefinitionPostProcessor
+ + 1. postProcessMergedBeanDefinition
+ + SmartInstantiationAwareBeanPostProcessor
+ + 1. determineCandidateConstructors
+ + 2. getEarlyBeanReference
+ 影响单个Bean
+ + Aware
+ + + Aware Group1（调用invokeInitMethods方法）
+ + + + BeanNameAware
+ + + + BeanClassLoaderAware
+ + + + BeanFactoryAware
+ + + Aware Group2（调用Aware和BeanPostProcessor# postProcessBeforeInitialization方法）
+ + + + EnvironmentAware
+ + + + EmbeddedValueResolverAware
+ + + + ApplicationContextAware(ResourceLoaderAware\ApplicationEventPublisherAware\MessageSourceAware)
+ + 生命周期
+ + + InitializingBean
+ + + DisposableBean

<br/>
原文链接：https://blog.csdn.net/qq_35634181/article/details/104473308