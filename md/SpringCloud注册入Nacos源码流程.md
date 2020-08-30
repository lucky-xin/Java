## AnnotationConfigServletWebServerApplication 调用refrash方法进行初始化
```java
    /**
	 * Create a new {@link AnnotationConfigServletWebServerApplicationContext} that needs
	 * to be populated through {@link #register} calls and then manually
	 * {@linkplain #refresh refreshed}.
	 */
	public AnnotationConfigServletWebServerApplicationContext() {
		this.reader = new AnnotatedBeanDefinitionReader(this);
		this.scanner = new ClassPathBeanDefinitionScanner(this);
	}

	/**
	 * Create a new {@link AnnotationConfigServletWebServerApplicationContext} with the
	 * given {@code DefaultListableBeanFactory}. The context needs to be populated through
	 * {@link #register} calls and then manually {@linkplain #refresh refreshed}.
	 * @param beanFactory the DefaultListableBeanFactory instance to use for this context
	 */
	public AnnotationConfigServletWebServerApplicationContext(DefaultListableBeanFactory beanFactory) {
		super(beanFactory);
		this.reader = new AnnotatedBeanDefinitionReader(this);
		this.scanner = new ClassPathBeanDefinitionScanner(this);
	}

	/**
	 * Create a new {@link AnnotationConfigServletWebServerApplicationContext}, deriving
	 * bean definitions from the given annotated classes and automatically refreshing the
	 * context.
	 * @param annotatedClasses one or more annotated classes, e.g. {@code @Configuration}
	 * classes
	 */
	public AnnotationConfigServletWebServerApplicationContext(Class<?>... annotatedClasses) {
		this();
		register(annotatedClasses);
		refresh();
	}

	/**
	 * Create a new {@link AnnotationConfigServletWebServerApplicationContext}, scanning
	 * for bean definitions in the given packages and automatically refreshing the
	 * context.
	 * @param basePackages the packages to check for annotated classes
	 */
	public AnnotationConfigServletWebServerApplicationContext(String... basePackages) {
		this();
		scan(basePackages);
		refresh();
	}
```

## 初始化完毕之后调用finishRefresh方法发布WebServerInitializedEvent事件
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
				// 创建WebServerStartStopLifecycle 入口
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
## onRefresh方法在AnnotationConfigServletWebServerApplication的父类ServletWebServerApplicationContext之中，向BeanFactory之中注册WebServerStartStopLifecycle
## 之后从BeanFactory之中获取
```java
    @Override
	protected void onRefresh() {
		super.onRefresh();
		try {
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
			ServletWebServerFactory factory = getWebServerFactory();
			this.webServer = factory.getWebServer(getSelfInitializer());
			getBeanFactory().registerSingleton("webServerGracefulShutdown",
					new WebServerGracefulShutdownLifecycle(this.webServer));
			getBeanFactory().registerSingleton("webServerStartStop",
					new WebServerStartStopLifecycle(this, this.webServer));
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
```

```java
    /**
	 * Finish the refresh of this context, invoking the LifecycleProcessor's
	 * onRefresh() method and publishing the
	 * {@link org.springframework.context.event.ContextRefreshedEvent}.
	 */
	protected void finishRefresh() {
		// Clear context-level resource caches (such as ASM metadata from scanning).
		clearResourceCaches();

		// Initialize lifecycle processor for this context.
		//创建DefaultLifecycleProcessor
		initLifecycleProcessor();

		// Propagate refresh to lifecycle processor first.
		// 创建LifecycleGroup并调用LifecycleGroup的start方法
		getLifecycleProcessor().onRefresh();

		// Publish the final event.
		publishEvent(new ContextRefreshedEvent(this));

		// Participate in LiveBeansView MBean, if active.
		LiveBeansView.registerApplicationContext(this);
	}
```
## 创建DefaultLifecycleProcessor并注册入BeanFactory之中，之后从BeanFactory获取并调用start方法
```java
	/**
	 * Initialize the LifecycleProcessor.
	 * Uses DefaultLifecycleProcessor if none defined in the context.
	 * @see org.springframework.context.support.DefaultLifecycleProcessor
	 */
	protected void initLifecycleProcessor() {
		ConfigurableListableBeanFactory beanFactory = getBeanFactory();
		if (beanFactory.containsLocalBean(LIFECYCLE_PROCESSOR_BEAN_NAME)) {
			this.lifecycleProcessor =
					beanFactory.getBean(LIFECYCLE_PROCESSOR_BEAN_NAME, LifecycleProcessor.class);
			if (logger.isTraceEnabled()) {
				logger.trace("Using LifecycleProcessor [" + this.lifecycleProcessor + "]");
			}
		}
		else {
			DefaultLifecycleProcessor defaultProcessor = new DefaultLifecycleProcessor();
			defaultProcessor.setBeanFactory(beanFactory);
			this.lifecycleProcessor = defaultProcessor;
			beanFactory.registerSingleton(LIFECYCLE_PROCESSOR_BEAN_NAME, this.lifecycleProcessor);
			if (logger.isTraceEnabled()) {
				logger.trace("No '" + LIFECYCLE_PROCESSOR_BEAN_NAME + "' bean, using " +
						"[" + this.lifecycleProcessor.getClass().getSimpleName() + "]");
			}
		}
	}
```
## DefaultLifecycleProcessor的onRefresh方法中从BeanFactory获取所有实现Lifecycle接口的Bean,会获取到之前注册的WebServerStartStopLifecycle,创建LifecycleGroup并调用LifecycleGroup的start方法
```java
    @Override
	public void onRefresh() {
		startBeans(true);
		this.running = true;
	}

	// Internal helpers

	private void startBeans(boolean autoStartupOnly) {
		Map<String, Lifecycle> lifecycleBeans = getLifecycleBeans();
		Map<Integer, LifecycleGroup> phases = new HashMap<>();
		lifecycleBeans.forEach((beanName, bean) -> {
			if (!autoStartupOnly || (bean instanceof SmartLifecycle && ((SmartLifecycle) bean).isAutoStartup())) {
				int phase = getPhase(bean);
				LifecycleGroup group = phases.get(phase);
				if (group == null) {
					group = new LifecycleGroup(phase, this.timeoutPerShutdownPhase, lifecycleBeans, autoStartupOnly);
					phases.put(phase, group);
				}
				group.add(beanName, bean);
			}
		});
		if (!phases.isEmpty()) {
			List<Integer> keys = new ArrayList<>(phases.keySet());
			Collections.sort(keys);
			for (Integer key : keys) {
				phases.get(key).start();
			}
		}
	}
	
	/**
	 * Retrieve all applicable Lifecycle beans: all singletons that have already been created,
	 * as well as all SmartLifecycle beans (even if they are marked as lazy-init).
	 * @return the Map of applicable beans, with bean names as keys and bean instances as values
	 */
	protected Map<String, Lifecycle> getLifecycleBeans() {
		ConfigurableListableBeanFactory beanFactory = getBeanFactory();
		Map<String, Lifecycle> beans = new LinkedHashMap<>();
		String[] beanNames = beanFactory.getBeanNamesForType(Lifecycle.class, false, false);
		for (String beanName : beanNames) {
			String beanNameToRegister = BeanFactoryUtils.transformedBeanName(beanName);
			boolean isFactoryBean = beanFactory.isFactoryBean(beanNameToRegister);
			String beanNameToCheck = (isFactoryBean ? BeanFactory.FACTORY_BEAN_PREFIX + beanName : beanName);
			if ((beanFactory.containsSingleton(beanNameToRegister) &&
					(!isFactoryBean || matchesBeanType(Lifecycle.class, beanNameToCheck, beanFactory))) ||
					matchesBeanType(SmartLifecycle.class, beanNameToCheck, beanFactory)) {
				Object bean = beanFactory.getBean(beanNameToCheck);
				if (bean != this && bean instanceof Lifecycle) {
					beans.put(beanNameToRegister, (Lifecycle) bean);
				}
			}
		}
		return beans;
	}

```
## LifecycleGroup的start方法，从BeanFactory这种获取之前注册的WebServerStartStopLifecycle并调用DefaultLifecycleProcessor的start方法
```java
    public void start() {
			if (this.members.isEmpty()) {
				return;
			}
			if (logger.isDebugEnabled()) {
				logger.debug("Starting beans in phase " + this.phase);
			}
			Collections.sort(this.members);
			for (LifecycleGroupMember member : this.members) {
				doStart(this.lifecycleBeans, member.name, this.autoStartupOnly);
			}
	}
	
	private void doStart(Map<String, ? extends Lifecycle> lifecycleBeans, String beanName, boolean autoStartupOnly) {
		Lifecycle bean = lifecycleBeans.remove(beanName);
		if (bean != null && bean != this) {
			String[] dependenciesForBean = getBeanFactory().getDependenciesForBean(beanName);
			for (String dependency : dependenciesForBean) {
				doStart(lifecycleBeans, dependency, autoStartupOnly);
			}
			if (!bean.isRunning() &&
					(!autoStartupOnly || !(bean instanceof SmartLifecycle) || ((SmartLifecycle) bean).isAutoStartup())) {
				if (logger.isTraceEnabled()) {
					logger.trace("Starting bean '" + beanName + "' of type [" + bean.getClass().getName() + "]");
				}
				try {
					bean.start();
				}
				catch (Throwable ex) {
					throw new ApplicationContextException("Failed to start bean '" + beanName + "'", ex);
				}
				if (logger.isDebugEnabled()) {
					logger.debug("Successfully started bean '" + beanName + "'");
				}
			}
		}
	}
		
```
## WebServerStartStopLifecycle的start方法之中发布ServletWebServerInitializedEvent
```java
package org.springframework.boot.web.servlet.context;

import org.springframework.boot.web.server.WebServer;
import org.springframework.context.SmartLifecycle;

/**
 * {@link SmartLifecycle} to start and stop the {@link WebServer} in a
 * {@link ServletWebServerApplicationContext}.
 *
 * @author Andy Wilkinson
 */
class WebServerStartStopLifecycle implements SmartLifecycle {

	private final ServletWebServerApplicationContext applicationContext;

	private final WebServer webServer;

	private volatile boolean running;

	WebServerStartStopLifecycle(ServletWebServerApplicationContext applicationContext, WebServer webServer) {
		this.applicationContext = applicationContext;
		this.webServer = webServer;
	}

	@Override
	public void start() {
		this.webServer.start();
		this.running = true;
		this.applicationContext
				.publishEvent(new ServletWebServerInitializedEvent(this.webServer, this.applicationContext));
	}

	@Override
	public void stop() {
		this.webServer.stop();
	}

	@Override
	public boolean isRunning() {
		return this.running;
	}

	@Override
	public int getPhase() {
		return Integer.MAX_VALUE - 1;
	}

}

```

```text
SpringBoot启动完毕发布了ServletWebServerInitializedEvent事件，告知已经初始化完毕，spring-cloud-commons模块
之中的抽象类org.springframework.cloud.client.serviceregistry.AbstractAutoServiceRegistration监听了WebServerInitializedEvent事件。
spring-cloud-starter-alibaba-nacos-discovery模块中NacosAutoServiceRegistration则继承了NacosAutoServiceRegistration类，
并实现自动注册SpringCloud服务
```
## AbstractAutoServiceRegistration 源码
```java
public abstract class AbstractAutoServiceRegistration<R extends Registration>
		implements AutoServiceRegistration, ApplicationContextAware,
		ApplicationListener<WebServerInitializedEvent> {

	private static final Log logger = LogFactory
			.getLog(AbstractAutoServiceRegistration.class);

	private final ServiceRegistry<R> serviceRegistry;

	private boolean autoStartup = true;

	private AtomicBoolean running = new AtomicBoolean(false);

	private int order = 0;

	private ApplicationContext context;

	private Environment environment;

	private AtomicInteger port = new AtomicInteger(0);

	private AutoServiceRegistrationProperties properties;

	@Deprecated
	protected AbstractAutoServiceRegistration(ServiceRegistry<R> serviceRegistry) {
		this.serviceRegistry = serviceRegistry;
	}

	protected AbstractAutoServiceRegistration(ServiceRegistry<R> serviceRegistry,
			AutoServiceRegistrationProperties properties) {
		this.serviceRegistry = serviceRegistry;
		this.properties = properties;
	}

	protected ApplicationContext getContext() {
		return this.context;
	}

	@Override
	@SuppressWarnings("deprecation")
	public void onApplicationEvent(WebServerInitializedEvent event) {
		bind(event);
	}

	@Deprecated
	public void bind(WebServerInitializedEvent event) {
		ApplicationContext context = event.getApplicationContext();
		if (context instanceof ConfigurableWebServerApplicationContext) {
			if ("management".equals(((ConfigurableWebServerApplicationContext) context)
					.getServerNamespace())) {
				return;
			}
		}
		this.port.compareAndSet(0, event.getWebServer().getPort());
		this.start();
	}

	@Override
	public void setApplicationContext(ApplicationContext applicationContext)
			throws BeansException {
		this.context = applicationContext;
		this.environment = this.context.getEnvironment();
	}

	@Deprecated
	protected Environment getEnvironment() {
		return this.environment;
	}

	@Deprecated
	protected AtomicInteger getPort() {
		return this.port;
	}

	public boolean isAutoStartup() {
		return this.autoStartup;
	}

	public void start() {
		if (!isEnabled()) {
			if (logger.isDebugEnabled()) {
				logger.debug("Discovery Lifecycle disabled. Not starting");
			}
			return;
		}

		// only initialize if nonSecurePort is greater than 0 and it isn't already running
		// because of containerPortInitializer below
		if (!this.running.get()) {
			this.context.publishEvent(
					new InstancePreRegisteredEvent(this, getRegistration()));
			register();
			if (shouldRegisterManagement()) {
				registerManagement();
			}
			this.context.publishEvent(
					new InstanceRegisteredEvent<>(this, getConfiguration()));
			this.running.compareAndSet(false, true);
		}

	}

	/**
	 * @return Whether the management service should be registered with the
	 * {@link ServiceRegistry}.
	 */
	protected boolean shouldRegisterManagement() {
		if (this.properties == null || this.properties.isRegisterManagement()) {
			return getManagementPort() != null
					&& ManagementServerPortUtils.isDifferent(this.context);
		}
		return false;
	}

	/**
	 * @return The object used to configure the registration.
	 */
	@Deprecated
	protected abstract Object getConfiguration();

	/**
	 * @return True, if this is enabled.
	 */
	protected abstract boolean isEnabled();

	/**
	 * @return The serviceId of the Management Service.
	 */
	@Deprecated
	protected String getManagementServiceId() {
		// TODO: configurable management suffix
		return this.context.getId() + ":management";
	}

	/**
	 * @return The service name of the Management Service.
	 */
	@Deprecated
	protected String getManagementServiceName() {
		// TODO: configurable management suffix
		return getAppName() + ":management";
	}

	/**
	 * @return The management server port.
	 */
	@Deprecated
	protected Integer getManagementPort() {
		return ManagementServerPortUtils.getPort(this.context);
	}

	/**
	 * @return The app name (currently the spring.application.name property).
	 */
	@Deprecated
	protected String getAppName() {
		return this.environment.getProperty("spring.application.name", "application");
	}

	@PreDestroy
	public void destroy() {
		stop();
	}

	public boolean isRunning() {
		return this.running.get();
	}

	protected AtomicBoolean getRunning() {
		return this.running;
	}

	public int getOrder() {
		return this.order;
	}

	public int getPhase() {
		return 0;
	}

	protected ServiceRegistry<R> getServiceRegistry() {
		return this.serviceRegistry;
	}

	protected abstract R getRegistration();

	protected abstract R getManagementRegistration();

	/**
	 * Register the local service with the {@link ServiceRegistry}.
	 */
	protected void register() {
		this.serviceRegistry.register(getRegistration());
	}

	/**
	 * Register the local management service with the {@link ServiceRegistry}.
	 */
	protected void registerManagement() {
		R registration = getManagementRegistration();
		if (registration != null) {
			this.serviceRegistry.register(registration);
		}
	}

	/**
	 * De-register the local service with the {@link ServiceRegistry}.
	 */
	protected void deregister() {
		this.serviceRegistry.deregister(getRegistration());
	}

	/**
	 * De-register the local management service with the {@link ServiceRegistry}.
	 */
	protected void deregisterManagement() {
		R registration = getManagementRegistration();
		if (registration != null) {
			this.serviceRegistry.deregister(registration);
		}
	}

	public void stop() {
		if (this.getRunning().compareAndSet(true, false) && isEnabled()) {
			deregister();
			if (shouldRegisterManagement()) {
				deregisterManagement();
			}
			this.serviceRegistry.close();
		}
	}

}

```

## NacosAutoServiceRegistration源码
```java
public class NacosAutoServiceRegistration
		extends AbstractAutoServiceRegistration<Registration> {

	private static final Logger log = LoggerFactory
			.getLogger(NacosAutoServiceRegistration.class);

	private NacosRegistration registration;

	public NacosAutoServiceRegistration(ServiceRegistry<Registration> serviceRegistry,
			AutoServiceRegistrationProperties autoServiceRegistrationProperties,
			NacosRegistration registration) {
		super(serviceRegistry, autoServiceRegistrationProperties);
		this.registration = registration;
	}

	@Deprecated
	public void setPort(int port) {
		getPort().set(port);
	}

	@Override
	protected NacosRegistration getRegistration() {
		if (this.registration.getPort() < 0 && this.getPort().get() > 0) {
			this.registration.setPort(this.getPort().get());
		}
		Assert.isTrue(this.registration.getPort() > 0, "service.port has not been set");
		return this.registration;
	}

	@Override
	protected NacosRegistration getManagementRegistration() {
		return null;
	}

	@Override
	protected void register() {
		if (!this.registration.getNacosDiscoveryProperties().isRegisterEnabled()) {
			log.debug("Registration disabled.");
			return;
		}
		if (this.registration.getPort() < 0) {
			this.registration.setPort(getPort().get());
		}
		super.register();
	}

	@Override
	protected void registerManagement() {
		if (!this.registration.getNacosDiscoveryProperties().isRegisterEnabled()) {
			return;
		}
		super.registerManagement();

	}

	@Override
	protected Object getConfiguration() {
		return this.registration.getNacosDiscoveryProperties();
	}

	@Override
	protected boolean isEnabled() {
		return this.registration.getNacosDiscoveryProperties().isRegisterEnabled();
	}

	@Override
	@SuppressWarnings("deprecation")
	protected String getAppName() {
		String appName = registration.getNacosDiscoveryProperties().getService();
		return StringUtils.isEmpty(appName) ? super.getAppName() : appName;
	}

}

```

## 收到WebServerInitializedEven之后调用org.springframework.cloud.client.serviceregistry.AbstractAutoServiceRegistration#start方法
```java
@Override
	@SuppressWarnings("deprecation")
	public void onApplicationEvent(WebServerInitializedEvent event) {
		bind(event);
	}

	@Deprecated
	public void bind(WebServerInitializedEvent event) {
		ApplicationContext context = event.getApplicationContext();
		if (context instanceof ConfigurableWebServerApplicationContext) {
			if ("management".equals(((ConfigurableWebServerApplicationContext) context)
					.getServerNamespace())) {
				return;
			}
		}
		this.port.compareAndSet(0, event.getWebServer().getPort());
		this.start();
	}
	
	public void start() {
		if (!isEnabled()) {
			if (logger.isDebugEnabled()) {
				logger.debug("Discovery Lifecycle disabled. Not starting");
			}
			return;
		}

		// only initialize if nonSecurePort is greater than 0 and it isn't already running
		// because of containerPortInitializer below
		if (!this.running.get()) {
			this.context.publishEvent(
					new InstancePreRegisteredEvent(this, getRegistration()));
			register();
			if (shouldRegisterManagement()) {
				registerManagement();
			}
			this.context.publishEvent(
					new InstanceRegisteredEvent<>(this, getConfiguration()));
			this.running.compareAndSet(false, true);
		}

	}
```

## 调用register完成SpringCloud注册，getRegistration为抽象方法，具体实现在NacosAutoServiceRegistration之中。serviceRegistry为注册服务接口，构造方法传入。
```java
    /**
	 * Register the local service with the {@link ServiceRegistry}.
	 */
	protected void register() {
		this.serviceRegistry.register(getRegistration());
	}
```
## getRegistration方法源码如下。返回的对象为NacosRegistration
```java
/**
 * @author xiaojing
 * @author <a href="mailto:mercyblitz@gmail.com">Mercy</a>
 */
public class NacosAutoServiceRegistration
		extends AbstractAutoServiceRegistration<Registration> {

	private static final Logger log = LoggerFactory
			.getLogger(NacosAutoServiceRegistration.class);

	private NacosRegistration registration;

	public NacosAutoServiceRegistration(ServiceRegistry<Registration> serviceRegistry,
			AutoServiceRegistrationProperties autoServiceRegistrationProperties,
			NacosRegistration registration) {
		super(serviceRegistry, autoServiceRegistrationProperties);
		this.registration = registration;
	}

	@Deprecated
	public void setPort(int port) {
		getPort().set(port);
	}

	@Override
	protected NacosRegistration getRegistration() {
		if (this.registration.getPort() < 0 && this.getPort().get() > 0) {
			this.registration.setPort(this.getPort().get());
		}
		Assert.isTrue(this.registration.getPort() > 0, "service.port has not been set");
		return this.registration;
	}

	@Override
	protected NacosRegistration getManagementRegistration() {
		return null;
	}

	@Override
	protected void register() {
		if (!this.registration.getNacosDiscoveryProperties().isRegisterEnabled()) {
			log.debug("Registration disabled.");
			return;
		}
		if (this.registration.getPort() < 0) {
			this.registration.setPort(getPort().get());
		}
		super.register();
	}

	@Override
	protected void registerManagement() {
		if (!this.registration.getNacosDiscoveryProperties().isRegisterEnabled()) {
			return;
		}
		super.registerManagement();

	}

	@Override
	protected Object getConfiguration() {
		return this.registration.getNacosDiscoveryProperties();
	}

	@Override
	protected boolean isEnabled() {
		return this.registration.getNacosDiscoveryProperties().isRegisterEnabled();
	}

	@Override
	@SuppressWarnings("deprecation")
	protected String getAppName() {
		String appName = registration.getNacosDiscoveryProperties().getService();
		return StringUtils.isEmpty(appName) ? super.getAppName() : appName;
	}

}
```
## NacosRegistration源码
```java
public class NacosRegistration implements Registration, ServiceInstance {

	/**
	 * The metadata key of management port.
	 */
	public static final String MANAGEMENT_PORT = "management.port";

	/**
	 * The metadata key of management context-path.
	 */
	public static final String MANAGEMENT_CONTEXT_PATH = "management.context-path";

	/**
	 * The metadata key of management address.
	 */
	public static final String MANAGEMENT_ADDRESS = "management.address";

	/**
	 * The metadata key of management endpoints web base path.
	 */
	public static final String MANAGEMENT_ENDPOINT_BASE_PATH = "management.endpoints.web.base-path";

	private NacosDiscoveryProperties nacosDiscoveryProperties;

	private ApplicationContext context;

	public NacosRegistration(NacosDiscoveryProperties nacosDiscoveryProperties,
			ApplicationContext context) {
		this.nacosDiscoveryProperties = nacosDiscoveryProperties;
		this.context = context;
	}

	@PostConstruct
	public void init() {

		Map<String, String> metadata = nacosDiscoveryProperties.getMetadata();
		Environment env = context.getEnvironment();

		String endpointBasePath = env.getProperty(MANAGEMENT_ENDPOINT_BASE_PATH);
		if (!StringUtils.isEmpty(endpointBasePath)) {
			metadata.put(MANAGEMENT_ENDPOINT_BASE_PATH, endpointBasePath);
		}

		Integer managementPort = ManagementServerPortUtils.getPort(context);
		if (null != managementPort) {
			metadata.put(MANAGEMENT_PORT, managementPort.toString());
			String contextPath = env
					.getProperty("management.server.servlet.context-path");
			String address = env.getProperty("management.server.address");
			if (!StringUtils.isEmpty(contextPath)) {
				metadata.put(MANAGEMENT_CONTEXT_PATH, contextPath);
			}
			if (!StringUtils.isEmpty(address)) {
				metadata.put(MANAGEMENT_ADDRESS, address);
			}
		}

		if (null != nacosDiscoveryProperties.getHeartBeatInterval()) {
			metadata.put(PreservedMetadataKeys.HEART_BEAT_INTERVAL,
					nacosDiscoveryProperties.getHeartBeatInterval().toString());
		}
		if (null != nacosDiscoveryProperties.getHeartBeatTimeout()) {
			metadata.put(PreservedMetadataKeys.HEART_BEAT_TIMEOUT,
					nacosDiscoveryProperties.getHeartBeatTimeout().toString());
		}
		if (null != nacosDiscoveryProperties.getIpDeleteTimeout()) {
			metadata.put(PreservedMetadataKeys.IP_DELETE_TIMEOUT,
					nacosDiscoveryProperties.getIpDeleteTimeout().toString());
		}
	}

	@Override
	public String getServiceId() {
		return nacosDiscoveryProperties.getService();
	}

	@Override
	public String getHost() {
		return nacosDiscoveryProperties.getIp();
	}

	@Override
	public int getPort() {
		return nacosDiscoveryProperties.getPort();
	}

	public void setPort(int port) {
		this.nacosDiscoveryProperties.setPort(port);
	}

	@Override
	public boolean isSecure() {
		return nacosDiscoveryProperties.isSecure();
	}

	@Override
	public URI getUri() {
		return DefaultServiceInstance.getUri(this);
	}

	@Override
	public Map<String, String> getMetadata() {
		return nacosDiscoveryProperties.getMetadata();
	}

	public boolean isRegisterEnabled() {
		return nacosDiscoveryProperties.isRegisterEnabled();
	}

	public String getCluster() {
		return nacosDiscoveryProperties.getClusterName();
	}

	public float getRegisterWeight() {
		return nacosDiscoveryProperties.getWeight();
	}

	public NacosDiscoveryProperties getNacosDiscoveryProperties() {
		return nacosDiscoveryProperties;
	}

	public NamingService getNacosNamingService() {
		return nacosDiscoveryProperties.namingServiceInstance();
	}

	@Override
	public String toString() {
		return "NacosRegistration{" + "nacosDiscoveryProperties="
				+ nacosDiscoveryProperties + '}';
	}

}

```
## serviceRegistry#register实现在NacosServiceRegistry之中
```java
public class NacosServiceRegistry implements ServiceRegistry<Registration> {

	private static final Logger log = LoggerFactory.getLogger(NacosServiceRegistry.class);

	private final NacosDiscoveryProperties nacosDiscoveryProperties;

	private final NamingService namingService;

	public NacosServiceRegistry(NacosDiscoveryProperties nacosDiscoveryProperties) {
		this.nacosDiscoveryProperties = nacosDiscoveryProperties;
		this.namingService = nacosDiscoveryProperties.namingServiceInstance();
	}

	@Override
	public void register(Registration registration) {

		if (StringUtils.isEmpty(registration.getServiceId())) {
			log.warn("No service to register for nacos client...");
			return;
		}

		String serviceId = registration.getServiceId();
		String group = nacosDiscoveryProperties.getGroup();

		Instance instance = getNacosInstanceFromRegistration(registration);

		try {
			namingService.registerInstance(serviceId, group, instance);
			log.info("nacos registry, {} {} {}:{} register finished", group, serviceId,
					instance.getIp(), instance.getPort());
		}
		catch (Exception e) {
			log.error("nacos registry, {} register failed...{},", serviceId,
					registration.toString(), e);
			// rethrow a RuntimeException if the registration is failed.
			// issue : https://github.com/alibaba/spring-cloud-alibaba/issues/1132
			rethrowRuntimeException(e);
		}
	}

	@Override
	public void deregister(Registration registration) {

		log.info("De-registering from Nacos Server now...");

		if (StringUtils.isEmpty(registration.getServiceId())) {
			log.warn("No dom to de-register for nacos client...");
			return;
		}

		NamingService namingService = nacosDiscoveryProperties.namingServiceInstance();
		String serviceId = registration.getServiceId();
		String group = nacosDiscoveryProperties.getGroup();

		try {
			namingService.deregisterInstance(serviceId, group, registration.getHost(),
					registration.getPort(), nacosDiscoveryProperties.getClusterName());
		}
		catch (Exception e) {
			log.error("ERR_NACOS_DEREGISTER, de-register failed...{},",
					registration.toString(), e);
		}

		log.info("De-registration finished.");
	}

	@Override
	public void close() {

	}

	@Override
	public void setStatus(Registration registration, String status) {

		if (!status.equalsIgnoreCase("UP") && !status.equalsIgnoreCase("DOWN")) {
			log.warn("can't support status {},please choose UP or DOWN", status);
			return;
		}

		String serviceId = registration.getServiceId();

		Instance instance = getNacosInstanceFromRegistration(registration);

		if (status.equalsIgnoreCase("DOWN")) {
			instance.setEnabled(false);
		}
		else {
			instance.setEnabled(true);
		}

		try {
			nacosDiscoveryProperties.namingMaintainServiceInstance()
					.updateInstance(serviceId, instance);
		}
		catch (Exception e) {
			throw new RuntimeException("update nacos instance status fail", e);
		}

	}

	@Override
	public Object getStatus(Registration registration) {

		String serviceName = registration.getServiceId();
		try {
			List<Instance> instances = nacosDiscoveryProperties.namingServiceInstance()
					.getAllInstances(serviceName);
			for (Instance instance : instances) {
				if (instance.getIp().equalsIgnoreCase(nacosDiscoveryProperties.getIp())
						&& instance.getPort() == nacosDiscoveryProperties.getPort()) {
					return instance.isEnabled() ? "UP" : "DOWN";
				}
			}
		}
		catch (Exception e) {
			log.error("get all instance of {} error,", serviceName, e);
		}
		return null;
	}

	private Instance getNacosInstanceFromRegistration(Registration registration) {
		Instance instance = new Instance();
		instance.setIp(registration.getHost());
		instance.setPort(registration.getPort());
		instance.setWeight(nacosDiscoveryProperties.getWeight());
		instance.setClusterName(nacosDiscoveryProperties.getClusterName());
		instance.setMetadata(registration.getMetadata());

		return instance;
	}

}

```
## com.alibaba.nacos.client.naming.NacosNamingService#registerInstance(java.lang.String, java.lang.String, com.alibaba.nacos.api.naming.pojo.Instance)
```java
    @Override
    public void registerInstance(String serviceName, String groupName, Instance instance) throws NacosException {
        if (instance.isEphemeral()) {
            BeatInfo beatInfo = beatReactor.buildBeatInfo(instance);
            beatReactor.addBeatInfo(NamingUtils.getGroupedName(serviceName, groupName), beatInfo);
        }
        serverProxy.registerService(NamingUtils.getGroupedName(serviceName, groupName), groupName, instance);
    }
```

## serverProxy为com.alibaba.nacos.client.naming.net.NamingProxy
```java
    public void registerService(String serviceName, String groupName, Instance instance) throws NacosException {

        NAMING_LOGGER.info("[REGISTER-SERVICE] {} registering service {} with instance: {}",
            namespaceId, serviceName, instance);

        final Map<String, String> params = new HashMap<String, String>(16);
        params.put(CommonParams.NAMESPACE_ID, namespaceId);
        params.put(CommonParams.SERVICE_NAME, serviceName);
        params.put(CommonParams.GROUP_NAME, groupName);
        params.put(CommonParams.CLUSTER_NAME, instance.getClusterName());
        params.put("ip", instance.getIp());
        params.put("port", String.valueOf(instance.getPort()));
        params.put("weight", String.valueOf(instance.getWeight()));
        params.put("enable", String.valueOf(instance.isEnabled()));
        params.put("healthy", String.valueOf(instance.isHealthy()));
        params.put("ephemeral", String.valueOf(instance.isEphemeral()));
        params.put("metadata", JacksonUtils.toJson(instance.getMetadata()));
        // UtilAndComs.NACOS_URL_INSTANCE=/nacos/v1/ns/instance
        reqAPI(UtilAndComs.NACOS_URL_INSTANCE, params, HttpMethod.POST);
    }
    
    public String reqAPI(String api, Map<String, String> params, String method) throws NacosException {
        return reqAPI(api, params, Collections.EMPTY_MAP, method);
    }
    
    public String reqAPI(String api, Map<String, String> params, Map<String, String> body, String method) throws NacosException {
        return reqAPI(api, params, body, getServerList(), method);
    }
    
    public String reqAPI(String api, Map<String, String> params, Map<String, String> body, List<String> servers, String method) throws NacosException {

        params.put(CommonParams.NAMESPACE_ID, getNamespaceId());

        if (CollectionUtils.isEmpty(servers) && StringUtils.isEmpty(nacosDomain)) {
            throw new NacosException(NacosException.INVALID_PARAM, "no server available");
        }

        NacosException exception = new NacosException();

        if (servers != null && !servers.isEmpty()) {

            Random random = new Random(System.currentTimeMillis());
            int index = random.nextInt(servers.size());

            for (int i = 0; i < servers.size(); i++) {
                String server = servers.get(index);
                try {
                    return callServer(api, params, body, server, method);
                } catch (NacosException e) {
                    exception = e;
                    if (NAMING_LOGGER.isDebugEnabled()) {
                        NAMING_LOGGER.debug("request {} failed.", server, e);
                    }
                }
                index = (index + 1) % servers.size();
            }
        }

        if (StringUtils.isNotBlank(nacosDomain)) {
            for (int i = 0; i < UtilAndComs.REQUEST_DOMAIN_RETRY_COUNT; i++) {
                try {
                    return callServer(api, params, body, nacosDomain, method);
                } catch (NacosException e) {
                    exception = e;
                    if (NAMING_LOGGER.isDebugEnabled()) {
                        NAMING_LOGGER.debug("request {} failed.", nacosDomain, e);
                    }
                }
            }
        }

        NAMING_LOGGER.error("request: {} failed, servers: {}, code: {}, msg: {}",
            api, servers, exception.getErrCode(), exception.getErrMsg());

        throw new NacosException(exception.getErrCode(), "failed to req API:" + api + " after all servers(" + servers + ") tried: "
            + exception.getMessage());

    }
    
    public String callServer(String api, Map<String, String> params, Map<String, String> body, String curServer, String method)
        throws NacosException {
        long start = System.currentTimeMillis();
        long end = 0;
        injectSecurityInfo(params);
        Header header = builderHeader();

        String url;
        if (curServer.startsWith(UtilAndComs.HTTPS) || curServer.startsWith(UtilAndComs.HTTP)) {
            url = curServer + api;
        } else {
            if (!curServer.contains(UtilAndComs.SERVER_ADDR_IP_SPLITER)) {
                curServer = curServer + UtilAndComs.SERVER_ADDR_IP_SPLITER + serverPort;
            }
            url = NamingHttpClientManager.getPrefix() + curServer + api;
        }

        try {
            HttpRestResult<String> restResult = nacosRestTemplate.exchangeForm(url, header, params, body, method, String.class);
            end = System.currentTimeMillis();

            MetricsMonitor.getNamingRequestMonitor(method, url, String.valueOf(restResult.getCode()))
                .observe(end - start);

            if (restResult.ok()) {
                return restResult.getData();
            }
            if (HttpStatus.SC_NOT_MODIFIED == restResult.getCode()) {
                return StringUtils.EMPTY;
            }
            throw new NacosException(restResult.getCode(), restResult.getData());
        } catch (Exception e) {
            NAMING_LOGGER.error("[NA] failed to request", e);
            throw new NacosException(NacosException.SERVER_ERROR, e);
        }
    }
    
    /**
     * Execute the HTTP method to the given URI template, writing the given request entity to the request, and
     * returns the response as {@link HttpRestResult}.
     *
     * @param url          url
     * @param header       http header param
     * @param paramValues  http query param
     * @param bodyValues   http body param
     * @param httpMethod   http method
     * @param responseType return type
     * @return {@link HttpRestResult}
     * @throws Exception ex
     */
    public <T> HttpRestResult<T> exchangeForm(String url, Header header,
                                              Map<String, String> paramValues, Map<String, String> bodyValues, String httpMethod, Type responseType) throws Exception{
        RequestHttpEntity requestHttpEntity = new RequestHttpEntity(
            header.setContentType(MediaType.APPLICATION_FORM_URLENCODED),
            Query.newInstance().initParams(paramValues),
            bodyValues);
        return execute(url, httpMethod, requestHttpEntity, responseType);
    }
    
    private <T> HttpRestResult<T> execute(String url, String httpMethod, RequestHttpEntity requestEntity,
            Type responseType) throws Exception {
        URI uri = HttpUtils.buildUri(url, requestEntity.getQuery());
        if (LOGGER.isDebugEnabled()) {
            LOGGER.debug("HTTP " + httpMethod + " " + url);
        }
        HttpClientResponse response = null;
        try {
            response = requestClient.execute(uri, httpMethod, requestEntity);
            return ResponseHandler.responseEntityExtractor(response, responseType);
        } finally {
            if (response != null) {
                response.close();
            }
        }
    }
```
