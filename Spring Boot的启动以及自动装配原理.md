### Spring Boot的启动
Spring Boot的启动包含两部分：  
1. spring容器的启动refreshContext(context);
2. servlet容器的启动(tomacat的启动) refreshContext(context); -->  onRefresh();

##### 1. spring容器的启动
Spring Boot应用的整个启动流程都封装在SpringApplication.run方法中，本质上其实就是在spring的基础之上做了封装，做了大量的扩展。

- run方法
```java
public ConfigurableApplicationContext run(String... args) {
        StopWatch stopWatch = new StopWatch();
        stopWatch.start();
        ConfigurableApplicationContext context = null;
        Collection<SpringBootExceptionReporter> exceptionReporters = new ArrayList<>();
        configureHeadlessProperty();
		//1.通过SpringFactoriesLoader查找并加载所有的SpringApplicationRunListeners，通过调用
		//starting()方法通知所有的SpringApplicationRunListeners：应用开始启动了
        SpringApplicationRunListeners listeners = getRunListeners(args);
        listeners.starting();
        try {
			//2.创建并配置当前应用将要使用的Environment
            ApplicationArguments applicationArguments = new DefaultApplicationArguments(
                    args);
            ConfigurableEnvironment environment = prepareEnvironment(listeners,
                    applicationArguments);
            configureIgnoreBeanInfo(environment);
			//3.打印banner
            Banner printedBanner = printBanner(environment);
			//4.根据是否是web项目，来创建不同的ApplicationContext容器
            context = createApplicationContext();
			//5.创建一系列FailureAnalyzer
            exceptionReporters = getSpringFactoriesInstances(
                    SpringBootExceptionReporter.class,
                    new Class[] { ConfigurableApplicationContext.class }, context);
			//6.初始化ApplicationContext
            prepareContext(context, environment, listeners, applicationArguments,
                    printedBanner);
			//7.调用ApplicationContext的refresh()方法,刷新容器
            refreshContext(context);
			//8.查找当前context中是否注册有CommandLineRunner和ApplicationRunner，如果有则遍历执行它们。
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
            handleRunFailure(context, listeners, exceptionReporters, ex);
            throw new IllegalStateException(ex);
        }
        listeners.running(context);
        return context;
    }
```
##### 核心方法分析

1. 通过SpringFactoriesLoader查找并加载所有的SpringApplicationRunListeners(spi机制)，通过调用starting()方法通知所有的SpringApplicationRunListeners：应用开始启动了。  
（SpringApplicationRunListeners其本质上就是一个事件发布者，它在SpringBoot应用启动的不同时间点发布不同应用事件类型(ApplicationEvent)，如果有哪些事件监听者(ApplicationListener)对这些事件感兴趣，则可以接收并且处理）  
**SpringApplicationRunListener只有一个实现类：EventPublishingRunListener。**

2. 创建并配置当前应用将要使用的Environment，Environment用于描述应用程序当前的运行环境，其抽象了两个方面的内容：配置文件(profile)和属性(properties)，不同的环境(eg：生产环境、预发布环境)可以使用不同的配置文件，而属性则可以从配置文件、环境变量、命令行参数等来源获取。因此，当Environment准备好后，在整个应用的任何时候，都可以从Environment中获取资源。  
```java
  判断Environment是否存在，不存在就创建（如果是web项目就创建StandardServletEnvironment，否则创建StandardEnvironment）  
  配置Environment：配置profile以及properties  
  调用SpringApplicationRunListener的environmentPrepared()方法，通知事件监听者：应用的Environment已经准备好 
```
3. 打印banner（可以自定义）  
4. 根据是否是web项目，来创建不同的ApplicationContext容器  
5. 创建一系列FailureAnalyzer，创建流程依然是通过SpringFactoriesLoader获取到所有实现FailureAnalyzer接口的class，然后在创建对应的实例。FailureAnalyzer用于分析故障并提供相关诊断信息。
6. 初始化ApplicationContext  
```shell
将准备好的Environment设置给ApplicationContext  
遍历调用所有的ApplicationContextInitializer的initialize()方法来对已经创建好的ApplicationContext进行进一步的处理  
调用SpringApplicationRunListener的contextPrepared()方法，通知所有的监听者：ApplicationContext已经准备完毕
将所有的bean加载到容器中
调用SpringApplicationRunListener的contextLoaded()方法，通知所有的监听者：ApplicationContext已经装载完毕
```
7. 调用ApplicationContext的refresh()方法,刷新容器  

这里的刷新和spring中刷新原理类似，这里重点关注invokeBeanFactoryPostProcessors(beanFactory);方法，主要完成获取到所有的BeanFactoryPostProcessor来对容器做一些额外的操作，通过源可以进入到PostProcessorRegistrationDelegate类的invokeBeanFactoryPostProcessors()方法，会获取类型为BeanDefinitionRegistryPostProcessor的beanorg.springframework.context.annotation.internalConfigurationAnnotationProcessor，对应的Class为ConfigurationClassPostProcessor。ConfigurationClassPostProcessor用于解析处理各种注解，包括：@Configuration、@ComponentScan、@Import、@PropertySource、@ImportResource、@Bean。当处理@import注解的时候，就会调用EnableAutoConfigurationImportSelector.selectImports()来完成自动配置功能  
	
tomcat的启动的地方也是在refresh方法中，具体就是onRefresh();方法  
	
8. 查找当前context中是否注册有CommandLineRunner和ApplicationRunner，如果有则遍历执行它们。

- SpringApplicationRunListener接口
```java 
public interface SpringApplicationRunListener {

     //刚执行run方法时
    void started();
     //环境建立好时候
    void environmentPrepared(ConfigurableEnvironment environment);
     //上下文建立好的时候
    void contextPrepared(ConfigurableApplicationContext context);
    //上下文载入配置时候
    void contextLoaded(ConfigurableApplicationContext context);
    //上下文刷新完成后，run方法执行完之前
    void finished(ConfigurableApplicationContext context, Throwable exception);

}

```
- SpringApplicationRunListener接口具体使用
新建类实现SpringApplicationRunListener,需要构造方法,里面两个参数SpringApplication sa, String[] args;
在resources下新建META-INF\spring.factories文件,文件里面将新建的实现类的类路径配置进去:
`org.springframework.boot.SpringApplicationRunListener=com.study.springbootplus.config.MyApplicationRunListener`
```java 

public class MyApplicationRunListener implements SpringApplicationRunListener {

  private final SpringApplication application;
  private final String[] args;

  public MyApplicationRunListener(SpringApplication sa, String[] args) {
    this.application = sa;
    this.args = args;
  }

  @Override
  public void starting() {
    System.out.println("服务启动RunnerTest  SpringApplicationRunListener的starting方法...");
  }

  @Override
  public void environmentPrepared(ConfigurableEnvironment environment) {
    System.out.println("服务启动RunnerTest  SpringApplicationRunListener的environmentPrepared方法...");
  }

  @Override
  public void contextPrepared(ConfigurableApplicationContext context) {
    System.out.println("服务启动RunnerTest  SpringApplicationRunListener的contextPrepared方法...");
  }

  @Override
  public void contextLoaded(ConfigurableApplicationContext context) {
    System.out.println("服务启动RunnerTest  SpringApplicationRunListener的contextLoaded方法...");
  }

  @Override
  public void running(ConfigurableApplicationContext context) {
    System.out.println("服务启动RunnerTest  SpringApplicationRunListener的running方法...");
  }

  @Override
  public void failed(ConfigurableApplicationContext context, Throwable exception) {
    System.out.println("服务启动RunnerTest  SpringApplicationRunListener的failed方法...");
  }

  @Override
  public void started(ConfigurableApplicationContext context) {
    System.out.println("服务启动RunnerTest  SpringApplicationRunListener的started方法...");
  }
}

```

**注意事项**
任何一个SpringApplicationRunListener实现类的构造方法都需要有两个构造参数;源码里面分析了,需要根据指定类型的构造方法初始化类;
SpringApplicationRunListener的配置在resources下新建META-INF\spring.factories文件,文件里面将新建的实现类的类路径配置进去,对应的key是org.springframework.boot.SpringApplicationRunListener;  

SpringApplicationRunListener属于应用程序启动层面的监听器,在springboot启动时候,调用run方法进行反射加载初始化。此时上下文还没有加载，如果通过@Compnant是起不了作用的.  

- createApplicationContext
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
					"Unable create a default ApplicationContext, " + "please specify an ApplicationContextClass",
					ex);
		}
	}
	return (ConfigurableApplicationContext) BeanUtils.instantiateClass(contextClass);
}
```



- tomacat的启动
tomcat的启动是在spring中调用refresh方法中进行的，具体会调用onRefresh()方法。这里做一个简单的了解
```java 
@Override
	public void refresh() throws BeansException, IllegalStateException {
		synchronized (this.startupShutdownMonitor) {
			// Prepare this context for refreshing.
			prepareRefresh();

			// Tell the subclass to refresh the internal bean factory.
			ConfigurableListableBeanFactory beanFactory = obtainFreshBeanFactory();

			// Prepare the bean factory for use in this context.
			prepareBeanFactory(beanFactory);

			try {
				// Allows post-processing of the bean factory in context subclasses.
				postProcessBeanFactory(beanFactory);

				// Invoke factory processors registered as beans in the context.
				invokeBeanFactoryPostProcessors(beanFactory);

				// Register bean processors that intercept bean creation.
				registerBeanPostProcessors(beanFactory);

				// Initialize message source for this context.
				initMessageSource();

				// Initialize event multicaster for this context.
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
- ServletWebServerApplicationContext
```java	
@Override
protected void onRefresh() {
	super.onRefresh();
	try {
  //创建webserver容器
		createWebServer();
	}
	catch (Throwable ex) {
		throw new ApplicationContextException("Unable to start web server", ex);
	}
}


private void createWebServer() {
	WebServer webServer = this.webServer;
	ServletContext servletContext = getServletContext();
	if (webServer == null && servletContext == null) {
  //工厂模式 得到一个创建服务的工厂
		ServletWebServerFactory factory = getWebServerFactory();
    //创建一个服务器我们这里举例tomcat 看下方详情代码
		this.webServer = factory.getWebServer(getSelfInitializer());
	}
	else if (servletContext != null) {
		try {
			getSelfInitializer().onStartup(servletContext);
		}
		catch (ServletException ex) {
			throw new ApplicationContextException("Cannot initialize servlet context", ex);
		}
	}
	initPropertySources();
}

@Override
public WebServer getWebServer(ServletContextInitializer... initializers) {
  //创建Tomcat容器
  Tomcat tomcat = new Tomcat();
  File baseDir = (this.baseDirectory != null) ? this.baseDirectory : createTempDir("tomcat");
  tomcat.setBaseDir(baseDir.getAbsolutePath());
  //创建连接器，默认NIO模式，可以通过WebServerFactoryCustomizer改变具体模式
  Connector connector = new Connector(this.protocol);
  tomcat.getService().addConnector(connector);
  //自定义连接器
  customizeConnector(connector);
  tomcat.setConnector(connector);
  tomcat.getHost().setAutoDeploy(false);
  configureEngine(tomcat.getEngine());
  //可以通过WebServerFactoryCustomizer添加额外的连接器，这边将这些连接器绑定到Tomcat
  for (Connector additionalConnector : this.additionalTomcatConnectors) {
    tomcat.getService().addConnector(additionalConnector);
  }
  //组测Listener、Filter和Servlet，自定义Context等操作
    //这个方法可以重点看下
  prepareContext(tomcat.getHost(), initializers);
  //创建TomcatWebServer，并调用start方法
  return getTomcatWebServer(tomcat);
}

public TomcatWebServer(Tomcat tomcat, boolean autoStart) {
  Assert.notNull(tomcat, "Tomcat Server must not be null");
  this.tomcat = tomcat;
  this.autoStart = autoStart;
  //触发Tomcat的启动流程
  initialize();
}
```

### springboot的自动配置原理  
**理念：约定大于配置！**

```java 
@SpringBootApplication
public class Application {
    public static void main(String[] args) {
        SpringApplication.run(Application.class, args);
    }
}
```
从上面代码可以看出，Annotation定义（@SpringBootApplication）和类定义（SpringApplication.run）最为耀眼，所以要揭开SpringBoot的神秘面纱，我们要从这两位开始就可以了。

```java 
@Target(ElementType.TYPE)            // 注解的适用范围，其中TYPE用于描述类、接口（包括包注解类型）或enum声明
@Retention(RetentionPolicy.RUNTIME)  // 注解的生命周期，保留到class文件中（三个生命周期）
@Documented                          // 表明这个注解应该被javadoc记录
@Inherited                           // 子类可以继承该注解
@SpringBootConfiguration             // 继承了Configuration，表示当前是注解类
@EnableAutoConfiguration             // 开启springboot的注解功能，springboot的四大神器之一，其借助@import的帮助
@ComponentScan(excludeFilters = {    // 扫描路径设置（具体使用待确认）
        @Filter(type = FilterType.CUSTOM, classes = TypeExcludeFilter.class),
        @Filter(type = FilterType.CUSTOM, classes = AutoConfigurationExcludeFilter.class) })
public @interface SpringBootApplication {
...
)
```
虽然定义使用了多个Annotation进行了原信息标注，但实际上重要的只有三个Annotation：
```java 
@Configuration
@EnableAutoConfiguration
@ComponentScan
```

- @Configuration（@SpringBootConfiguration点开查看发现里面还是应用了@Configuration）  
从Spring3.0，@Configuration用于定义配置类，可替换xml配置文件，被注解的类内部包含有一个或多个被@Bean注解的方法，这些方法将会被AnnotationConfigApplicationContext或AnnotationConfigWebApplicationContext类进行扫描，并用于构建bean定义，初始化Spring容器。  

@Configuation等价于<Beans></Beans>,这里的启动类标注了@Configuration之后，本身其实也是一个IoC容器的配置类。  

- @ComponentScan
@ComponentScan这个注解在Spring中很重要，它对应XML配置中的元素，@ComponentScan的功能其实就是自动扫描并加载符合条件的组件（比如@Component和@Repository等）或者bean定义，最终将这些bean定义加载到IoC容器中。  

我们可以通过basePackages等属性来细粒度的定制@ComponentScan自动扫描的范围，如果不指定，则默认Spring框架实现会从声明@ComponentScan所在类的package进行扫描。  

注：所以SpringBoot的启动类最好是放在root package下，因为默认不指定basePackages。  

- @EnableAutoConfiguration
@EnableAutoConfiguration是借助@Import的帮助，将所有符合自动配置条件的bean定义加载到IoC容器，仅此而已！  

@EnableAutoConfiguration作为一个复合Annotation,其自身定义关键信息如下：  
```java 
@SuppressWarnings("deprecation")
@Target(ElementType.TYPE)
@Retention(RetentionPolicy.RUNTIME)
@Documented
@Inherited
@AutoConfigurationPackage
@Import(EnableAutoConfigurationImportSelector.class)
public @interface EnableAutoConfiguration {
    ...
}
```
两个比较重要的注解：
@AutoConfigurationPackage：自动配置包
@Import: 导入自动配置的组件

- AutoConfigurationPackage注解：
```java 
static class Registrar implements ImportBeanDefinitionRegistrar, DeterminableImports {

	@Override
	public void registerBeanDefinitions(AnnotationMetadata metadata, BeanDefinitionRegistry registry) {
		register(registry, new PackageImport(metadata).getPackageName());
	}

	@Override
	public Set<Object> determineImports(AnnotationMetadata metadata) {
		return Collections.singleton(new PackageImport(metadata));
	}

}
```
它其实是注册了一个Bean的定义。
new PackageImport(metadata).getPackageName()，它其实返回了当前主程序类的同级以及子级的包组件。

- Import(AutoConfigurationImportSelector.class)注解：
AutoConfigurationImportSelector 实现了 DeferredImportSelector接口， 该接口继承了 ImportSelector接口;
ImportSelector有一个方法为：selectImports。

DeferredImportSelector该接口是ImportSelector接口的一个子接口，那么它是如何使用的呢？我们来看看案例，我们通过@Import注解把需要实例化的类加载到spring容器中如何实现：
```java
/**
 * 需要实例化的类
 */
public class DeferredBean {
}

/**
 * @descript 自定义类实现DeferredImportSelector接口
 */
public class DeferredImportSelectorDemo implements DeferredImportSelector {

    @Override
    public String[] selectImports(AnnotationMetadata importingClassMetadata) {
        System.out.println("=====DeferredImportSelectorDemo.selectImports");
        return new String[]{DeferredBean.class.getName()};
    }

    /**
     * 返回了一个实现了Group接口的类
     * @return
     */
    @Override
    public Class<? extends Group> getImportGroup() {
        return DeferredImportSelectorGroupDemo.class;
    }

    private static class DeferredImportSelectorGroupDemo implements Group {

        List<Entry> list = new ArrayList<>();

        /**
         * 收集需要实例化的类
         * @param metadata
         * @param selector
         */
        @Override
        public void process(AnnotationMetadata metadata, DeferredImportSelector selector) {
            System.out.println("=====DeferredImportSelectorGroupDemo.process");
            String[] strings = selector.selectImports(metadata);
            for (String string : strings) {
                list.add(new Entry(metadata,string));
            }
        }

        /**把收集的类返回给spring 容器
         *
         * @return
         */
        @Override
        public Iterable<Entry> selectImports() {
            System.out.println("=====DeferredImportSelectorGroupDemo.selectImports");
            return list;
        }
    }
}

@Component
//Import虽然是实例化一个类，Import进来的类可以实现一些接口
@Import({DeferredImportSelectorDemo.class})
public class ImportBean {
}
```
通过上面的代码当我们启动spring boot项目时就可以把DeferredBean实例对象加载到spring容器中，所以spring boot也是通过这个机制实现的，接下来我们就分析spring boot是如何实现的。

- AutoConfigurationImportSelector
```java 
@Override
public String[] selectImports(AnnotationMetadata annotationMetadata) {
	if (!isEnabled(annotationMetadata)) {
		return NO_IMPORTS;
	}
	AutoConfigurationMetadata autoConfigurationMetadata = AutoConfigurationMetadataLoader
			.loadMetadata(this.beanClassLoader);
	AutoConfigurationEntry autoConfigurationEntry = getAutoConfigurationEntry(autoConfigurationMetadata,
			annotationMetadata);
	return StringUtils.toStringArray(autoConfigurationEntry.getConfigurations());
}
	
/**
 * Return the {@link AutoConfigurationEntry} based on the {@link AnnotationMetadata}
 * of the importing {@link Configuration @Configuration} class.
 * @param autoConfigurationMetadata the auto-configuration metadata
 * @param annotationMetadata the annotation metadata of the configuration class
 * @return the auto-configurations that should be imported
 */
protected AutoConfigurationEntry getAutoConfigurationEntry(AutoConfigurationMetadata autoConfigurationMetadata,
		AnnotationMetadata annotationMetadata) {
	if (!isEnabled(annotationMetadata)) {
		return EMPTY_ENTRY;
	}
	//获取@SpringBootApplication的配置属性 主类中的配置信息，attributes对象中包含了需要排除的类
	AnnotationAttributes attributes = getAttributes(annotationMetadata);
	//获取候选的所有类的名称，通过spi机制进行类的加载，会加载"META-INF/spring.factories" 文件中 org.springframework.boot.autoconfigure.EnableAutoConfiguration 作为查找的Key,获取对应的一组@Configuration类 如下图所示
	List<String> configurations = getCandidateConfigurations(annotationMetadata, attributes);
	configurations = removeDuplicates(configurations);
	Set<String> exclusions = getExclusions(annotationMetadata, attributes);
	checkExcludedClasses(configurations, exclusions);
	//从候选的类中删除需要排除的类
	configurations.removeAll(exclusions);
	//SPI的扩展，获取过滤器实例对候选的类过滤
	configurations = filter(configurations, autoConfigurationMetadata);
	fireAutoConfigurationImportEvents(configurations, exclusions);
	//把候选的所有类包装成AutoConfigurationEntry对象
	return new AutoConfigurationEntry(configurations, exclusions);
}

protected List<String> getCandidateConfigurations(AnnotationMetadata metadata, AnnotationAttributes attributes) {
	//SPI的方式获取EnableAutoConfiguration.class类型的所有类的名称
	List<String> configurations = SpringFactoriesLoader.loadFactoryNames(getSpringFactoriesLoaderFactoryClass(),
			getBeanClassLoader());
	Assert.notEmpty(configurations, "No auto configuration classes found in META-INF/spring.factories. If you "
			+ "are using a custom packaging, make sure that file is correct.");
	return configurations;
}
```
因为该类AutoConfigurationImportSelector实现了DeferredImportSelector接口，所以启动时就会调用到AutoConfigurationGroup类的方法
```java 
@Override
public Class<? extends Group> getImportGroup() {
	return AutoConfigurationGroup.class;
}
```
看到这里和我们写的案例是不是有点类似了，肯定在AutoConfigurationImportSelector这个类中实现了一个内部类AutoConfigurationGroup，接下来我们看该类，
不出所料确实存在！
- AutoConfigurationGroup
AutoConfigurationGroup该类又实现了Group接口
```java 

//该方法收集需要实例化的方法
@Override
public void process(AnnotationMetadata annotationMetadata, DeferredImportSelector deferredImportSelector) {
	Assert.state(deferredImportSelector instanceof AutoConfigurationImportSelector,
			() -> String.format("Only %s implementations are supported, got %s",
					AutoConfigurationImportSelector.class.getSimpleName(),
					deferredImportSelector.getClass().getName()));
	//核心代码，SPI的方式获取需要实例化的类。因为在getAutoConfigurationEntry方法中返回的就是AutoConfigurationEntry对象，这里所以进行了转化，AutoConfigurationEntry类中包装了所有需要实例化的类的集合
	AutoConfigurationEntry autoConfigurationEntry = ((AutoConfigurationImportSelector) deferredImportSelector)
			.getAutoConfigurationEntry(getAutoConfigurationMetadata(), annotationMetadata);
	this.autoConfigurationEntries.add(autoConfigurationEntry);
	//循环所有需要实例化的类，并建立类和注解的映射关系
	for (String importClassName : autoConfigurationEntry.getConfigurations()) {
		this.entries.putIfAbsent(importClassName, annotationMetadata);
	}
}

@Override
public Iterable<Entry> selectImports() {
	if (this.autoConfigurationEntries.isEmpty()) {
		return Collections.emptyList();
	}
	//获取需要排除类的集合
	Set<String> allExclusions = this.autoConfigurationEntries.stream()
			.map(AutoConfigurationEntry::getExclusions).flatMap(Collection::stream).collect(Collectors.toSet());
			
	//获取所有需要实例化的类的集合
	Set<String> processedConfigurations = this.autoConfigurationEntries.stream()
			.map(AutoConfigurationEntry::getConfigurations).flatMap(Collection::stream)
			.collect(Collectors.toCollection(LinkedHashSet::new));
	//删除需要排除的类
	processedConfigurations.removeAll(allExclusions);
	//把需要实例化的类包装成Entry的集合，必须这么写，spring要根据entry实例化对象
	return sortAutoConfigurations(processedConfigurations, getAutoConfigurationMetadata()).stream()
			.map((importClassName) -> new Entry(this.entries.get(importClassName), importClassName))
			.collect(Collectors.toList());
}

```
