---
title: "基于 Spring Boot + Disconf 的配置中心"
date: 2019-11-03T13:01:01+08:00
tags: ["spring-boot", "disconf", "配置中心", "工具", '分享']
---


## disconf 简介

disconf 是百度的某位员工开源的分布式配置中心，[文档地址](https://disconf.readthedocs.io/zh_CN/latest/index.html)，C/S 架构，客户端通过监听 zookeeper 中更新触发配置拉取，服务端自带一个简陋前端可以用于图形化查看/更新配置。相比于 spring boot admin 中提供的 refresh context 功能，优势在于可以一次更改，应用于所有同类机器配置，而 spring boot admin 需要对每个实例进行单独修改。除此之外，spring boot admin 在界面及交互上吊打 disconf。

### 设计架构

* server 端包含 springmvc 用于处理业务逻辑，mysql 用于存储配置文件，两个 redis 用于用户登录，zk 用于配置更新的通知， web 端用于用户登录后统一查看/更新配置
* client 端以 pom 方式引入项目内即可（需要魔改一下，见后）

### 运行流程

![process.jpeg](/media/disconf/diconf-client-process.jpg)

**启动事件A：**
以下按顺序发生。
* A3：扫描静态注解类数据，并注入到配置仓库里。
* A4+A2：根据仓库里的配置文件、配置项，去 disconf-web 平台里下载配置数据。这里会有主备竞争
* A5：将下载得到的配置数据值注入到仓库里。
* A6：根据仓库里的配置文件、配置项，去ZK上监控结点。
* A7+A2：根据XML配置定义，到 disconf-web 平台里下载配置文件，放在仓库里，并监控ZK结点。这里会有主备竞争。
* A8：A1-A6均是处理静态类数据。A7是处理动态类数据，包括：实例化配置的回调函数类；将配置的值注入到配置实体里。

**更新配置事件B：**
以下按顺序发生。
* B1：管理员在 Disconf-web 平台上更新配置。
* B2：Disconf-web 平台发送配置更新消息给ZK指定的结点。
* B3：ZK通知 Disconf-cient 模块。
* B4：与A4一样。
* B5：与A5一样。
* B6：基本与A4一样，唯一的区别是，这里还会将配置的新值注入到配置实体里。

**主备机切换事件C：**
以下按顺序发生。
* C1：发生主机挂机事件。
* C2：ZK通知所有被影响到的备机。
* C4：与A2一样。
* C5：与A4一样。
* C6：与A5一样。
* C7：与A6一样。

以上完全引用自 [disconf 官方文档](https://disconf.readthedocs.io/zh_CN/latest/design/src/disconf-client%E8%AF%A6%E7%BB%86%E8%AE%BE%E8%AE%A1%E6%96%87%E6%A1%A3.html)

这边使用时，只使用了基于 xml 的静态配置部分，即配置文件的托管和下载，动态加载部分，由于 disconf 要求配置文件的拆分方式需要按不同组件划分，并对应不同的 java config bean，而且和 spring 自动加载的 bean 的配合也不好，所以放弃使用。

## disconf 使用准备（魔改）

由于 disconf 已经三年多没有更新（最新一次版本发布在 20160911），使用的时候需要对 client 端做一些 hack，这边主要做了两处修改：

1. 依赖的 spring 提供的 hook 函数已经先是 deprecated 现在已经被删除了，
2. 依赖的 zookeeper 版本过低，和项目中作为注册中心使用的 zookeeper 冲突

`ReloadingPropertyPlaceholderConfigurer.java`
```java
 import org.springframework.context.ApplicationContext;
 import org.springframework.context.ApplicationContextAware;
 import org.springframework.util.ObjectUtils;
+import org.springframework.util.PropertyPlaceholderHelper;
 import org.springframework.util.StringValueResolver;

         // then, business as usual. no recursive reloading placeholders please.
-        return super.parseStringValue(buf.toString(), props, visitedPlaceholders);
+        PropertyPlaceholderHelper helper = new PropertyPlaceholderHelper(placeholderPrefix, placeholderSuffix, valueSeparator, ignoreUnresolvablePlaceholders);
+        return helper.replacePlaceholders(strVal, props);
     }
```

修改 `pom.xml` 升级一些第三方依赖版本

```xml
diff --git a/pom.xml b/pom.xml
index fc62fef0..dd203736 100644
--- a/pom.xml
+++ b/pom.xml
@@ -89,7 +89,7 @@
             <dependency>
                 <groupId>com.google.code.gson</groupId>
                 <artifactId>gson</artifactId>
-                <version>2.2.4</version>
+                <version>2.8.5</version>
             </dependency>
 
             <dependency>
@@ -151,7 +151,7 @@
             <dependency>
                 <groupId>org.apache.zookeeper</groupId>
                 <artifactId>zookeeper</artifactId>
-                <version>3.3.6</version>
+                <version>3.4.10</version>
                 <exclusions>
                     <exclusion>
                         <groupId>com.sun.jmx</groupId>
@@ -165,6 +165,10 @@
                         <groupId>javax.jms</groupId>
                         <artifactId>jms</artifactId>
                     </exclusion>
+                    <exclusion>
+                        <groupId>org.slf4j</groupId>
+                        <artifactId>slf4j-log4j12</artifactId>
+                    </exclusion>
                 </exclusions>
             </dependency>
```


修改后重新打包引入项目。

disconf server 端有别人准备好的 [docker-compose 文件](https://github.com/zq2599/docker_disconf)，直接下载使用即可。这里有一点需要注意，tomcat 依赖的服务都是以 link 方式注入容器的，所以 server 告知 client 的 zookeeper 的连接信息不是 ip（还好不是）而是 servername（zkhost），需要在本地配置下 hosts 文件，指向 zookeeper 的连接地址。

## 集成到项目中

考虑配置中心需要完成的事情：

1. 项目配置统一存储、查看
2. 一处修改，直接下发到多个环境、节点，省去修改配置需要重新打包部署的麻烦
3. 配置更新热加载

disconf 提供了不同环境的区分，但是这边没有使用，而是继续使用启动参数中 `-Dspring.profiles.active=dev` 的方式做区分。

disconf 虽然也提供了热加载，但是对代码侵入性高，需要使用它提供的 `@DisconfItem` 之类的注解，并且对于数据库这类由 spring 负责读取配置文件然后自动初始化并加载的第三方组件，根本找不到放注解的地方，需要深入 spring 启动的加载过程才好做，看了一些文档和视频，不是短期内能搞懂的。

期间了解了一下 spring cloud 提供的 config client + config server 的机制，非常傻瓜式，config server 负责连接存储配置的后端，配置更新后（以 git 后端为例，是 `git commit` 操作后），用户调用 spring boot actuator 提供的 `http get http://localhost:8080/actuator/refresh` endpoint 触发 config client 更新带有 `@RefreshScope` 注解的配置，实现配置更新热加载。

spring cloud config 优势在于和 spring boot 完美结合，原有代码完全不需要改动，只需要在需要更新的类/方法上加 `@RefreshScope` 注解。缺点是没有图形化界面（勉强算上 spring boot admin 的话，缺点在上面说过了，多实例的应用配置需要单独修改）。

在某天晚上回家的路上，经[邹扒皮](https://zou.cool/)同学提醒，为什么要费劲吧啦的去研究 spring 的启动加载呢，disconf 更新配置以后，通知 spring 去做更新就好了呀，这不正是用户调用 `/actuator/refresh` 接口做的事情吗。后面的事情就比较顺理成章了。

disconf 提供了配置更新时的接口供用户实现 hook 类，在这个 hook 中调用 `/actuator/refresh` 接口就可以通知到 spring 去做配置更新。

所以整个流程是这样的：

1. 在 disconf 提供的 web 中上传需要统一配置的 properties 文件
2. 启动应用，加载远端属性（如果没有则使用本地的）
3. 在 disconf web 端修改属性，应用中引入的 disconf client 监听到属性变化，执行用户实现的 hook 方法
4. hook 方法中调用 `http get http://localhost:8080/actuator/refresh` 通知 spring 配置已更新，请重新加载带有 `@RefreshScope` 注解的类或方法
5. spring 重新加载，完成热更新
6. spring 无法重新加载/兼容性有问题，则需要手动重新启动应用，此时回到第二步

嗯，还是比较顺滑的。

## 遇到的问题

1. 定时任务注解 `@Scheduled` 和 `@RefreshScope` 不兼容
2. 加密组件 jasypt-spring-boot-starter 的解密流程在 spring 重新加载 bean 时没有被执行到，加密属性直接作为属性值传给 bean 导致报错

问题一找到一个[解法](http://stuartingram.com/2016/11/07/joy-and-pain-with-schedule-and-refreshscope-in-springboot-2/)，但是这边没有尝试，因为目前看定时任务的热加载还不是强需求，重启可以加载最新配置就可以满足需求。

问题二在 jasypt 的 gitter 频道里找到了解决办法

![jasypt.png](/media/disconf/jasypt.jpg)

具体来说，把 `@EnableEncryptableProperties` 注解去掉，使用

```java
new SpringApplicationBuilder()
        .environment(new StandardEncryptableEnvironment())
        .sources(ManagerApplication.class)
        .run(args);
```

代替

```java
SpringApplication.run(ManagerApplication.class, args);
```

相应的，引入的 pom 也从 jasypt-spring-boot-starter 改为 jasypt-spring-boot

---

以上大概就是在集成和使用 disconf 的过程中遇到的问题以及解决，可能不全，因为做这个功能拖的时间实在太久。那么，就这样结束吧。

参考链接：

[1]. [disconf 官方文档](https://disconf.readthedocs.io/zh_CN/latest/index.html)

[2]. [disconf server 端 docker-compose 方式安装](https://github.com/zq2599/docker_disconf)

[3]. [Disconf+spring boot+mybatis](https://blog.kazaff.me/2016/07/15/disconf+spring%20boot+mybatis/?nsukey=80nDxFilbD6GC4bgyWWWrWixPUwI01dszm5ayo%2BLgHiCUGGA0576Ap6SA7%2Bvbab28SUtfjHPqDWhc3Sc3lvVcf4j3J%2F0s1WzN%2BTvEhuV3c9QXy1Brr80d7bUt8r30dXrYY%2FQY7qBXMwiKRZCxIuaJaf6GRd06JWnsZMcA%2B%2FcJnXorFncVr3ezE5vwwMvAwlCqcqhLQNX9z0BSFI3xkQuKw%3D%3D#%E4%B8%8Emybatis%E7%BB%93%E5%90%88)

[4]. [spring cloud context](https://cloud.spring.io/spring-cloud-commons/multi/multi__spring_cloud_context_application_context_services.html)