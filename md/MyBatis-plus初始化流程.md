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
# 接下来看看MapperFactoryBean源码![MapperFactoryBean类类](https://github.com/lucky-xin/Learning/blob/gh-pages/image/MapperFactoryBean.PNG)
# 1. 实现了FactoryBean接口,FactoryBean为创建bean工厂，在DefaultListableBeanFactory超类AbstractFactoryBean的doGetBean方法之中获取bean时会先去singleton缓存获取
# 如果如果拿到bean对象则判断该bean是否为FactoryBean如果是则调用FactoryBean的getObject方法获取对象。
# 2. 实现了InitializingBean接口为初始化完成之后回调，用于初始化mapper把mapper注册到Configuration实际上是mapper注册中心MapperRegistry
# afterPropertiesSet方法具体实现在超类DaoSupport
```java
	@Override
	public final void afterPropertiesSet() throws IllegalArgumentException, BeanInitializationException {
		// Let abstract subclasses check their configuration.
		// MapperFactoryBean类重写该方法用于初始化mapper 
		checkDaoConfig();

		// Let concrete implementations initialize themselves.
		try {
			initDao();
		}
		catch (Exception ex) {
			throw new BeanInitializationException("Initialization of DAO failed", ex);
		}
	}
```
```java
public class MapperFactoryBean<T> extends SqlSessionDaoSupport implements FactoryBean<T> {

  private Class<T> mapperInterface;

  private boolean addToConfig = true;

  public MapperFactoryBean() {
    //intentionally empty 
  }
  
  public MapperFactoryBean(Class<T> mapperInterface) {
    this.mapperInterface = mapperInterface;
  }

  /**
   * {@inheritDoc}
   */
  @Override
  protected void checkDaoConfig() {
    super.checkDaoConfig();

    notNull(this.mapperInterface, "Property 'mapperInterface' is required");
    // 初始化mapper,把mapper信息注册到注册中心之中
    Configuration configuration = getSqlSession().getConfiguration();
    if (this.addToConfig && !configuration.hasMapper(this.mapperInterface)) {
      try {
        configuration.addMapper(this.mapperInterface);
      } catch (Exception e) {
        logger.error("Error while adding the mapper '" + this.mapperInterface + "' to configuration.", e);
        throw new IllegalArgumentException(e);
      } finally {
        ErrorContext.instance().reset();
      }
    }
  }

  /**
   * {@inheritDoc}
   */
  @Override
  public T getObject() throws Exception {
    return getSqlSession().getMapper(this.mapperInterface);
  }

  /**
   * {@inheritDoc}
   */
  @Override
  public Class<T> getObjectType() {
    return this.mapperInterface;
  }

  /**
   * {@inheritDoc}
   */
  @Override
  public boolean isSingleton() {
    return true;
  }

  //------------- mutators --------------

  /**
   * Sets the mapper interface of the MyBatis mapper
   *
   * @param mapperInterface class of the interface
   */
  public void setMapperInterface(Class<T> mapperInterface) {
    this.mapperInterface = mapperInterface;
  }

  /**
   * Return the mapper interface of the MyBatis mapper
   *
   * @return class of the interface
   */
  public Class<T> getMapperInterface() {
    return mapperInterface;
  }

  /**
   * If addToConfig is false the mapper will not be added to MyBatis. This means
   * it must have been included in mybatis-config.xml.
   * <p>
   * If it is true, the mapper will be added to MyBatis in the case it is not already
   * registered.
   * <p>
   * By default addToConfig is true.
   *
   * @param addToConfig a flag that whether add mapper to MyBatis or not
   */
  public void setAddToConfig(boolean addToConfig) {
    this.addToConfig = addToConfig;
  }

  /**
   * Return the flag for addition into MyBatis config.
   *
   * @return true if the mapper will be added to MyBatis in the case it is not already
   * registered.
   */
  public boolean isAddToConfig() {
    return addToConfig;
  }
}

```
# Configuration的注册mapper方法.具体委派到MybatisMapperRegistry的addMapper方法源码如下
```java
    @Override
    public <T> void addMapper(Class<T> type) {
        if (type.isInterface()) {
            if (hasMapper(type)) {
                // TODO 如果之前注入 直接返回
                return;
                // TODO 这里就不抛异常了
//                throw new BindingException("Type " + type + " is already known to the MapperRegistry.");
            }
            boolean loadCompleted = false;
            try {
                // TODO 这里也换成 MybatisMapperProxyFactory 而不是 MapperProxyFactory
                knownMappers.put(type, new MybatisMapperProxyFactory<>(type));
                // It's important that the type is added before the parser is run
                // otherwise the binding may automatically be attempted by the
                // mapper parser. If the type is already known, it won't try.
                // TODO 这里也换成 MybatisMapperAnnotationBuilder 而不是 MapperAnnotationBuilder
                MybatisMapperAnnotationBuilder parser = new MybatisMapperAnnotationBuilder(config, type);
                // 解析xml,生成statement语句信息
                parser.parse();
                loadCompleted = true;
            } finally {
                if (!loadCompleted) {
                    knownMappers.remove(type);
                }
            }
        }
    }
```
# 具体解析在MybatisMapperAnnotationBuilder类的parse方法。遍历mapper的所有方法生成statement信息并注册到注册中心MapperRegistry。
# 等运行时会根据namespace+statement来获取某一个statement。SqlSession所有操作都基于statement。这样就完成了mapper初始化以及把mapper注册到注册中心之中。
```java
 @Override
    public void parse() {
        String resource = type.toString();
        if (!configuration.isResourceLoaded(resource)) {
            // 加载mapper的xml文件
            loadXmlResource();
            configuration.addLoadedResource(resource);
            final String typeName = type.getName();
            assistant.setCurrentNamespace(typeName);
            parseCache();
            parseCacheRef();
            SqlParserHelper.initSqlParserInfoCache(type);
            Method[] methods = type.getMethods();
            for (Method method : methods) {
                try {
                    // issue #237
                    if (!method.isBridge()) {
                        parseStatement(method);
                        SqlParserHelper.initSqlParserInfoCache(typeName, method);
                    }
                } catch (IncompleteElementException e) {
                    // TODO 使用 MybatisMethodResolver 而不是 MethodResolver
                    configuration.addIncompleteMethod(new MybatisMethodResolver(this, method));
                }
            }
            // TODO 注入 CURD 动态 SQL , 放在在最后, because 可能会有人会用注解重写sql
            if (GlobalConfigUtils.isSupperMapperChildren(configuration, type)) {
                GlobalConfigUtils.getSqlInjector(configuration).inspectInject(assistant, type);
            }
        }
        parsePendingMethods();
    }
```