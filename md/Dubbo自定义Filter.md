```text
使用了@PreAuthorize("@pms.hasPermission('PERMISSION_ADD_MENU')")来控制方法是否有访问权限时，
如果是SpringCloud微服务并且携带token访问，则上下文会SecurityContextHolder.getContext().getAuthentication()可以拿到
Authentication认证信息，如果是Dubbo则不行(RPC实现),需要自定义Filter来传递Authentication
```
```java
package biz.datainsights.common.security.filter;
/**
 * @author Luchaoxin
 * @version V 1.0
 * @Description: 自定义token校验filter, 用于Dubbo之间传递token,消费设置token
 * @date 2019-08-26 22:32
 */
@Activate(group = Constants.CONSUMER)
public class ConsumerTokenFilter implements Filter {

    private Logger logger = Logger.getLogger(ConsumerTokenFilter.class);

    @Setter
    private OAuth2ClientContext clientContext;

    @Override
    public Result invoke(Invoker<?> invoker, Invocation invocation) throws RpcException {
        if (Objects.nonNull(clientContext)) {
            OAuth2AccessToken accessToken = clientContext.getAccessToken();
            if (Objects.nonNull(accessToken)) {
                logger.info("token is:" + accessToken.getValue());
                RpcContext.getContext().setAttachment(HttpHeaders.AUTHORIZATION, accessToken.getValue());
            }
        }
        return invoker.invoke(invocation);
    }
}
```

```java
package biz.datainsights.common.security.filter;

/**
 * @author Luchaoxin
 * @version V 1.0
 * @Description: 自定义token校验filter, 用于Dubbo之间传递token，
 *   提供者从上下文获取并根据token加载OAuth2Authentication,并存入上下文
 * @date 2019-08-26 22:32
 */
@Activate(group = Constants.PROVIDER)
public class ProviderTokenFilter implements Filter {

    private Logger logger = Logger.getLogger(ProviderTokenFilter.class);

    @Setter
    private ResourceServerTokenServices services;

    @Override
    public Result invoke(Invoker<?> invoker, Invocation invocation) throws RpcException {
        String token = RpcContext.getContext().getAttachment(HttpHeaders.AUTHORIZATION);
        logger.info("token is:" + token);
        if (StringUtils.isNoneBlank(token) && Objects.nonNull(services)) {
            token = BearerTokenExtractorUtil.extractToken(token);
            if (StringUtils.isNoneBlank(token)) {
                logger.info("token is:" + token);
                OAuth2Authentication authResult = services.loadAuthentication(token);
                SecurityContextHolder.getContext().setAuthentication(authResult);
            }
        }
        Result result = invoker.invoke(invocation);
        SecurityContextHolder.clearContext();
        return result;
    }
}

```
效果如下
# Consumer 日志
```log
[INFO ] 2019-09-05 13:24:02,557 biz.datainsights.common.security.filter.ConsumerTokenFilter.invoke(ConsumerTokenFilter.java:33)
token is:bc3c159a-7285-4138-b65d-39442483c2c0
```

# Provider 日志
```log
[INFO ] 2019-09-05 05:24:02,562 method:biz.datainsights.common.security.filter.ProviderTokenFilter.invoke(ProviderTokenFilter.java:34)
token is:bc3c159a-7285-4138-b65d-39442483c2c0
```
# 具体实现认识Activate。group可以指定分组如provider或者consumer，value则可以控制url之中出现该参数才执行此Filter
```java
package org.apache.dubbo.common.extension;
/**
 * Activate. This annotation is useful for automatically activate certain extensions with the given criteria,
 * for examples: <code>@Activate</code> can be used to load certain <code>Filter</code> extension when there are
 * multiple implementations.
 * <ol>
 * <li>{@link Activate#group()} specifies group criteria. Framework SPI defines the valid group values.
 * <li>{@link Activate#value()} specifies parameter key in {@link URL} criteria.
 * </ol>
 * SPI provider can call {@link ExtensionLoader#getActivateExtension(URL, String, String)} to find out all activated
 * extensions with the given criteria.
 *
 * @see SPI
 * @see URL
 * @see ExtensionLoader
 */
@Documented
@Retention(RetentionPolicy.RUNTIME)
@Target({ElementType.TYPE, ElementType.METHOD})
public @interface Activate {
    /**
     * Activate the current extension when one of the groups matches. The group passed into
     * {@link ExtensionLoader#getActivateExtension(URL, String, String)} will be used for matching.
     *
     * @return group names to match
     * @see ExtensionLoader#getActivateExtension(URL, String, String)
     */
    String[] group() default {};

    /**
     * Activate the current extension when the specified keys appear in the URL's parameters.
     * <p>
     * For example, given <code>@Activate("cache, validation")</code>, the current extension will be return only when
     * there's either <code>cache</code> or <code>validation</code> key appeared in the URL's parameters.
     * </p>
     *
     * @return URL parameter keys
     * @see ExtensionLoader#getActivateExtension(URL, String)
     * @see ExtensionLoader#getActivateExtension(URL, String, String)
     */
    String[] value() default {};

    /**
     * Relative ordering info, optional
     * Deprecated since 2.7.0
     *
     * @return extension list which should be put before the current one
     */
    @Deprecated
    String[] before() default {};

    /**
     * Relative ordering info, optional
     * Deprecated since 2.7.0
     *
     * @return extension list which should be put after the current one
     */
    @Deprecated
    String[] after() default {};

    /**
     * Absolute ordering info, optional
     *
     * @return absolute ordering info
     */
    int order() default 0;
}
```
```text
Dobbo之中的Bean并没有使用Spring IoC进行管理，所以@Autowired也就不能注入对象了。
可以为需要注入的对象添加Setter方法既可以自动注入
具体实现：
Dobbu基于SPI实现，使用org.apache.dubbo.common.extension.ExtensionLoader来加载bean。具体实现
```
```java
    public T getAdaptiveExtension() {
        Object instance = cachedAdaptiveInstance.get();
        if (instance == null) {
            if (createAdaptiveInstanceError == null) {
                synchronized (cachedAdaptiveInstance) {
                    instance = cachedAdaptiveInstance.get();
                    if (instance == null) {
                        try {
                            instance = createAdaptiveExtension();
                            cachedAdaptiveInstance.set(instance);
                        } catch (Throwable t) {
                            createAdaptiveInstanceError = t;
                            throw new IllegalStateException("Failed to create adaptive instance: " + t.toString(), t);
                        }
                    }
                }
            } else {
                throw new IllegalStateException("Failed to create adaptive instance: " + createAdaptiveInstanceError.toString(), createAdaptiveInstanceError);
            }
        }

        return (T) instance;
    }

    private T createAdaptiveExtension() {
        try {
            // 创建对象之后委派injectExtension方法注入属性
            return injectExtension((T) getAdaptiveExtensionClass().newInstance());
        } catch (Exception e) {
            throw new IllegalStateException("Can't create adaptive extension " + type + ", cause: " + e.getMessage(), e);
        }
    }

    private T injectExtension(T instance) {
        try {
            if (objectFactory != null) {
                for (Method method : instance.getClass().getMethods()) {
                    // 遍历所有方法,如果是Setter方法且是public则往下执行
                    if (isSetter(method)) {
                        /**
                         * Check {@link DisableInject} to see if we need auto injection for this property
                         */
                        if (method.getAnnotation(DisableInject.class) != null) {
                            continue;
                        }
                        Class<?> pt = method.getParameterTypes()[0];
                        if (ReflectUtils.isPrimitives(pt)) {
                            continue;
                        }
                        try {
                        // 获取属性名称
                            String property = getSetterProperty(method);
                        //  private String getSetterProperty(Method method) {
                        //          return method.getName().length() > 3 ? method.getName().substring(3, 4).toLowerCase() + method.getName().substring(4) : "";
                        //      }
                            // 获取属性值objectFactory为org.apache.dubbo.config.spring.extension.SpringExtensionFactory
                            Object object = objectFactory.getExtension(pt, property);
                            if (object != null) {
                            // 使用setter方法设置值
                                method.invoke(instance, object);
                            }
                        } catch (Exception e) {
                            logger.error("Failed to inject via method " + method.getName()
                                    + " of interface " + type.getName() + ": " + e.getMessage(), e);
                        }
                    }
                }
            }
        } catch (Exception e) {
            logger.error(e.getMessage(), e);
        }
        return instance;
    }
```
# org.apache.dubbo.config.spring.extension.SpringExtensionFactory源码如下，持有ApplicationContext也就相当于IoC了，可以拿到Spring IoC之中对象，这样Dubbo就可以注入Spring Bean
```java
package org.apache.dubbo.config.spring.extension;

/**
 * SpringExtensionFactory
 */
public class SpringExtensionFactory implements ExtensionFactory {
    private static final Logger logger = LoggerFactory.getLogger(SpringExtensionFactory.class);

    private static final Set<ApplicationContext> contexts = new ConcurrentHashSet<ApplicationContext>();
    private static final ApplicationListener shutdownHookListener = new ShutdownHookListener();

    public static void addApplicationContext(ApplicationContext context) {
        contexts.add(context);
        if (context instanceof ConfigurableApplicationContext) {
            ((ConfigurableApplicationContext) context).registerShutdownHook();
            DubboShutdownHook.getDubboShutdownHook().unregister();
        }
        BeanFactoryUtils.addApplicationListener(context, shutdownHookListener);
    }

    public static void removeApplicationContext(ApplicationContext context) {
        contexts.remove(context);
    }

    public static Set<ApplicationContext> getContexts() {
        return contexts;
    }

    // currently for test purpose
    public static void clearContexts() {
        contexts.clear();
    }

    @Override
    @SuppressWarnings("unchecked")
    public <T> T getExtension(Class<T> type, String name) {

        //SPI should be get from SpiExtensionFactory
        if (type.isInterface() && type.isAnnotationPresent(SPI.class)) {
            return null;
        }
        // 遍历IoC判断是否有该对象
        for (ApplicationContext context : contexts) {
            if (context.containsBean(name)) {
                Object bean = context.getBean(name);
                if (type.isInstance(bean)) {
                    return (T) bean;
                }
            }
        }

        logger.warn("No spring extension (bean) named:" + name + ", try to find an extension (bean) of type " + type.getName());

        if (Object.class == type) {
            return null;
        }
        // 如果有该对象则尝试获取该对象并返回
        for (ApplicationContext context : contexts) {
            try {
                return context.getBean(type);
            } catch (NoUniqueBeanDefinitionException multiBeanExe) {
                logger.warn("Find more than 1 spring extensions (beans) of type " + type.getName() + ", will stop auto injection. Please make sure you have specified the concrete parameter type and there's only one extension of that type.");
            } catch (NoSuchBeanDefinitionException noBeanExe) {
                if (logger.isDebugEnabled()) {
                    logger.debug("Error when get spring extension(bean) for type:" + type.getName(), noBeanExe);
                }
            }
        }

        logger.warn("No spring extension (bean) named:" + name + ", type:" + type.getName() + " found, stop get bean.");

        return null;
    }

    private static class ShutdownHookListener implements ApplicationListener {
        @Override
        public void onApplicationEvent(ApplicationEvent event) {
            if (event instanceof ContextClosedEvent) {
                DubboShutdownHook shutdownHook = DubboShutdownHook.getDubboShutdownHook();
                shutdownHook.doDestroy();
            }
        }
    }
}

```
