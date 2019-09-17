# JPA中Repository怎么从接口变成bean

如果在启动类中添加注解@EnableJpaRepositories，JpaRepositoriesRegistrar将会激活注解，返回JpaRepositoryConfigExtension

```java
class JpaRepositoriesRegistrar extends RepositoryBeanDefinitionRegistrarSupport {

	@Override
	protected Class<? extends Annotation> getAnnotation() {
		return EnableJpaRepositories.class;
	}

	@Override
    //限定factorybean的类型
	protected RepositoryConfigurationExtension getExtension() {
		return new JpaRepositoryConfigExtension();
	}
}
```

JpaRepositoriesRegistrar继承了RepositoryBeanDefinitionRegistrarSupport并最终继承了ImportBeanDefinitionRegistrar的重要方法registerBeanDefinitions,为每个repository注册Repository Bean(类型是JpaRepositoryFacrotyBean)。

```java
public void registerBeanDefinitions(AnnotationMetadata annotationMetadata, BeanDefinitionRegistry registry) {

		Assert.notNull(annotationMetadata, "AnnotationMetadata must not be null!");
		Assert.notNull(registry, "BeanDefinitionRegistry must not be null!");
		Assert.notNull(resourceLoader, "ResourceLoader must not be null!");

		// Guard against calls for sub-classes
		if (annotationMetadata.getAnnotationAttributes(getAnnotation().getName()) == null) {
			return;
		}

		AnnotationRepositoryConfigurationSource configurationSource = new AnnotationRepositoryConfigurationSource(
				annotationMetadata, getAnnotation(), resourceLoader, environment, registry);

		RepositoryConfigurationExtension extension = getExtension();
		RepositoryConfigurationUtils.exposeRegistration(extension, registry, configurationSource);

		RepositoryConfigurationDelegate delegate = new RepositoryConfigurationDelegate(configurationSource, resourceLoader,
				environment);
		//前面获取extension,通过delegate注册
		delegate.registerRepositoriesIn(registry, extension);
	}
```

```java
//定义RepositoryBeanDefinitionBuilder
RepositoryBeanDefinitionBuilder builder = new RepositoryBeanDefinitionBuilder(registry, extension,
				configurationSource, resourceLoader, environment);
//拉取所有Repository接口，为每个Repository创建BeanDefinitionBuilder
BeanDefinitionBuilder definitionBuilder = builder.build(configuration);
//获取beanDefinition，并返回
AbstractBeanDefinition beanDefinition = definitionBuilder.getBeanDefinition();
definitions.add(new BeanComponentDefinition(beanDefinition, beanName));
```

至此，我们仅仅做到了扫描并识别出所有集成了Repository接口的JPA接口，并吧他们作为JpaRepositoryFactoryBean进行注册。

实例化JpaRepositoryFactoryBean时，由于接口实现了InitializingBean接口，FactoryBean初始化后执行了afterPropertiesSet，自动生成代理Bean实现，并托管spring管理。

```java
public void afterPropertiesSet() {

		this.factory = createRepositoryFactory();
		this.factory.setQueryLookupStrategyKey(queryLookupStrategyKey);
		this.factory.setNamedQueries(namedQueries);
		this.factory.setEvaluationContextProvider(
				evaluationContextProvider.orElseGet(() -> QueryMethodEvaluationContextProvider.DEFAULT));
		this.factory.setBeanClassLoader(classLoader);
		this.factory.setBeanFactory(beanFactory);

		if (publisher != null) {
			this.factory.addRepositoryProxyPostProcessor(new EventPublishingRepositoryProxyPostProcessor(publisher));
		}

		repositoryBaseClass.ifPresent(this.factory::setRepositoryBaseClass);

		RepositoryFragments customImplementationFragment = customImplementation //
				.map(RepositoryFragments::just) //
				.orElseGet(RepositoryFragments::empty);

		RepositoryFragments repositoryFragmentsToUse = this.repositoryFragments //
				.orElseGet(RepositoryFragments::empty) //
				.append(customImplementationFragment);

		this.repositoryMetadata = this.factory.getRepositoryMetadata(repositoryInterface);

		// Make sure the aggregate root type is present in the MappingContext (e.g. for auditing)
		this.mappingContext.ifPresent(it -> it.getPersistentEntity(repositoryMetadata.getDomainType()));

		//生成代理类
		this.repository = Lazy.of(() -> this.factory.getRepository(repositoryInterface, repositoryFragmentsToUse));

		if (!lazyInit) {
			this.repository.get();
		}
	}
```

在接口中定义的方法，通过在getRepository中添加Advice实现

```Java
result.addAdvice(new DefaultMethodInvokingMethodInterceptor());
result.addAdvice(new QueryExecutorMethodInterceptor(information, projectionFactory));
```

具体执行在AbstractJpaQuery.execute执行具体查询，语法解析在Part中。

```java
private Object doExecute(JpaQueryExecution execution, Object[] values) {

		Object result = execution.execute(this, values);

		ParametersParameterAccessor accessor = new ParametersParameterAccessor(method.getParameters(), values);
		ResultProcessor withDynamicProjection = method.getResultProcessor().withDynamicProjection(accessor);

		return withDynamicProjection.processResult(result, new TupleConverter(withDynamicProjection.getReturnedType()));
	}
```

