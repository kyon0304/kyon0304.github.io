---
title: "Spring 框架缓存故障自动切换"
date: 2020-09-29T15:40:36+08:00
lastMod: 2020-09-29T15:40:36+08:00
tags: [spring, cache, redis, '分享']
enableRelated: false
enableOutdatedInfoWarning: true
description: 缓存只是提高访问速度，如何配置 Spring CacheManager，使得缓存不可用时，不影响应用正常运行，不影响接口正常返回。
---

## 现状
缓存只是提高访问速度，应用本身没有很高的并发访问量，缓存不可用时，数据库也能顶住。但是缓存挂掉以后，Spring CacheManager 默认会抛出异常，方法直接就异常退出了。

## 目标
缓存不可用时，不影响应用正常运行，不影响接口正常返回。

## 解决方案
### 方案一
最直接的方案，Spring 定义的 error handler 默认是抛出异常，覆盖 handler 并 catch 住异常不要抛出就可以不影响正常处理流程。

```java
public class CacheErrorLoggingHandler extends SimpleCacheErrorHandler {
    private Logger logger = LoggerFactory.getLogger(CacheErrorLoggingHandler.class);

    private ClientResources clientResources;

    public void setClientResources(ClientResources clientResources) {
        this.clientResources = clientResources;
    }

    @Override
    public void handleCacheGetError(RuntimeException exception, Cache cache, Object key) {
        logger.error("Fail to get cache with key [{}]", key, exception);
    }

    @Override
    public void handleCachePutError(RuntimeException exception, Cache cache, Object key, Object value) {
        logger.error("Fail to put cache with key [{}]", key, exception);
    }

    @Override
    public void handleCacheEvictError(RuntimeException exception, Cache cache, Object key) {
        logger.error("Fail to evict cache with key [{}]", key, exception);
    }

    @Override
    public void handleCacheClearError(RuntimeException exception, Cache cache) {
        logger.error("Fail to clear cache", exception);
    }
}
```

另外需要注意，不是直接 `@Component` 的方式注入 Spring 容器，而是需要在缓存配置类中配置：

```java
@EnableCaching
@PropertySource("classpath:cache-${spring.profiles.active}.properties")
@Configuration
public class CacheConfig extends CachingConfigurerSupport {
    @Override
    public CacheErrorHandler errorHandler() {
        return new CacheErrorLoggingHandler();;
    }
}
```

但是这样处理有个问题，就是请求会很慢，每次都要等 redis 连接超时时间x2，因为 CacheManager 的实现是，如果缓存 get 失败了，会从比如数据库里取，那么还会做 put 操作尝试更新缓存，这样 get 等一遍，put 再等一遍。

于是考虑可不可以在缓存不可用的时候，就直接请求比如数据库，不要再在挂掉的缓存这里绕一道弯子。

### 方案二

首先考虑不用缓存的情况如何处理，Spring 提供了 NoOpCacheManager，提供没有缓存时 CacheManager 相关操作的 pass on，这个比较简单直接了，直接使用就好。

然后考虑如何做切换，Spring 中给提供了一个 CompositeCacheManager，内部有一个 ArrayList 类型的 cacheManagers 变量，可以存储多个不同的 CacheManager，给切换提供了可能。

不过实际跟了下 CompositeCacheManager 的行为，跟我的意图并不一致，它虽然会遍历所有的 CacheManager，但是只要这个 CacheManager 有相应的 CacheName 就会返回，并不会判断这个 CacheManager 的 backstore 是否可用。换句话说，CompositeCacheManager 的使用场景是这样的：不同的 CacheManager 管理不同 CacheName 的缓存，根据 CacheName 获取缓存时，如果当前 CacheManager 下没有，就返回 null，CompositeCacheManager 就会去下一个 CacheManager 寻找这个 CacheName。

而我的需求是，根据缓存状态动态增添 CompositeCacheManager 中的 cacheManagers 变量，比如当 Redis 正常运行时，它就是第一顺位被使用的，而当 Redis 挂掉以后，可以将它从 cacheManagers 中去掉，避免每次请求带缓存的接口都需要等待连接超时。

然而，CompositeCacheManager 中 cacheManagers 变量定义是 final 的，不允许修改，暴露的方法也是只允许赋值，不允许修改，于是自己照猫画虎写了一个 CustomCompositeCacheManager，实现可以动态增添 CacheManager 的功能，代码如下：

```java
public class CustomCompositeCacheManager implements CacheManager, InitializingBean {
    private final Logger logger = LoggerFactory.getLogger(CustomCompositeCacheManager.class);

    private LinkedList<CacheManager> cacheManagers = new LinkedList<>();

    private CacheManager redisCacheManager;

    private boolean fallbackToNoOpCache = false;

    public CustomCompositeCacheManager() {
    }

    public CustomCompositeCacheManager(CacheManager... cacheManagers) {
        addCacheManagers(Arrays.asList(cacheManagers));
    }

    @Override
    @Nullable
    public Cache getCache(String name) {
        for (CacheManager cacheManager : this.cacheManagers) {
            Cache cache = cacheManager.getCache(name);
            if (cache != null) {
                return cache;
            }
        }
        return null;
    }

    @Override
    public Collection<String> getCacheNames() {
        Set<String> names = new LinkedHashSet<>();
        for (CacheManager manager : this.cacheManagers) {
            names.addAll(manager.getCacheNames());
        }
        return Collections.unmodifiableSet(names);
    }

    @Override
    public void afterPropertiesSet() throws Exception {
        if (this.fallbackToNoOpCache) {
            this.cacheManagers.add(new NoOpCacheManager());
        }
    }

    public void setFallbackToNoOpCache(boolean fallbackToNoOpCache) {
        this.fallbackToNoOpCache = fallbackToNoOpCache;
    }

    public void addCacheManagers(Collection<CacheManager> cacheManagers) {
        this.cacheManagers.addAll(cacheManagers);
        redisCacheManager = this.cacheManagers.get(0);
    }

    public void deactivateRedisCacheManager() {
        if (!cacheManagers.getFirst().equals(redisCacheManager)) {
            logger.warn("First cache manager is not redis cache manager");
            return;
        }
        cacheManagers.removeFirst();
    }

    public void activateRedisCacheManager() {
        if (cacheManagers.getFirst().equals(redisCacheManager)) {
            logger.warn("First cache manager already is redis cache manager");
            return;
        }
        cacheManagers.addFirst(redisCacheManager);
    }
}
```

因为只给当前项目用，所以对 RedisCacheManager 做了特别对待，有两个函数 activateRedisCacheManager 和 deactivateRedisCacheManager，专门用于添加和删除它。那么什么时候调用这两个函数呢？也即，怎么判断 Redis 上线了、挂掉了呢？

（未被采取的）第一反应是，HealthCheck，Springboot 流派中，最常用的就是 spring-boot-actuator 了，参考它的 RedisHealthIndicator 代码如下：

```java
@Override
protected void doHealthCheck(Health.Builder builder) throws Exception {
    RedisConnection connection = RedisConnectionUtils.getConnection(this.redisConnectionFactory);
    try {
        if (connection instanceof RedisClusterConnection) {
            ClusterInfo clusterInfo = ((RedisClusterConnection) connection).clusterGetClusterInfo();
            builder.up().withDetail("cluster_size", clusterInfo.getClusterSize())
                    .withDetail("slots_up", clusterInfo.getSlotsOk())
                    .withDetail("slots_fail", clusterInfo.getSlotsFail());
        }
        else {
            // 因为项目中用到的 redis 是哨兵机制部署的，所以关键的就是下面这一行了
            Properties info = connection.info();
            builder.up().withDetail(VERSION, info.getProperty(REDIS_VERSION));
        }
    }
    finally {
        RedisConnectionUtils.releaseConnection(connection, this.redisConnectionFactory);
    }
}
```
然而效果并不好，首先，它是被调用的一方，也就是说如果我要用类似的方式检测，还需要自己再起一个线程不断查询；其次，如果 redis 挂了，connection.info() 就会先等待超时，然后抛出异常，和正常的缓存操作数据流程没有什么区别，作为一个检测是否可用的机制，感觉很不优雅。

兜兜转转，发现 lettuce 提供了[事件监听机制](https://github.com/lettuce-io/lettuce-core/wiki/Connection-Events)，可以注册监听 reids 的上线、下线事件，正中下怀，于是修改 CacheConfig 配置类，添加事件注册如下：

```java
@EnableCaching
@PropertySource("classpath:cache-${spring.profiles.active}.properties")
@Configuration
public class CacheConfig extends CachingConfigurerSupport {
    @Autowired
    private LettuceConnectionFactory lettuceConnectionFactory;

    @Autowired
    @Lazy
    private CustomCompositeCacheManager frequentCacheManager;

    @Override
    public CacheErrorHandler errorHandler() {
        return new CacheErrorLoggingHandler();;
    }

    @PostConstruct
    public void registerListener() {
        LettuceClientConfiguration clientConfiguration = lettuceConnectionFactory.getClientConfiguration();
        Optional<ClientResources> clientResources = clientConfiguration.getClientResources();

        clientResources.ifPresent(resources -> {
            resources.eventBus().get().subscribe(e -> {
                logger.info("Caught cache event: {}", e);
                if (e instanceof ConnectionDeactivatedEvent) {
                    logger.info("ConnectionDeactivatedEvent");
                    frequentCacheManager.deactivateRedisCacheManager();
                } else if (e instanceof ConnectionActivatedEvent) {
                    logger.info("ConnectionActivatedEvent");
                    frequentCacheManager.activateRedisCacheManager();
                }
            });
        });
    }
}
```

忍不住赞美 lettuce，测试程序运行过程中，将 redis 挂掉再启动，事件通知非常灵敏，整个运行流程完全如我预测：

- redis 服务下线，deactivateRedisCacheManager 方法被触发，RedisCacheManager 被从可用 cacheManagers 中移除，再次请求带缓存的方法时，由 cacheManagers 中下一顺位的 CacheManager （比如 NoOpCacheManager）处理
- redis 服务上线，activateRedisCacheManager() 方法被触发，RedisCacheManager 作为第一个节点重新回到 cacheManagers 队列，再次请求带缓存的方法时，优先缓存到 redis 中


## 与多级缓存对比

提到缓存，就肯定会想到多级缓存（吧？）本篇讨论的虽然也涉及多个缓存（假如把 NoOpCacheManager 也看作一种特殊缓存），但是侧重点不同：

- 多级缓存主要目标是进一步提高访问速度，在 redis 这种远程的分布式缓存之上，再加一层本地内存缓存，比如 caffeine 或者 ehcache，更新缓存时，要考虑多级缓存中的数据一致性，显然也不是 CacheManager 的适用场景。
- 本篇主要考虑缓存故障以及故障修复情况下，应用使用的缓存动态切换。需要考虑的是数据 stale 问题，比如故障期间数据已过期，由于 redis 故障未得到及时更新，修复后程序再次切换使用 redis，其中保存的已过期数据就会影响程序逻辑。
 
## 问题

### 程序启动时 redis 未就绪

然而还是有问题的，如果程序启动前，redis 就挂了，那么 lettuce 是不会发布任何事件的，即使后来 redis 又上线了，也不会有事件通知，以后如果去请求带缓存的方法，就会次次进入 ErrorHandler 处理流程，也就是事件监听器没有起到任何作用。

此处猜测（未验证），lettuce 的事件发布，是订阅了 redis 的事件频道，所以如果最开始没有正确连接到 reids 的话 ，程序中订阅的 lettuce 的事件也就废了。

不过对于 standalone 的 redis 上线事件，又是怎么监听到的呢？不理解呀。

### redis 故障期间缓存数据过期

一个是设置比较短的缓存过期时间，尽量减少遇到这种情况的概率 😂
一个是比较保险的办法，但是也比较低效，就是故障修复后切回使用 redis 时，把所有缓存 evict 掉，等待下次请求重新 put。

## 题外话

关于缓存穿透问题，就是不怀好意的用户使用大量不存在的参数查询，击穿缓存层直接打到数据库，从而达到打挂数据的目的，最开始看到的办法是，来一个请求就缓存一个并且把返回值设置为 null 之类的，尽量减少到数据库的请求。虽然好像也能行，但是如果人家用的海量不重复的请求，不就顺带击垮你的 redis 了吗？😂

后来看到一种解决办法是用布隆过滤器，感觉脑袋被敲了一下，哎呀，这个厉害呀！内存占用又少，又拦住了恶意请求。点赞！👍
