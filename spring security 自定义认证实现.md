# spring security restful api 认证实现

实现功能: 自定义接口，字段实现登录。实现**remember-me**功能.。一个 **restful api** 功能的**web**服务。

首先，得实现自定义接口实现登录，这里假设**URL**为  **POST /user/login** 数据类型为 **json**， 官方使用的是 **AbstractAuthenticationProcessingFilter**, 只接受**form**表单形式的内容。所以我们需要替换它.

新建一个filter， 实现自 **UsernamePasswordAuthenticationFilter** 的父类 **AbstractAuthenticationProcessingFilter**， 覆写它的 **attemptAuthentication** 方法

```java
// 需要实现父类的构造函数，传入一个登录的url， 这里就是 /user/login
public DMAuthenticationFilter(String defaultFilterProcessesUrl) {
    super(defaultFilterProcessesUrl);
}

@Override
public Authentication attemptAuthentication(HttpServletRequest request, HttpServletResponse response) throws AuthenticationException {
    // 判断请求头类型，只接受json格式的数据
    if(request.getContentType().equals(MediaType.APPLICATION_JSON_UTF8_VALUE)
       ||request.getContentType().equals(MediaType.APPLICATION_JSON_VALUE)){
        log.info("dm authentication start");
        UsernamePasswordAuthenticationToken authRequest = null;
        try{
            InputStream is = request.getInputStream();
            // 将 request的 stream 流反序列化为自定义的一个 登录封装对象LoginRequestVO， 里面目前只有username和password字段， 反序列化框架使用的是 jackson
            LoginRequestVO loginRequestVO = JSONSnakeUtils.readValue(is, LoginRequestVO.class);
            authRequest = new UsernamePasswordAuthenticationToken(
                loginRequestVO.getUsername(), loginRequestVO.getPassword());
        }catch (IOException e) {
            e.printStackTrace();
            authRequest = new UsernamePasswordAuthenticationToken(
                "", "");
        }finally {
            authRequest.setDetails(authenticationDetailsSource.buildDetails(request));
            return this.getAuthenticationManager().authenticate(authRequest);
        }
    }
    return null;
}
```

然后需要配置替换，把这个filter替换掉默认的 UsernamePasswordAuthenticationFilter。新建 SecurityConfig 类，继承 WebSecurityConfigurerAdapter 接口， 加上 @EnableWebSecurity 注解

```java
@EnableWebSecurity
@Configuration
public class SecurityConfig extends WebSecurityConfigurerAdapter {

    @Autowired
    private UserService userService;

    @Bean
    public AccessDeniedHandler accessDeniedHandler(){
        return new RestAccessDeniedHanlder();
    }

    @Bean
    public PasswordEncoder passwordEncoder(){
        return new BCryptPasswordEncoder();
    }

    @Bean
    public AuthenticationEntryPoint authenticationEntryPoint(){
        return new UrlAuthenticationEntryPoint();
    }

    @Override
    protected void configure(HttpSecurity http) throws Exception {
        http.authorizeRequests()
                .antMatchers("/user/login").permitAll()
                .anyRequest().authenticated().and()
            .csrf().disable()
            .exceptionHandling().accessDeniedHandler(accessDeniedHandler()).and()
            .exceptionHandling().authenticationEntryPoint(authenticationEntryPoint()).and()
            .addFilterAt(dmAuthenticationFilter(), UsernamePasswordAuthenticationFilter.class);
    }

    @Bean
    public DMAuthenticationFilter dmAuthenticationFilter() throws Exception {
        DMAuthenticationFilter dmAuthenticationFilter = new DMAuthenticationFilter("/user/login");
        dmAuthenticationFilter.setAuthenticationManager(authenticationManager());
        dmAuthenticationFilter.setAuthenticationSuccessHandler(new LoginSuccessHandler());
        dmAuthenticationFilter.setAuthenticationFailureHandler(new LoginFailedHandler());
        dmAuthenticationFilter.setRememberMeServices(rememberMeServices());
        return dmAuthenticationFilter;
    }

    @Override
    public void configure(WebSecurity web) throws Exception {
        web.ignoring().antMatchers(HttpMethod.OPTIONS, "/**")
                .antMatchers("/resource/**");
    }

    @Override
    protected void configure(AuthenticationManagerBuilder auth) throws Exception {
        auth.userDetailsService(userService);
    }

    @Override
    public AuthenticationManager authenticationManagerBean() throws Exception {
        return super.authenticationManagerBean();
    }

}


```

**userService** 是自定义的查询用户类，实现 **UserDetailsService** 类的**loadUserByUsername**方法。到目前为止，已经完成了自定义**json**格式的前后端交互登录交互流程。还需要加上**remember-me**功能。本篇日志的重点就是这个功能的添加

spring security 实现remember-me功能的关键服务类是 **RememberMeServices**， 这里使用的是 **TokenBasedRememberMeServices** 实现类， 这个类有个抽象父类  **AbstractRememberMeServices**， 当携带 **remmember-me** 的**cookie** 登录时会走这个类的**autoLogin**方法

```java
@Override
public final Authentication autoLogin(HttpServletRequest request,
                                      HttpServletResponse response) {
    String rememberMeCookie = extractRememberMeCookie(request);

    if (rememberMeCookie == null) {
        return null;
    }

    logger.debug("Remember-me cookie detected");

    if (rememberMeCookie.length() == 0) {
        logger.debug("Cookie was empty");
        cancelCookie(request, response);
        return null;
    }

    UserDetails user = null;

    try {
        // 从 remember-me cookie 中获取用户名和其他一些用户数据，组成一个String数组
        String[] cookieTokens = decodeCookie(rememberMeCookie);
        // 这里调用实现类的 processAutoLoginCookie 处理登录方法
        user = processAutoLoginCookie(cookieTokens, request, response);
        userDetailsChecker.check(user);

        logger.debug("Remember-me cookie accepted");

        return createSuccessfulAuthentication(request, user);
    }
    catch (CookieTheftException cte) {
        cancelCookie(request, response);
        throw cte;
    }
    catch (UsernameNotFoundException noUser) {
        logger.debug("Remember-me login was valid but corresponding user not found.",
                     noUser);
    }
    catch (InvalidCookieException invalidCookie) {
        logger.debug("Invalid remember-me cookie: " + invalidCookie.getMessage());
    }
    catch (AccountStatusException statusInvalid) {
        logger.debug("Invalid UserDetails: " + statusInvalid.getMessage());
    }
    catch (RememberMeAuthenticationException e) {
        logger.debug(e.getMessage());
    }

    cancelCookie(request, response);
    return null;
}
```

跟进实现类的**processAutoLoginCookie**方法

```java
@Override
protected UserDetails processAutoLoginCookie(String[] cookieTokens,
                                             HttpServletRequest request, HttpServletResponse response) {

    if (cookieTokens.length != 3) {
        throw new InvalidCookieException("Cookie token did not contain 3"
                                         + " tokens, but contained '" + Arrays.asList(cookieTokens) + "'");
    }

    long tokenExpiryTime;

    try {
        tokenExpiryTime = new Long(cookieTokens[1]).longValue();
    }
    catch (NumberFormatException nfe) {
        throw new InvalidCookieException(
            "Cookie token[1] did not contain a valid number (contained '"
            + cookieTokens[1] + "')");
    }

    if (isTokenExpired(tokenExpiryTime)) {
        throw new InvalidCookieException("Cookie token[1] has expired (expired on '"
                                         + new Date(tokenExpiryTime) + "'; current time is '" + new Date()
                                         + "')");
    }

    // Check the user exists.
    // Defer lookup until after expiry time checked, to possibly avoid expensive
    // database call.
    
    // 调用自定义的userService的loadUserByUsername方法来获取用户信息
    UserDetails userDetails = getUserDetailsService().loadUserByUsername(
        cookieTokens[0]);

    // Check signature of token matches remaining details.
    // Must do this after user lookup, as we need the DAO-derived password.
    // If efficiency was a major issue, just add in a UserCache implementation,
    // but recall that this method is usually only called once per HttpSession - if
    // the token is valid,
    // it will cause SecurityContextHolder population, whilst if invalid, will cause
    // the cookie to be cancelled.
    String expectedTokenSignature = makeTokenSignature(tokenExpiryTime,
                                                       userDetails.getUsername(), userDetails.getPassword());

    if (!equals(expectedTokenSignature, cookieTokens[2])) {
        throw new InvalidCookieException("Cookie token[2] contained signature '"
                                         + cookieTokens[2] + "' but expected '" + expectedTokenSignature + "'");
    }

    return userDetails;
}
```

好，处理**remember-me cookie**的登录流程结束，来看看**remember-me cookie**的生产流程吧，当登陆成功后，会调用 **AbstractAuthenticationProcessingFilter** 的  **successfulAuthentication** 方法，它会调用 

```java
rememberMeServices.loginSuccess(request, response, authResult);
```

这个方法来处理**remember-me**的**cookie**问题。回到上面的 **AbstractRememberMeServices** 的  **loginSuccess** 方法，它首先有个判断判断是否为**remember-me**的请求

```java
if (!rememberMeRequested(request, parameter)) {
    logger.debug("Remember-me login not requested.");
    return;
}
```

看他的实现:

```java
protected boolean rememberMeRequested(HttpServletRequest request, String parameter) {
    // 这个通过初始化这个bean的时候可以传递这个参数设置
    if (alwaysRemember) {
        return true;
    }
    // 从request中获取parameter这个参数，判断是否需要设置rememberme这个cookie
    String paramValue = request.getParameter(parameter);
	
    // ......

    return false;
}
```

由于我们是**json**格式的数据，所以这里肯定得重写, 继承**TokenBasedRememberMeServices** 类，这里命名为 **DMTokenBasedRememberMeServices**，重写这个**rememberMeRequested** 方法，添加**alwaysRemember**变量

```java
// 从 request 的 Inputstream 流中获取请求的数据
InputStream is = request.getInputStream();
LoginRequestVO loginRequestVO = JSONSnakeUtils.readValue(is, LoginRequestVO.class);  // 转换为自定义的登录对象
if(loginRequestVO.getRememberMe() != null && loginRequestVO.getRememberMe()){
    return true;  // 如果请求中有remember-me 字段并且值为true则返回true
}
```

在这里就遇到问题了，前面的自定义请求中已经从**request**中获取了一次**stream**流，这里再次获取是获取不到的。这个问题怕是许多新手都会忽略的问题，解决方案有两种，一是将里面的数据写入缓存，存入**request**的一个**attribute**中或者**session**中，通过**getAttribute/setAttribute**方法可以多次获取到。另一种解决方案是用一个 **HttpServletRequestWrapper** 来包装请求，缓存请求数据，看具体实现代码:

```java
 ResettableStreamHttpServletRequest wrappedRequest = new ResettableStreamHttpServletRequest(
                (HttpServletRequest) request);
        // wrappedRequest.getInputStream().read();
String body = IOUtils.toString(wrappedRequest.getReader());
auditor.audit(wrappedRequest.getRequestURI(),wrappedRequest.getUserPrincipal(), body);
wrappedRequest.resetInputStream();
chain.doFilter(wrappedRequest, response);

```

把这个隐形的**bug**修改完后，然后把开始的**filter**加上这段逻辑，就可以往我们自定义的**spring security**类中加上**remember-me**的相关配置了

```java
@Override
protected void configure(HttpSecurity http) throws Exception {
    http.authorizeRequests()
        .antMatchers("/user/login").permitAll()
        .antMatchers("/user/registry").permitAll()
        .antMatchers("/error/**").permitAll()
        .antMatchers("/agent/**").permitAll()
        .antMatchers("/api/resource").permitAll()
        .antMatchers("/enums/**").permitAll()
        .antMatchers("/manager/agent/**").permitAll()
        .anyRequest().authenticated().and()
        .csrf().disable()
        .exceptionHandling().accessDeniedHandler(accessDeniedHandler()).and()
        .exceptionHandling().authenticationEntryPoint(authenticationEntryPoint()).and()
        .addFilterAt(dmAuthenticationFilter(), UsernamePasswordAuthenticationFilter.class)
        .addFilterBefore(rememberMeAuthenticationFilter(), DMAuthenticationFilter.class);
}

public RememberMeAuthenticationFilter rememberMeAuthenticationFilter() throws Exception {
    return new RememberMeAuthenticationFilter(authenticationManagerBean(),rememberMeServices());
}

@Bean
public RememberMeServices rememberMeServices(){
    DMTokenBasedRememberMeServices tokenBasedRememberMeServices = new DMTokenBasedRememberMeServices("steve", userService);
    tokenBasedRememberMeServices.setCookieName("dm-remember-me");
    tokenBasedRememberMeServices.setTokenValiditySeconds(timeout);
//  tokenBasedRememberMeServices.setAlwaysRemember(true);   // 这里的 true 就是上面的 alwaysRemember 变量
    return tokenBasedRememberMeServices;
}

public RememberMeAuthenticationProvider rememberMeAuthenticationProvider(){
    return new RememberMeAuthenticationProvider("steve");    // 这个字符串是个加密的key，加密cookie用的
}

@Override
protected void configure(AuthenticationManagerBuilder auth) throws Exception {
    auth.userDetailsService(userService);
    auth.authenticationProvider(rememberMeAuthenticationProvider());  // 需要注入 provider
}



```

尽情尝试吧 。