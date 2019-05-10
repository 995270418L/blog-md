# Spring security 整合 Spring data ldap 实现对LDAP服务端认证

### 需求

用户的权限信息保存在内网的Active Directory(简称AD)上面，需要通过客户端的形式从AD上拉取用户的组信息，对应到我们自己统一的用户认证体系里面去。简单的做法就是将AD上的**组**概念对应到传统的RBAC模型里面的**角色**概念。

### 技术实现

新建spring boot项目, 引入如下依赖:

```xml
 <parent>
     <groupId>org.springframework.boot</groupId>
     <artifactId>spring-boot-starter-parent</artifactId>
     <version>2.1.3.RELEASE</version>
     <relativePath/> <!-- lookup parent from repository -->
</parent> 
<dependencies>
        <!--common pom start-->
    <dependency>
        <groupId>org.springframework.boot</groupId>
        <artifactId>spring-boot-starter</artifactId>
    </dependency>
    <dependency>
        <groupId>org.springframework.boot</groupId>
        <artifactId>spring-boot-starter-web</artifactId>
    </dependency>
    <dependency>
        <groupId>org.projectlombok</groupId>
        <artifactId>lombok</artifactId>
    </dependency>
    <dependency>
        <groupId>org.springframework.boot</groupId>
        <artifactId>spring-boot-starter-test</artifactId>
        <scope>test</scope>
    </dependency>
    <dependency>
        <groupId>org.springframework.boot</groupId>
        <artifactId>spring-boot-starter-security</artifactId>
    </dependency>
    <!--common pom end-->

    <dependency>
        <groupId>org.springframework.security</groupId>
        <artifactId>spring-security-ldap</artifactId>
    </dependency>
    <dependency>
        <groupId>org.springframework.boot</groupId>
        <artifactId>spring-boot-starter-data-ldap</artifactId>
    </dependency>
</dependencies>
```

**spring-security-ldap** 中实现对 LDAP 服务端的认证类是 **ActiveDirectoryLdapAuthenticationProvider**

**spring-boot-starter-data-ldap** 的主要作用是将 LDAP服务端(这里指AD) 的用户信息进行简单的封装，方便CRUD 操作。

然后就发现这两个东西的矛盾了。下面这个需要在配置文件中配置指定的用户信息才能进行用户信息的操作，而**spring-security-ldap** 是动态的输入用户信息的。学习发现**spring-boot-starter-data-ldap** 对用户信息的操作是通过一个**LdapTemplate**类来实现的，打开对应的 **LdapAutoConfiguration** 文件发现它是通过一个**ContextSource** 类 **new** 出来的，这是个接口，找到默认的实现为**LdapContextSource**类，看源码，发现我们是可以自定义实现它的

```java
@Bean
@ConditionalOnMissingBean  // 当上下文中不存在这个Bean的时候才会执行这个方法
public LdapContextSource ldapContextSource() {
    LdapContextSource source = new LdapContextSource();
    source.setUserDn(this.properties.getUsername());
    source.setPassword(this.properties.getPassword());
    source.setAnonymousReadOnly(this.properties.getAnonymousReadOnly());
    source.setBase(this.properties.getBase());
    source.setUrls(this.properties.determineUrls(this.environment));
    source.setBaseEnvironmentProperties(
        Collections.unmodifiableMap(this.properties.getBaseEnvironment()));
    return source;
}
```

那么我们只需要在使用**LdapContextSource**前动态的把用户名和密码传入进去就可以实现对用户信息的**CRUD**操作了。 发现 **ActiveDirectoryLdapAuthenticationProvider** 是**final**的，所以我们只能自己实现一个**CustomActiveDirectoryLdapAuthenticationProvider** 继承自 **AbstractLdapAuthenticationProvider** 类，把原来的这个类的代码复制到我们这个自定义的类中，这里贴上新加的代码逻辑

```java
// 新加这个ContextSource属性
private ContextSource contextSource;

public ContextSource getContextSource(){
    return contextSource;
}

public void setContextSource(ContextSource contextSource){
    this.contextSource = contextSource;
}

private DirContext bindAsUser(String username, String password) {
    // TODO. add DNS lookup based on domain
    final String bindUrl = url;

    Hashtable<String, Object> env = new Hashtable<>();
    env.put(Context.SECURITY_AUTHENTICATION, "simple");
    String bindPrincipal = createBindPrincipal(username);
    env.put(Context.SECURITY_PRINCIPAL, bindPrincipal);
    env.put(Context.PROVIDER_URL, bindUrl);
    env.put(Context.SECURITY_CREDENTIALS, password);
    env.put(Context.INITIAL_CONTEXT_FACTORY, "com.sun.jndi.ldap.LdapCtxFactory");
    env.put(Context.OBJECT_FACTORIES, DefaultDirObjectFactory.class.getName());
    env.putAll(this.contextEnvironmentProperties);
    // 在这里实现将用户名和密码动态的设置进去
    if(this.contextSource != null && contextSource instanceof LdapContextSource){
        LdapContextSource ldapContextSource = (LdapContextSource) contextSource;
        ldapContextSource.setUserDn(username);
        ldapContextSource.setPassword(password);
    }
    try {
        return contextFactory.createContext(env);
    }
    catch (NamingException e) {
        if ((e instanceof AuthenticationException)
            || (e instanceof OperationNotSupportedException)) {
            handleBindException(bindPrincipal, e);
            throw badCredentials(e);
        }
        else {
            throw LdapUtils.convertLdapException(e);
        }
    }
}
```

查看我们的安全配置:

```java
@Slf4j
@EnableWebSecurity
@Configuration
public class SecurityConfig extends WebSecurityConfigurerAdapter {

    @Value("${ldap.urls}")
    private String ldapUrl;

    @Value("${ldap.base}")
    private String ldapBasic;

    @Override
    protected void configure(HttpSecurity http) throws Exception {
        http
                .authorizeRequests()
                    .antMatchers("/signin/**").permitAll()
                    .anyRequest().authenticated().and()
                .csrf().disable()
                .formLogin().successForwardUrl("/index").loginProcessingUrl("/signin").and()
                .logout()
                    .permitAll()
                    .and();
    }

    @Override
    protected void configure(AuthenticationManagerBuilder auth){
        auth
              .authenticationProvider(customActiveDirectoryLdapAuthenticationProvider());
    }

    @Bean
    public CustomActiveDirectoryLdapAuthenticationProvider customActiveDirectoryLdapAuthenticationProvider() {
        CustomActiveDirectoryLdapAuthenticationProvider provider =  new CustomActiveDirectoryLdapAuthenticationProvider(null, ldapUrl);
        provider.setContextSource(ldapContextSource());
        return provider;
    }

    @Bean
    public LdapContextSource ldapContextSource(){
        LdapContextSource ldapContextSource = new LdapContextSource();
        ldapContextSource.setBase(ldapBasic);
        ldapContextSource.setUrl(ldapUrl);
        return ldapContextSource;
    }
}
```

然后再配置一下 **spring-boot-starter-data-ldap** 的 **repository** 封装**CRUD**操作

```java
@Service
public class PersonRepository{

    @Autowired
    private LdapTemplate ldapTemplate;

    public Person create(Person person){
        ldapTemplate.create(person);
        return person;
    }

    public Person findByPrincipalName(String userPricpleName){
        String filter = "(&(objectCategory=Person)(userPrincipalName="+userPricpleName+"))";
        return ldapTemplate.findOne(query().base("CN=Users").where("userPrincipalName").is(userPricpleName), Person.class);
    }

    public Person modifyPerson(Person person){
        ldapTemplate.update(person);
        return person;
    }

    public void deletePerson(Person person){
        ldapTemplate.delete(person);
    }

}
```

看一下我们的 **person** 实体定义

```java
@Data
@Entry(objectClasses = {"top","person","organizationalPerson","user"}, base="dc=corp,dc=datamesh,dc=com")
public class Person {

    @Id
    @JsonIgnore
    private Name dn;   // CN=XingHou Liu,CN=Users
    @Attribute(name="sAMAccountName")
    private String name; // xinghouliu
    @Attribute(name="userPrincipalName")
    private String principalName; // xinghouliu@datamesh.com
    @Attribute(name="memberOf")
    private List<String> memberOf; // CN=DataMesh Senior Developers,CN=Managed Service Accounts,DC=corp,DC=datamesh,DC=com

    public Person() {
    }

    public Person(String name) {
        this.dn = LdapNameBuilder.newInstance().add("cn", "Users").add("cn", name).build();
    }

    public void setName(String name) {
        this.name = name;
        if(this.dn == null){
            this.dn = this.dn = LdapNameBuilder.newInstance().add("cn", "Users").add("cn", name).build();
        }
    }

}
```

 然后大体的框架就实现完了，写接口吧:

```java
@GetMapping("create")
public void add(){
    Person person = new Person("steve");
    person.setMemberOf(Arrays.asList("CN=Senior Developers,CN=Managed Service Accounts,DC=example,DC=com"));
    person.setName("steveliu");
    person.setPrincipalName("steve@example.com");
    personRepository.create(person);
}

@GetMapping("delete")
public void delete(String name){
    Person person = new Person();
    person.setName(name);
    personRepository.deletePerson(person);
}

@GetMapping("/")
public String index(){
    LdapUserDetailsImpl object = (LdapUserDetailsImpl) SecurityContextHolder.getContext().getAuthentication().getPrincipal();
    System.out.println("object: " + object);
    Person person = personRepository.findByPrincipalName(object.getUsername());
    System.out.println("person: "+ person);
    return "index";
}
```

postman 调试就行了。

注意: 还有另一种配置**LdapAuthentication**的方法，就是在AuthenticationManagerBuilder这里配置ldapAuthentication方法的一些属性，但是这种方法需要AD配置**不需要绑定即可查询信息**这个权限。

