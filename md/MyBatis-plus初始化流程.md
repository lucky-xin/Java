# 入口为配置扫描包注解 @MapperScan("x.x.mapper") 源码如下
```java
@MapperScan("x.x.mapper")
public class Application {

    public static void main(String[] args) {
        SpringApplication.run(Application.class, args);
    }

}
```
# MapperScan注解会委派MapperScannerRegistrar进行mapper扫描并把mapper注册到ioc之中
```java
@Retention(RetentionPolicy.RUNTIME)
@Target(ElementType.TYPE)
@Documented
@Import(MapperScannerRegistrar.class)
@Repeatable(MapperScans.class)
public @interface MapperScan {
    
}
```
# MapperScannerRegistrar源码如下 创建ClassPathMapperScanner并委派ClassPathMapperScanner进行扫描注册
```java
public class MapperScannerRegistrar implements ImportBeanDefinitionRegistrar, ResourceLoaderAware {

  private ResourceLoader resourceLoader;

  /**
   * {@inheritDoc}
   */
  @Override
  public void setResourceLoader(ResourceLoader resourceLoader) {
    this.resourceLoader = resourceLoader;
  }

  /**
   * {@inheritDoc}
   */
  @Override
  public void registerBeanDefinitions(AnnotationMetadata importingClassMetadata, BeanDefinitionRegistry registry) {
    AnnotationAttributes mapperScanAttrs = AnnotationAttributes
        .fromMap(importingClassMetadata.getAnnotationAttributes(MapperScan.class.getName()));
    if (mapperScanAttrs != null) {
      registerBeanDefinitions(mapperScanAttrs, registry);
    }
  }

  void registerBeanDefinitions(AnnotationAttributes annoAttrs, BeanDefinitionRegistry registry) {

    ClassPathMapperScanner scanner = new ClassPathMapperScanner(registry);

    // this check is needed in Spring 3.1
    Optional.ofNullable(resourceLoader).ifPresent(scanner::setResourceLoader);

    Class<? extends Annotation> annotationClass = annoAttrs.getClass("annotationClass");
    if (!Annotation.class.equals(annotationClass)) {
      scanner.setAnnotationClass(annotationClass);
    }

    Class<?> markerInterface = annoAttrs.getClass("markerInterface");
    if (!Class.class.equals(markerInterface)) {
      scanner.setMarkerInterface(markerInterface);
    }

    Class<? extends BeanNameGenerator> generatorClass = annoAttrs.getClass("nameGenerator");
    if (!BeanNameGenerator.class.equals(generatorClass)) {
      scanner.setBeanNameGenerator(BeanUtils.instantiateClass(generatorClass));
    }

    Class<? extends MapperFactoryBean> mapperFactoryBeanClass = annoAttrs.getClass("factoryBean");
    if (!MapperFactoryBean.class.equals(mapperFactoryBeanClass)) {
      scanner.setMapperFactoryBeanClass(mapperFactoryBeanClass);
    }

    scanner.setSqlSessionTemplateBeanName(annoAttrs.getString("sqlSessionTemplateRef"));
    scanner.setSqlSessionFactoryBeanName(annoAttrs.getString("sqlSessionFactoryRef"));

    List<String> basePackages = new ArrayList<>();
    basePackages.addAll(
        Arrays.stream(annoAttrs.getStringArray("value"))
            .filter(StringUtils::hasText)
            .collect(Collectors.toList()));

    basePackages.addAll(
        Arrays.stream(annoAttrs.getStringArray("basePackages"))
            .filter(StringUtils::hasText)
            .collect(Collectors.toList()));

    basePackages.addAll(
        Arrays.stream(annoAttrs.getClassArray("basePackageClasses"))
            .map(ClassUtils::getPackageName)
            .collect(Collectors.toList()));

    scanner.registerFilters();
    scanner.doScan(StringUtils.toStringArray(basePackages));
  }

  /**
   * A {@link MapperScannerRegistrar} for {@link MapperScans}.
   * @since 2.0.0
   */
  static class RepeatingRegistrar extends MapperScannerRegistrar {
    /**
     * {@inheritDoc}
     */
    @Override
    public void registerBeanDefinitions(AnnotationMetadata importingClassMetadata,
        BeanDefinitionRegistry registry) {
      AnnotationAttributes mapperScansAttrs = AnnotationAttributes
          .fromMap(importingClassMetadata.getAnnotationAttributes(MapperScans.class.getName()));
      if (mapperScansAttrs != null) {
        Arrays.stream(mapperScansAttrs.getAnnotationArray("value"))
            .forEach(mapperScanAttrs -> registerBeanDefinitions(mapperScanAttrs, registry));
      }
    }
  }

}
```
# ClassPathMapperScanner的doScan源码指定mapper扫描包委派父类扫描
```java
  /**
   * Calls the parent search that will search and register all the candidates.
   * Then the registered objects are post processed to set them as
   * MapperFactoryBeans
   */
  @Override
  public Set<BeanDefinitionHolder> doScan(String... basePackages) {
    Set<BeanDefinitionHolder> beanDefinitions = super.doScan(basePackages);

    if (beanDefinitions.isEmpty()) {
      LOGGER.warn(() -> "No MyBatis mapper was found in '" + Arrays.toString(basePackages) + "' package. Please check your configuration.");
    } else {
      processBeanDefinitions(beanDefinitions);
    }

    return beanDefinitions;
  }
```
# ClassPathBeanDefinitionScanner扫描指定的包返回bean定义封装对象BeanDefinitionHolder
```java
	/**
	 * Perform a scan within the specified base packages,
	 * returning the registered bean definitions.
	 * <p>This method does <i>not</i> register an annotation config processor
	 * but rather leaves this up to the caller.
	 * @param basePackages the packages to check for annotated classes
	 * @return set of beans registered if any for tooling registration purposes (never {@code null})
	 */
	protected Set<BeanDefinitionHolder> doScan(String... basePackages) {
		Assert.notEmpty(basePackages, "At least one base package must be specified");
		Set<BeanDefinitionHolder> beanDefinitions = new LinkedHashSet<>();
		for (String basePackage : basePackages) {
			Set<BeanDefinition> candidates = findCandidateComponents(basePackage);
			for (BeanDefinition candidate : candidates) {
				ScopeMetadata scopeMetadata = this.scopeMetadataResolver.resolveScopeMetadata(candidate);
				candidate.setScope(scopeMetadata.getScopeName());
				String beanName = this.beanNameGenerator.generateBeanName(candidate, this.registry);
				if (candidate instanceof AbstractBeanDefinition) {
					postProcessBeanDefinition((AbstractBeanDefinition) candidate, beanName);
				}
				if (candidate instanceof AnnotatedBeanDefinition) {
					AnnotationConfigUtils.processCommonDefinitionAnnotations((AnnotatedBeanDefinition) candidate);
				}
				if (checkCandidate(beanName, candidate)) {
					BeanDefinitionHolder definitionHolder = new BeanDefinitionHolder(candidate, beanName);
					definitionHolder =
							AnnotationConfigUtils.applyScopedProxyMode(scopeMetadata, definitionHolder, this.registry);
					beanDefinitions.add(definitionHolder);
					// 注册BeanDefinitionHolder到ioc之中
					registerBeanDefinition(definitionHolder, this.registry);
				}
			}
		}
		return beanDefinitions;
	}
```
# 扫描得到所有mapper的定义对象BeanDefinitionHolder后，调用processBeanDefinitions方法进行后置处理,因为Mapper为接口不能直接把接口定义注
# 入ioc之中要使用BeanFactory实现类MapperFactoryBean来创建mapper代理对象，也就是说BeanDefinitionHolder之中的BeanDefinition也是对
# MapperFactoryBean的定义。扫描mapper包拿到的是mapper接口定义（BeanDefinitionHolder），需要转换为MapperFactoryBean的BeanDefinitionHolder

```java
  private void processBeanDefinitions(Set<BeanDefinitionHolder> beanDefinitions) {
    GenericBeanDefinition definition;
    for (BeanDefinitionHolder holder : beanDefinitions) {
      definition = (GenericBeanDefinition) holder.getBeanDefinition();
      String beanClassName = definition.getBeanClassName();
      LOGGER.debug(() -> "Creating MapperFactoryBean with name '" + holder.getBeanName()
          + "' and '" + beanClassName + "' mapperInterface");

      // the mapper interface is the original class of the bean
      // but, the actual class of the bean is MapperFactoryBean
      // 设置MapperFactoryBean构造函数参数为mapper接口的class
      definition.getConstructorArgumentValues().addGenericArgumentValue(beanClassName); // issue #59
      //设置BeanDefinitionHolder的bean class为MapperFactoryBean
      definition.setBeanClass(this.mapperFactoryBeanClass);
      // 添加MapperFactoryBean的addToConfig属性
      definition.getPropertyValues().add("addToConfig", this.addToConfig);

      boolean explicitFactoryUsed = false;
      //添加MapperFactoryBean的sqlSessionFactory属性
      if (StringUtils.hasText(this.sqlSessionFactoryBeanName)) {
        definition.getPropertyValues().add("sqlSessionFactory", new RuntimeBeanReference(this.sqlSessionFactoryBeanName));
        explicitFactoryUsed = true;
      } else if (this.sqlSessionFactory != null) {
        definition.getPropertyValues().add("sqlSessionFactory", this.sqlSessionFactory);
        explicitFactoryUsed = true;
      }
      // 添加MapperFactoryBean的sqlSessionTemplate属性
      if (StringUtils.hasText(this.sqlSessionTemplateBeanName)) {
        if (explicitFactoryUsed) {
          LOGGER.warn(() -> "Cannot use both: sqlSessionTemplate and sqlSessionFactory together. sqlSessionFactory is ignored.");
        }
        definition.getPropertyValues().add("sqlSessionTemplate", new RuntimeBeanReference(this.sqlSessionTemplateBeanName));
        explicitFactoryUsed = true;
      } else if (this.sqlSessionTemplate != null) {
        if (explicitFactoryUsed) {
          LOGGER.warn(() -> "Cannot use both: sqlSessionTemplate and sqlSessionFactory together. sqlSessionFactory is ignored.");
        }
        definition.getPropertyValues().add("sqlSessionTemplate", this.sqlSessionTemplate);
        explicitFactoryUsed = true;
      }
      // 设置注入类型为根据类型注入
      if (!explicitFactoryUsed) {
        LOGGER.debug(() -> "Enabling autowire by type for MapperFactoryBean with name '" + holder.getBeanName() + "'.");
        definition.setAutowireMode(AbstractBeanDefinition.AUTOWIRE_BY_TYPE);
      }
    }
  }
```
# 这样就完成了mapper接口定义注册，后面初始化其他bean对象，有依赖mapper对象的会去初始化mapper,也就是初始化MapperFactoryBean
# 接下来看看MapperFactoryBean源码
```java

```