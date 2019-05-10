# Spring Boot 应用程序JMX正确关闭姿势

将 **springboot** **build** 成 **windows** 服务，使用的是官方推荐的 [**sample**](<https://github.com/snicoll/spring-boot-daemon>) ， 代码有点老，依赖改了改，贴在下面

```maven
  properties部分
  <properties>
      <dist.dir>${project.build.directory}/dist</dist.dir>
      <dist.project.id>${project.artifactId}</dist.project.id>
      <dist.project.name>Master</dist.project.name>
      <dist.start.class>com.steve.MasterApplication</dist.start.class>  <!--springboot 项目的入口类-->
      <dist.project.description>
      	Master service.
      </dist.project.description>
      <dist.jmx.port>30001</dist.jmx.port>
      <java.version>1.8</java.version>
      <project.build.sourceEncoding>UTF-8</project.build.sourceEncoding>
  </properties>
    
  新增依赖 部分
  
  <dependency>
      <groupId>com.sun.winsw</groupId>
      <artifactId>winsw</artifactId>
      <version>2.2.0</version>
      <classifier>bin</classifier>
      <type>exe</type>
  </dependency>
  <dependency>
      <groupId>commons-daemon</groupId>
      <artifactId>commons-daemon</artifactId>
      <version>1.0.15</version>
  </dependency>
  
  build 部分
  
  <build>
      <plugins>
          <plugin>
              <groupId>org.apache.maven.plugins</groupId>
              <artifactId>maven-dependency-plugin</artifactId>
              <version>2.10</version>
              <executions>
                  <execution>
                      <id>copy</id>
                      <phase>package</phase>
                      <goals>
                      	<goal>copy</goal>
                      </goals>
                      <configuration>
                          <artifactItems>
                              <artifactItem>
                                  <groupId>com.sun.winsw</groupId>
                                  <artifactId>winsw</artifactId>
                                  <classifier>bin</classifier>
                                  <type>exe</type>
                                  <destFileName>service.exe</destFileName>
                              </artifactItem>
                          </artifactItems>
                          <outputDirectory>${dist.dir}</outputDirectory>
                      </configuration>
                  </execution>
              </executions>
          </plugin>
          <plugin>
              <groupId>org.apache.maven.plugins</groupId>
              <artifactId>maven-resources-plugin</artifactId>
              <version>2.7</version>
              <executions>
                  <execution>
                  <id>copy-resources</id>
                  <phase>process-resources</phase>
                  <goals>
                  	<goal>copy-resources</goal>
                  </goals>
                  <configuration>
                      <outputDirectory>${dist.dir}</outputDirectory>
                      <resources>
                          <resource>
                              <directory>src/main/dist</directory>
                              <filtering>true</filtering>
                          </resource>
                      </resources>
                  </configuration>
                  </execution>
              </executions>
          </plugin>
          <plugin>
              <groupId>org.apache.maven.plugins</groupId>
              <artifactId>maven-assembly-plugin</artifactId>
              <version>2.5.5</version>
              <configuration>
              <descriptors>
                  <descriptor>src/main/assembly/unix.xml</descriptor>
                  <descriptor>src/main/assembly/windows.xml</descriptor>
              </descriptors>
              </configuration>
              <executions>
                  <execution>
                      <id>assembly</id>
                      <phase>package</phase>
                      <goals>
                      	<goal>single</goal>
                      </goals>
                  </execution>
              </executions>
          </plugin>
      </plugins>
  </build>
  
repository (不可少)
   <repositories>
       <repository>
           <id>alimaven</id>
           <name>aliyun maven</name>
           <url>http://maven.aliyun.com/nexus/content/groups/public/</url>
           <releases>
           	<enabled>true</enabled>
           </releases>
           <snapshots>
           	<enabled>false</enabled>
           </snapshots>
       </repository>
       <repository>
           <id>jenkins</id>
           <name>Jenkins Repository</name>
           <url>http://repo.jenkins-ci.org/releases</url>
           <snapshots>
           	<enabled>false</enabled>
           </snapshots>
       </repository>
   </repositories>
```
这个项目是 **windows** 服务和 **linux** 服务都帮我们做了。所以需要引入 **commons-daemon** 这个依赖，改造我们的入口类.

```java
@SpringBootApplication
public class MasterApplication implements Daemon{

    public static void main(String[] args) {
        SpringApplication sp = new SpringApplication(AgentApplication.class);
        Environment environment = sp.run(args).getEnvironment();
    }


    private Class<?> springBootApp;

    private ConfigurableApplicationContext content;

    @Override
    public void init(DaemonContext context) throws DaemonInitException, Exception {
        this.springBootApp = ClassUtils.resolveClassName(context.getArguments()[0],
                AgentApplication.class.getClassLoader());
    }

    @Override
    public void start() throws Exception {
        this.content = SpringApplication.run(springBootApp);
    }

    @Override
    public void stop() throws Exception {
        this.content.close();
    }

    @Override
    public void destroy() {

    }
}
```

在 **src/mian **下面添加**sample**对应的文件，添加完成后项目结构图

![1555666436049](C:\Users\DataMesh\AppData\Roaming\Typora\typora-user-images\1555666436049.png)

打包后**target**目录生成三个包，可以修改 **assembly** 文件夹中对应系统的**xml**文件更改对应系统的打包生成的压缩文件格式。建议**linux**改为**tar.gz**。 下面为打包好的文件图

![1555666730134](C:\Users\DataMesh\AppData\Roaming\Typora\typora-user-images\1555666730134.png)

一共有三个，一个是**jar**， 一个是**linux**下的，一个是**windows**下的。解压缩 **windows**系统的**zip**包，使用系统管理员打开**cmd**窗口

```cmd
xxx.exe install    // 安装服务
net start xxx.exe  // 启动服务
net stop xxx.exe   // 关闭服务

sc delete xxx   // xxx 为安装的服务名, 删除对应的服务
sc query xxx    // 查询对应服务状态
```

问题就出在关闭服务上，怎么都关闭不了。 发现是由于应用启动了一个定时任务，开了其他的线程。那很简单，自然而然会想到往 **JVM** 里注册 **shutdownHook** ，贴个伪码：

```java
 Runtime.getRuntime().addShutdownHook(new Thread(){
     @Override
     public void run() {
         // 关闭线程
     }
 });
```

再次打包尝试，发现还是失败。没办法，只能看个日志，发现有这样一个类 **SpringApplicationAdminMXBean** ， 关闭的时候调用了它的实现类 **SpringApplicationAdmin** 的**shutdown**方法。看看这个类的源码

```java
private class SpringApplicationAdmin implements SpringApplicationAdminMXBean {
    private SpringApplicationAdmin() {
    }

    public boolean isReady() {
        return SpringApplicationAdminMXBeanRegistrar.this.ready;
    }

    public boolean isEmbeddedWebApplication() {
        return SpringApplicationAdminMXBeanRegistrar.this.embeddedWebApplication;
    }

    public String getProperty(String key) {
        return SpringApplicationAdminMXBeanRegistrar.this.environment.getProperty(key);
    }

    public void shutdown() {
        SpringApplicationAdminMXBeanRegistrar.logger.info("Application shutdown requested.");
        SpringApplicationAdminMXBeanRegistrar.this.applicationContext.close();
    }
}
```

这个类是 **SpringApplicationAdminMXBeanRegistrar** 的私有内部类。我开始还妄想 实现**SpringApplicationAdminMXBean**重写**shutdown**方法， 哈哈，太天真了。 没办法，只能在这里打个断点，看看调用栈，看看执行这个方法前和方法后执行了啥，可不可以实现重写。找到了**spring**代码的一个套路, 发现了每次关闭的时候**spring**都会**publishEvent** ， 这一下就来了灵感， 只需监听 **spring** 的 **event** 就行了， 通过JMX端口关闭的时候发布的是**ContextClosedEvent**方式的**event**

```java
@Component
public class DMSpringApplicationListener implements ApplicationListener {

    @Autowired
    private AgentInit agentInit;

    @Override
    public void onApplicationEvent(ApplicationEvent event) {
        if(event instanceof ContextClosedEvent){
           	// 关闭其他线程任务
        }
    }
}

```

再次重试，ok，解决。