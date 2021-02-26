---
title: "使线上 Spring 应用更好部署和调试"
date: 2020-02-17T21:32:03+08:00
lastMod: 2020-02-18T00:04:03+08:00
tags: ['分享', "spring", "配置中心", "spring boot", "服务发现"]
enableRelated: false
enableOutdatedInfoWarning: true
images:
- /media/spring-config/figure-spring-config-1.png
---

运行一个配置中心，可以方便快捷的更改线上服务多个实例的配置（通过 git commit&git push），配置中心通过服务发现暴露自己的 IP，client 中无需写死 server 地址，在 k8s 中部署也更自然。

运行一个可视化的应用详情 server，可以直观的看到应用健康监控、应用运行的包的构建时间及代码 commit id，应用当前生效的属性并可以动态修改，应用精确到类的日志级别，并可以动态修改。

要做到以上的事情，主要涉及四个组件的配置：

- spring-cloud-config
- spring-boot-admin
- spring-zookeeper-discovery
- spring-boot-actuator

## 效果展示

配置完成后效果图如下，主要是可视化的应用详情部分的展示，关于配置中心的使用方式的话，目前没有界面，主要就是对 spring-cloud-config server 后端数据的修改，然后 spring 有机制可以让它们动态生效，至少应用重启后可以覆盖 jar 包中的属性配置。

{{< figure src="/media/spring-config/figure-spring-config-1.png" alt="figure-1" >}}

一个美观页面，主要展示了三种信息：
- 服务运行了多长时间
- 当前起的服务: configserver 和 cloud-adapter-aws，是从服务的 `spring.application.name` 取值
- 每个服务有几个实例，目前本地测试环境起的，都只有一个实例
- 服务的版本信息，从 pom.xml 文件种读的 `<version>0.1.0</version>`

{{< figure src="/media/spring-config/figure-spring-config-5.png" alt="figure-5" class="big" >}}

右上角切换到 `Applications` 的页面，以数字+列表形式展示了服务统计信息，当服务比较多的时候这个页面会更直观一些，另外当一个服务有多个实例时，可以从这个页面区分不同实例进入到详情页面。

{{< figure src="/media/spring-config/figure-spring-config-2.png" alt="figure-2" class="big" >}}

这里进入 `cloud-adapter-aws` 服务的一个实例页面，左侧导航栏有三部分，默认进入 `Details` 这一项。

页面上我会比较关注的部分，

1. git 信息，包括 commit id，时间，分支
2. build 信息，包括版本号，构建时间，当线上出现问题时，可以通过 1，2 这两条信息快速定位到部署对应的包以及代码
3. 健康状态监控，由 /actuator/health 返回的信息稍微规整了一下，可以快速看到依赖的第三方应用比如 mq、zk、db（如果使用了的话） 等组件的健康状态展示出来。另外也可以通过配置包含一些详细信息比如下面的 4 和 5
4. 使用了 spring cloud discovery 的话，会把可以看到的 client 也展示出来
5. 使用了 spring cloud config 的话，会把 config 加载的地址展示出来


{{< figure src="/media/spring-config/figure-spring-config-3.png" alt="figure-3" class="big" >}}

这个页面展示了应用从各个地方加载的属性值，包括 application.properties, application-${PROFILE}.properties, bootstrap.properties, system properties 以及从 spring cloud config server 拉取下来的属性值。

分成上下部分，可以先从下半部分搜索属性 key，看到原属性值，然后在上半部分选取要修改的属性 key，然后填入想要修改的目标值后，`Refresh Context`，就可以动态修改属性值。

{{< figure src="/media/spring-config/figure-spring-config-4.png" alt="figure-4" class="big" >}}

日志级别页面，可以非常方便而且精确地查看某个包的日志级别并动态修改，在定位线上问题时，非常好用。

## 配置中心

spring-cloud-config 分为 server 和 client 两部分，server 是配置中心，负责适配各种后端存储配置，这里选取的是 git 作为存储后端，client 负责从 server 拉取配置

### spring-cloud-config server 配置

server 部分配置，引入依赖

```xml
<!-- pom.xml -->

<!-- Spring Cloud Config Server -->
<dependency>
    <groupId>org.springframework.cloud</groupId>
    <artifactId>spring-cloud-config-server</artifactId>
</dependency>
```

应用入口处标注注解

```Java
@EnableConfigServer
@SpringBootApplication
public class AdminApplication {

    public static void main(String[] args) {
        SpringApplication.run(AdminApplication.class, args);
    }
}
```

并对 config server 的后端配置如下，暂时配置为本地地址做测试用

```bash
##################################### Spring Cloud Config Server Start ##################################################
# 用户需要修改配置时，修改这个库中对应的文件，并提交&push
# 应用可以动态修改，如果没有生效或进入一种未知状态，可以重启应用，这时新修改的属性肯定就可以生效了
spring.cloud.config.server.git.uri=file://${user.home}/config-repo
##################################### Spring Cloud Config Server End ####################################################
```

这样的话，应该就可以用了，在 client 配置部分指定 server 的地址。但是我这边希望 client 可以通过服务发现直接得到 server 地址，这样不同环境部署时，不需要分别配置 server 地址，会更灵活一些，而且对失败迁移也更友好。

这里选取的是基于 zookeeper 的服务发现机制，所以另外引入依赖 `spring-cloud-starter-zookeeper-discovery`，由于使用的是 3.4.10 版本的 zk，所以重新引入一下，logger 统一使用的是 logback，所以也需要从 zk 的依赖中去除对 slf4j 的依赖。

```xml
<!-- Spring Cloud Zookeeper Discovery -->
<dependency>
    <groupId>org.springframework.cloud</groupId>
    <artifactId>spring-cloud-starter-zookeeper-discovery</artifactId>
    <version>${spring-cloud-zk-discovery.version}</version>
    <exclusions>
        <exclusion>
            <groupId>org.apache.zookeeper</groupId>
            <artifactId>zookeeper</artifactId>
        </exclusion>
    </exclusions>
</dependency>
<dependency>
    <groupId>org.apache.zookeeper</groupId>
    <artifactId>zookeeper</artifactId>
    <version>3.4.10</version>
    <exclusions>
        <exclusion>
            <groupId>org.slf4j</groupId>
            <artifactId>slf4j-log4j12</artifactId>
        </exclusion>
    </exclusions>
</dependency>
```

并在入口类添加注解 `@EnableDiscoveryClient`，于是变成：

```Java
@EnableConfigServer
@EnableDiscoveryClient
@SpringBootApplication
public class AdminApplication {

    public static void main(String[] args) {
        SpringApplication.run(AdminApplication.class, args);
    }
}
```

然后在 `application.properties` 中添加如下配置

```bash
################################### Spring Cloud ZooKeeper Discovery Start ##############################################
spring.cloud.zookeeper.discovery.enabled=true
#当前服务在 zk 注册时，使用 IP 进行注册，不配置的话默认会取 hostname，对于设置了自己 hostname 但是调用方无法解析的情况不够友好         
spring.cloud.zookeeper.discovery.prefer-ip-address=true 
#配置 spring cloud zk discovery 使用的 zk 地址
spring.cloud.zookeeper.connect-string=192.168.31.171:2181
# 由于我本地安装了 VMware 和 virtual box，以及开了 VPN，会有虚拟网卡，设置这个属性后就不会去拿这些虚拟网卡的 IP 在 zk 中注册
spring.cloud.inetutils.ignoredInterfaces=VMware.*, Hyper-V.*, TAP-Windows.*, VirtualBox.*
################################### Spring Cloud ZooKeeper Discovery End ################################################
```

加完如上配置，spring-cloud-config 的 server 部分就算配置完毕了

### spring-cloud-config client 配置

接下来看下 client 部分的配置

首先依然是在 `pom.xml` 中引入依赖，这里同时引入了 spring-cloud-starter-config 和 spring-cloud-starter-zookeeper-discovery，不再分开说明

```xml
<!-- spring cloud config client-->
<dependency>
    <groupId>org.springframework.cloud</groupId>
    <artifactId>spring-cloud-starter-config</artifactId>
</dependency>
<!-- discovery client -->
<dependency>
    <groupId>org.springframework.cloud</groupId>
    <artifactId>spring-cloud-starter-zookeeper-discovery</artifactId>
    <version>${spring-cloud-zk-discovery.version}</version>
    <exclusions>
        <exclusion>
            <groupId>org.apache.zookeeper</groupId>
            <artifactId>zookeeper</artifactId>
        </exclusion>
    </exclusions>
</dependency>
<dependency>
    <groupId>org.apache.zookeeper</groupId>
    <artifactId>zookeeper</artifactId>
    <version>3.4.10</version>
    <exclusions>
        <exclusion>
            <groupId>org.slf4j</groupId>
            <artifactId>slf4j-log4j12</artifactId>
        </exclusion>
    </exclusions>
</dependency>
```

然后在入口类加注解，打开服务发现

```java
@SpringBootApplication
@EnableDiscoveryClient
public class AwsApplication extends SpringBootServletInitializer {

    private static final Logger logger = LoggerFactory.getLogger(AwsApplication.class);

    public static void main(String[] args) {
        ApplicationContext application = SpringApplication.run(AwsApplication.class, args);
        if (logger.isDebugEnabled()) {
            String[] beanDefinitionNames = application.getBeanDefinitionNames();
            for (String beanName : beanDefinitionNames) {
                logger.debug(beanName);
            }
        }
    }
}
```

配置 `application.propeties`

```sh
################################### Spring Cloud ZooKeeper Discovery Start ##############################################
# 在 zk 中注册当前服务
spring.cloud.zookeeper.discovery.register=true
# 使能发现 zk 中注册的服务
spring.cloud.zookeeper.discovery.enabled=true
# 使用服务所在主机 IP 在 zk 中进行注册
spring.cloud.zookeeper.discovery.prefer-ip-address=true
# 使能通过服务发现获得 spring-cloud-config server 地址
spring.cloud.config.discovery.enabled=true
# 指定 zk 中作为 spring-cloud-config server 的服务，默认是 configserver，如果 spring-cloud-config server 那边的 spring.application.name 不是 configserver 的话，可以用这个属性进行跟随
spring.cloud.config.discovery.service-id=configserver
# 打开 spring cloud config 功能，其实只要有 bootstrap.propterties|yml 文件就会自动打开
spring.cloud.config.enabled=true
# 和 spring.cloud.config.profile 一起指定 spring-cloud-config server 拉取的文件：`${spring.cloud.config.name}-${spring.cloud.config.profile}.properties|yml`
spring.cloud.config.profile=${spring.profiles.active}
# 和 spring.cloud.config.name 一起指定 spring-cloud-config server 拉取的文件 ：`${spring.cloud.config.name}-${spring.cloud.config.profile}.properties|yml`
spring.cloud.config.name=${spring.application.name}
# 可以是个数组，指定优先拉取的 git 分支/commit id 等信息
spring.cloud.config.label=master
################################### Spring Cloud ZooKeeper Discovery End ################################################
```

以上，就完成了 spring-cloud-config 的 client 和 server 端的配置，并且支持通过服务发现来动态获取 server 地址。


## 可视化

配置的存取搞定了，下一步来配置可视化部分。这时就用到了 spring-boot-admin 和 spring-boot-actuator。其中 actuator 会从 spring 应用中提取得到数据，包括配置、健康监控以及日志级别等等，并通过 http 的 endpoint 暴露出来，spring boot admin 则会定时请求这些 endpoint 汇总这些信息并通过一个比较美观的前端页面展示出来。

### actuator 配置

引入 pom 依赖

```xml
<!-- spring-boot-actuator-->
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-actuator</artifactId>
    <version>2.1.6.RELEASE</version>
</dependency>
```

配置文件添加如下neir

```bash
############################################# Actuator Start ############################################################
# 要在 http endpoint 中暴露的 actuator 内容
management.endpoints.web.exposure.include=health,info,env,loggers,refresh
# 是否允许在页面上进行 RefreshContext 操作（动态更新属性），需要上面一条属性中包含 refresh
management.endpoint.refresh.enabled=true
# 展示详细信息，比如 zk 的连接地址，如果不配置，可能只显示 up 或 down
management.endpoint.health.show-details=always
# 展示应用详情信息
management.endpoint.info.enabled=true
# 展示环境信息
management.endpoint.env.enabled=true
# 对环境信息中指定属性的值进行脱敏
management.endpoint.env.keys-to-sanitize=password,secret,key,token
# 展示日志级别页面
management.endpoint.loggers.enabled=true
############################################# Actuator End ##############################################################
```

另外如果需要在应用详情中展示 git 和 build info，还需要在 pom 中进行配置生成一下

```xml
<build>
    <finalName>cloud-adapter-aws</finalName>
    <plugins>
        <plugin>
            <groupId>org.springframework.boot</groupId>
            <artifactId>spring-boot-maven-plugin</artifactId>
            <executions>
                <execution>
                    <goals>
                        <goal>build-info</goal>
                    </goals>
                    <phase>package</phase>
                </execution>
            </executions>
        </plugin>
        <plugin>
            <groupId>pl.project13.maven</groupId>
            <artifactId>git-commit-id-plugin</artifactId>
            <executions>
                <execution>
                    <id>get-id-info</id>
                    <goals>
                        <goal>revision</goal>
                    </goals>
                    <phase>package</phase>
                </execution>
                <execution>
                    <id>validate-git-info</id>
                    <goals>
                        <goal>validateRevision</goal>
                    </goals>
                </execution>
            </executions>
        </plugin>
    </plugins>
</build>
```

这样在打包的时候就会自动生成 git 和 build 信息。

### spring-boot-admin 配置

现在数据有了，接下来需要有人去做汇总并展示，于是 spring-boot-admin 出场。

spring-boot-admin 也是分为 server 和 client 两部分，和 spring-cloud-config 相反，spring-boot-admin 是 server 拉取 client 数据，所以，一般来说，需要每个应用引入 spring-boot-admin-client 依赖，并且配置 spring-boot-admin server 地址，但是既然 spring-cloud-config client 可以通过服务发现找到 server，那么 spring-boot-admin 的 server 也可以通过服务发现找到 client，而且这样配置以后，client 端不仅不再需要配置 server 地址，而且还可以去掉对 spring-boot-admin-client 的依赖。

这里我把 spring-boot-admin server 和 spring-cloud-config server 放到了一起，节省一些空间和端口占用，使用服务发现的 spring-boot-admin server 配置比较简单，只需要在 pom 中引入 starter 依赖，然后在入口类加注解，其他配置都由 starter 帮助完成，下面看一下具体配置

pom 依赖：

```xml
<!-- Spring Boot Admin Server -->
<dependency>
    <groupId>de.codecentric</groupId>
    <artifactId>spring-boot-admin-starter-server</artifactId>
    <version>${spring-boot-admin.version}</version>
</dependency>
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-web</artifactId>
</dependency>
```

入口类新增注解 `@EnableAdminServer`，终于得到最终体

```Java
@EnableAdminServer
@EnableConfigServer
@EnableDiscoveryClient
@SpringBootApplication
public class AdminApplication {

    public static void main(String[] args) {
        SpringApplication.run(AdminApplication.class, args);
    }
}
```

但是由于 spring-boot-admin server 和 spring-cloud-config server 的合并，引入了一个新问题，就是两个都会用到 8888 端口的 `/` 这个 endpoint，于是这边需要给它们岔开，spring-cloud-config server 用 `/config` 这个 uri，于是需要做以下额外配置：

```sh
# spring-cloud-config server 对外提供服务的网址新增 /config。
# 比如原来 cloud-adapter-aws client 获取 dev 环境属性的请求是 http://localhost:8888/cloud-adapter-aws/dev 
# 配置这个属性后变为 http://localhost:8888/config/cloud-adapter-aws/dev 
spring.cloud.config.server.prefix=/config
# 告知 client 从 server 拉取配置时，需要在 uri /config 下，否则 client 就会从根目录去拉取
spring.cloud.zookeeper.discovery.metadata.configPath=/config
```

由于使用了服务发现，spring-boot-admin client 部分不需要做任何配置，server 会自动从 zk 中拿到它们的地址，并请求 /actuator/health 等 endpoint 获取数据，进行汇总，统一展示。


明天再看一下加密属性部分，后续如果有时间，还要看下 spring boot admin 网页密码登录

写的有点乱，先这样，以后再改。
