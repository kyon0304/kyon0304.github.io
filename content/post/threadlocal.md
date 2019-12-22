---
title: "Threadlocal 梳理"
date: 2019-12-22T18:24:10+08:00
---

## 使用场景

用于同一线程中，不同类、不同方法间数据共享。没有 ThreadLocal 的帮助，这些需要共享的数据，就必须通过参数、返回值进行传递，将会变得非常繁琐。常见的使用场景，比如存储登录用户信息，同一个 request 的 trace id 等。

## 实现

ThreadLocal 包含两个静态内部类，ThreadLocalMap 和 Entry，如下所示，其中 Entry 是弱引用的子类，referent 是 threadlocal 对象，value 成员变量则是真正存储用户设置的值的地方。

内部类 ThreadLocalMap 名字中虽然包含 Map，但与一般的 hashmap 并没有什么关系，其实是内部维护了一个 Entry[] 类型的数组，索引自己的内部类 Entry 的实例时，有两种方式，一种是通过 hashcode 计算快速定位，另外一种是快速定位失效时，使用 threadlocal 实例循环比对 entry 的 referant。

```Java
public class ThreadLocal<T> {    
    static class ThreadLocalMap {
        static class Entry extends WeakReference<ThreadLocal<?>> {
            /** The value associated with this ThreadLocal. */
            Object value;

            Entry(ThreadLocal<?> k, Object v) {
                super(k);
                value = v;
            }
        }
    }
}
```

使用 ThreadLocal 时，代码调用逻辑。

调用以下代码时 ThreadLocal 的执行逻辑

```Java
ThreadLocal<Integer> threadLocal = new ThreadLocal<Integer>() {
    @Override
    protected void initValue() {
        return 5;
    }
};
threadLocal.get();
```

ThreadLocal 的构造函数是一个空函数，所以 `new` 的时候没有什么特殊操作，接下来看下 `get` 函数都做了哪些事情：

```java
public T get() {
    Thread t = Thread.currentThread();
    ThreadLocalMap map = getMap(t);
    if (map != null) {
        ThreadLocalMap.Entry e = map.getEntry(this);
        if (e != null) {
            @SuppressWarnings("unchecked")
            T result = (T)e.value;
            return result;
        }
    }
    return setInitialValue();
}

ThreadLocalMap getMap(Thread t) {
    return t.threadLocals;
}
```

首先获取当前线程，然后获取当前线程持有的 ThreadLocalMap 对象，这个对象是 Thread 对象持有的成员变量，具体管理则由 ThreadLocal 类实现。

```java
private Entry getEntry(ThreadLocal<?> key) {
    int i = key.threadLocalHashCode & (table.length - 1);
    Entry e = table[i];
    if (e != null && e.get() == key)
        return e;
    else
        return getEntryAfterMiss(key, i, e);
}
```

得到 ThreadLocalMap 对象后，再使用当前 threadLocal 对象获取 Entry 对象。如果可以获取到，则返回 Entry 中存储的 value 值，如果获取不到，则进行初始化。

```java
private T setInitialValue() {
    T value = initialValue();
    Thread t = Thread.currentThread();
    ThreadLocalMap map = getMap(t);
    if (map != null)
        map.set(this, value);
    else
        createMap(t, value);
    return value;
}

private void ThreadLocalMap.set(ThreadLocal<?> key, Object value) {
    Entry[] tab = table;
    int len = tab.length;
    int i = key.threadLocalHashCode & (len-1);

    for (Entry e = tab[i];
            e != null;
            e = tab[i = nextIndex(i, len)]) {
        ThreadLocal<?> k = e.get();

        if (k == key) {
            e.value = value;
            return;
        }

        if (k == null) {
            replaceStaleEntry(key, value, i);
            return;
        }
    }

    tab[i] = new Entry(key, value);
    int sz = ++size;
    if (!cleanSomeSlots(i, sz) && sz >= threshold)
        rehash();
}

void createMap(Thread t, T firstValue) {
    t.threadLocals = new ThreadLocalMap(this, firstValue);
}

ThreadLocalMap.ThreadLocalMap(ThreadLocal<?> firstKey, Object firstValue) {
    table = new Entry[INITIAL_CAPACITY];
    int i = firstKey.threadLocalHashCode & (INITIAL_CAPACITY - 1);
    table[i] = new Entry(firstKey, firstValue);
    size = 1;
    setThreshold(INITIAL_CAPACITY);
}
```

初始化的流程，首先调用 `initialValue()` 获取初始值，我们创建 threadLocal 对象时 override 了这个方法，因此会返回 5，获取值后，再使用当前线程获取 threadLocals 即这里的变量 map，如果 map 存在，则将值放入对应的 entry，不存在则创建。从这里可以看到，当不同线程访问同一个 threadlocal 对象时，获取 map 也是要根据当前线程来得到，因此不同线程会有各自独享的本地变量。


最后，用一张图来表示一下

![threadlocal.png](/media/threadlocal/threadlocal.png)

总结：每个线程通过持有 ThreadLocal.ThreadLocalMap 实例，来访问到真正需要的数据：存储在 Entry 实例中的成员变量 value。而不同的线程 thread-1, thread-2 即使访问相同的 ThreadLocal 实例，也会为它们创建各自线程独享的成员变量，ThreadLocalMap，从而持有线程独享的本地变量。threadlocal 实例的 get 方法，只是将这个流程包装了一下。

## 垃圾回收

ThreadLocal 中的 Entry 类是弱引用的子类，弱引用的使用方式是，如果弱引用对象指向的对象，只有弱引用这一条路径，则该对象会在下一次 YGC 中被回收。Entry 的 referant 是 threadlocal 对象，也即，当只有 Entry 指向这个 threadlocal 实例时，不会劫持对象，影响垃圾回收。另一方面来讲，当 Entry 实例检测到 referant 指向为 null 时，会将成员变量 value 也设置为 null，方便垃圾回收。

但是为了方便使用，threadlocal 对象都是声明为静态的，这就是说，它不会随着线程结束而销毁，这就意味着两个问题

1. 内存泄漏，因为类变量不会随着线程终止而销毁
2. 更严重的是，当使用线程池时，threadlocal 中的值可能会被重复使用，导致错误的赋值

解决这个问题的办法是线程结束前，调用 ThreadLocal 实例的 remove 方法

```java
public void remove() {
    ThreadLocalMap m = getMap(Thread.currentThread());
    if (m != null)
        m.remove(this);
}
```

## 一种使用方式

一种便利函数，将 threadlocal 存储的本地变量设置为 map，这样需要用到时可以随时 put/get，而不需要为每个要使用的全局变量都单独创建一个 threadlocal 实例。

```java
public class ThreadLocalUtil {

    private static final Logger logger = LoggerFactory.getLogger(ThreadLocalUtil.class);

    private static ThreadLocal<ThreadContext> threadLocal = new ThreadLocal<>();

    public static void put(String key, Object value) {
        logger.debug("===== put key: [{}] value: [{}] in threadLocal =====", key, SerializeUtil.toJsonString(value));

        ThreadContext threadContext = threadLocal.get();
        if (Objects.isNull(threadContext)) {
            threadContext = new ThreadContext();
            threadLocal.set(threadContext);
        }
        threadContext.put(key, value);
    }

    public static Object get(String key) {
        logger.debug("===== get value of key: [{}] from threadLocal =====", key);

        ThreadContext context = threadLocal.get();
        if (Objects.isNull(context)) {
            return null;
        }

        return context.get(key);
    }

    public static <T> T get(String key, Class<T> clazz) {
        logger.debug("===== get value of key: [{}] class: [{}] from threadLocal =====", key, clazz.getName());

        ThreadContext context = threadLocal.get();
        if (Objects.isNull(context)) {
            return null;
        }

        return context.get(key, clazz);
    }

    public static void clear() {
        logger.debug("====== clear threadLocal =====");
        threadLocal.remove();
    }

    public static void setThreadLocal(ThreadContext threadLocal) {

    }

    public static ThreadLocal<ThreadContext> getThreadLocal() {
        return threadLocal;
    }
}

public class ThreadContext {

    private Map<String, Object> contextMap;

    public ThreadContext() {
        this.contextMap = new HashMap<>();
    }

    public <T> T get(String key, Class<T> clazz) {
        return clazz.cast(contextMap.get(key));
    }

    public Object get(String key) {
        return contextMap.get(key);
    }

    public void put(String key, Object value) {
        contextMap.put(key, value);
    }

    public Map<String, Object> getContextMap() {
        return contextMap;
    }

    public void setContextMap(Map<String, Object> contextMap) {
        this.contextMap = contextMap;
    }
}
```

## 其他

因为目前还没有用到，这里先备注一下，以后要用到的话，可以搜一下用法：ThreadLocal 是支持子线程继承的，即将父线程的本地变量拷贝到子线程中使用。

## 参考

- [码出高效：Java开发手册](https://book.douban.com/subject/30333948/)
- Java ThreadLocal 实现源码
