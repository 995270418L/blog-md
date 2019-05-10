Spring cloud Feign 执行请求源码解析

回到[第一篇](https://www.jianshu.com/p/abc33207dda8)看执行流程，找到 **SynchronousMethodHandler** 的 **invoke** 方法，看第一步，通过 **buildTemplateFromArgs.create(argv);** 方法构建一个 **RequestTemplate**，看这个方法的内部实现

```java
@Override
    public RequestTemplate create(Object[] argv) {
      // 大量用到了 metadata 对象，是 MethodMetadata 类型的。 看来有必要对 metadata 这个对象有深入的了解了。
      RequestTemplate mutable = RequestTemplate.from(metadata.template());
      if (metadata.urlIndex() != null) {
        int urlIndex = metadata.urlIndex();
        checkArgument(argv[urlIndex] != null, "URI parameter %s was null", urlIndex);
        mutable.target(String.valueOf(argv[urlIndex]));
      }
      Map<String, Object> varBuilder = new LinkedHashMap<String, Object>();
      for (Entry<Integer, Collection<String>> entry : metadata.indexToName().entrySet()) {
        int i = entry.getKey();
        Object value = argv[entry.getKey()];
        if (value != null) { // Null values are skipped.
          if (indexToExpander.containsKey(i)) {
            value = expandElements(indexToExpander.get(i), value);
          }
          for (String name : entry.getValue()) {
            varBuilder.put(name, value);
          }
        }
      }
      // .......
    }
```

为了便于后面解析源码，这里我们回到前面看看 **metadata** 对象的创建过程。这个属性是在 **BuildTemplateByResolvingArgs** 这个静态内部类里面的，它的上层类是 **ReflectiveFeign**， 在第一篇文章我们知道， 动态代理类的生成就是通过这个类的 **newInstance** 方法做到的，再看一遍这个方法

```java
@Override
public <T> T newInstance(Target<T> target) {
    // 这个 target 对象是 HardCodeTarget 类型的，里面保存了我们加上@FeignClient注解接口的Type、name、url三个信息。
    // targetToHandlersByName 是 ParseHandlersByName 类型的， 这个对象里面有很多我们配置的元数据（metadata），那肯定得看看这个apply方法了
    Map<String, MethodHandler> nameToHandler = targetToHandlersByName.apply(target);
    Map<Method, MethodHandler> methodToHandler = new LinkedHashMap<Method, MethodHandler>();
    List<DefaultMethodHandler> defaultMethodHandlers = new LinkedList<DefaultMethodHandler>();

   // ...... 省略无效代码
}
```

看  **ParseHandlersByName** 的 **apply** 方法

```java
public Map<String, MethodHandler> apply(Target key) {
    // 这里我们的主角 MethodMetadata 出现了，看来就在这个方法里了，key.type 方法就是标注了@FeignClient注解的接口了
    List<MethodMetadata> metadata = contract.parseAndValidatateMetadata(key.type());
    Map<String, MethodHandler> result = new LinkedHashMap<String, MethodHandler>();
    for (MethodMetadata md : metadata) {
        // 这里就会把metadata 通过构造函数的方式传给 BuildTemplateByResolvingArgs 对象。
        BuildTemplateByResolvingArgs buildTemplate;
        if (!md.formParams().isEmpty() && md.template().bodyTemplate() == null) {
            buildTemplate = new BuildFormEncodedTemplateFromArgs(md, encoder, queryMapEncoder);
        } else if (md.bodyIndex() != null) {
            buildTemplate = new BuildEncodedTemplateFromArgs(md, encoder, queryMapEncoder);
        } else {
            buildTemplate = new BuildTemplateByResolvingArgs(md, queryMapEncoder);
        }
        result.put(md.configKey(),
                   factory.create(key, md, buildTemplate, options, decoder, errorDecoder));
    }
    return result;
}
```

看 **contract.parseAndValidatateMetadata(key.type())**  方法实现（ **contract** 默认实现是 **BaseContract** ）

```java
@Override
public List<MethodMetadata> parseAndValidatateMetadata(Class<?> targetType) {
    // ..... 
    Map<String, MethodMetadata> result = new LinkedHashMap<String, MethodMetadata>();
    for (Method method : targetType.getMethods()) {
        // 判断是否是 default 修饰的 method
        if (method.getDeclaringClass() == Object.class ||
            (method.getModifiers() & Modifier.STATIC) != 0 ||
            Util.isDefault(method)) {
            continue;
        }
        // 这里根据接口名和方法名生成metedata，看具体实现细节
        MethodMetadata metadata = parseAndValidateMetadata(targetType, method);
        checkState(!result.containsKey(metadata.configKey()), "Overrides unsupported: %s",
                   metadata.configKey());
        result.put(metadata.configKey(), metadata);
    }
    return new ArrayList<>(result.values());
}
```

我们知道 **Spring cloud Feign** 可以使用 **Spring mvc** 的方式配置调用接口，埋个伏笔，继续往下看 **parseAndValidateMetadata** 方法的实现细节

```java
protected MethodMetadata parseAndValidateMetadata(Class<?> targetType, Method method) {
    // new 了这个对象
    MethodMetadata data = new MethodMetadata();
    // 就是指定方法的返回类型
    data.returnType(Types.resolve(targetType, targetType, method.getGenericReturnType()));
    // configKey格式: 接口名#方法名(参数类型...)
    data.configKey(Feign.configKey(targetType, method));
    // 判断接口是否继承了别的接口
    if (targetType.getInterfaces().length == 1) {
        processAnnotationOnClass(data, targetType.getInterfaces()[0]);
    }
	// 这里就会解析标识了 spring mvc 的注解，并把数据放入 data 这个对象里面
    processAnnotationOnClass(data, targetType);
    for (Annotation methodAnnotation : method.getAnnotations()) {
        processAnnotationOnMethod(data, methodAnnotation, method);
    }
    // ...... 剩余的代码就没有看到的必要了
    return data;
}
```

这里做个短暂的总结， 记录一下 **metadata** 的各个字段的意义

```yaml
configKey：接口名#方法名(参数类型...)
returnType: 方法的返回类型
urlIndex： 记录参数里面 类型属于URI.class的index位置（下标从0开始）
bodyIndex：只要参数类型不是URI.class,就记录参数的下标位置
bodyType：上面这个index对应的类型
headerMapIndex： 
queryMapIndex： 
queryMapEncoded：
template： RequestTemplate 类型，封装了一些请求的metadata(方法、uri)
formParams：
indexToName：
indexToExpanderClass： 
indexToEncoded：
indexToExpander： @Param 注解的Map

```

未解释的大都和参数相关，这边就不一一去尝试解释了，有兴趣的都可以**debug**看看。到这里为止，**metadata**的生成已经看完了，每个方法对应一个**metadata**，回到文章开头部分，接下来看看RequestTemplate的构成, 看**create**方法

```java
// argv 就是我们要调用的方法的参数列表
@Override
public RequestTemplate create(Object[] argv) {
    // 这里就是根据 metadata 的 template 模板new一个RequestTemplate
    RequestTemplate mutable = RequestTemplate.from(metadata.template());
    if (metadata.urlIndex() != null) {
        int urlIndex = metadata.urlIndex();
        checkArgument(argv[urlIndex] != null, "URI parameter %s was null", urlIndex);
        mutable.target(String.valueOf(argv[urlIndex]));
    }
    Map<String, Object> varBuilder = new LinkedHashMap<String, Object>();
    for (Entry<Integer, Collection<String>> entry : metadata.indexToName().entrySet()) {
        int i = entry.getKey();
        // 这里也是和参数有关
        Object value = argv[entry.getKey()];
        if (value != null) { // Null values are skipped.
            if (indexToExpander.containsKey(i)) {
                value = expandElements(indexToExpander.get(i), value);
            }
            for (String name : entry.getValue()) {
                varBuilder.put(name, value);
            }
        }
    }
    // resolve 方法是对argv中请求参数进行编码的地方，然后生成一个新的RequestTemplate对象。
    RequestTemplate template = resolve(argv, mutable, varBuilder);
    // 默认我觉得应该是用不到这些东西的
    if (metadata.queryMapIndex() != null) {
        // add query map parameters after initial resolve so that they take
        // precedence over any predefined values
        Object value = argv[metadata.queryMapIndex()];
        Map<String, Object> queryMap = toQueryMap(value);
        template = addQueryMapQueryParameters(queryMap, template);
    }
    // 这个参数也是一样的
    if (metadata.headerMapIndex() != null) {
        template =
            addHeaderMapHeaders((Map<String, Object>) argv[metadata.headerMapIndex()], template);
    }

    return template;
}
```

看看 **ReflectiveFeign** 的 **resolve** 方法（当**Feign**请求方法类型为**POST** 时走的这个）

```java
@Override
protected RequestTemplate resolve(Object[] argv,
                                  RequestTemplate mutable,
                                  Map<String, Object> variables) {
    // 默认你的请求参数不用注解(@RequestParam,@RequestBody)标记的话, Feign 会当做RequestBody处理，方法为post，只能有一个这种参数，所以这里可以这样玩。
    Object body = argv[metadata.bodyIndex()];
    checkArgument(body != null, "Body parameter %s was null", metadata.bodyIndex());
    try {
        // 对body进行压缩，默认使用SpringEncode这个类处理的，对mutable进行填充数据。有兴趣的可以细看
        encoder.encode(body, metadata.bodyType(), mutable);
    } catch (EncodeException e) {
        throw e;
    } catch (RuntimeException e) {
        throw new EncodeException(e.getMessage(), e);
    }
    // 执行父类的resolve方法
    return super.resolve(argv, mutable, variables);
}
```

当 **Feign** 请求方法类型为 **GET** 类型时走的下面这个

```java
public RequestTemplate resolve(Map<String, ?> variables) {

    StringBuilder uri = new StringBuilder();

    /* create a new template form this one, but explicitly */
    RequestTemplate resolved = RequestTemplate.from(this);

    if (this.uriTemplate == null) {
        /* create a new uri template using the default root */
        this.uriTemplate = UriTemplate.create("", !this.decodeSlash, this.charset);
    }

    uri.append(this.uriTemplate.expand(variables));

    /*
     * for simplicity, combine the queries into the uri and use the resulting uri to seed the
     * resolved template.
     */
    // 组装了查询参数，组成请求的字符串
    if (!this.queries.isEmpty()) {
        /*
       * since we only want to keep resolved query values, reset any queries on the resolved copy
       */
        resolved.queries(Collections.emptyMap());
        StringBuilder query = new StringBuilder();
        Iterator<QueryTemplate> queryTemplates = this.queries.values().iterator();

        while (queryTemplates.hasNext()) {
            QueryTemplate queryTemplate = queryTemplates.next();
            String queryExpanded = queryTemplate.expand(variables);
            if (Util.isNotBlank(queryExpanded)) {
                query.append(queryTemplate.expand(variables));
                if (queryTemplates.hasNext()) {
                    query.append("&");
                }
            }
        }

        String queryString = query.toString();
        if (!queryString.isEmpty()) {
            Matcher queryMatcher = QUERY_STRING_PATTERN.matcher(uri);
            if (queryMatcher.find()) {
                /* the uri already has a query, so any additional queries should be appended */
                uri.append("&");
            } else {
                uri.append("?");
            }
            uri.append(queryString);
        }
    }

    /* add the uri to result */
    resolved.uri(uri.toString());

    /* headers */
    if (!this.headers.isEmpty()) {
        /*
       * same as the query string, we only want to keep resolved values, so clear the header map on
       * the resolved instance
       */
        resolved.headers(Collections.emptyMap());
        for (HeaderTemplate headerTemplate : this.headers.values()) {
            /* resolve the header */
            String header = headerTemplate.expand(variables);
            if (!header.isEmpty()) {
                /* split off the header values and add it to the resolved template */
                String headerValues = header.substring(header.indexOf(" ") + 1);
                if (!headerValues.isEmpty()) {
                    resolved.header(headerTemplate.getName(), headerValues);
                }
            }
        }
    }

    resolved.body(this.body.expand(variables));

    /* mark the new template resolved */
    resolved.resolved = true;
    return resolved;
}
```

上面官方源码都写得清清楚楚了。就不怎么解释了。然后回到 **create** 方法后面的流程，回到 **SynchronousMethodHandler** 的 **invoke** 方法，看 **executeAndDecode** 方法

```java
// template 为上面都构建好了的 template
Object executeAndDecode(RequestTemplate template) throws Throwable {
    // 这个方法里面会根据当前方法的template和类的HardCodeTarget属性创建一个请求。项目如果使用了注册中心，这里还没有表明具体的ip地址。
    Request request = targetRequest(template);

    if (logLevel != Logger.Level.NONE) {
      logger.logRequest(metadata.configKey(), logLevel, request);
    }

    Response response;
    long start = System.nanoTime();
    try {
      // 如果项目使用了Ribbon进行负载均衡策略，这里就会使用 LoadBalancerFeignClient 作为 Client 接口的实现。
      response = client.execute(request, options);
    } catch (IOException e) {
      if (logLevel != Logger.Level.NONE) {
        logger.logIOException(metadata.configKey(), logLevel, e, elapsedTime(start));
      }
      throw errorExecuting(request, e);
    }
    long elapsedTime = TimeUnit.NANOSECONDS.toMillis(System.nanoTime() - start);

  	// ...... 省略剩余的代码
  }
```

看  **LoadBalancerFeignClient** 的 **execute** 方法

```java
@Override
public Response execute(Request request, Request.Options options) throws IOException {
    try {
        // 构建 URI 对象 eg: https://invoke-application-name/path
        URI asUri = URI.create(request.url());
        // 获取要调用的客户端在注册中心的 application-name eg: invoke-application-name
        String clientName = asUri.getHost();
        // 去除application-name 的部分  eg: http:///path
        URI uriWithoutHost = cleanUrl(request.url(), clientName);
        // 这个 delegate 感觉像是 spring 的老套路了， 这里就是具体执行请求的客户端了实现了（有okHttpClient，ApacheHttpClient和默认的实现）
        FeignLoadBalancer.RibbonRequest ribbonRequest = new FeignLoadBalancer.RibbonRequest(
            this.delegate, request, uriWithoutHost);
        
        IClientConfig requestConfig = getClientConfig(options, clientName);
        // 真正构建请求的是在这个方法里
        return lbClient(clientName).executeWithLoadBalancer(ribbonRequest,
                                                            requestConfig).toResponse();
    }
    catch (ClientException e) {
        IOException io = findIOException(e);
        if (io != null) {
            throw io;
        }
        throw new RuntimeException(e);
    }
}
```

看 **AbstractLoadBalancerAwareClient** 的 **executeWithLoadBalancer** 方法

```java
 public T executeWithLoadBalancer(final S request, final IClientConfig requestConfig) throws ClientException {
     // 这里就给 LoadBalancerCommand 设置了一些参数， 有LoadBanlanceContext 和 RetryHandler 和 LoadBanlanceUri
     LoadBalancerCommand<T> command = buildLoadBalancerCommand(request, requestConfig);
     
     try {
         // 这里的 submit 方法
         return command.submit(
             
             new ServerOperation<T>() {
                 // ...... 省略具体实现代码
             })
             .toBlocking()
             .single();
     } catch (Exception e) {
         Throwable t = e.getCause();
         if (t instanceof ClientException) {
             throw (ClientException) t;
         } else {
             throw new ClientException(e);
         }
     }

 }
```

**submit** 方法内部实现

```java
  public Observable<T> submit(final ServerOperation<T> operation) {
        final ExecutionInfoContext context = new ExecutionInfoContext();
        
        if (listenerInvoker != null) {
            try {
                listenerInvoker.onExecutionStart();
            } catch (AbortExecutionException e) {
                return Observable.error(e);
            }
        }

        final int maxRetrysSame = retryHandler.getMaxRetriesOnSameServer();
        final int maxRetrysNext = retryHandler.getMaxRetriesOnNextServer();

        // Use the load balancer 默认server为空，进入 selectServer 方法
        Observable<T> o = 
                (server == null ? selectServer() : Observable.just(server))
                .concatMap(new Func1<Server, Observable<T>>() {
                   // ...... 我们看到 selectServer方法就行了。剩余的下回再解释，不然文章太长了。
        });
    }
```

看 **selectServer** 方法

```java
private Observable<Server> selectServer() {
    return Observable.create(new OnSubscribe<Server>() {
        // 看这个回调方法， create方式就是注册了一个 Observable hook
        @Override
        public void call(Subscriber<? super Server> next) {
            try {
               	// 这里就涉及到了具体的选择服务端的流程了，这个 loadBalancerContext 是 FeignLoadBalancer 类型的
                Server server = loadBalancerContext.getServerFromLoadBalancer(loadBalancerURI, loadBalancerKey);   
                next.onNext(server);
                next.onCompleted();
            } catch (Exception e) {
                next.onError(e);
            }
        }
    });
}
```

看 **LoadBalancerContext** 的 **getServerFromLoadBalancer** 方法， 这里面无用代码太多了，我们就只看这部分

```java
// 这个对象封装了所有的服务端地址信息, 这里扯来就比较长了，涉及到ribbon的很多细节，这里就不一一细说了，关注我的文集，我以后有空会把Ribbon也分析的透透彻彻的。
ILoadBalancer lb = getLoadBalancer();
if (host == null) {
    // Partial URI or no URI Case
    // well we have to just get the right instances from lb - or we fall back
    if (lb != null){
        // 根据Ribbon的负载均衡策略选择指定的服务器
        Server svc = lb.chooseServer(loadBalancerKey);
```

就到这里为止吧，选择完了服务器，剩下的只是执行请求了。 欢迎有问题留言。 关注我的[文集]([https://www.jianshu.com/nb/34247971](https://www.jianshu.com/nb/34247971)
)，有不一样的精彩。

