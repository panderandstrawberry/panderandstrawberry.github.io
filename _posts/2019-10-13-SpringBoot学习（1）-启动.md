# SpringBoot 学习
## 学习思路
+ run()方法启动流程以及事件
+ 自动配置原理

### 启动流程介绍
我们从最开始的启动说起
```java
@SpringBootApplication
public class DemoApplication {

    public static void main(String[] args) {
        SpringApplication.run(DemoApplication.class, args);
    }

}
```
这里可以看见启动类中主要值执行了`SpringApplication.run()`这个方法，这个也是整个SpringBoot项目
启动的地方，一路跟进去可以发现，执行了如下的代码
```java
public static ConfigurableApplicationContext run(Class<?>[] primarySources,
        String[] args) {
    return new SpringApplication(primarySources).run(args);
}
```
可以看见先创建了`SpringApplication`这个对象，之后执行了这个对象的`run`方法,按照惯例，先看创建对象的过程
```java
public SpringApplication(ResourceLoader resourceLoader, Class<?>... primarySources) {
    //这里的 resourceLoade是一个null值
    this.resourceLoader = resourceLoader;
    Assert.notNull(primarySources, "PrimarySources must not be null");
    this.primarySources = new LinkedHashSet<>(Arrays.asList(primarySources));
    //判断当前容器是否是一个web应用
    this.webApplicationType = WebApplicationType.deduceFromClasspath();
    //设置初始化器
    setInitializers((Collection) getSpringFactoriesInstances(
            ApplicationContextInitializer.class));
    //设置监听器
    setListeners((Collection) getSpringFactoriesInstances(ApplicationListener.class));
    //返回当前程序的主类
    this.mainApplicationClass = deduceMainApplicationClass();
}
```
> 初始化器和监听器着这里起到什么作用？
+ 初始化器ApplicationContextInitializer
```java
public interface ApplicationContextInitializer<C extends ConfigurableApplicationContext> {

	/**
	 * Initialize the given application context.
	 * @param applicationContext the application to configure
	 */
	void initialize(C applicationContext);

}
```
ApplicationContextInitializer是一个回调接口，它会在ConfigurableApplicationContext的refresh()方法调用之前被调用,做一些容器的初始化工作。
这一点我们也可以通过*SpringApplication的实例run方法的实现代码得到验证①*
可以看到接口传入`applicationContext`对象，可以直接操作IOC容器

那么 `ApplicationContextInitializer`是怎么加入容器的，可以跟进`setInitializers()`这个方法
```java
setInitializers((Collection) getSpringFactoriesInstances(ApplicationContextInitializer.class));


private <T> Collection<T> getSpringFactoriesInstances(Class<T> type,
    Class<?>[] parameterTypes, Object... args) {
    ClassLoader classLoader = getClassLoader();
    // Use names and ensure unique to protect against duplicates
    Set<String> names = new LinkedHashSet<>(
            SpringFactoriesLoader.loadFactoryNames(type, classLoader));
    List<T> instances = createSpringFactoriesInstances(type, parameterTypes,
            classLoader, args, names);
    AnnotationAwareOrderComparator.sort(instances);
    return instances;
}
```
这里需要注意到的是`SpringFactoriesLoader.loadFactoryNames(type, classLoader))`这个方法，我们后边会遇到很多，这个也是
Springboot中比较重要的一个方法，看下内部实现
```java
private static Map<String, List<String>> loadSpringFactories(@Nullable ClassLoader classLoader) {
		MultiValueMap<String, String> result = cache.get(classLoader);
		if (result != null) {
			return result;
		}

		try {
			Enumeration<URL> urls = (classLoader != null ?
			//这里看见这里取了FACTORIES_RESOURCE_LOCATION这个资源文件，而这个常量是
			//public static final String FACTORIES_RESOURCE_LOCATION = "META-INF/spring.factories";
					classLoader.getResources(FACTORIES_RESOURCE_LOCATION) :
					ClassLoader.getSystemResources(FACTORIES_RESOURCE_LOCATION));
			result = new LinkedMultiValueMap<>();
			while (urls.hasMoreElements()) {
				URL url = urls.nextElement();
				UrlResource resource = new UrlResource(url);
				Properties properties = PropertiesLoaderUtils.loadProperties(resource);
				for (Map.Entry<?, ?> entry : properties.entrySet()) {
					String factoryClassName = ((String) entry.getKey()).trim();
					for (String factoryName : StringUtils.commaDelimitedListToStringArray((String) entry.getValue())) {
						result.add(factoryClassName, factoryName.trim());
					}
				}
			}
			cache.put(classLoader, result);
			return result;
		}
		catch (IOException ex) {
			throw new IllegalArgumentException("Unable to load factories from location [" +
					FACTORIES_RESOURCE_LOCATION + "]", ex);
		}
	}
```
这个方法这个总结下来就是，类路径下所有资源文件 `META-INF/spring.factories `内的信息，那个这个文件到底配置了什么东西

META-INF/spring.factories部分文件内容
```java
# Initializers
org.springframework.context.ApplicationContextInitializer=\
org.springframework.boot.autoconfigure.SharedMetadataReaderFactoryContextInitializer,\
org.springframework.boot.autoconfigure.logging.ConditionEvaluationReportLoggingListener

# Application Listeners
org.springframework.context.ApplicationListener=\
org.springframework.boot.autoconfigure.BackgroundPreinitializer

# Auto Configuration Import Listeners
org.springframework.boot.autoconfigure.AutoConfigurationImportListener=\
org.springframework.boot.autoconfigure.condition.ConditionEvaluationReportAutoConfigurationImportListener

# Auto Configuration Import Filters
org.springframework.boot.autoconfigure.AutoConfigurationImportFilter=\
org.springframework.boot.autoconfigure.condition.OnBeanCondition,\
org.springframework.boot.autoconfigure.condition.OnClassCondition,\
org.springframework.boot.autoconfigure.condition.OnWebApplicationCondition
```

这里我们看见了ApplicationContextInitializer字眼，正式我们刚才提到的接口，所以 综合起来，就是很久传入的接口类获取到配置文件中
配置的实现类的全类名，获取到全类名之后，很容易就能想到 通过反射，创建接口的实现对象，同时缓存起来，刚有提到，容器刷新之前调用这些
接口
```java
public void setInitializers(
        Collection<? extends ApplicationContextInitializer<?>> initializers) {
    this.initializers = new ArrayList<>();
    this.initializers.addAll(initializers);
}
```
初始化器除了配置方式之外，也自行给容器中添加初始化器，实现方式可参考：

[ApplicationContextInitializer的三种使用方法](https://blog.csdn.net/leileibest_437147623/article/details/81074174 "ApplicationContextInitializer的三种使用方法")

+ 监听器ApplicationListener

同样的原理，相信大家也能猜到`ApplicationListener`的实现原理，那么这里说下Listener的作用
先看下接口：

```java
@FunctionalInterface
public interface ApplicationListener<E extends ApplicationEvent> extends EventListener {

	/**
	 * Handle an application event.
	 * @param event the event to respond to
	 */
	void onApplicationEvent(E event);

}
```
1.ApplicationContext事件机制是观察者设计模式的实现，通过ApplicationEvent类和ApplicationListener接口，可以实现ApplicationContext事件处理。

2.如果容器中有一个ApplicationListener Bean，每当ApplicationContext发布ApplicationEvent时，ApplicationListener Bean将自动被触发。这种事件机制都必须需要程序显示的触发。

3.其中spring有一些内置的事件，当完成某种操作时会发出某些事件动作。比如监听ContextRefreshedEvent事件，当所有的bean都初始化完成并被成功装载后会触发该事件，实现ApplicationListener<ContextRefreshedEvent>接口可以收到监听动作，然后可以写自己的逻辑。

4.同样事件可以自定义、监听也可以自定义，完全根据自己的业务逻辑来处理。

之所以需要在这个地方注册监听器，是因为在spring启动的一些地方发布事件需要用到，需要优先加载（自己理解，这里需要子啊进行研究）

_ _ _

至此 `new SpringApplication()`已经分析完了，记下来分析下`run()`发方法内执行的逻辑，**敲重点**

这部分是源码文件，如果感觉看着头痛，可以看我第二段精简部分代码
```java
public ConfigurableApplicationContext run(String... args) {
    StopWatch stopWatch = new StopWatch();
    stopWatch.start();
    ConfigurableApplicationContext context = null;
    Collection<SpringBootExceptionReporter> exceptionReporters = new ArrayList<>();
    configureHeadlessProperty();
    SpringApplicationRunListeners listeners = getRunListeners(args);
    listeners.starting();
    try {
        ApplicationArguments applicationArguments = new DefaultApplicationArguments(
                args);
        ConfigurableEnvironment environment = prepareEnvironment(listeners,
                applicationArguments);
        configureIgnoreBeanInfo(environment);
        Banner printedBanner = printBanner(environment);
        context = createApplicationContext();
        exceptionReporters = getSpringFactoriesInstances(
                SpringBootExceptionReporter.class,
                new Class[] { ConfigurableApplicationContext.class }, context);
        prepareContext(context, environment, listeners, applicationArguments,
                printedBanner);
        refreshContext(context);
        afterRefresh(context, applicationArguments);
        stopWatch.stop();
        if (this.logStartupInfo) {
            new StartupInfoLogger(this.mainApplicationClass)
                    .logStarted(getApplicationLog(), stopWatch);
        }
        listeners.started(context);
        callRunners(context, applicationArguments);
    }
    catch (Throwable ex) {
        handleRunFailure(context, ex, exceptionReporters, listeners);
        throw new IllegalStateException(ex);
    }

    try {
        listeners.running(context);
    }
    catch (Throwable ex) {
        handleRunFailure(context, ex, exceptionReporters, null);
        throw new IllegalStateException(ex);
    }
    return context;
}

```

注意 以下这段代码是删掉了在Springboot启动过程中无关紧要的部分，希望同过此部分代码 分析启动的整个流程


```java
public ConfigurableApplicationContext run(String... args) {
    ConfigurableApplicationContext context = null;
    //1开启监听
    SpringApplicationRunListeners listeners = getRunListeners(args);
    listeners.starting();
    ApplicationArguments applicationArguments = new DefaultApplicationArguments(
            args);
    
    //2准备环境
    ConfigurableEnvironment environment = prepareEnvironment(listeners,
            applicationArguments);
    
    //3.创建上下文
    context = createApplicationContext();
    prepareContext(context, environment, listeners, applicationArguments,
            printedBanner);
    
    //4.刷新容器
    refreshContext(context);
    afterRefresh(context, applicationArguments);
    listeners.started(context);
    
    //5.启动完成，回调Runners接口
    callRunners(context, applicationArguments);
    listeners.running(context);
    return context;
}
```

整个过程可以分解为如上5部分，现在一个个分析。

1.开启监听
方法很直接，获取RunListeners,之后开启，相信之前有对`SpringFactoriesLoader.loadFactoryNames(type, classLoader))`
这个方法的分析的经验，这里对于获取RunListeners的方式不看源码大家都能分析出来，那么这里主要看下，这个接口内部主要做了什么
```java
public interface SpringApplicationRunListener {
    //开始
	void starting();	
	//环境准备完成
	void environmentPrepared(ConfigurableEnvironment environment);
	//上下文准备完成
	void contextPrepared(ConfigurableApplicationContext context);
	//上下文加载完成
	void contextLoaded(ConfigurableApplicationContext context);
	//启动完成
	void started(ConfigurableApplicationContext context);
	//运行
	void running(ConfigurableApplicationContext context);
	//失败
	void failed(ConfigurableApplicationContext context, Throwable exception);

}

```
可以看到，这个接口就是包含了SpringBoot的整齐启动过程的监听，触发时机最显著的就是获取完成之后的
listeners.starting();
跟进代码会发现，循环调用每一个监听器的start方法
```java
public void starting() {
    for (SpringApplicationRunListener listener : this.listeners) {
        listener.starting();
    }
}
```
再举例`prepareContext()`方法
```java
private void prepareContext(ConfigurableApplicationContext context,
			ConfigurableEnvironment environment, SpringApplicationRunListeners listeners,
			ApplicationArguments applicationArguments, Banner printedBanner) {
		context.setEnvironment(environment);
		postProcessApplicationContext(context);
		applyInitializers(context);
		listeners.contextPrepared(context);
		if (this.logStartupInfo) {
			logStartupInfo(context.getParent() == null);
			logStartupProfileInfo(context);
		}
		// Add boot specific singleton beans
		ConfigurableListableBeanFactory beanFactory = context.getBeanFactory();
		beanFactory.registerSingleton("springApplicationArguments", applicationArguments);
		if (printedBanner != null) {
			beanFactory.registerSingleton("springBootBanner", printedBanner);
		}
		if (beanFactory instanceof DefaultListableBeanFactory) {
			((DefaultListableBeanFactory) beanFactory)
					.setAllowBeanDefinitionOverriding(this.allowBeanDefinitionOverriding);
		}
		// Load the sources
		Set<Object> sources = getAllSources();
		Assert.notEmpty(sources, "Sources must not be empty");
		load(context, sources.toArray(new Object[0]));
		listeners.contextLoaded(context);
	}
```
在准备上下文之后，回调`listeners.contextLoaded(context);`监听器的方法，因此要监控SpringBoot的启动过程
就可以实现此接口。

2.prepareEnvironment创建环境，还未深入研究 ，此处不做分析

3.准备上下文
此处重点分析 ``context = createApplicationContext();``

可以看到，上下文环境是通过这个方法创建出来的
```java

protected ConfigurableApplicationContext createApplicationContext() {
    Class<?> contextClass = this.applicationContextClass;
    if (contextClass == null) {
        try {
            switch (this.webApplicationType) {
            case SERVLET:
                contextClass = Class.forName(DEFAULT_SERVLET_WEB_CONTEXT_CLASS);
                break;
            case REACTIVE:
                contextClass = Class.forName(DEFAULT_REACTIVE_WEB_CONTEXT_CLASS);
                break;
            default:
                contextClass = Class.forName(DEFAULT_CONTEXT_CLASS);
            }
        }
        catch (ClassNotFoundException ex) {
            throw new IllegalStateException(
                    "Unable create a default ApplicationContext, "
                            + "please specify an ApplicationContextClass",
                    ex);
        }
    }
    return (ConfigurableApplicationContext) BeanUtils.instantiateClass(contextClass);
}
```
这段代码的意思很明显，就是根据不同的上下类型去实例化不同的上下文。

前边①的地方提到，在容器刷新之前，会先执行初始化器的接口，就在prepareContext（）方法中的

```java
private void prepareContext(ConfigurableApplicationContext context,
        ConfigurableEnvironment environment, SpringApplicationRunListeners listeners,
        ApplicationArguments applicationArguments, Banner printedBanner) {
    context.setEnvironment(environment);
    postProcessApplicationContext(context);
    //回调初始化器方法
    applyInitializers(context);
    listeners.contextPrepared(context);
    if (this.logStartupInfo) {
        logStartupInfo(context.getParent() == null);
        logStartupProfileInfo(context);
    }
    // Add boot specific singleton beans
    ConfigurableListableBeanFactory beanFactory = context.getBeanFactory();
    beanFactory.registerSingleton("springApplicationArguments", applicationArguments);
    if (printedBanner != null) {
        beanFactory.registerSingleton("springBootBanner", printedBanner);
    }
    if (beanFactory instanceof DefaultListableBeanFactory) {
        ((DefaultListableBeanFactory) beanFactory)
                .setAllowBeanDefinitionOverriding(this.allowBeanDefinitionOverriding);
    }
    // Load the sources
    Set<Object> sources = getAllSources();
    Assert.notEmpty(sources, "Sources must not be empty");
    load(context, sources.toArray(new Object[0]));
    listeners.contextLoaded(context);
}
	
		
/**
* 回调初始化器方法
 */		
protected void applyInitializers(ConfigurableApplicationContext context) {
    for (ApplicationContextInitializer initializer : getInitializers()) {
        Class<?> requiredType = GenericTypeResolver.resolveTypeArgument(
                initializer.getClass(), ApplicationContextInitializer.class);
        Assert.isInstanceOf(requiredType, context, "Unable to call initializer.");
        initializer.initialize(context);
    }
}
```

5.刷新容器




















