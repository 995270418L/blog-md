## springboot + mybatis 手动加载数据源实现

怎么在**springboot**项目启动后通过**web**接口手动配置数据源呢？

**SpringBoot** 初始化和**Mybatis**相关的类流程如下：

```java
初始化 HikaraDataSource Bean ==>> 初始化SqlSessionFactory Bean,配置Mapper.xml地址  ==>> 使用 MapperScannerConfigurer 扫描 mapper 包下的 mapper 类，并将 Mapper 注入到 spring IOC 中作为 Mapper Bean
```

这样我们在**Service**层就能直接通过**Autowired**注解使用这些**mapper Bean**， 但是**Spring IOC** 只会在初始化的时候加载这些**Mapper bean**并且为相应的**Service**注入对应的**mapper**实例。 而且Mybatis的这些Bean也依赖于SqlSessionFactory的初始化。看起来陷入了死锁一样。但还是有解决方案。

第一种： 手动配置数据源后，将 **SqlSessionFactory** 和**MapperScannerConfigurer** 两个**Bean**手动注入到**Spring IOC**中， Mapper就根据这个 **SqlSessionFactory** 手动生成，伪码如下：

```java
public M mapper(){
    SqlSessionFactory sqlSessionFactory = springUtil.getBean(SqlSessionFactory.class)
    SqlSession sqlSession = sqlSessionFacoty.openSession()
    mapper = sqlSession.getMapper(clazz);  // clazz 是一个全局变量，表示当前实现类的类型。
    return mapper;
}	

```

但这还会引发一起其他问题，对springboot的侵入性太大，而且事务也无法使用。如果真要用这种方法还是建议使用**SqlSessionTemplate**， 它会自动管理**sqlSession**的生命周期，自动释放资源。 这里主要推荐第二种（需要对源码有点研究）

第二种(强烈推荐)： 代理模式。首先假设，初始化**Mapper bean**的时候需要的数据源只是一个外壳而已，并不一定会拿数据源做什么事情，所以尝试一下试试。新建代理类 **HikariDataSourceProxy**，包装一下真实的**DataSource**类

```java
@Data
public class HikariDataSourceProxy implements DataSource {
	
    private HikariDataSource hikariDataSource; 、、 

    
    @Override
    public Connection getConnection() throws SQLException {
        return hikariDataSource.getConnection();
    }

    @Override
    public Connection getConnection(String username, String password) throws SQLException {
        return hikariDataSource.getConnection(username, password);
    }

    @Override
    public <T> T unwrap(Class<T> iface) throws SQLException {
        return hikariDataSource.unwrap(iface);
    }
    
    // ... 省略部分Override 方法
}
```

在初始化的时候，像**Springboot**动态加载外部数据源那样配置**Bean**即可

```java
@Configuration
public class BeanConfiguration{
    
    @Bean
    public DataSource dataSource() {
        return new HikariDataSourceProxy();
    }

    @Bean
    public SqlSessionFactory sqlSessionFactory(DataSource dataSource) throws Exception {
        SqlSessionFactoryBean sqlSessionFactoryBean = new SqlSessionFactoryBean();
        sqlSessionFactoryBean.setDataSource(dataSource);
        String[] locations = {"classpath*:/mybatis/*Mapper.xml"};
        sqlSessionFactoryBean.setMapperLocations(Stream.of(Optional.ofNullable(locations).orElse(new String[0])).flatMap(e -> Stream.of(getResources(e))).toArray(Resource[]::new));
        return sqlSessionFactoryBean.getObject();
    }

    @Bean
    public MapperScannerConfigurer scannerConfigurer() {
        MapperScannerConfigurer configurer = new MapperScannerConfigurer();
        configurer.setSqlSessionFactoryBeanName("sqlSessionFactory");
        configurer.setBasePackage("com.ybwx.spring.mybatis.mapper");
        return configurer;
    }

    private Resource[] getResources(String location) {
        try {
            return resourceResolver.getResources(location);
        } catch (IOException e) {
            return new Resource[0];
        }
    }
    
}
```

在获取到外部数据源配置后，将真实的**HikariDataSource**注入到**DataSource Bean**中去即可，这样实现了最小的侵入性

```java
// 标砖的mysql数据源外部配置
private DataSource initHikariDataSource(DataBaseRequestVO database) {
        String url = "jdbc:mysql://{host}:{port}/{db}?allowPublicKeyRetrieval=true&useUnicode=true&useSSL=false&characterEncoding=utf8&serverTimezone=GMT%2B8";
        url = url.replace("{host}", database.getHost());
        url = url.replace("{port}", String.valueOf(database.getPort()));
        url = url.replace("{db}", String.valueOf(database.getDbName()));
        log.info("mysql database url: {}", url);
        HikariConfig hikariConfig = new HikariConfig();
        hikariConfig.setJdbcUrl(url);
        hikariConfig.setUsername(database.getUsername());
        hikariConfig.setPassword(database.getPassword());
        hikariConfig.addDataSourceProperty("cachePrepStmts", true); //是否自定义配置，为true时下面两个参数才生效
        hikariConfig.setMaximumPoolSize(5);
        hikariConfig.setMinimumIdle(300000);
        hikariConfig.setIdleTimeout(600000);
        hikariConfig.setMaxLifetime(3000);
        hikariConfig.addDataSourceProperty("prepStmtCacheSqlLimit", 2048); //单条语句最大长度默认256，官方推荐2048
        HikariDataSource hikariDataSource = null;
        try {
            hikariDataSource = new HikariDataSource(hikariConfig);
            DataSource dataSource = springService.obtainBean(DataSource.class); // springComponent 是自己封装的spring 工具类，便于从 IOC 容器中获取Bean， 这里获取到初始化加载的DataSource Bean，
            if(dataSource instanceof HikariDataSourceProxy){ // 实际上是 HikariDataSourceProxy 类
                HikariDataSourceProxy hikariDataSourceProxy = (HikariDataSourceProxy) dataSource;
                hikariDataSourceProxy.setHikariDataSource(hikariDataSource);
            }
        } catch (HikariPool.PoolInitializationException e) {
            e.printStackTrace();
            if (e.getMessage().contains("Access denied")) {
                throw new LogicException("database username or password not courrent");
            } else if (e.getMessage().contains("Unknown database")) {
                throw new LogicException("database not exists ");
            } else {
                throw new LogicException("data parameters not right ");
            }
        }
        return hikariDataSource;
    }

```

这里再贴一下自己封装的**springService** 类， 详情可以参考我的 **github** 项目  <https://github.com/995270418L/spring-cloud-demonstration/tree/master/ybwx-spring-mybatis>

```java
@Service
public class SpringService implements ApplicationContextAware {

    private ApplicationContext applicationContext;

    @Override
    public void setApplicationContext(ApplicationContext applicationContext) throws BeansException {
        this.applicationContext = applicationContext;
    }

    public void injectBean(String beanName, BeanDefinition beanDefinition){
        if(applicationContext != null){
            DefaultListableBeanFactory beanFactory = (DefaultListableBeanFactory) applicationContext.getAutowireCapableBeanFactory();
            beanFactory.registerBeanDefinition(beanName, beanDefinition);
        }
    }

    /**
     * 注入单个Bean使用，若需要重复注入同一个Bean，使用上面的方法
     * @param object
     */
    public void injectBean(Object object){
        if(applicationContext != null){
            DefaultListableBeanFactory beanFactory = (DefaultListableBeanFactory) applicationContext.getAutowireCapableBeanFactory();
            char[] nameArray = object.getClass().getSimpleName().toCharArray();
            nameArray[0] += 32;
            beanFactory.registerSingleton(new String(nameArray), object);
        }
    }

    public Object getBean(String beanName){
        if(applicationContext != null){
            return applicationContext.getBean(beanName);
        }
        return null;
    }

    public <T> T getBean(Class<T> clazz){
        if(applicationContext != null){
            return applicationContext.getBean(clazz);
        }
        return null;
    }
}

```

这种方法实现了最小的入侵性，合理的利用代理模式是不是很优雅呢？