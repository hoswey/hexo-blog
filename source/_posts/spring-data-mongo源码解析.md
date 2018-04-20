title: spring-data-mongo源码解析
tags: []
categories: []
date: 2018-04-04 09:50:00
---

## 背景
由于项目已经很重的使用了mongodb, 对于mongodb的访问主要采用[spring-data-mongo](https://docs.spring.io/spring-data/mongodb/docs/2.0.6.RELEASE/reference/html/),该框架对于一些简单的查询语句都能直接通过[Query methods](https://docs.spring.io/spring-data/mongodb/docs/2.0.6.RELEASE/reference/html/#mongodb.repositories.queries),需要手工写查询语句的情况也很少，这种magic般的使用方法也引起了我的好奇，以前写查询语句的时候规范也是说，对于进行sql查询的方法应当和查询的条件保持一致，例如findByXXX之类，没想到spring data只需要按照类似这种规范，就能直接生成对应的查询语句。
## 源码分析
**目的**

1. 了解spring接口类方法的原理。
2. 了解MongoRepository接口crud方法能直接调用的原理。

基于spring boot写了一个简单的[demo](https://github.com/hoswey/example-spring-data-mongo.git)

[ArticleRepositroy](https://github.com/hoswey/example-spring-data-mongo/blob/master/src/main/java/com/example/spring/mongo/demo/repository/ArticleRepository.java)

```java
/**
* 无需任何实现，使用findByAuthor时，spring data会自动生成mongodb查询语句
**/
public interface ArticleRepository extends MongoRepository<Article, String> {
	
  Article findByAuthor(String author);
}
```
spring是如何实现的，通过接口就可能注入实现具体功能，一般都是通过代理实现，那么只要找出spring是如何初始化该bean问题就可以解决。
### Spring Boot配置
该demo是使用spring boot作为示例，那么按照spring boot的特点，一般都是由各种XXXAutoConfiguration进行初始化。

**相关配置类**

- org.springframework.boot.autoconfigure.mongo，MongoClient,MongoTemplate相关配置
{% asset_img pasted-0.png example.jpg %}
- org.springframework.boot.autoconfigure.data.mongo，data mongo相关配置，涉及到核心类Repository
{% asset_img pasted-1.png example.jpg %}

spring data mongo主要是由这两个package下的配置类初始化,对于Repository类的初始化，核心类是
MongoRepositoriesAutoConfigureRegistrar

```java
class MongoRepositoriesAutoConfigureRegistrar
		extends AbstractRepositoryConfigurationSourceSupport {

	@Override
	protected Class<? extends Annotation> getAnnotation() {
		return EnableMongoRepositories.class;
	}

	@Override
	protected Class<?> getConfiguration() {
		return EnableMongoRepositoriesConfiguration.class;
	}

	@Override
	protected RepositoryConfigurationExtension getRepositoryConfigurationExtension() {
		return new MongoRepositoryConfigurationExtension();
	}

	@EnableMongoRepositories
	private static class EnableMongoRepositoriesConfiguration {
	}
}
```

该类继承了**AbstractRepositoryConfigurationSourceSupport**并且实现了三个抽象方法，简单查看了AbstractRepositoryConfigurationSourceSupport的子类，可以发现spring data很多模块都是继承该类，所以**AbstractRepositoryConfigurationSourceSupport**是Spring data初始化的核心

- AbstractRepositoryConfigurationSourceSupport
	- CassandraRepositoriesAutoConfigureRegistrar 
	- MongoRepositoriesAutoConfigureRegistrar 
	- ElasticsearchRepositoriesRegistrar 
	- LdapRepositoriesRegistrar 
	- CassandraReactiveRepositoriesAutoConfigureRegistrar 
	- CouchbaseRepositoriesRegistrar 
	- JpaRepositoriesAutoConfigureRegistrar 
	- CouchbaseReactiveRepositoriesRegistrar 
	- RedisRepositoriesAutoConfigureRegistrar 
	- Neo4jRepositoriesAutoConfigureRegistrar 
	- SolrRepositoriesRegistrar 
	- MongoReactiveRepositoriesAutoConfigureRegistrar

- MongoRepositoriesAutoConfigureRegistrar实现的三个方法都是简单的返回对象，这里主要有两个对象
	- EnableMongoRepositories 配置类，方法getRepositoryFactoryBeanClassName返回MongoRepositoryFactoryBean，该工厂bean返回了Repositoryd的代理
	- RepositoryConfigurationExtension 该类主要是在Spring Data通用的bean初始化阶段加入mongodb特有的一些配置

<details><summary>EnableMongoRepositories</summary>

```java
public @interface EnableMongoRepositories {

	/**
	 * Alias for the {@link #basePackages()} attribute. Allows for more concise annotation declarations e.g.:
	 * {@code @EnableMongoRepositories("org.my.pkg")} instead of {@code @EnableMongoRepositories(basePackages="org.my.pkg")}.
	 */
	String[] value() default {};

	/**
	 * Base packages to scan for annotated components. {@link #value()} is an alias for (and mutually exclusive with) this
	 * attribute. Use {@link #basePackageClasses()} for a type-safe alternative to String-based package names.
	 */
	String[] basePackages() default {};

	/**
	 * Type-safe alternative to {@link #basePackages()} for specifying the packages to scan for annotated components. The
	 * package of each class specified will be scanned. Consider creating a special no-op marker class or interface in
	 * each package that serves no purpose other than being referenced by this attribute.
	 */
	Class<?>[] basePackageClasses() default {};

	/**
	 * Specifies which types are eligible for component scanning. Further narrows the set of candidate components from
	 * everything in {@link #basePackages()} to everything in the base packages that matches the given filter or filters.
	 */
	Filter[] includeFilters() default {};

	/**
	 * Specifies which types are not eligible for component scanning.
	 */
	Filter[] excludeFilters() default {};

	/**
	 * Returns the postfix to be used when looking up custom repository implementations. Defaults to {@literal Impl}. So
	 * for a repository named {@code PersonRepository} the corresponding implementation class will be looked up scanning
	 * for {@code PersonRepositoryImpl}.
	 * 
	 * @return
	 */
	String repositoryImplementationPostfix() default "Impl";

	/**
	 * Configures the location of where to find the Spring Data named queries properties file. Will default to
	 * {@code META-INFO/mongo-named-queries.properties}.
	 * 
	 * @return
	 */
	String namedQueriesLocation() default "";

	/**
	 * Returns the key of the {@link QueryLookupStrategy} to be used for lookup queries for query methods. Defaults to
	 * {@link Key#CREATE_IF_NOT_FOUND}.
	 * 
	 * @return
	 */
	Key queryLookupStrategy() default Key.CREATE_IF_NOT_FOUND;

	/**
	 * Returns the {@link FactoryBean} class to be used for each repository instance. Defaults to
	 * {@link MongoRepositoryFactoryBean}.
	 * 
	 * @return
	 */
	Class<?> repositoryFactoryBeanClass() default MongoRepositoryFactoryBean.class;

	/**
	 * Configure the repository base class to be used to create repository proxies for this particular configuration.
	 * 
	 * @return
	 * @since 1.8
	 */
	Class<?> repositoryBaseClass() default DefaultRepositoryBaseClass.class;

	/**
	 * Configures the name of the {@link MongoTemplate} bean to be used with the repositories detected.
	 * 
	 * @return
	 */
	String mongoTemplateRef() default "mongoTemplate";

	/**
	 * Whether to automatically create indexes for query methods defined in the repository interface.
	 * 
	 * @return
	 */
	boolean createIndexesForQueryMethods() default false;

	/**
	 * Configures whether nested repository-interfaces (e.g. defined as inner classes) should be discovered by the
	 * repositories infrastructure.
	 */
	boolean considerNestedRepositories() default false;
}
```
</p>
</details>

<details><summary>EnableMongoRepositoriesConfiguration</summary>

```java
	@EnableMongoRepositories
	private static class EnableMongoRepositoriesConfiguration {

	}
```
</p>
</details>

<details><summary>RepositoryConfigurationExtension</summary>

```java
public interface RepositoryConfigurationExtension {

	/**
	 * Returns the descriptive name of the module.
	 * 
	 * @return
	 */
	String getModuleName();

	/**
	 * Returns all {@link RepositoryConfiguration}s obtained through the given {@link RepositoryConfigurationSource}.
	 * 
	 * @param configSource must not be {@literal null}.
	 * @param loader must not be {@literal null}.
	 * @deprecated call or implement
	 *             {@link #getRepositoryConfigurations(RepositoryConfigurationSource, ResourceLoader, boolean)} instead.
	 * @return
	 */
	@Deprecated
	<T extends RepositoryConfigurationSource> Collection<RepositoryConfiguration<T>> getRepositoryConfigurations(
			T configSource, ResourceLoader loader);

	/**
	 * Returns all {@link RepositoryConfiguration}s obtained through the given {@link RepositoryConfigurationSource}.
	 * 
	 * @param configSource
	 * @param loader
	 * @param strictMatchesOnly whether to return strict repository matches only. Handing in {@literal true} will cause
	 *          the repository interfaces and domain types handled to be checked whether they are managed by the current
	 *          store.
	 * @return
	 * @since 1.9
	 */
	<T extends RepositoryConfigurationSource> Collection<RepositoryConfiguration<T>> getRepositoryConfigurations(
			T configSource, ResourceLoader loader, boolean strictMatchesOnly);

	/**
	 * Returns the default location of the Spring Data named queries.
	 * 
	 * @return must not be {@literal null} or empty.
	 */
	String getDefaultNamedQueryLocation();

	/**
	 * Returns the name of the repository factory class to be used.
	 * 
	 * @return
	 */
	String getRepositoryFactoryBeanClassName();

	/**
	 * Callback to register additional bean definitions for a {@literal repositories} root node. This usually includes
	 * beans you have to set up once independently of the number of repositories to be created. Will be called before any
	 * repositories bean definitions have been registered.
	 * 
	 * @param registry
	 * @param source
	 */
	void registerBeansForRoot(BeanDefinitionRegistry registry, RepositoryConfigurationSource configurationSource);

	/**
	 * Callback to post process the {@link BeanDefinition} and tweak the configuration if necessary.
	 * 
	 * @param builder will never be {@literal null}.
	 * @param config will never be {@literal null}.
	 */
	void postProcess(BeanDefinitionBuilder builder, RepositoryConfigurationSource config);

	/**
	 * Callback to post process the {@link BeanDefinition} built from annotations and tweak the configuration if
	 * necessary.
	 * 
	 * @param builder will never be {@literal null}.
	 * @param config will never be {@literal null}.
	 */
	void postProcess(BeanDefinitionBuilder builder, AnnotationRepositoryConfigurationSource config);

	/**
	 * Callback to post process the {@link BeanDefinition} built from XML and tweak the configuration if necessary.
	 * 
	 * @param builder will never be {@literal null}.
	 * @param config will never be {@literal null}.
	 */
	void postProcess(BeanDefinitionBuilder builder, XmlRepositoryConfigurationSource config);
```
</p>
</details>

### 初始化流程

#### 入口

MongoRepositoriesAutoConfigureRegistrar的父类AbstractRepositoryConfigurationSourceSupport
**AbstractRepositoryConfigurationSourceSupport**，该类主要实现了[ImportBeanDefinitionRegistrar](https://github.com/spring-projects/spring-framework/blob/28b2a4d46db39f7c6c25ba8a4ace825af3c4cbcb/spring-context/src/main/java/org/springframework/context/annotation/ImportBeanDefinitionRegistrar.java#L61)的registerBeanDefinitions方法

```java
	@Override
	public void registerBeanDefinitions(AnnotationMetadata importingClassMetadata,
			BeanDefinitionRegistry registry) {
		new RepositoryConfigurationDelegate(getConfigurationSource(registry),
				this.resourceLoader, this.environment).registerRepositoriesIn(registry,
						getRepositoryConfigurationExtension());
	}
```
#### 初始化总流程
registerBeanDefinitions. registerBeanDefinitions调用了RepositoryConfigurationDelegate.registerRepositoriesIn方法
这个方法包含了spring data mongo对于mongo repository的初始化的整个流程，主要逻辑是通过configurationSource（由EnableMongoRepositories构造），以及RepositoryConfigurationExtension扫描BasePackages下面的类，构造beanDefinition并且注册到spring registry。

```java
	/**
	 * Registers the found repositories in the given {@link BeanDefinitionRegistry}.
	 * 
	 * @param registry
	 * @param extension
	 * @return {@link BeanComponentDefinition}s for all repository bean definitions found.
	 */
	public List<BeanComponentDefinition> registerRepositoriesIn(BeanDefinitionRegistry registry,
			RepositoryConfigurationExtension extension) {

		extension.registerBeansForRoot(registry, configurationSource);

		RepositoryBeanDefinitionBuilder builder = new RepositoryBeanDefinitionBuilder(registry, extension, resourceLoader,
				environment);
		List<BeanComponentDefinition> definitions = new ArrayList<>();

		for (RepositoryConfiguration<? extends RepositoryConfigurationSource> configuration : extension
				.getRepositoryConfigurations(configurationSource, resourceLoader, inMultiStoreMode)) {

			//BeanDefintion构造的主要逻辑
			BeanDefinitionBuilder definitionBuilder = builder.build(configuration);

			extension.postProcess(definitionBuilder, configurationSource);

			if (isXml) {
				extension.postProcess(definitionBuilder, (XmlRepositoryConfigurationSource) configurationSource);
			} else {
				extension.postProcess(definitionBuilder, (AnnotationRepositoryConfigurationSource) configurationSource);
			}

			AbstractBeanDefinition beanDefinition = definitionBuilder.getBeanDefinition();
			String beanName = configurationSource.generateBeanName(beanDefinition);

			if (LOGGER.isDebugEnabled()) {
				LOGGER.debug(REPOSITORY_REGISTRATION, extension.getModuleName(), beanName,
						configuration.getRepositoryInterface(), configuration.getRepositoryFactoryBeanClassName());
			}

			beanDefinition.setAttribute(FACTORY_BEAN_OBJECT_TYPE, configuration.getRepositoryInterface());

			registry.registerBeanDefinition(beanName, beanDefinition);
			definitions.add(new BeanComponentDefinition(beanDefinition, beanName));
		}

		return definitions;
	}
```

#### BeanDefinition构造
registerRepositoriesIn调用了RepositoryBeanDefinitionBuilder.build方法

```java
public BeanDefinitionBuilder build(RepositoryConfiguration<?> configuration) {

		Assert.notNull(registry, "BeanDefinitionRegistry must not be null!");
		Assert.notNull(resourceLoader, "ResourceLoader must not be null!");

		//设置类为MongoRepositoryFactoryBean
		BeanDefinitionBuilder builder = BeanDefinitionBuilder
				.rootBeanDefinition(configuration.getRepositoryFactoryBeanClassName());

		builder.getRawBeanDefinition().setSource(configuration.getSource());
		builder.addConstructorArgValue(configuration.getRepositoryInterface());
		builder.addPropertyValue("queryLookupStrategyKey", configuration.getQueryLookupStrategyKey());
		builder.addPropertyValue("lazyInit", configuration.isLazyInit());

		configuration.getRepositoryBaseClassName()//
				.ifPresent(it -> builder.addPropertyValue("repositoryBaseClass", it));

		NamedQueriesBeanDefinitionBuilder definitionBuilder = new NamedQueriesBeanDefinitionBuilder(
				extension.getDefaultNamedQueryLocation());
		configuration.getNamedQueriesLocation().ifPresent(definitionBuilder::setLocations);

		builder.addPropertyValue("namedQueries", definitionBuilder.build(configuration.getSource()));

		//2.0版本已经不建议使用的方法，和Framement类似
		registerCustomImplementation(configuration).ifPresent(it -> {
			builder.addPropertyReference("customImplementation", it);
			builder.addDependsOn(it);
		});

		BeanDefinitionBuilder fragmentsBuilder = BeanDefinitionBuilder
				.rootBeanDefinition(RepositoryFragmentsFactoryBean.class);

		List<String> fragmentBeanNames = registerRepositoryFragmentsImplementation(configuration) //
				.map(RepositoryFragmentConfiguration::getFragmentBeanName) //
				.collect(Collectors.toList());

		fragmentsBuilder.addConstructorArgValue(fragmentBeanNames);

		//注册Fragmemnt,也就是这种做法
		//https://docs.spring.io/spring-data/mongodb/docs/2.0.6.RELEASE/reference/html/#repositories.custom-implementations
		builder.addPropertyValue("repositoryFragments",
				ParsingUtils.getSourceBeanDefinition(fragmentsBuilder, configuration.getSource()));

		RootBeanDefinition evaluationContextProviderDefinition = new RootBeanDefinition(
				ExtensionAwareEvaluationContextProvider.class);
		evaluationContextProviderDefinition.setSource(configuration.getSource());

		builder.addPropertyValue("evaluationContextProvider", evaluationContextProviderDefinition);

		return builder;
	}
```

#### MongoRepositoryFactoryBean
MongoRepositoriesAutoConfigureRegistrar的职责最终就是把所有扫到的Repository构造成BeanDefinition注册到Registry, BeanDefinition的BaseClass是MongoRepositoryFactoryBean，所以探究这个FactoryBean的逻辑是了解spring data核心的关键

**类图**
{% plantuml %}

interface FactoryBean
class MongoRepositoryFactoryBean
class RepositoryFactoryBeanSupport {
	{field} -T repository
	{method} +afterPropertiesSet
	{method} +T getObject
}
MongoRepositoryFactoryBean --|> RepositoryFactoryBeanSupport
RepositoryFactoryBeanSupport ..|> FactoryBean
{% endplantuml %}

**序列图**
{% plantuml %}
participant MongoRepositoryFactoryBean

MongoRepositoryFactoryBean -> MongoRepositoryFactoryBean: afterPropertiesSet

{% endplantuml %}

**序列图

## 特别
SurroundingTransactionDetectorMethodInterceptor