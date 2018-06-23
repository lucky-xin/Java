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
## 在AopConfigUtils之中完成注册如下
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
