
回到上篇的开头部分，进入到 **FeignClientsRegistrar** 类的执行流程

```java
@Override
public void registerBeanDefinitions(AnnotationMetadata metadata, BeanDefinitionRegistry registry) {
    // 这个方法是将 Feign 默认的全局配置注册进 Spirng 的 IOC 中
    registerDefaultConfiguration(metadata, registry);
    // 这个方法是将 Feign client 和 对应的每个 configuration 注册进 Spring IOC中
    registerFeignClients(metadata, registry);
}
```

看 **registerDefaultConfiguration** 方法

```java
// metadata 类的实现是 StandardAnnotationMetadata 类
// registry 类的实现是 DefaultListableBeanFactory 类
private void registerDefaultConfiguration(AnnotationMetadata metadata,
                                          BeanDefinitionRegistry registry) {
    // 获取 EnableFeignClients 注解的五个属性，构建一个 LinekHaskMap
    Map<String, Object> defaultAttrs = metadata
        .getAnnotationAttributes(EnableFeignClients.class.getName(), true);
    // 当配置了 defaultConfiguration 属性的时候，进这个判断
    if (defaultAttrs != null && defaultAttrs.containsKey("defaultConfiguration")) {
        String name;
        // metadata 的 hasEnclosingClass 这个方法判断的是加上 @EnableFeignClients 注解的类的Class字节码的 getEnclosingClass方法，作用是判断当前类是不是内部类，是的话，就是上一层类的名字，不是的话，就为null
        if (metadata.hasEnclosingClass()) {
            name = "default." + metadata.getEnclosingClassName();
        }
        else {
            name = "default." + metadata.getClassName();
        }
        // 这个方法实现了具体的注册逻辑
        registerClientConfiguration(registry, name,
                                    defaultAttrs.get("defaultConfiguration"));
    }
}
```

看 **registerClientConfiguration** 方法

```java
// name 是一个 default. 开头的字符串， configuration 是配置的 defaultConfiguration 字符串
private void registerClientConfiguration(BeanDefinitionRegistry registry, Object name,
                                         Object configuration) {
    // BeanDefinitionBuilder 内部维护了一个 AbstractBeanDefinition 类型的 beanDefinition 变量，这里将FeignClientSpecification.class 这个class字节码赋值给 beanDefinition 的 beanClass 变量。可以自己看一下内部实现
    BeanDefinitionBuilder builder = BeanDefinitionBuilder
        .genericBeanDefinition(FeignClientSpecification.class);
    builder.addConstructorArgValue(name);
    builder.addConstructorArgValue(configuration);
    registry.registerBeanDefinition(
        name + "." + FeignClientSpecification.class.getSimpleName(),
        builder.getBeanDefinition()); // 这里就获取到了 beanDefinition 这个变量，是 GenericBeanDefinition 类型的。
}
```

这里简单的看一下 **registry** 的 **registerBeanDefinition** 方法，在 **DefaultListableBeanFactory** 内部实现

```java
@Override
public void registerBeanDefinition(String beanName, BeanDefinition beanDefinition)
    throws BeanDefinitionStoreException {

    Assert.hasText(beanName, "Bean name must not be empty");
    Assert.notNull(beanDefinition, "BeanDefinition must not be null");
    // 这里的 beanDefinition 是 GenericBeanDefinition， 继承自 AbstractBeanDefinition 类
    if (beanDefinition instanceof AbstractBeanDefinition) {
        try {
            // 这个方法没啥看的，里面判断了一个变量是否为空，默认这个变量是空的，意味着在这里其实什么都没做
            ((AbstractBeanDefinition) beanDefinition).validate();
        }
        catch (BeanDefinitionValidationException ex) {
            throw new BeanDefinitionStoreException(beanDefinition.getResourceDescription(), beanName,
                                                   "Validation of bean definition failed", ex);
        }
    }
    // 尝试从 beanDefinition 的Map中获取这个名字的Map， 第一次肯定为空
    BeanDefinition existingDefinition = this.beanDefinitionMap.get(beanName);
    if (existingDefinition != null) {
        // ......
    }
    else {
        if (hasBeanCreationStarted()) {
            // Cannot modify startup-time collection elements anymore (for stable iteration)
            synchronized (this.beanDefinitionMap) {
                this.beanDefinitionMap.put(beanName, beanDefinition); // 这里就放进去了
               // .....
            }
        }
        else {
            // Still in startup registration phase
            this.beanDefinitionMap.put(beanName, beanDefinition);
            this.beanDefinitionNames.add(beanName);
            this.manualSingletonNames.remove(beanName);
        }
        this.frozenBeanDefinitionNames = null;
    }
    // .....
	}
```

这里不必要的代码就不看了，到这里为止，我们就把全局的 **Feign** 配置类 **BeanDefinition** 和对应的名字放入这个 **beanDeifinitionMap** 中去了，再看指定的 **Feign client beanDefinition** 创建过程。

```java
public void registerFeignClients(AnnotationMetadata metadata,
			BeanDefinitionRegistry registry) {
    // 我们在 @EnableFeignClients 注解中配置了 basePackages 的值，这个 scanner 的作用就是扫描包下 @FeignClient 注解的所有接口
    ClassPathScanningCandidateComponentProvider scanner = getScanner();
    scanner.setResourceLoader(this.resourceLoader);

    Set<String> basePackages;

    Map<String, Object> attrs = metadata
        .getAnnotationAttributes(EnableFeignClients.class.getName());
    AnnotationTypeFilter annotationTypeFilter = new AnnotationTypeFilter(
        FeignClient.class);
    // 一般的使用方式是配置 basePackges 扫包的方式，硬编码 client 的方式还是比较少的，所以这里的 clients 数组长度为0
    final Class<?>[] clients = attrs == null ? null
        : (Class<?>[]) attrs.get("clients");
    if (clients == null || clients.length == 0) {
        // 这里就是添加了一个过滤器，过滤出只包含指定注解的接口
        scanner.addIncludeFilter(annotationTypeFilter);
        // 获取 basePackages 的值
        basePackages = getBasePackages(metadata);
    } else {
  		// ....... 省略不必要的代码
    }
    for (String basePackage : basePackages) {
        // 构建了一个 ScannedGenericBeanDefinition 的 Set 集合，也是 AnnotatedBeanDefinition的实现类
        Set<BeanDefinition> candidateComponents = scanner
            .findCandidateComponents(basePackage);
        for (BeanDefinition candidateComponent : candidateComponents) {
            if (candidateComponent instanceof AnnotatedBeanDefinition) {
                // verify annotated class is an interface
                AnnotatedBeanDefinition beanDefinition = (AnnotatedBeanDefinition) candidateComponent;
                AnnotationMetadata annotationMetadata = beanDefinition.getMetadata();
                Assert.isTrue(annotationMetadata.isInterface(),
                              "@FeignClient can only be specified on an interface");

                Map<String, Object> attributes = annotationMetadata
                    .getAnnotationAttributes(
                    FeignClient.class.getCanonicalName());

                String name = getClientName(attributes);
                // 看上文的分析，上文就是往registry 里放了一个 defaultConfiguration Feign全局的配置类，这里是放对应于FeignClient对应的configuration
                registerClientConfiguration(registry, name,
                                            attributes.get("configuration"));
                // 这里实现了feign client bean的动态构建
                registerFeignClient(registry, annotationMetadata, attributes);
            }
        }
    }
}
```

看 **Feign client Bean** 的装配方法 **registerFeignClient**

```java
private void registerFeignClient(BeanDefinitionRegistry registry,
			AnnotationMetadata annotationMetadata, Map<String, Object> attributes) {
    // 这个 className 就是加了 FeignClient 注解的
    String className = annotationMetadata.getClassName();
    // 内部创建一个 beanClass为 FeignClientFactoryBean 类型的 GenericBeanDefinition beanDefinition。
    BeanDefinitionBuilder definition = BeanDefinitionBuilder
        .genericBeanDefinition(FeignClientFactoryBean.class);
    validate(attributes);
    // 给这个 beanDefinition 的 beanClass 设置属性值 
    definition.addPropertyValue("url", getUrl(attributes));
    definition.addPropertyValue("path", getPath(attributes));
    String name = getName(attributes);
    definition.addPropertyValue("name", name);
    String contextId = getContextId(attributes);
    // 这个属性值值得关注，里面涉及一个 context 对象，见下文分析。是本文的关键点
    definition.addPropertyValue("contextId", contextId);
    definition.addPropertyValue("type", className);
    definition.addPropertyValue("decode404", attributes.get("decode404"));
    definition.addPropertyValue("fallback", attributes.get("fallback"));
    definition.addPropertyValue("fallbackFactory", attributes.get("fallbackFactory"));
    definition.setAutowireMode(AbstractBeanDefinition.AUTOWIRE_BY_TYPE);
    
    String alias = contextId + "FeignClient";
    AbstractBeanDefinition beanDefinition = definition.getBeanDefinition();
    boolean primary = (Boolean)attributes.get("primary"); // has a default, won't be null
    beanDefinition.setPrimary(primary);
    // 看看有没有 @Qualifier 注解
    String qualifier = getQualifier(attributes);
    if (StringUtils.hasText(qualifier)) {
        alias = qualifier;
    }
    // 构建一个 BeanDefinitionHolder 类
    BeanDefinitionHolder holder = new BeanDefinitionHolder(beanDefinition, className,
                                                           new String[] { alias });
    // 包装了一层具体的注册逻辑，默认注册进 BeanDefinitionMap 的 key 是 bean 的名字， 这里面把Bean的别名也注册进去了
    BeanDefinitionReaderUtils.registerBeanDefinition(holder, registry);
}
```

看到这里把 **Feign client** 和 **Feign configuration** 的 **BeanDefinition** 配置流程都看得差不多了，接下来看具体的创建 **Bean** 流程。

根据[上文分析](<https://www.jianshu.com/p/abc33207dda8>)可知， 是通过 **FeignClientFactoryBean** 的 **getObject** 方法创建的具体 **Bean** 实例的，最终源码

```java

@Override
public Object getObject() throws Exception {
    // 调用的 getTarget 方法
    return getTarget();
}

/**
	 * @param <T> the target type of the Feign client
	 * @return a {@link Feign} client created with the specified data and the context information
	 */
<T> T getTarget() {
    // 重点在这里
    FeignContext context = applicationContext.getBean(FeignContext.class);
    // 根据指定的context创建一个具体的 Feign.Builder 对象
    Feign.Builder builder = feign(context);

    // .......
}

```

可以看到， **getTarget** 方法中开始通过 **applicationContext** 对象从 **Spring IOC** 容器中获取到了 一个 **FeignContext** 类型的 **Bean**，

那么这个 **FeignContext** 什么时候创建的呢， 在 **Spring cloud starter openfeign** 里面的 **FeignAutoConfiguration** 类里面，看它的部分源码

```java
@Configuration
@ConditionalOnClass(Feign.class)
@EnableConfigurationProperties({FeignClientProperties.class, FeignHttpClientProperties.class})
public class FeignAutoConfiguration {

    // 这个 FeignClientSpecification 配置类不就是上面构建BeanDefinition配置类型的BeanDefinitionBuilder传进去的参数嘛，在这里用到了
	@Autowired(required = false)
	private List< FeignClientSpecification> configurations = new ArrayList<>();
	
    // ......

	@Bean
	public FeignContext feignContext() {
		FeignContext context = new FeignContext();
        // 传入了 configurations 配置
		context.setConfigurations(this.configurations);
		return context;
	}
    // ......
}
```

可以看一下 构建 **Feign.Builder** 对象的方法了

```java
protected Feign.Builder feign(FeignContext context) {
    FeignLoggerFactory loggerFactory = get(context, FeignLoggerFactory.class);
    Logger logger = loggerFactory.create(this.type);

    // @formatter:off  调用了这里
    Feign.Builder builder = get(context, Feign.Builder.class)
        // required values
        .logger(logger)
        .encoder(get(context, Encoder.class))
        .decoder(get(context, Decoder.class))
        .contract(get(context, Contract.class));
    // @formatter:on

    configureFeign(context, builder);

    return builder;
}

protected <T> T get(FeignContext context, Class<T> type) {
    //根据这个contextId 获取到指定的 Feign.Builder 实例
    T instance = context.getInstance(this.contextId, type);
    if (instance == null) {
        throw new IllegalStateException("No bean found of type " + type + " for "
                                        + this.contextId);
    }
    return instance;
}
```

到这里就全都 **over** 了，明白了吗？ 欢迎有问题留言。 关注我的文集，有不一样的精彩。