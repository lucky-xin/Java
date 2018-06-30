# SpringBoot事务实现原理

## 基于Spring AOP实现 [AOP解析](https://github.com/lucky-xin/Learning/blob/gh-pages/md/Spring%E6%BA%90%E7%A0%81%E5%AD%A6%E4%B9%A0%E4%B9%8BAOP.md)

## 基于AOP实现，把有Transactional注解的方法封装成Advisor，提交事务之前先保存提交事务之前状态（TransactionStatus），然后提交事务。如果事务执行失败，则回滚到保存状态点TransactionStatus。成功就不回滚。具体实现见源码分析。

## 1.通过注解`@EnableTransactionManagement`打开事务配置，通过ProxyTransactionManagementConfiguration注册AdvisorBeanFactoryTransactionAttributeSourceAdvisor
```java
@SpringBootApplication(scanBasePackages = "com.xin.springboot")
@ServletComponentScan//扫描监听类
@EnableAspectJAutoProxy//打开AOP自动配置
@EnableTransactionManagement//打开事务自动配置
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
## 2.EnableTransactionManagement注解导入TransactionManagementConfigurationSelector类完成相关配置。
```java
public class TransactionManagementConfigurationSelector extends AdviceModeImportSelector<EnableTransactionManagement> {

	/**
	 * {@inheritDoc}
	 * @return {@link ProxyTransactionManagementConfiguration} or
	 * {@code AspectJTransactionManagementConfiguration} for {@code PROXY} and
	 * {@code ASPECTJ} values of {@link EnableTransactionManagement#mode()}, respectively
	 */
	@Override
	protected String[] selectImports(AdviceMode adviceMode) {
		switch (adviceMode) {
			case PROXY://默认配置
				return new String[] {AutoProxyRegistrar.class.getName(), 
				ProxyTransactionManagementConfiguration.class.getName()};
			case ASPECTJ:
				return new String[] {TransactionManagementConfigUtils.TRANSACTION_ASPECT_CONFIGURATION_CLASS_NAME};
			default:
				return null;
		}
	}

}
```
## 3.使用ProxyTransactionManagementConfiguration类来完成配置。把BeanFactoryTransactionAttributeSourceAdvisor注册到IoC之中,
生成AOP代理时会去扫描所有Advisor实现类。通过Advisor的成员变量Pointcut来确定当前的bean的某个方法是否和Advisor匹配(通过matches方法判断)，如果
匹配则为当前类生成代理类，把匹配的方法封装成拦截器MethodInterceptor。
```java
@Configuration
public class ProxyTransactionManagementConfiguration extends AbstractTransactionManagementConfiguration {

    // 注册Advisor到IoC容器之中
	@Bean(name = TransactionManagementConfigUtils.TRANSACTION_ADVISOR_BEAN_NAME)
	@Role(BeanDefinition.ROLE_INFRASTRUCTURE)
	public BeanFactoryTransactionAttributeSourceAdvisor transactionAdvisor() {
		BeanFactoryTransactionAttributeSourceAdvisor advisor = new BeanFactoryTransactionAttributeSourceAdvisor();
		advisor.setTransactionAttributeSource(transactionAttributeSource());
		// 配置Advice
		advisor.setAdvice(transactionInterceptor());
		if (this.enableTx != null) {
			advisor.setOrder(this.enableTx.<Integer>getNumber("order"));
		}
		return advisor;
	}

	@Bean
	@Role(BeanDefinition.ROLE_INFRASTRUCTURE)
	public TransactionAttributeSource transactionAttributeSource() {
		return new AnnotationTransactionAttributeSource();
	}

	@Bean
	@Role(BeanDefinition.ROLE_INFRASTRUCTURE)
	public TransactionInterceptor transactionInterceptor() {
		TransactionInterceptor interceptor = new TransactionInterceptor();
		interceptor.setTransactionAttributeSource(transactionAttributeSource());
		if (this.txManager != null) {
			interceptor.setTransactionManager(this.txManager);
		}
		return interceptor;
	}

}
```
## 4.判断是否要为bean的方法生成事务方法。AOP实现原理判断是否为方法生成拦截器，是通过Advisor的成员变量Pointcut的通过matches
方法判断来判断如果matches返回true则生成拦截器。BeanFactoryTransactionAttributeSourceAdvisor的Pointcut为TransactionAttributeSourcePointcut。
BeanFactoryTransactionAttributeSourceAdvisor源码如下
```java
public class BeanFactoryTransactionAttributeSourceAdvisor extends AbstractBeanFactoryPointcutAdvisor {

	@Nullable
	private TransactionAttributeSource transactionAttributeSource;

	private final TransactionAttributeSourcePointcut pointcut = new TransactionAttributeSourcePointcut() {
		@Override
		@Nullable
		protected TransactionAttributeSource getTransactionAttributeSource() {
			return transactionAttributeSource;
		}
	};


	/**
	 * Set the transaction attribute source which is used to find transaction
	 * attributes. This should usually be identical to the source reference
	 * set on the transaction interceptor itself.
	 * @see TransactionInterceptor#setTransactionAttributeSource
	 */
	public void setTransactionAttributeSource(TransactionAttributeSource transactionAttributeSource) {
		this.transactionAttributeSource = transactionAttributeSource;
	}

	/**
	 * Set the {@link ClassFilter} to use for this pointcut.
	 * Default is {@link ClassFilter#TRUE}.
	 */
	public void setClassFilter(ClassFilter classFilter) {
		this.pointcut.setClassFilter(classFilter);
	}

	@Override
	public Pointcut getPointcut() {
		return this.pointcut;
	}

}
```

## BeanFactoryTransactionAttributeSourceAdvisor的Pointcut为TransactionAttributeSourcePointcut。看源码知
TransactionAttributeSourcePointcut判断是否为方法生成拦截器时委派到TransactionAttributeSource类的getTransactionAttribute方法之中。
在配置类ProxyTransactionManagementConfiguration之中可知TransactionAttributeSource为AnnotationTransactionAttributeSource

```java
abstract class TransactionAttributeSourcePointcut extends StaticMethodMatcherPointcut implements Serializable {

	@Override
	public boolean matches(Method method, @Nullable Class<?> targetClass) {
		if (targetClass != null && TransactionalProxy.class.isAssignableFrom(targetClass)) {
			return false;
		}
		TransactionAttributeSource tas = getTransactionAttributeSource();
		return (tas == null || tas.getTransactionAttribute(method, targetClass) != null);
	}

	@Override
	public boolean equals(Object other) {
		if (this == other) {
			return true;
		}
		if (!(other instanceof TransactionAttributeSourcePointcut)) {
			return false;
		}
		TransactionAttributeSourcePointcut otherPc = (TransactionAttributeSourcePointcut) other;
		return ObjectUtils.nullSafeEquals(getTransactionAttributeSource(), otherPc.getTransactionAttributeSource());
	}

	@Override
	public int hashCode() {
		return TransactionAttributeSourcePointcut.class.hashCode();
	}

	@Override
	public String toString() {
		return getClass().getName() + ": " + getTransactionAttributeSource();
	}


	/**
	 * Obtain the underlying TransactionAttributeSource (may be {@code null}).
	 * To be implemented by subclasses.
	 */
	@Nullable
	protected abstract TransactionAttributeSource getTransactionAttributeSource();

}
```

## AnnotationTransactionAttributeSource类结构如下图![](https://github.com/lucky-xin/Learning/blob/gh-pages/image/AnnotationTransactionAttributeSource.png)
getTransactionAttribute方法具体在超类AbstractFallbackTransactionAttributeSource之中
```java_holder_method_tree
public TransactionAttribute getTransactionAttribute(Method method, @Nullable Class<?> targetClass) {
		if (method.getDeclaringClass() == Object.class) {
			return null;
		}

		// First, see if we have a cached value.
		Object cacheKey = getCacheKey(method, targetClass);
		Object cached = this.attributeCache.get(cacheKey);
		if (cached != null) {
			// Value will either be canonical value indicating there is no transaction attribute,
			// or an actual transaction attribute.
			if (cached == NULL_TRANSACTION_ATTRIBUTE) {
				return null;
			}
			else {
				return (TransactionAttribute) cached;
			}
		}
		else {
			// We need to work it out.
			// 查找Transactional注解入口
			TransactionAttribute txAttr = computeTransactionAttribute(method, targetClass);
			// Put it in the cache.
			if (txAttr == null) {
				this.attributeCache.put(cacheKey, NULL_TRANSACTION_ATTRIBUTE);
			}
			else {
				String methodIdentification = ClassUtils.getQualifiedMethodName(method, targetClass);
				if (txAttr instanceof DefaultTransactionAttribute) {
					((DefaultTransactionAttribute) txAttr).setDescriptor(methodIdentification);
				}
				if (logger.isDebugEnabled()) {
					logger.debug("Adding transactional method '" + methodIdentification + "' with attribute: " + txAttr);
				}
				this.attributeCache.put(cacheKey, txAttr);
			}
			return txAttr;
		}
	}
```
```java_holder_method_tree
protected TransactionAttribute computeTransactionAttribute(Method method, @Nullable Class<?> targetClass) {
		// Don't allow no-public methods as required.
		if (allowPublicMethodsOnly() && !Modifier.isPublic(method.getModifiers())) {
			return null;
		}

		// The method may be on an interface, but we need attributes from the target class.
		// If the target class is null, the method will be unchanged.
		Method specificMethod = AopUtils.getMostSpecificMethod(method, targetClass);

		// First try is the method in the target class.
		// 查找方法是否有Transactional注解，findTransactionAttribute为抽象方法具体实现在子类
		TransactionAttribute txAttr = findTransactionAttribute(specificMethod);
		if (txAttr != null) {
			return txAttr;
		}

		// Second try is the transaction attribute on the target class.
		txAttr = findTransactionAttribute(specificMethod.getDeclaringClass());
		if (txAttr != null && ClassUtils.isUserLevelMethod(method)) {
			return txAttr;
		}

		if (specificMethod != method) {
			// Fallback is to look at the original method.
			txAttr = findTransactionAttribute(method);
			if (txAttr != null) {
				return txAttr;
			}
			// Last fallback is the class of the original method.
			txAttr = findTransactionAttribute(method.getDeclaringClass());
			if (txAttr != null && ClassUtils.isUserLevelMethod(method)) {
				return txAttr;
			}
		}

		return null;
	}
```
## 最后查找Transactional注解任务分派到AnnotationTransactionAttributeSource类中使用SpringTransactionAnnotationParser解析注解
如果SpringTransactionAnnotationParser找到Transactional注解则解析注解获取相关配置。
```java_holder_method_tree
    @Override
	@Nullable
	protected TransactionAttribute findTransactionAttribute(Method method) {
		return determineTransactionAttribute(method);
	}
	
	@Nullable
    protected TransactionAttribute determineTransactionAttribute(AnnotatedElement ae) {
    	for (TransactionAnnotationParser annotationParser : this.annotationParsers) {
    			TransactionAttribute attr = annotationParser.parseTransactionAnnotation(ae);
    			if (attr != null) {
    				return attr;
    			}
    	}
    	return null;
    }
    	
```
## SpringTransactionAnnotationParser找到Transactional注解则解析注解获取相关配置，并使用TransactionAttribute封装注解相关信息然后返回
```java
public class SpringTransactionAnnotationParser implements TransactionAnnotationParser, Serializable {

	@Override
	@Nullable
	public TransactionAttribute parseTransactionAnnotation(AnnotatedElement ae) {
		AnnotationAttributes attributes = AnnotatedElementUtils.findMergedAnnotationAttributes(
				ae, Transactional.class, false, false);
		if (attributes != null) {
			return parseTransactionAnnotation(attributes);
		}
		else {
			return null;
		}
	}

	public TransactionAttribute parseTransactionAnnotation(Transactional ann) {
		return parseTransactionAnnotation(AnnotationUtils.getAnnotationAttributes(ann, false, false));
	}

	protected TransactionAttribute parseTransactionAnnotation(AnnotationAttributes attributes) {
		RuleBasedTransactionAttribute rbta = new RuleBasedTransactionAttribute();
		Propagation propagation = attributes.getEnum("propagation");
		rbta.setPropagationBehavior(propagation.value());
		Isolation isolation = attributes.getEnum("isolation");
		rbta.setIsolationLevel(isolation.value());
		rbta.setTimeout(attributes.getNumber("timeout").intValue());
		rbta.setReadOnly(attributes.getBoolean("readOnly"));
		rbta.setQualifier(attributes.getString("value"));
		ArrayList<RollbackRuleAttribute> rollBackRules = new ArrayList<>();
		Class<?>[] rbf = attributes.getClassArray("rollbackFor");
		for (Class<?> rbRule : rbf) {
			RollbackRuleAttribute rule = new RollbackRuleAttribute(rbRule);
			rollBackRules.add(rule);
		}
		String[] rbfc = attributes.getStringArray("rollbackForClassName");
		for (String rbRule : rbfc) {
			RollbackRuleAttribute rule = new RollbackRuleAttribute(rbRule);
			rollBackRules.add(rule);
		}
		Class<?>[] nrbf = attributes.getClassArray("noRollbackFor");
		for (Class<?> rbRule : nrbf) {
			NoRollbackRuleAttribute rule = new NoRollbackRuleAttribute(rbRule);
			rollBackRules.add(rule);
		}
		String[] nrbfc = attributes.getStringArray("noRollbackForClassName");
		for (String rbRule : nrbfc) {
			NoRollbackRuleAttribute rule = new NoRollbackRuleAttribute(rbRule);
			rollBackRules.add(rule);
		}
		rbta.getRollbackRules().addAll(rollBackRules);
		return rbta;
	}

	@Override
	public boolean equals(Object other) {
		return (this == other || other instanceof SpringTransactionAnnotationParser);
	}

	@Override
	public int hashCode() {
		return SpringTransactionAnnotationParser.class.hashCode();
	}

}
```
## 判断bean匹配Advisor之后为bean生成AOP代理对象。为方法生成拦截器之后。接下来调用bean对象的事务方法，
实际调用AOP代理对象的拦截方法。AOP代理对象方法调用时使用Advice（切面）实现拦截。最后把切面封装成MethodInterceptor。通过MethodInterceptor的invoke来实现拦截。Spring 事务拦截器为
TransactionInterceptor。TransactionInterceptor类结构图如下![](https://github.com/lucky-xin/Learning/blob/gh-pages/image/TransactionInterceptor.png)
```java
public class TransactionInterceptor extends TransactionAspectSupport implements MethodInterceptor, Serializable {

	/**
	 * Create a new TransactionInterceptor.
	 * <p>Transaction manager and transaction attributes still need to be set.
	 * @see #setTransactionManager
	 * @see #setTransactionAttributes(java.util.Properties)
	 * @see #setTransactionAttributeSource(TransactionAttributeSource)
	 */
	public TransactionInterceptor() {
	}

	/**
	 * Create a new TransactionInterceptor.
	 * @param ptm the default transaction manager to perform the actual transaction management
	 * @param attributes the transaction attributes in properties format
	 * @see #setTransactionManager
	 * @see #setTransactionAttributes(java.util.Properties)
	 */
	public TransactionInterceptor(PlatformTransactionManager ptm, Properties attributes) {
		setTransactionManager(ptm);
		setTransactionAttributes(attributes);
	}

	/**
	 * Create a new TransactionInterceptor.
	 * @param ptm the default transaction manager to perform the actual transaction management
	 * @param tas the attribute source to be used to find transaction attributes
	 * @see #setTransactionManager
	 * @see #setTransactionAttributeSource(TransactionAttributeSource)
	 */
	public TransactionInterceptor(PlatformTransactionManager ptm, TransactionAttributeSource tas) {
		setTransactionManager(ptm);
		setTransactionAttributeSource(tas);
	}


	@Override
	@Nullable
	public Object invoke(final MethodInvocation invocation) throws Throwable {
		// Work out the target class: may be {@code null}.
		// The TransactionAttributeSource should be passed the target class
		// as well as the method, which may be from an interface.
		Class<?> targetClass = (invocation.getThis() != null ? AopUtils.getTargetClass(invocation.getThis()) : null);

		// Adapt to TransactionAspectSupport's invokeWithinTransaction...
		// 事务执行方法
		return invokeWithinTransaction(invocation.getMethod(), targetClass, invocation::proceed);
	}


	//---------------------------------------------------------------------
	// Serialization support
	//---------------------------------------------------------------------

	private void writeObject(ObjectOutputStream oos) throws IOException {
		// Rely on default serialization, although this class itself doesn't carry state anyway...
		oos.defaultWriteObject();

		// Deserialize superclass fields.
		oos.writeObject(getTransactionManagerBeanName());
		oos.writeObject(getTransactionManager());
		oos.writeObject(getTransactionAttributeSource());
		oos.writeObject(getBeanFactory());
	}

	private void readObject(ObjectInputStream ois) throws IOException, ClassNotFoundException {
		// Rely on default serialization, although this class itself doesn't carry state anyway...
		ois.defaultReadObject();

		// Serialize all relevant superclass fields.
		// Superclass can't implement Serializable because it also serves as base class
		// for AspectJ aspects (which are not allowed to implement Serializable)!
		setTransactionManagerBeanName((String) ois.readObject());
		setTransactionManager((PlatformTransactionManager) ois.readObject());
		setTransactionAttributeSource((TransactionAttributeSource) ois.readObject());
		setBeanFactory((BeanFactory) ois.readObject());
	}

}

```
## 事务执行方法具体实现在事务拦截器TransactionInterceptor的超类TransactionAspectSupport的invokeWithinTransaction方法之中.
通过createTransactionIfNecessary方法创建TransactionInfo对象保存事务提交之前状态（TransactionStatus），如果异常则回滚到事务执行之前的TransactionInfo保存消息点。
```java_holder_method_tree
protected Object invokeWithinTransaction(Method method, @Nullable Class<?> targetClass,
			final InvocationCallback invocation) throws Throwable {

		// If the transaction attribute is null, the method is non-transactional.
		TransactionAttributeSource tas = getTransactionAttributeSource();
		final TransactionAttribute txAttr = (tas != null ? tas.getTransactionAttribute(method, targetClass) : null);
		final PlatformTransactionManager tm = determineTransactionManager(txAttr);
		final String joinpointIdentification = methodIdentification(method, targetClass, txAttr);

		if (txAttr == null || !(tm instanceof CallbackPreferringPlatformTransactionManager)) {
			// Standard transaction demarcation with getTransaction and commit/rollback calls.
			TransactionInfo txInfo = createTransactionIfNecessary(tm, txAttr, joinpointIdentification);
			Object retVal = null;
			try {
				// This is an around advice: Invoke the next interceptor in the chain.
				// This will normally result in a target object being invoked.
				retVal = invocation.proceedWithInvocation();
			}
			catch (Throwable ex) {
				// target invocation exception
				completeTransactionAfterThrowing(txInfo, ex);
				throw ex;
			}
			finally {
				cleanupTransactionInfo(txInfo);
			}
			commitTransactionAfterReturning(txInfo);
			return retVal;
		}

		else {
			final ThrowableHolder throwableHolder = new ThrowableHolder();

			// It's a CallbackPreferringPlatformTransactionManager: pass a TransactionCallback in.
			try {
				Object result = ((CallbackPreferringPlatformTransactionManager) tm).execute(txAttr, status -> {
					TransactionInfo txInfo = prepareTransactionInfo(tm, txAttr, joinpointIdentification, status);
					try {
						return invocation.proceedWithInvocation();
					}
					catch (Throwable ex) {
						if (txAttr.rollbackOn(ex)) {
							// A RuntimeException: will lead to a rollback.
							if (ex instanceof RuntimeException) {
								throw (RuntimeException) ex;
							}
							else {
								throw new ThrowableHolderException(ex);
							}
						}
						else {
							// A normal return value: will lead to a commit.
							throwableHolder.throwable = ex;
							return null;
						}
					}
					finally {
						cleanupTransactionInfo(txInfo);
					}
				});

				// Check result state: It might indicate a Throwable to rethrow.
				if (throwableHolder.throwable != null) {
					throw throwableHolder.throwable;
				}
				return result;
			}
			catch (ThrowableHolderException ex) {
				throw ex.getCause();
			}
			catch (TransactionSystemException ex2) {
				if (throwableHolder.throwable != null) {
					logger.error("Application exception overridden by commit exception", throwableHolder.throwable);
					ex2.initApplicationException(throwableHolder.throwable);
				}
				throw ex2;
			}
			catch (Throwable ex2) {
				if (throwableHolder.throwable != null) {
					logger.error("Application exception overridden by commit exception", throwableHolder.throwable);
				}
				throw ex2;
			}
		}
	}

```
## 创建TransactionInfo对象封装事务提交之前信息，TransactionInfo持有TransactionStatus对象，TransactionStatus封装了保存点信息。
```java

public abstract class AbstractTransactionStatus implements TransactionStatus {

	private boolean rollbackOnly = false;

	private boolean completed = false;

	@Nullable
	private Object savepoint;


	//---------------------------------------------------------------------
	// Handling of current transaction state
	//---------------------------------------------------------------------

	@Override
	public void setRollbackOnly() {
		this.rollbackOnly = true;
	}

	/**
	 * Determine the rollback-only flag via checking both the local rollback-only flag
	 * of this TransactionStatus and the global rollback-only flag of the underlying
	 * transaction, if any.
	 * @see #isLocalRollbackOnly()
	 * @see #isGlobalRollbackOnly()
	 */
	@Override
	public boolean isRollbackOnly() {
		return (isLocalRollbackOnly() || isGlobalRollbackOnly());
	}

	/**
	 * Determine the rollback-only flag via checking this TransactionStatus.
	 * <p>Will only return "true" if the application called {@code setRollbackOnly}
	 * on this TransactionStatus object.
	 */
	public boolean isLocalRollbackOnly() {
		return this.rollbackOnly;
	}

	/**
	 * Template method for determining the global rollback-only flag of the
	 * underlying transaction, if any.
	 * <p>This implementation always returns {@code false}.
	 */
	public boolean isGlobalRollbackOnly() {
		return false;
	}

	/**
	 * This implementations is empty, considering flush as a no-op.
	 */
	@Override
	public void flush() {
	}

	/**
	 * Mark this transaction as completed, that is, committed or rolled back.
	 */
	public void setCompleted() {
		this.completed = true;
	}

	@Override
	public boolean isCompleted() {
		return this.completed;
	}


	//---------------------------------------------------------------------
	// Handling of current savepoint state
	//---------------------------------------------------------------------

	/**
	 * Set a savepoint for this transaction. Useful for PROPAGATION_NESTED.
	 * @see org.springframework.transaction.TransactionDefinition#PROPAGATION_NESTED
	 */
	protected void setSavepoint(@Nullable Object savepoint) {
		this.savepoint = savepoint;
	}

	/**
	 * Get the savepoint for this transaction, if any.
	 */
	@Nullable
	protected Object getSavepoint() {
		return this.savepoint;
	}

	@Override
	public boolean hasSavepoint() {
		return (this.savepoint != null);
	}

	/**
	 * Create a savepoint and hold it for the transaction.
	 * @throws org.springframework.transaction.NestedTransactionNotSupportedException
	 * if the underlying transaction does not support savepoints
	 */
	public void createAndHoldSavepoint() throws TransactionException {
		setSavepoint(getSavepointManager().createSavepoint());
	}

	/**
	 * Roll back to the savepoint that is held for the transaction
	 * and release the savepoint right afterwards.
	 */
	public void rollbackToHeldSavepoint() throws TransactionException {
		Object savepoint = getSavepoint();
		if (savepoint == null) {
			throw new TransactionUsageException(
					"Cannot roll back to savepoint - no savepoint associated with current transaction");
		}
		getSavepointManager().rollbackToSavepoint(savepoint);
		getSavepointManager().releaseSavepoint(savepoint);
		setSavepoint(null);
	}

	/**
	 * Release the savepoint that is held for the transaction.
	 */
	public void releaseHeldSavepoint() throws TransactionException {
		Object savepoint = getSavepoint();
		if (savepoint == null) {
			throw new TransactionUsageException(
					"Cannot release savepoint - no savepoint associated with current transaction");
		}
		getSavepointManager().releaseSavepoint(savepoint);
		setSavepoint(null);
	}


	//---------------------------------------------------------------------
	// Implementation of SavepointManager
	//---------------------------------------------------------------------

	/**
	 * This implementation delegates to a SavepointManager for the
	 * underlying transaction, if possible.
	 * @see #getSavepointManager()
	 * @see SavepointManager#createSavepoint()
	 */
	@Override
	public Object createSavepoint() throws TransactionException {
		return getSavepointManager().createSavepoint();
	}

	/**
	 * This implementation delegates to a SavepointManager for the
	 * underlying transaction, if possible.
	 * @see #getSavepointManager()
	 * @see SavepointManager#rollbackToSavepoint(Object)
	 */
	@Override
	public void rollbackToSavepoint(Object savepoint) throws TransactionException {
		getSavepointManager().rollbackToSavepoint(savepoint);
	}

	/**
	 * This implementation delegates to a SavepointManager for the
	 * underlying transaction, if possible.
	 * @see #getSavepointManager()
	 * @see SavepointManager#releaseSavepoint(Object)
	 */
	@Override
	public void releaseSavepoint(Object savepoint) throws TransactionException {
		getSavepointManager().releaseSavepoint(savepoint);
	}

	/**
	 * Return a SavepointManager for the underlying transaction, if possible.
	 * <p>Default implementation always throws a NestedTransactionNotSupportedException.
	 * @throws org.springframework.transaction.NestedTransactionNotSupportedException
	 * if the underlying transaction does not support savepoints
	 */
	protected SavepointManager getSavepointManager() {
		throw new NestedTransactionNotSupportedException("This transaction does not support savepoints");
	}
	
}

```
## 创建TransactionInfo对象
```java_holder_method_tree
protected TransactionInfo createTransactionIfNecessary(@Nullable PlatformTransactionManager tm,
			@Nullable TransactionAttribute txAttr, final String joinpointIdentification) {

		// If no name specified, apply method identification as transaction name.
		if (txAttr != null && txAttr.getName() == null) {
			txAttr = new DelegatingTransactionAttribute(txAttr) {
				@Override
				public String getName() {
					return joinpointIdentification;
				}
			};
		}

		TransactionStatus status = null;
		if (txAttr != null) {
			if (tm != null) {
				status = tm.getTransaction(txAttr);
			}
			else {
				if (logger.isDebugEnabled()) {
					logger.debug("Skipping transactional joinpoint [" + joinpointIdentification +
							"] because no transaction manager has been configured");
				}
			}
		}
		return prepareTransactionInfo(tm, txAttr, joinpointIdentification, status);
	}
```

## 发生了异常回滚事务,completeTransactionAfterThrowing方法源码。如下
```java_holder_method_tree
protected void completeTransactionAfterThrowing(@Nullable TransactionInfo txInfo, Throwable ex) {
		if (txInfo != null && txInfo.getTransactionStatus() != null) {
			if (logger.isTraceEnabled()) {
				logger.trace("Completing transaction for [" + txInfo.getJoinpointIdentification() +
						"] after exception: " + ex);
			}
			if (txInfo.transactionAttribute != null && txInfo.transactionAttribute.rollbackOn(ex)) {
				try {
					txInfo.getTransactionManager().rollback(txInfo.getTransactionStatus());
				}
				catch (TransactionSystemException ex2) {
					logger.error("Application exception overridden by rollback exception", ex);
					ex2.initApplicationException(ex);
					throw ex2;
				}
				catch (RuntimeException | Error ex2) {
					logger.error("Application exception overridden by rollback exception", ex);
					throw ex2;
				}
			}
			else {
				// We don't roll back on this exception.
				// Will still roll back if TransactionStatus.isRollbackOnly() is true.
				try {
				// 调用AbstractPlatformTransactionManager的rollback方法
					txInfo.getTransactionManager().commit(txInfo.getTransactionStatus());
				}
				catch (TransactionSystemException ex2) {
					logger.error("Application exception overridden by commit exception", ex);
					ex2.initApplicationException(ex);
					throw ex2;
				}
				catch (RuntimeException | Error ex2) {
					logger.error("Application exception overridden by commit exception", ex);
					throw ex2;
				}
			}
		}
	}
```
## AbstractPlatformTransactionManager的rollback方法
```java_holder_method_tree
public final void rollback(TransactionStatus status) throws TransactionException {
		if (status.isCompleted()) {
			throw new IllegalTransactionStateException(
					"Transaction is already completed - do not call commit or rollback more than once per transaction");
		}

		DefaultTransactionStatus defStatus = (DefaultTransactionStatus) status;
		processRollback(defStatus, false);
	}

	/**
	 * Process an actual rollback.
	 * The completed flag has already been checked.
	 * @param status object representing the transaction
	 * @throws TransactionException in case of rollback failure
	 */
	private void processRollback(DefaultTransactionStatus status, boolean unexpected) {
		try {
			boolean unexpectedRollback = unexpected;

			try {
				triggerBeforeCompletion(status);

				if (status.hasSavepoint()) {
					if (status.isDebug()) {
						logger.debug("Rolling back transaction to savepoint");
					}
					// 回滚到保存点
					status.rollbackToHeldSavepoint();
				}
				else if (status.isNewTransaction()) {
					if (status.isDebug()) {
						logger.debug("Initiating transaction rollback");
					}
					doRollback(status);
				}
				else {
					// Participating in larger transaction
					if (status.hasTransaction()) {
						if (status.isLocalRollbackOnly() || isGlobalRollbackOnParticipationFailure()) {
							if (status.isDebug()) {
								logger.debug("Participating transaction failed - marking existing transaction as rollback-only");
							}
							doSetRollbackOnly(status);
						}
						else {
							if (status.isDebug()) {
								logger.debug("Participating transaction failed - letting transaction originator decide on rollback");
							}
						}
					}
					else {
						logger.debug("Should roll back transaction but cannot - no transaction available");
					}
					// Unexpected rollback only matters here if we're asked to fail early
					if (!isFailEarlyOnGlobalRollbackOnly()) {
						unexpectedRollback = false;
					}
				}
			}
			catch (RuntimeException | Error ex) {
				triggerAfterCompletion(status, TransactionSynchronization.STATUS_UNKNOWN);
				throw ex;
			}

			triggerAfterCompletion(status, TransactionSynchronization.STATUS_ROLLED_BACK);

			// Raise UnexpectedRollbackException if we had a global rollback-only marker
			if (unexpectedRollback) {
				throw new UnexpectedRollbackException(
						"Transaction rolled back because it has been marked as rollback-only");
			}
		}
		finally {
			cleanupAfterCompletion(status);
		}
	}
```
## 最后会调用TransactionStatus的回滚方法rollbackToHeldSavepoint回滚到保存点，恢复现场。
```java_holder_method_tree
public void rollbackToHeldSavepoint() throws TransactionException {
		Object savepoint = getSavepoint();
		if (savepoint == null) {
			throw new TransactionUsageException(
					"Cannot roll back to savepoint - no savepoint associated with current transaction");
		}
		getSavepointManager().rollbackToSavepoint(savepoint);
		getSavepointManager().releaseSavepoint(savepoint);
		setSavepoint(null);
	}
	@Override
    public void rollbackToSavepoint(Object savepoint) throws TransactionException {
    	getSavepointManager().rollbackToSavepoint(savepoint);
    }
```
## rollbackToSavepoint方法最终会调用java.sql.Connection的rollback(Savepoint)方法回滚到保存点。
JdbcTransactionObjectSupport回滚实现如下
```java_holder_method_tree
public void rollbackToSavepoint(Object savepoint) throws TransactionException {
        try {
            this.getConnectionHolderForSavepoint().getConnection().rollback((Savepoint)savepoint);
        } catch (Throwable var3) {
            throw new TransactionSystemException("Could not roll back to JDBC savepoint", var3);
        }
    }
```

