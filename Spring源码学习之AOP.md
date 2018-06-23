# Spring源码学习之AOP

## SpringBoot开启AOP配置如下图添加`@EnableAspectJAutoProxy`注解，会自动完成相关配置
```java
@SpringBootApplication(scanBasePackages = "com.xin.springboot")
@ServletComponentScan//扫描监听类
@EnableAspectJAutoProxy
public class XinSpringbootApplication extends SpringBootServletInitializer {

    public static void main(String[] args) {

        SpringApplication.run(XinSpringbootApplication.class, args);
    }

    @Override
    protected SpringApplicationBuilder configure(SpringApplicationBuilder builder) {
        builder.sources(XinSpringbootApplication.class);
        return super.configure(builder);
    }
}

```
## 注解`@EnableAspectJAutoProxy`功能如下
```java
@Target(ElementType.TYPE)
@Retention(RetentionPolicy.RUNTIME)
@Documented
@Import(AspectJAutoProxyRegistrar.class)//注册AnnotationAwareAspectJAutoProxyCreator到BeanDefinitionRegistry之中。
public @interface EnableAspectJAutoProxy {

	/**
	 * Indicate whether subclass-based (CGLIB) proxies are to be created as opposed
	 * to standard Java interface-based proxies. The default is {@code false}.
	 */
	boolean proxyTargetClass() default false;

	/**
	 * Indicate that the proxy should be exposed by the AOP framework as a {@code ThreadLocal}
	 * for retrieval via the {@link org.springframework.aop.framework.AopContext} class.
	 * Off by default, i.e. no guarantees that {@code AopContext} access will work.
	 * @since 4.3.1
	 */
	boolean exposeProxy() default false;

}
```

## 注解@EnableAspectJAutoProxy使用AspectJAutoProxyRegistrar注册AnnotationAwareAspectJAutoProxyCreator到BeanDefinitionRegistry之中。BeanDefinitionRegistry为DefaultListableBeanFactory,AspectJAutoProxyRegistrar类如下
```java
/**
 * Registers an {@link org.springframework.aop.aspectj.annotation.AnnotationAwareAspectJAutoProxyCreator
 * AnnotationAwareAspectJAutoProxyCreator} against the current {@link BeanDefinitionRegistry}
 * as appropriate based on a given @{@link EnableAspectJAutoProxy} annotation.
 *
 * @author Chris Beams
 * @author Juergen Hoeller
 * @since 3.1
 * @see EnableAspectJAutoProxy
 */
class AspectJAutoProxyRegistrar implements ImportBeanDefinitionRegistrar {

	/**
	 * Register, escalate, and configure the AspectJ auto proxy creator based on the value
	 * of the @{@link EnableAspectJAutoProxy#proxyTargetClass()} attribute on the importing
	 * {@code @Configuration} class.
	 */
	@Override
	public void registerBeanDefinitions(
			AnnotationMetadata importingClassMetadata, BeanDefinitionRegistry registry) {
		
		//注册AnnotationAwareAspectJAutoProxyCreator
		AopConfigUtils.registerAspectJAnnotationAutoProxyCreatorIfNecessary(registry);

		AnnotationAttributes enableAspectJAutoProxy =
				AnnotationConfigUtils.attributesFor(importingClassMetadata, EnableAspectJAutoProxy.class);
		if (enableAspectJAutoProxy != null) {
			if (enableAspectJAutoProxy.getBoolean("proxyTargetClass")) {
				AopConfigUtils.forceAutoProxyCreatorToUseClassProxying(registry);
			}
			if (enableAspectJAutoProxy.getBoolean("exposeProxy")) {
				AopConfigUtils.forceAutoProxyCreatorToExposeProxy(registry);
			}
		}
	}

}
```
## 在`AopConfigUtils`之中完成注册如下
```java
@Nullable
public static BeanDefinition registerAspectJAnnotationAutoProxyCreatorIfNecessary(BeanDefinitionRegistry registry) {
	return registerAspectJAnnotationAutoProxyCreatorIfNecessary(registry, null);
}

@Nullable
public static BeanDefinition registerAspectJAnnotationAutoProxyCreatorIfNecessary(BeanDefinitionRegistry registry,
		@Nullable Object source) {

	return registerOrEscalateApcAsRequired(AnnotationAwareAspectJAutoProxyCreator.class, registry, source);
}

@Nullable
private static BeanDefinition registerOrEscalateApcAsRequired(Class<?> cls, BeanDefinitionRegistry registry,
		@Nullable Object source) {

	Assert.notNull(registry, "BeanDefinitionRegistry must not be null");

	if (registry.containsBeanDefinition(AUTO_PROXY_CREATOR_BEAN_NAME)) {
		BeanDefinition apcDefinition = registry.getBeanDefinition(AUTO_PROXY_CREATOR_BEAN_NAME);
		if (!cls.getName().equals(apcDefinition.getBeanClassName())) {
			int currentPriority = findPriorityForClass(apcDefinition.getBeanClassName());
			int requiredPriority = findPriorityForClass(cls);
			if (currentPriority < requiredPriority) {
				apcDefinition.setBeanClassName(cls.getName());
			}
		}
		return null;
	}

	RootBeanDefinition beanDefinition = new RootBeanDefinition(cls);
	beanDefinition.setSource(source);
	beanDefinition.getPropertyValues().add("order", Ordered.HIGHEST_PRECEDENCE);
	beanDefinition.setRole(BeanDefinition.ROLE_INFRASTRUCTURE);
	registry.registerBeanDefinition(AUTO_PROXY_CREATOR_BEAN_NAME, beanDefinition);
	return beanDefinition;
}

```
## 注册AnnotationAwareAspectJAutoProxyCreator之后，在获取bean对象时是使用AnnotationAwareAspectJAutoProxyCreator来创建代理对象
AnnotationAwareAspectJAutoProxyCreator类结构图如下
 ![](https://github.com/lucky-xin/Learning/blob/gh-pages/image/AnnotationAwareAspectJAutoProxyCreator.png)
## AnnotationAwareAspectJAutoProxyCreator实现了InstantiationAwareBeanPostProcessor接口，在创建对象时会查找InstantiationAwareBeanPostProcessor并使用InstantiationAwareBeanPostProcessor来创建代理对象,如果成功创建了代理对象则直接返回该代理对象，否则根据BeanDefinition定义来创建对象，具体实现在AbstractAutowireCapableBeanFactory之中，看代码会看到
```java
/**
 * Central method of this class: creates a bean instance,
 * populates the bean instance, applies post-processors, etc.
 * @see #doCreateBean
 */
@Override
protected Object createBean(String beanName, RootBeanDefinition mbd, @Nullable Object[] args)
		throws BeanCreationException {

	if (logger.isDebugEnabled()) {
		logger.debug("Creating instance of bean '" + beanName + "'");
	}
	RootBeanDefinition mbdToUse = mbd;

	// Make sure bean class is actually resolved at this point, and
	// clone the bean definition in case of a dynamically resolved Class
	// which cannot be stored in the shared merged bean definition.
	Class<?> resolvedClass = resolveBeanClass(mbd, beanName);
	if (resolvedClass != null && !mbd.hasBeanClass() && mbd.getBeanClassName() != null) {
		mbdToUse = new RootBeanDefinition(mbd);
		mbdToUse.setBeanClass(resolvedClass);
	}

	// Prepare method overrides.
	try {
		mbdToUse.prepareMethodOverrides();
	}
	catch (BeanDefinitionValidationException ex) {
		throw new BeanDefinitionStoreException(mbdToUse.getResourceDescription(),
				beanName, "Validation of method overrides failed", ex);
	}

	try {
		// Give BeanPostProcessors a chance to return a proxy instead of the target bean instance.
		//遍历所有BeanPostProcessor找到InstantiationAwareBeanPostProcessor实现类来创建代理对象
		//如果成功创建了代理对象，则返回该代理对象，不在往下执行
		Object bean = resolveBeforeInstantiation(beanName, mbdToUse);
		if (bean != null) {
			return bean;
		}
	}
	catch (Throwable ex) {
		throw new BeanCreationException(mbdToUse.getResourceDescription(), beanName,
				"BeanPostProcessor before instantiation of bean failed", ex);
	}

	try {
		// 如果没有找到InstantiationAwareBeanPostProcessor则根据BeanDefinition信息来创建对象
		Object beanInstance = doCreateBean(beanName, mbdToUse, args);
		if (logger.isDebugEnabled()) {
			logger.debug("Finished creating instance of bean '" + beanName + "'");
		}
		return beanInstance;
	}
	catch (BeanCreationException | ImplicitlyAppearedSingletonException ex) {
		// A previously detected exception with proper bean creation context already,
		// or illegal singleton state to be communicated up to DefaultSingletonBeanRegistry.
		throw ex;
	}
	catch (Throwable ex) {
		throw new BeanCreationException(
				mbdToUse.getResourceDescription(), beanName, "Unexpected exception during bean creation", ex);
	}
}
````
## 进入resolveBeforeInstantiation方法之中,查找InstantiationAwareBeanPostProcessor实现类
```java
/**
 * Apply before-instantiation post-processors, resolving whether there is a
 * before-instantiation shortcut for the specified bean.
 * @param beanName the name of the bean
 * @param mbd the bean definition for the bean
 * @return the shortcut-determined bean instance, or {@code null} if none
 */
@Nullable
protected Object resolveBeforeInstantiation(String beanName, RootBeanDefinition mbd) {
	Object bean = null;
	if (!Boolean.FALSE.equals(mbd.beforeInstantiationResolved)) {
		// Make sure bean class is actually resolved at this point.
		//判断是否有InstantiationAwareBeanPostProcessor实现类
		if (!mbd.isSynthetic() && hasInstantiationAwareBeanPostProcessors()) {
			Class<?> targetType = determineTargetType(beanName, mbd);
			if (targetType != null) {
				bean = applyBeanPostProcessorsBeforeInstantiation(targetType, beanName);
				if (bean != null) {
					bean = applyBeanPostProcessorsAfterInitialization(bean, beanName);
				}
			}
		}
		mbd.beforeInstantiationResolved = (bean != null);
	}
	return bean;
}
````
## 遍历查找InstantiationAwareBeanPostProcessor，并使用InstantiationAwareBeanPostProcessor来创建代理对象
```java
/**
 * Apply InstantiationAwareBeanPostProcessors to the specified bean definition
 * (by class and name), invoking their {@code postProcessBeforeInstantiation} methods.
 * <p>Any returned object will be used as the bean instead of actually instantiating
 * the target bean. A {@code null} return value from the post-processor will
 * result in the target bean being instantiated.
 * @param beanClass the class of the bean to be instantiated
 * @param beanName the name of the bean
 * @return the bean object to use instead of a default instance of the target bean, or {@code null}
 * @see InstantiationAwareBeanPostProcessor#postProcessBeforeInstantiation
 */
@Nullable
protected Object applyBeanPostProcessorsBeforeInstantiation(Class<?> beanClass, String beanName) {
	//遍历查找InstantiationAwareBeanPostProcessor，并使用InstantiationAwareBeanPostProcessor来创建代理对象
	for (BeanPostProcessor bp : getBeanPostProcessors()) {
		if (bp instanceof InstantiationAwareBeanPostProcessor) {
			InstantiationAwareBeanPostProcessor ibp = (InstantiationAwareBeanPostProcessor) bp;
			Object result = ibp.postProcessBeforeInstantiation(beanClass, beanName);
			if (result != null) {
				return result;
			}
		}
	}
	return null;
}
```
## 调用AnnotationAwareAspectJAutoProxyCreator类的postProcessBeforeInstantiation方法来创建代理对象。postProcessBeforeInstantiation方法实现在AnnotationAwareAspectJAutoProxyCreator超类AbstractAutoProxyCreator的postProcessBeforeInstantiation方法之中.判断该Class是否有Aspect注解,如果有则根据Class信息创建Advisor并返回，每一个方法，每一个变量对应一个Advisor，具体实现如下代码
```java
@Override
public Object postProcessBeforeInstantiation(Class<?> beanClass, String beanName) throws BeansException {
	Object cacheKey = getCacheKey(beanClass, beanName);

	if (!StringUtils.hasLength(beanName) || !this.targetSourcedBeans.contains(beanName)) {
		if (this.advisedBeans.containsKey(cacheKey)) {
			return null;
		}
		if (isInfrastructureClass(beanClass) || shouldSkip(beanClass, beanName)) {
			this.advisedBeans.put(cacheKey, Boolean.FALSE);
			return null;
		}
	}

	// Create proxy here if we have a custom TargetSource.
	// Suppresses unnecessary default instantiation of the target bean:
	// The TargetSource will handle target instances in a custom fashion.
	TargetSource targetSource = getCustomTargetSource(beanClass, beanName);
	if (targetSource != null) {
		if (StringUtils.hasLength(beanName)) {
			this.targetSourcedBeans.add(beanName);
		}
		//查找Aspect注解并创建Advisor，
		Object[] specificInterceptors = getAdvicesAndAdvisorsForBean(beanClass, beanName, targetSource);
		Object proxy = createProxy(beanClass, beanName, specificInterceptors, targetSource);
		this.proxyTypes.put(cacheKey, proxy.getClass());
		return proxy;
	}

	return null;
}
```
## 进入getAdvicesAndAdvisorsForBean之中，getAdvicesAndAdvisorsForBean为AbstractAutoProxyCreator类的抽象方法，具体实现在子类AbstractAdvisorAutoProxyCreator之中，最后查找分派到AnnotationAwareAspectJAutoProxyCreator的方法findCandidateAdvisors，并使用findAdvisorsThatCanApply方法过滤Advisor，只获取该bean的Advisor
```java
@Override
@Nullable
protected Object[] getAdvicesAndAdvisorsForBean(
		Class<?> beanClass, String beanName, @Nullable TargetSource targetSource) {

	List<Advisor> advisors = findEligibleAdvisors(beanClass, beanName);
	if (advisors.isEmpty()) {
		return DO_NOT_PROXY;
	}
	return advisors.toArray();
}

/**
 * Find all eligible Advisors for auto-proxying this class.
 * @param beanClass the clazz to find advisors for
 * @param beanName the name of the currently proxied bean
 * @return the empty List, not {@code null},
 * if there are no pointcuts or interceptors
 * @see #findCandidateAdvisors
 * @see #sortAdvisors
 * @see #extendAdvisors
 */
protected List<Advisor> findEligibleAdvisors(Class<?> beanClass, String beanName) {
	List<Advisor> candidateAdvisors = findCandidateAdvisors();//AnnotationAwareAspectJAutoProxyCreator重写了findCandidateAdvisors方法
	List<Advisor> eligibleAdvisors = findAdvisorsThatCanApply(candidateAdvisors, beanClass, beanName);//只获取该bean的Advisor
	extendAdvisors(eligibleAdvisors);
	if (!eligibleAdvisors.isEmpty()) {
		eligibleAdvisors = sortAdvisors(eligibleAdvisors);
	}
	return eligibleAdvisors;
}
````
## AnnotationAwareAspectJAutoProxyCreator重写了findCandidateAdvisors方法具体实现看如下代码aspectJAdvisorsBuilder为BeanFactoryAspectJAdvisorsBuilder
```java
@Override
protected List<Advisor> findCandidateAdvisors() {
	// Add all the Spring advisors found according to superclass rules.
	List<Advisor> advisors = super.findCandidateAdvisors();
	// Build Advisors for all AspectJ aspects in the bean factory.
	if (this.aspectJAdvisorsBuilder != null) {
		//查找Class的Aspect注解入口
		advisors.addAll(this.aspectJAdvisorsBuilder.buildAspectJAdvisors());
	}
	return advisors;
}
```
## BeanFactoryAspectJAdvisorsBuilder类如下图
![](https://github.com/lucky-xin/Learning/blob/gh-pages/image/BeanFactoryAspectJAdvisorsBuilder.png)
## BeanFactoryAspectJAdvisorsBuilder类的方法buildAspectJAdvisors，第一次调用buildAspectJAdvisors方法时aspectBeanNames为null,后面直接读取aspectBeanNames。第一次初始化时会遍历DefaultListableBeanFactory获取所有已经注册的bean,获取bean的Class信息，判断是否有Aspect注解，如果有则缓存起来，
然后遍历具体实现如下
```java
/**
 * Look for AspectJ-annotated aspect beans in the current bean factory,
 * and return to a list of Spring AOP Advisors representing them.
 * <p>Creates a Spring Advisor for each AspectJ advice method.
 * @return the list of {@link org.springframework.aop.Advisor} beans
 * @see #isEligibleBean
 */
public List<Advisor> buildAspectJAdvisors() {
	List<String> aspectNames = this.aspectBeanNames;

	if (aspectNames == null) {
		synchronized (this) {
	aspectNames = this.aspectBeanNames;
	if (aspectNames == null) {
		List<Advisor> advisors = new LinkedList<>();
		aspectNames = new LinkedList<>();
		String[] beanNames = BeanFactoryUtils.beanNamesForTypeIncludingAncestors(
				this.beanFactory, Object.class, true, false);
		for (String beanName : beanNames) {
			if (!isEligibleBean(beanName)) {
				continue;
			}
			// We must be careful not to instantiate beans eagerly as in this case they
			// would be cached by the Spring container but would not have been weaved.
			Class<?> beanType = this.beanFactory.getType(beanName);
			if (beanType == null) {
				continue;
			}
			if (this.advisorFactory.isAspect(beanType)) {
				// 把当前的bean名称存入缓存
				aspectNames.add(beanName);
				// AspectMetadata封装了注解信息
				AspectMetadata amd = new AspectMetadata(beanType, beanName);
				if (amd.getAjType().getPerClause().getKind() == PerClauseKind.SINGLETON) {
					MetadataAwareAspectInstanceFactory factory =
							new BeanFactoryAspectInstanceFactory(this.beanFactory, beanName);
					List<Advisor> classAdvisors = this.advisorFactory.getAdvisors(factory);
					if (this.beanFactory.isSingleton(beanName)) {
						this.advisorsCache.put(beanName, classAdvisors);
					}
					else {
						this.aspectFactoryCache.put(beanName, factory);
					}
					advisors.addAll(classAdvisors);
				}
				else {
					// Per target or per this.
					if (this.beanFactory.isSingleton(beanName)) {
						throw new IllegalArgumentException("Bean with name '" + beanName +
								"' is a singleton, but aspect instantiation model is not singleton");
					}
					MetadataAwareAspectInstanceFactory factory =
							new PrototypeAspectInstanceFactory(this.beanFactory, beanName);
					this.aspectFactoryCache.put(beanName, factory);
					// 根据class获取Advisor
					advisors.addAll(this.advisorFactory.getAdvisors(factory));
				}
			}
		}
		this.aspectBeanNames = aspectNames;
		return advisors;
	}
		}
	}
```
## 最后委派到MetadataAwareAspectInstanceFactory的方法getAdvisors获取Advisor.通过方法getAdvisorMethods判断获取该Class所有有Pointcut注解的方法
```java
@Override
public List<Advisor> getAdvisors(MetadataAwareAspectInstanceFactory aspectInstanceFactory) {
	Class<?> aspectClass = aspectInstanceFactory.getAspectMetadata().getAspectClass();
	String aspectName = aspectInstanceFactory.getAspectMetadata().getAspectName();
	validate(aspectClass);

	// We need to wrap the MetadataAwareAspectInstanceFactory with a decorator
	// so that it will only instantiate once.
	MetadataAwareAspectInstanceFactory lazySingletonAspectInstanceFactory =
			new LazySingletonAspectInstanceFactoryDecorator(aspectInstanceFactory);

	List<Advisor> advisors = new LinkedList<>();
	for (Method method : getAdvisorMethods(aspectClass)) {
		// 获取方法注解并生成Advis
		Advisor advisor = getAdvisor(method, lazySingletonAspectInstanceFactory, advisors.size(), aspectName);
		if (advisor != null) {
			advisors.add(advisor);
		}
	}

	// If it's a per target aspect, emit the dummy instantiating aspect.
	if (!advisors.isEmpty() && lazySingletonAspectInstanceFactory.getAspectMetadata().isLazilyInstantiated()) {
		Advisor instantiationAdvisor = new SyntheticInstantiationAdvisor(lazySingletonAspectInstanceFactory);
		advisors.add(0, instantiationAdvisor);
	}

	// Find introduction fields.
	for (Field field : aspectClass.getDeclaredFields()) {
		Advisor advisor = getDeclareParentsAdvisor(field);
		if (advisor != null) {
			advisors.add(advisor);
		}
	}

	return advisors;
}

private List<Method> getAdvisorMethods(Class<?> aspectClass) {
	final List<Method> methods = new LinkedList<>();
	ReflectionUtils.doWithMethods(aspectClass, method -> {
		// Exclude pointcuts
		if (AnnotationUtils.getAnnotation(method, Pointcut.class) == null) {
			methods.add(method);
		}
	});
	methods.sort(METHOD_COMPARATOR);
	return methods;
}
```
## 进入MetadataAwareAspectInstanceFactory类的方法getAdvisor。查找方法是否有Before, Around, After, AfterReturning, AfterThrowing, Pointcut注解，找到就使用AspectJExpressionPointcut封装切入点信息并创建InstantiationModelAwarePointcutAdvisorImpl对象
```java
@Override
@Nullable
public Advisor getAdvisor(Method candidateAdviceMethod, MetadataAwareAspectInstanceFactory aspectInstanceFactory,
		int declarationOrderInAspect, String aspectName) {

	validate(aspectInstanceFactory.getAspectMetadata().getAspectClass());

	AspectJExpressionPointcut expressionPointcut = getPointcut(
			candidateAdviceMethod, aspectInstanceFactory.getAspectMetadata().getAspectClass());
	if (expressionPointcut == null) {
		return null;
	}

	return new InstantiationModelAwarePointcutAdvisorImpl(expressionPointcut, candidateAdviceMethod,
			this, aspectInstanceFactory, declarationOrderInAspect, aspectName);
}
```
## 进入getPointcut方法通过AbstractAspectJAdvisorFactory类的方法findAspectJAnnotationOnMethod查找方法是否有Before, Around, After, AfterReturning, AfterThrowing, Pointcut注解找到则使用AspectJAnnotation封装注解信息并返回。如果找到注解则创建AspectJExpressionPointcut封装切入点信息并返回
```java
@Nullable
private AspectJExpressionPointcut getPointcut(Method candidateAdviceMethod, Class<?> candidateAspectClass) {
	AspectJAnnotation<?> aspectJAnnotation =
			AbstractAspectJAdvisorFactory.findAspectJAnnotationOnMethod(candidateAdviceMethod);
	if (aspectJAnnotation == null) {
		return null;
	}

	AspectJExpressionPointcut ajexp =
			new AspectJExpressionPointcut(candidateAspectClass, new String[0], new Class<?>[0]);
	ajexp.setExpression(aspectJAnnotation.getPointcutExpression());
	if (this.beanFactory != null) {
		ajexp.setBeanFactory(this.beanFactory);
	}
	return ajexp;
}
```
## AbstractAspectJAdvisorFactory类的方法findAspectJAnnotationOnMethod具体实现如下
```java
/**
 * Find and return the first AspectJ annotation on the given method
 * (there <i>should</i> only be one anyway...)
 */
@SuppressWarnings("unchecked")
@Nullable
protected static AspectJAnnotation<?> findAspectJAnnotationOnMethod(Method method) {
	Class<?>[] classesToLookFor = new Class<?>[] {
			Before.class, Around.class, After.class, AfterReturning.class, AfterThrowing.class, Pointcut.class};
	for (Class<?> c : classesToLookFor) {
		AspectJAnnotation<?> foundAnnotation = findAnnotation(method, (Class<Annotation>) c);
		if (foundAnnotation != null) {
			return foundAnnotation;
		}
	}
	return null;
}

@Nullable
private static <A extends Annotation> AspectJAnnotation<A> findAnnotation(Method method, Class<A> toLookFor) {
	A result = AnnotationUtils.findAnnotation(method, toLookFor);
	if (result != null) {
		return new AspectJAnnotation<>(result);
	}
	else {
		return null;
	}
}

```
## 到此已经把该Class所有有Before, Around, After, AfterReturning, AfterThrowing, Pointcut注解的方法封装成Advisor。接下来遍历所有成员查找所有有DeclareParents注解的成员并封装成Advisor，具体实现如下代码
```java
/**
 * Build a {@link org.springframework.aop.aspectj.DeclareParentsAdvisor}
 * for the given introduction field.
 * <p>Resulting Advisors will need to be evaluated for targets.
 * @param introductionField the field to introspect
 * @return the Advisor instance, or {@code null} if not an Advisor
 */
@Nullable
private Advisor getDeclareParentsAdvisor(Field introductionField) {
	DeclareParents declareParents = introductionField.getAnnotation(DeclareParents.class);
	if (declareParents == null) {
		// Not an introduction field
		return null;
	}

	if (DeclareParents.class == declareParents.defaultImpl()) {
		throw new IllegalStateException("'defaultImpl' attribute must be set on DeclareParents");
	}

	return new DeclareParentsAdvisor(
			introductionField.getType(), declareParents.value(), declareParents.defaultImpl());
}
```
## 根据该Class生成Advisor，存入list之中。此Advisor为InstantiationModelAwarePointcutAdvisorImpl
获取Advisor之后返回到AnnotationAwareAspectJAutoProxyCreator的超类AbstractAutoProxyCreator类的postProcessBeforeInstantiation方法，调用createProxy方法创建代理对象
```
/**
 * Create an AOP proxy for the given bean.
 * @param beanClass the class of the bean
 * @param beanName the name of the bean
 * @param specificInterceptors the set of interceptors that is
 * specific to this bean (may be empty, but not null)
 * @param targetSource the TargetSource for the proxy,
 * already pre-configured to access the bean
 * @return the AOP proxy for the bean
 * @see #buildAdvisors
 */
protected Object createProxy(Class<?> beanClass, @Nullable String beanName,
		@Nullable Object[] specificInterceptors, TargetSource targetSource) {

	if (this.beanFactory instanceof ConfigurableListableBeanFactory) {
		AutoProxyUtils.exposeTargetClass((ConfigurableListableBeanFactory) this.beanFactory, beanName, beanClass);
	}

	ProxyFactory proxyFactory = new ProxyFactory();
	proxyFactory.copyFrom(this);

	if (!proxyFactory.isProxyTargetClass()) {
		if (shouldProxyTargetClass(beanClass, beanName)) {
			proxyFactory.setProxyTargetClass(true);
		}
		else {
			evaluateProxyInterfaces(beanClass, proxyFactory);
		}
	}

	Advisor[] advisors = buildAdvisors(beanName, specificInterceptors);
	proxyFactory.addAdvisors(advisors);
	proxyFactory.setTargetSource(targetSource);
	customizeProxyFactory(proxyFactory);

	proxyFactory.setFrozen(this.freezeProxy);
	if (advisorsPreFiltered()) {
		proxyFactory.setPreFiltered(true);
	}

	return proxyFactory.getProxy(getProxyClassLoader());
}

```
## 调用buildAdvisors方法封装对应的Advisor。使用DefaultAdvisorAdapterRegistry来根据注解类型生成DefaultPointcutAdvisor，MethodBeforeAdviceAdapter，MethodBeforeAdviceAdapter，ThrowsAdviceAdapter

```
/**
 * Determine the advisors for the given bean, including the specific interceptors
 * as well as the common interceptor, all adapted to the Advisor interface.
 * @param beanName the name of the bean
 * @param specificInterceptors the set of interceptors that is
 * specific to this bean (may be empty, but not null)
 * @return the list of Advisors for the given bean
 */
protected Advisor[] buildAdvisors(@Nullable String beanName, @Nullable Object[] specificInterceptors) {
	// Handle prototypes correctly...
	Advisor[] commonInterceptors = resolveInterceptorNames();

	List<Object> allInterceptors = new ArrayList<>();
	if (specificInterceptors != null) {
		allInterceptors.addAll(Arrays.asList(specificInterceptors));
		if (commonInterceptors.length > 0) {
			if (this.applyCommonInterceptorsFirst) {
				allInterceptors.addAll(0, Arrays.asList(commonInterceptors));
			}
			else {
				allInterceptors.addAll(Arrays.asList(commonInterceptors));
			}
		}
	}
	if (logger.isDebugEnabled()) {
		int nrOfCommonInterceptors = commonInterceptors.length;
		int nrOfSpecificInterceptors = (specificInterceptors != null ? specificInterceptors.length : 0);
		logger.debug("Creating implicit proxy for bean '" + beanName + "' with " + nrOfCommonInterceptors +
				" common interceptors and " + nrOfSpecificInterceptors + " specific interceptors");
	}

	Advisor[] advisors = new Advisor[allInterceptors.size()];
	for (int i = 0; i < allInterceptors.size(); i++) {
		advisors[i] = this.advisorAdapterRegistry.wrap(allInterceptors.get(i));
	}
	return advisors;
}
```
## DefaultAdvisorAdapterRegistry类如下。因为返回specificInterceptors都为InstantiationModelAwarePointcutAdvisorImpl，在wrap方法之中直接返回该Advisor
```
public class DefaultAdvisorAdapterRegistry implements AdvisorAdapterRegistry, Serializable {

	private final List<AdvisorAdapter> adapters = new ArrayList<>(3);


	/**
	 * Create a new DefaultAdvisorAdapterRegistry, registering well-known adapters.
	 */
	public DefaultAdvisorAdapterRegistry() {
		registerAdvisorAdapter(new MethodBeforeAdviceAdapter());
		registerAdvisorAdapter(new AfterReturningAdviceAdapter());
		registerAdvisorAdapter(new ThrowsAdviceAdapter());
	}


	@Override
	public Advisor wrap(Object adviceObject) throws UnknownAdviceTypeException {
		if (adviceObject instanceof Advisor) {//adviceObject为InstantiationModelAwarePointcutAdvisorImpl
			return (Advisor) adviceObject;
		}
		if (!(adviceObject instanceof Advice)) {
			throw new UnknownAdviceTypeException(adviceObject);
		}
		Advice advice = (Advice) adviceObject;
		if (advice instanceof MethodInterceptor) {
			// So well-known it doesn't even need an adapter.
			return new DefaultPointcutAdvisor(advice);
		}
		for (AdvisorAdapter adapter : this.adapters) {
			// Check that it is supported.
			if (adapter.supportsAdvice(advice)) {
				return new DefaultPointcutAdvisor(advice);
			}
		}
		throw new UnknownAdviceTypeException(advice);
	}

	@Override
	public MethodInterceptor[] getInterceptors(Advisor advisor) throws UnknownAdviceTypeException {
		List<MethodInterceptor> interceptors = new ArrayList<>(3);
		Advice advice = advisor.getAdvice();
		if (advice instanceof MethodInterceptor) {
			interceptors.add((MethodInterceptor) advice);
		}
		for (AdvisorAdapter adapter : this.adapters) {
			if (adapter.supportsAdvice(advice)) {
				interceptors.add(adapter.getInterceptor(advisor));
			}
		}
		if (interceptors.isEmpty()) {
			throw new UnknownAdviceTypeException(advisor.getAdvice());
		}
		return interceptors.toArray(new MethodInterceptor[0]);
	}

	@Override
	public void registerAdvisorAdapter(AdvisorAdapter adapter) {
		this.adapters.add(adapter);
	}

}
```
## 使用ProxyFactory来创建代理对象
```
/**
 * Create a new proxy according to the settings in this factory.
 * <p>Can be called repeatedly. Effect will vary if we've added
 * or removed interfaces. Can add and remove interceptors.
 * <p>Uses the given class loader (if necessary for proxy creation).
 * @param classLoader the class loader to create the proxy with
 * (or {@code null} for the low-level proxy facility's default)
 * @return the proxy object
 */
public Object getProxy(@Nullable ClassLoader classLoader) {
	return createAopProxy().getProxy(classLoader);
}
```
## 在ProxyFactory超类ProxyCreatorSupport之中。最后调用DefaultAopProxyFactory创建代理对象
```
/**
 * Subclasses should call this to get a new AOP proxy. They should <b>not</b>
 * create an AOP proxy with {@code this} as an argument.
 */
protected final synchronized AopProxy createAopProxy() {
	if (!this.active) {
		activate();
	}
	//getAopProxyFactory方法返回DefaultAopProxyFactory
	return getAopProxyFactory().createAopProxy(this);
}

```

## DefaultAopProxyFactory类如下。如果Class实现了接口则JdkDynamicAopProxy类生成代理对象，否则使用ObjenesisCglibAopProxy创建代理对象

```
public class DefaultAopProxyFactory implements AopProxyFactory, Serializable {

	@Override
	public AopProxy createAopProxy(AdvisedSupport config) throws AopConfigException {
		if (config.isOptimize() || config.isProxyTargetClass() || hasNoUserSuppliedProxyInterfaces(config)) {
			Class<?> targetClass = config.getTargetClass();
			if (targetClass == null) {
				throw new AopConfigException("TargetSource cannot determine target class: " +
						"Either an interface or a target is required for proxy creation.");
			}
			if (targetClass.isInterface() || Proxy.isProxyClass(targetClass)) {
				return new JdkDynamicAopProxy(config);
			}
			return new ObjenesisCglibAopProxy(config);
		}
		else {
			return new JdkDynamicAopProxy(config);
		}
	}

	/**
	 * Determine whether the supplied {@link AdvisedSupport} has only the
	 * {@link org.springframework.aop.SpringProxy} interface specified
	 * (or no proxy interfaces specified at all).
	 */
	private boolean hasNoUserSuppliedProxyInterfaces(AdvisedSupport config) {
		Class<?>[] ifcs = config.getProxiedInterfaces();
		return (ifcs.length == 0 || (ifcs.length == 1 && SpringProxy.class.isAssignableFrom(ifcs[0])));
	}

}
```

```java

private static class DynamicAdvisedInterceptor implements MethodInterceptor, Serializable {

private final AdvisedSupport advised;

public DynamicAdvisedInterceptor(AdvisedSupport advised) {
	this.advised = advised;
}

@Override
@Nullable
public Object intercept(Object proxy, Method method, Object[] args, MethodProxy methodProxy) throws Throwable {
	Object oldProxy = null;
	boolean setProxyContext = false;
	Object target = null;
	TargetSource targetSource = this.advised.getTargetSource();
	try {
		if (this.advised.exposeProxy) {
			// Make invocation available if necessary.
			oldProxy = AopContext.setCurrentProxy(proxy);
			setProxyContext = true;
		}
		// Get as late as possible to minimize the time we "own" the target, in case it comes from a pool...
		target = targetSource.getTarget();
		Class<?> targetClass = (target != null ? target.getClass() : null);
		List<Object> chain = this.advised.getInterceptorsAndDynamicInterceptionAdvice(method, targetClass);
		Object retVal;
		// Check whether we only have one InvokerInterceptor: that is,
		// no real advice, but just reflective invocation of the target.
		if (chain.isEmpty() && Modifier.isPublic(method.getModifiers())) {
			// We can skip creating a MethodInvocation: just invoke the target directly.
			// Note that the final invoker must be an InvokerInterceptor, so we know
			// it does nothing but a reflective operation on the target, and no hot
			// swapping or fancy proxying.
			Object[] argsToUse = AopProxyUtils.adaptArgumentsIfNecessary(method, args);
			retVal = methodProxy.invoke(target, argsToUse);
		}
		else {
			// We need to create a method invocation...
			retVal = new CglibMethodInvocation(proxy, target, method, args, targetClass, chain, methodProxy).proceed();
		}
		retVal = processReturnType(proxy, target, method, retVal);
		return retVal;
	}
	finally {
		if (target != null && !targetSource.isStatic()) {
			targetSource.releaseTarget(target);
		}
		if (setProxyContext) {
			// Restore old proxy.
			AopContext.setCurrentProxy(oldProxy);
		}
	}
}

@Override
public boolean equals(Object other) {
	return (this == other ||
			(other instanceof DynamicAdvisedInterceptor &&
					this.advised.equals(((DynamicAdvisedInterceptor) other).advised)));
}

/**
 * CGLIB uses this to drive proxy creation.
 */
@Override
public int hashCode() {
	return this.advised.hashCode();
}
}
```
