从使用开始解析, 需要在入口类上加上 **@EnableFeignClients** 注解，看这个注解的部分源码。

```java
@Retention(RetentionPolicy.RUNTIME)
@Target(ElementType.TYPE)
@Documented
@Import(FeignClientsRegistrar.class)
public @interface EnableFeignClients {
    //...
}
```

**Import** 了 **FeignClientsRegistrar** 这个类， 看这个类的部分源码

```java
// 实现了 ImportBeanDefinitionRegistrar 这个接口，它的作用是动态的注入bean到spring IOC中
class FeignClientsRegistrar implements ImportBeanDefinitionRegistrar,
		ResourceLoaderAware, EnvironmentAware {

	// ......
	@Override
	public void registerBeanDefinitions(AnnotationMetadata metadata,
			BeanDefinitionRegistry registry) {
		registerDefaultConfiguration(metadata, registry);
        // 注册被 @FeignClients 注解标识的接口，这里不用想，肯定是通过JDK动态代理来实现注入的，
		registerFeignClients(metadata, registry);
	}
            
     private void registerFeignClient(BeanDefinitionRegistry registry,
			AnnotationMetadata annotationMetadata, Map<String, Object> attributes) {
		String className = annotationMetadata.getClassName();
		// 使用 FeignClientFactoryBean 来创建 Bean
        BeanDefinitionBuilder definition = BeanDefinitionBuilder
				.genericBeanDefinition(FeignClientFactoryBean.class);
        
        // ......

		BeanDefinitionHolder holder = new BeanDefinitionHolder(beanDefinition, className,
				new String[] { alias });
        // 将这个 bean 注入 registry
		BeanDefinitionReaderUtils.registerBeanDefinition(holder, registry);
	}
    // ......
}
```

使用了 **FeignClientFactoryBean** 来创建指定的 **bean**，它实现了 **FactoryBean**  接口，通过它的**getObject** 方法可以创建**Bean**，看它的 **getObject** 方法，内部调用了**getTarget**方法：

```java
<T> T getTarget() {
		FeignContext context = applicationContext.getBean(FeignContext.class);
		Feign.Builder builder = feign(context);

		if (!StringUtils.hasText(this.url)) {
			// ....
            // 这个方法创建的 target 对象
			return (T) loadBalance(builder, context, new HardCodedTarget<>(this.type,
					this.name, url));
		}
    // ....
}

protected <T> T loadBalance(Feign.Builder builder, FeignContext context,
			HardCodedTarget<T> target) {
		Client client = getOptional(context, Client.class);
		if (client != null) {
			builder.client(client);
			// 根据context 获取对应的Target实现类，看 HystrixTargeter 的target 方法实现
            Targeter targeter = get(context, Targeter.class);
			return targeter.target(this, builder, context, target);
		}

		throw new IllegalStateException(
				"No Feign Client for loadBalancing defined. Did you forget to include spring-cloud-starter-netflix-ribbon?");
}
```

看 **HystrixTargeter** 的 **target** 方法实现

```java
@Override
public <T> T target(FeignClientFactoryBean factory, Feign.Builder feign, FeignContext context,Target.HardCodedTarget<T> target) {
    if (!(feign instanceof feign.hystrix.HystrixFeign.Builder)) {
        return feign.target(target);
    }
    feign.hystrix.HystrixFeign.Builder builder = (feign.hystrix.HystrixFeign.Builder) feign;
    SetterFactory setterFactory = getOptional(factory.getName(), context, SetterFactory.class);
    if (setterFactory != null) {
        builder.setterFactory(setterFactory);
    }
    Class<?> fallback = factory.getFallback();
    if (fallback != void.class) {
        return targetWithFallback(factory.getName(), context, target, builder, fallback);
    }
    Class<?> fallbackFactory = factory.getFallbackFactory();
    if (fallbackFactory != void.class) {
        return targetWithFallbackFactory(factory.getName(), context, target, builder, fallbackFactory);
    }
    // 当没有配置 fallback 或者 fallbackfactory 属性的时候，执行下面这个方法
    return feign.target(target);
}
```

```java
public <T> T target(Target<T> target) {
    return build().newInstance(target);
}
public Feign build() {
    // 通过
    SynchronousMethodHandler.Factory synchronousMethodHandlerFactory =
        new SynchronousMethodHandler.Factory(client, retryer, requestInterceptors, logger, logLevel, decode404, closeAfterDecode, propagationPolicy);
    ParseHandlersByName handlersByName =
        new ParseHandlersByName(contract, options, encoder, decoder, queryMapEncoder,
                                errorDecoder, synchronousMethodHandlerFactory);
    // 可以看到，这里使用的是 ReflectiveFeign 来动态生成指定的类实现
    return new ReflectiveFeign(handlersByName, invocationHandlerFactory, queryMapEncoder);
}
```

看 **ReflectiveFeign** 的 **newInstance** 方法

```java
public <T> T newInstance(Target<T> target) {
    // 为注解标识的接口里的每个方法创建一个 SynchronousMethodHandler 对象， 由它来执行对应的调用流程
    Map<String, MethodHandler> nameToHandler = targetToHandlersByName.apply(target);
    Map<Method, MethodHandler> methodToHandler = new LinkedHashMap<Method, MethodHandler>();
    List<DefaultMethodHandler> defaultMethodHandlers = new LinkedList<DefaultMethodHandler>();
    for (Method method : target.type().getMethods()) {
        if (method.getDeclaringClass() == Object.class) {
            continue;
        } else if (Util.isDefault(method)) {
            DefaultMethodHandler handler = new DefaultMethodHandler(method);
            defaultMethodHandlers.add(handler);
            methodToHandler.put(method, handler);
        } else {
            methodToHandler.put(method, nameToHandler.get(Feign.configKey(target.type(), method)));
        }
    }
    // 这里就调用了 InvocationHandlerFactory.create 方法
    InvocationHandler handler = factory.create(target, methodToHandler);
    T proxy = (T) Proxy.newProxyInstance(target.type().getClassLoader(),
                                         new Class<?>[] {target.type()}, handler);
    for (DefaultMethodHandler defaultMethodHandler : defaultMethodHandlers) {
        defaultMethodHandler.bindTo(proxy);
    }
    return proxy;
}

// .......
public Map<String, MethodHandler> apply(Target key) {
     List<MethodMetadata> metadata = contract.parseAndValidatateMetadata(key.type());
     Map<String, MethodHandler> result = new LinkedHashMap<String, MethodHandler>();
     for (MethodMetadata md : metadata) {
         BuildTemplateByResolvingArgs buildTemplate;
         if (!md.formParams().isEmpty() && md.template().bodyTemplate() == null) {
             buildTemplate = new BuildFormEncodedTemplateFromArgs(md, encoder, queryMapEncoder);
         } else if (md.bodyIndex() != null) {
             buildTemplate = new BuildEncodedTemplateFromArgs(md, encoder, queryMapEncoder);
         } else {
             buildTemplate = new BuildTemplateByResolvingArgs(md, queryMapEncoder);
         }
         // 创建了一个 SynchronousMethodHandler 对象
         result.put(md.configKey(),
                    factory.create(key, md, buildTemplate, options, decoder, errorDecoder));
     }
     return result;
 }

```

来看  **InvocationHandlerFactory.create** 方法，里面调用了 **ReflectiveFeign.FeignInvocationHandler** 方法

```java
// 实现了 reflect 包下的 InvocationHandler 类，会自动执行 invoke 方法
static class FeignInvocationHandler implements InvocationHandler {

    private final Target target;
    private final Map<Method, MethodHandler> dispatch;

    FeignInvocationHandler(Target target, Map<Method, MethodHandler> dispatch) {
      this.target = checkNotNull(target, "target");
      this.dispatch = checkNotNull(dispatch, "dispatch for %s", target);
    }

    @Override
    public Object invoke(Object proxy, Method method, Object[] args) throws Throwable {
      if ("equals".equals(method.getName())) {
        try {
          Object otherHandler =
              args.length > 0 && args[0] != null ? Proxy.getInvocationHandler(args[0]) : null;
          return equals(otherHandler);
        } catch (IllegalArgumentException e) {
          return false;
        }
      } else if ("hashCode".equals(method.getName())) {
        return hashCode();
      } else if ("toString".equals(method.getName())) {
        return toString();
      }
      // 这个 dispatch 就是上面 newInstance 方法里面的 methodToHandler map对象，会获取到一个SynchronousMethodHandler 对象，执行它的 invoke 方法。
      return dispatch.get(method).invoke(args);
    }
 }
```

看 **SynchronousMethodHandler** 的 **invoke** 方法

```java
@Override
public Object invoke(Object[] argv) throws Throwable {
    // 构建一个 RequestTemplate 请求对象
    RequestTemplate template = buildTemplateFromArgs.create(argv);
    Retryer retryer = this.retryer.clone();
    while (true) {
        try {
            // 这里执行请求并解码结果
            return executeAndDecode(template);
        } catch (RetryableException e) {
            try {
                retryer.continueOrPropagate(e);
            } catch (RetryableException th) {
                Throwable cause = th.getCause();
                if (propagationPolicy == UNWRAP && cause != null) {
                    throw cause;
                } else {
                    throw th;
                }
            }
            if (logLevel != Logger.Level.NONE) {
                logger.logRetry(metadata.configKey(), logLevel);
            }
            continue;
        }
    }
}
```

看 **executeAndDecode** 方法：

```java
  Object executeAndDecode(RequestTemplate template) throws Throwable {
    // 构建请求
    Request request = targetRequest(template);

    if (logLevel != Logger.Level.NONE) {
      logger.logRequest(metadata.configKey(), logLevel, request);
    }

    Response response;
    long start = System.nanoTime();
    try {
      // 执行请求 
      response = client.execute(request, options);
    } catch (IOException e) {
      if (logLevel != Logger.Level.NONE) {
        logger.logIOException(metadata.configKey(), logLevel, e, elapsedTime(start));
      }
      throw errorExecuting(request, e);
    }
    long elapsedTime = TimeUnit.NANOSECONDS.toMillis(System.nanoTime() - start);

    boolean shouldClose = true;
    try {
      if (logLevel != Logger.Level.NONE) {
        response =
            logger.logAndRebufferResponse(metadata.configKey(), logLevel, response, elapsedTime);
      }
      if (Response.class == metadata.returnType()) {
        if (response.body() == null) {
          return response;
        }
        if (response.body().length() == null ||
            response.body().length() > MAX_RESPONSE_BUFFER_SIZE) {
          shouldClose = false;
          return response;
        }
        // Ensure the response body is disconnected
        byte[] bodyData = Util.toByteArray(response.body().asInputStream());
        return response.toBuilder().body(bodyData).build();
      }
      // 如果响应的状态码为200的时候走这里
      if (response.status() >= 200 && response.status() < 300) {
        if (void.class == metadata.returnType()) {
          return null;
        } else {
          Object result = decode(response);
          shouldClose = closeAfterDecode;
          return result;
        }
      } else if (decode404 && response.status() == 404 && void.class != metadata.returnType()) {
        Object result = decode(response);
        shouldClose = closeAfterDecode;
        return result;
      } else {
        throw errorDecoder.decode(metadata.configKey(), response);
      }
    } catch (IOException e) {
      if (logLevel != Logger.Level.NONE) {
        logger.logIOException(metadata.configKey(), logLevel, e, elapsedTime);
      }
      throw errorReading(request, response, e);
    } finally {
      if (shouldClose) {
        ensureClosed(response.body());
      }
    }
  }
```

到这里我们已经看完了 **Feign** 执行的大致流程， 还有一些细节没进去看。下次再说吧。关注我的文集