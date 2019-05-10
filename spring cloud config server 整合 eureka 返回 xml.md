# spring cloud config server 整合 eureka 返回 xml 

这是在整合注册中心时出现的问题(别的注册中心没试，但应该也会出现)， 版本说明

```xml
 <dependency>
      <groupId>org.springframework.cloud</groupId>
      <artifactId>spring-cloud-dependencies</artifactId>
      <version>Greenwich.RELEASE</version>
      <type>pom</type>
      <scope>import</scope>
</dependency>
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-dependencies</artifactId>
    <version>2.1.3.RELEASE</version>
    <type>pom</type>
    <scope>import</scope>
</dependency>
```

访问配置中心的一个某个配置文件信息，返回**xml**格式数据（正常来讲应该是json数据），解决方法, 在入口类实现  **org.springframework.web.servlet.config.annotation.WebMvcConfigurer** 接口，重写 **configureContentNegotiation** **default**方法：

```java
@Override
public void configureContentNegotiation(ContentNegotiationConfigurer configurer) {
    configurer.ignoreAcceptHeader(true).defaultContentType(MediaType.APPLICATION_JSON_UTF8);
}
```

这个方法会忽略浏览器发送的 **accept header**，强制返回 **json** 格式的数据。修改了 **sevelet** 的创建工厂配置。

