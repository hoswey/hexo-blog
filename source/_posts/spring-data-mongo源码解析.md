title: spring-data-mongo源码解析
tags: []
categories: []
date: 2018-04-04 09:50:00
---

# 背景
由于项目已经很重的使用了mongodb, 对于mongodb的访问主要采用[spring-data-mongo](https://docs.spring.io/spring-data/mongodb/docs/2.0.6.RELEASE/reference/html/),该框架对于一些简单的查询语句都能直接通过[Query methods](https://docs.spring.io/spring-data/mongodb/docs/2.0.6.RELEASE/reference/html/#mongodb.repositories.queries),需要手工写查询语句的情况也很少，这种magic般的使用方法也引起了我的好奇，以前写查询语句的时候规范也是说，对于进行sql查询的方法应当和查询的条件保持一致，例如findByXXX之类，没想到spring data只需要按照类似这种规范，就能直接生成对应的查询语句，那么底层是怎么实现的？

# 源码分析
## 目的
了解spring接口类方法的原理。


## 准备工作
通过spring boot快速搭建[demo](https://github.com/hoswey/example-spring-data-mongo.git),该demo包含一个MongoRepository,通过main函数执行一些简单的查询方法。

[ArticleRepositroy](https://github.com/hoswey/example-spring-data-mongo/blob/master/src/main/java/com/example/spring/mongo/demo/repository/ArticleRepository.java)

```java
/**
* 无需任何实现，使用findByAuthor时，spring data会自动生成mongodb查询语句
**/
public interface ArticleRepository extends MongoRepository<Article, String>,
    CustomizedArticleRepository {

  List<Article> findByAuthorOrderByIdDescAllIgnoreCase(String author);

  List<Article> findByNumOfLikeIsGreaterThan(int numOfLike);
}
```

## 源码分析
### Spring Boot配置
要了解spring data mongo原理，最直接的方法就是看repository是怎么初始化的，由于该demo是基于spring boot，按照spring boot的特点，初始化的控制都是由各种AutoConfiguration引导。

直接在IDE里搜索 \*Mongo\*AutoConfiguration, 可以发现以下配置类

#### 相关配置类

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

该类继承了**AbstractRepositoryConfigurationSourceSupport**并且实现了三个抽象方法，查看了AbstractRepositoryConfigurationSourceSupport的子类，可以发现spring data很多模块都是继承该类，所以**AbstractRepositoryConfigurationSourceSupport**是Spring data初始化的核心

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

#### MongoRepositoriesAutoConfigureRegistrar

该类实现的三个方法都是返回了配置类POJO，这里主要有两个对象
1. EnableMongoRepositories 配置类，方法getRepositoryFactoryBeanClassName返回MongoRepositoryFactoryBean，该工厂bean返回了Repositoryd的代理
1. RepositoryConfigurationExtension 该类主要是在Spring Data通用的bean初始化阶段加入mongodb特有的一些配置，其大部分方法都是接受了BeanDefinitionBuilder进行一些

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
**AbstractRepositoryConfigurationSourceSupport**，该类主要实现了[ImportBeanDefinitionRegistrar](https://github.com/spring-projects/spring-framework/blob/28b2a4d46db39f7c6c25ba8a4ace825af3c4cbcb/spring-context/src/main/java/org/springframework/context/annotation/ImportBeanDefinitionRegistrar.java#L61)的registerBeanDefinitions方法，registerBeanDefinitions是Spring初始化的时候调用，让程序能够自主注册一些bean到spring容器里

```java
	@Override
	public void registerBeanDefinitions(AnnotationMetadata importingClassMetadata,
			BeanDefinitionRegistry registry) {
		new RepositoryConfigurationDelegate(getConfigurationSource(registry),
				this.resourceLoader, this.environment).registerRepositoriesIn(registry,
						getRepositoryConfigurationExtension());
	}
```
#### 流程
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

### BeanDefinition构造
registerRepositoriesIn调用了RepositoryBeanDefinitionBuilder.build方法，注意到BeanDefinitionBuilder
				.rootBeanDefinition(configuration.getRepositoryFactoryBeanClassName())，这里把MongoRepositoryFactoryBean设置为BeanDefinition的class,意味着Bean的初始化由MongoRepositoryFactoryBean进行控制

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

### MongoRepositoryFactoryBean
MongoRepositoriesAutoConfigureRegistrar的职责最终就是把所有扫到的Repository构造成BeanDefinition注册到Registry, BeanDefinition的BaseClass是MongoRepositoryFactoryBean，所以该FactoryBean是spring data mongo的核心。

从该Sequence Diagram可以看出Repository最终的实现是通过Proxy,这里只列出了两个关键的Interceptor

1. QueryExecutorMethodInterceptor - 该类主要实现了[Query methods](https://docs.spring.io/spring-data/mongodb/docs/2.0.6.RELEASE/reference/html/#mongodb.repositories.queries)
1. ImplementationMethodExecutionInterceptor - 该类主要实现了[repositories.custom-implementations](https://docs.spring.io/spring-data/mongodb/docs/2.0.6.RELEASE/reference/html/#repositories.custom-implementations)

**Sequence Diagram**

{% plantuml %}
participant MongoRepositoryFactoryBean

MongoRepositoryFactoryBean -> MongoRepositoryFactoryBean: afterPropertiesSet
create MongoRepositoryFactory
MongoRepositoryFactoryBean -> MongoRepositoryFactory: new
MongoRepositoryFactoryBean -> MongoRepositoryFactory: setRepositoryBaseClass
MongoRepositoryFactoryBean -> MongoRepositoryFactory: getRepository
create ProxyFactory
MongoRepositoryFactory -> ProxyFactory: new
MongoRepositoryFactory -> ProxyFactory: addAdvice(QueryExecutorMethodInterceptor)
MongoRepositoryFactory -> ProxyFactory: addAdvice(ImplementationMethodExecutionInterceptor)

MongoRepositoryFactoryBean <-- MongoRepositoryFactory: Repository

{% endplantuml %}

**Class Diagram**

{% plantuml %}

interface FactoryBean
class MongoRepositoryFactoryBean
class RepositoryFactoryBeanSupport {
	{field} -T repository
	{method} +afterPropertiesSet
	{method} +T getObject
}
class MongoRepositoryFactory
class ProxyFactory
class QueryExecutorMethodInterceptor
class ImplementationMethodExecutionInterceptor
interface MethodInterceptor

MongoRepositoryFactoryBean --|> RepositoryFactoryBeanSupport
RepositoryFactoryBeanSupport ..|> FactoryBean
RepositoryFactoryBeanSupport ..> MongoRepositoryFactory
MongoRepositoryFactory ..> ProxyFactory
MongoRepositoryFactory ..> QueryExecutorMethodInterceptor
MongoRepositoryFactory ..> ImplementationMethodExecutionInterceptor
QueryExecutorMethodInterceptor ..|> MethodInterceptor
ImplementationMethodExecutionInterceptor ..|> MethodInterceptor

{% endplantuml %}

### QueryExecutorMethodInterceptor
从该Sequence Diagram可看出QueryExecutorMethodInterceptor最终构造了PartTree，PartTree又Predicate组成

#### Sequence Diagram
{% plantuml %}

QueryExecutorMethodInterceptor -> MongoQueryLookupStrategy:resolveQuery
create PartTreeMongoQuery
MongoQueryLookupStrategy -> PartTreeMongoQuery: new
create PartTree
PartTreeMongoQuery -> PartTree: new
create Predicate
PartTree -> Predicate: new

{% endplantuml %}


#### Class Diagram
{% plantuml %}
interface QueryLookupStrategy
class MongoQueryLookupStrategy
interface RepositoryQuery
class PartTreeMongoQuery
interface MethodInterceptor
class QueryExecutorMethodInterceptor
class PartTree
QueryExecutorMethodInterceptor ..|> MethodInterceptor
QueryExecutorMethodInterceptor ..> QueryLookupStrategy 
MongoQueryLookupStrategy ..|> QueryLookupStrategy
PartTreeMongoQuery ..|> RepositoryQuery
MongoQueryLookupStrategy ..> PartTreeMongoQuery
PartTreeMongoQuery *.. PartTree
PartTree *.. Predicate
Predicate *.. OrPart : n
OrPart *.. Part : n

{% endplantuml %}

### PartTree
[PartTree](https://github.com/spring-projects/spring-data-commons/blob/70ac316b400937d7e1dff71c1605a4205fc818bd/src/main/java/org/springframework/data/repository/query/parser/PartTree.java#L84)首先通过方法名(source参数)解析出语句类型（查询/更新/删除），然后把余下的部分传给了Predicate, 例如findByAuthorOrTitle(以下的解析把这个方法名作为例子），那么传给Predicate的就是**AuthorOrTitle**

下面这些静态变量则是定义了合法的方法名前缀，从这些变量可知道查询方法的前缀不仅仅只支持“find",😊同时也支持“read/get/query/steam".这些可在官方文档没提及。



```java
private static final String KEYWORD_TEMPLATE = "(%s)(?=(\\p{Lu}|\\P{InBASIC_LATIN}))";
private static final String QUERY_PATTERN = "find|read|get|query|stream";
private static final String COUNT_PATTERN = "count";
private static final String EXISTS_PATTERN = "exists";
private static final String DELETE_PATTERN = "delete|remove";
private static final Pattern PREFIX_TEMPLATE = Pattern.compile( //
		"^(" + QUERY_PATTERN + "|" + COUNT_PATTERN + "|" + EXISTS_PATTERN + "|" + DELETE_PATTERN + ")((\\p{Lu}.*?))??By");


public PartTree(String source, Class<?> domainClass) {

	Assert.notNull(source, "Source must not be null");
	Assert.notNull(domainClass, "Domain class must not be null");

	Matcher matcher = PREFIX_TEMPLATE.matcher(source);

	if (!matcher.find()) {
		this.subject = new Subject(Optional.empty());
		this.predicate = new Predicate(source, domainClass);
	} else {
		this.subject = new Subject(Optional.of(matcher.group(0)));
		this.predicate = new Predicate(source.substring(matcher.group().length()), domainClass);
	}
```

### Predicate
那么[Predicate](https://github.com/spring-projects/spring-data-commons/blob/70ac316b400937d7e1dff71c1605a4205fc818bd/src/main/java/org/springframework/data/repository/query/parser/PartTree.java#L361)的逻辑又是怎样的呢？Predicate主要是根据repository的方法名，通过关键字“Or"，把方法名split成多个子字符串，构造OrPart, 多个OrPart最终在mongo的查询里就会解析成了$or.例如**AuthorOrTitle**就会被拆成**Author**和**Title**然后构造成两个OrPart对象。


```java
public Predicate(String predicate, Class<?> domainClass) {

	String[] parts = split(detectAndSetAllIgnoreCase(predicate), ORDER_BY);

	if (parts.length > 2) {
		throw new IllegalArgumentException("OrderBy must not be used more than once in a method name!");
	}

	this.nodes = Arrays.stream(split(parts[0], "Or")) //
			.filter(StringUtils::hasText) //
			.map(part -> new OrPart(part, domainClass, alwaysIgnoreCase)) //
			.collect(Collectors.toList());

	this.orderBySource = parts.length == 2 ? new OrderBySource(parts[1], Optional.of(domainClass))
					: OrderBySource.EMPTY;
```

### OrPart
从[OrPart](https://github.com/spring-projects/spring-data-commons/blob/70ac316b400937d7e1dff71c1605a4205fc818bd/src/main/java/org/springframework/data/repository/query/parser/PartTree.java#L233)的构造函数可以看出，这里是把Source拆成构造成List\<Part\>,以**Author**作为例子，由于也不包含"And",所以只会有一个Part对象。

```java
OrPart(String source, Class<?> domainClass, boolean alwaysIgnoreCase) {

	String[] split = split(source, "And");

	this.children = Arrays.stream(split)//
			.filter(StringUtils::hasText)//
			.map(part -> new Part(part, domainClass, alwaysIgnoreCase))//
			.collect(Collectors.toList());
}
```

### Part
[Part](https://github.com/spring-projects/spring-data-commons/blob/70ac316b400937d7e1dff71c1605a4205fc818bd/src/main/java/org/springframework/data/repository/query/parser/Part.java#L69)主要设置了两个成员变量

```java
	public Part(String source, Class<?> clazz, boolean alwaysIgnoreCase) {

		Assert.hasText(source, "Part source must not be null or empty!");
		Assert.notNull(clazz, "Type must not be null!");

		String partToUse = detectAndSetIgnoreCase(source);

		if (alwaysIgnoreCase && ignoreCase != IgnoreCaseType.ALWAYS) {
			this.ignoreCase = IgnoreCaseType.WHEN_POSSIBLE;
		}

		this.type = Type.fromProperty(partToUse);
		this.propertyPath = PropertyPath.from(type.extractProperty(partToUse), clazz);
	}
```	

1. [type](https://github.com/spring-projects/spring-data-commons/blob/70ac316b400937d7e1dff71c1605a4205fc818bd/src/main/java/org/springframework/data/repository/query/parser/Part.java#L149) - 主要保存了该条件的类型，BETWEEN, IS_NOT_NULL等等， 对于例子中的**author**关键字，则会解析为Type.SIMPLE_PROPERTY,表示EQUAL.
2. propertyPath - 属性名， **author**

**看到这里，初始化的逻辑基本完毕， Repository的方法最终会被解析并且保存成PartTree, 查询的时候只需要从PartTree生成mongo语句**

### MongoQueryCreator
[MongoQueryCreator](https://github.com/spring-projects/spring-data-mongodb/blob/17d61004261c388a98bf1cc932318db69073db2f/spring-data-mongodb/src/main/java/org/springframework/data/mongodb/repository/query/MongoQueryCreator.java#L174)从该方法可看出从Part构建出Criteria的过程。

```java
private Criteria from(Part part, MongoPersistentProperty property, Criteria criteria, Iterator<Object> parameters) {

	Type type = part.getType();

	switch (type) {
		case AFTER:
		case GREATER_THAN:
			return criteria.gt(parameters.next());
		case GREATER_THAN_EQUAL:
			return criteria.gte(parameters.next());
		case BEFORE:
		case LESS_THAN:
			return criteria.lt(parameters.next());
		case LESS_THAN_EQUAL:
			return criteria.lte(parameters.next());
		case BETWEEN:
			return criteria.gt(parameters.next()).lt(parameters.next());
		case IS_NOT_NULL:
			return criteria.ne(null);
		case IS_NULL:
			return criteria.is(null);
		case NOT_IN:
```

# 总结
1. 扫描路径下继承Repository的接口（当有多个spring data模块存在时，则会限定到MongoRepository)的类生成代理。
2. 解析接口名生成PartTree,执行的时候根据对应PartTree生成Criteria条件，传给MongoTemplate.

# 彩蛋
1. 😆看源码过程中发现[SurroundingTransactionDetectorMethodInterceptor](https://github.com/spring-projects/spring-data-commons/blob/70ac316b400937d7e1dff71c1605a4205fc818bd/src/main/java/org/springframework/data/repository/core/support/SurroundingTransactionDetectorMethodInterceptor.java#L35)枚举类可继承接口 