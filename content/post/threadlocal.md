---
title: "Threadlocal 梳理"
date: 2019-12-22T18:24:10+08:00
---

## 使用场景

用于同一线程中，不同类、不同方法间数据共享。没有 ThreadLocal 的帮助，这些需要共享的数据，就必须通过参数、返回值进行传递，将会变得非常繁琐。常见的使用场景，比如存储登录用户信息，同一个 request 的 trace id 等。

## 实现

ThreadLocalMap 是 ThreadLocal 的静态内部类，Entry 是 ThreadLocalMap 的静态内部类，其中 Entry 是弱引用的子类，referent 指向 threadlocal 对象，value 成员变量则是真正存储用户设置的值的地方。

### 如何存储

首先，用一张图来表示一下 threadlocal 是如何存储数据的：

![threadlocal.png](/media/threadlocal/threadlocal.png)

可以看到，用户关心的数据存储在 ThreadLocal 内部静态类 Entry 的 value 成员变量中，Entry 类型的变量有多个，组成数组 `table`，由 ThreadLocalMap 的实例所持有，而 ThreadLocalMap 实例则是线程的成员变量 threadlocals。因此，当多线程环境下访问同一个类实例的 ThreadLocal 变量时，其实每个线程都有各自的 ThreadLocalMap 变量 threadlocals，从而持有各自的 Entry 及其 value。这就是 ThreadLocal 如何做到线程安全的，不同线程使用的是不同副本。

ThreadLocalMap 内部维护了一个 Entry[] 类型的数组变量 `table`，索引 Entry 的实例时，有两种方式，一种是通过 hashcode 计算快速定位，另外一种是快速定位失效时，使用 threadlocal 实例循环比对 entry 的 referant。

ThreadLocal, ThreadLocalMap, Entry 三者的套娃定义如下：

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

### 获取过程

上面说明了 ThreadLocal 数据是如何存储的，那么当使用 ThreadLocal 时，是如何创建和获取数据的呢？一个简单的使用 ThreadLocal 的代码片段如下所示：

```Java
ThreadLocal<Integer> threadLocal = new ThreadLocal<Integer>() {
    @Override
    protected void initValue() {
        return 5;
    }
};
threadLocal.get();
```

相较于创建，我们先来看下 ThreadLocal 更高频使用的 `get()` 方法：

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

可以看到，在这个方法中会去获取当前线程`t`，然后使用 `t` 获取当前线程持有的 ThreadLocalMap 实例变量 `map`，然后根据 `map` 和 `this` 得到对应的 Entry 实例 `e`，最终获取用户数据 `e.value`。如果获取不到，则调用 `setInitialValue()` 进行初始化。

获取当前线程 `t` 和获取 `map` 都是比较直白的过程，直接略过，先看下 `ThreadLocalMap.Entry e = map.getEntry(this)` 是如何获取 Entry `e` 的：

```java
/**
    * Get the entry associated with key.  This method
    * itself handles only the fast path: a direct hit of existing
    * key. It otherwise relays to getEntryAfterMiss.  This is
    * designed to maximize performance for direct hits, in part
    * by making this method readily inlinable.
    *
    * @param  key the thread local object
    * @return the entry associated with key, or null if no such
    */
private Entry ThreadLocalMap.getEntry(ThreadLocal<?> key) {
    int i = key.threadLocalHashCode & (table.length - 1);
    Entry e = table[i];
    if (e != null && e.get() == key)
        return e;
    else
        return getEntryAfterMiss(key, i, e);
}
/**
    * Version of getEntry method for use when key is not found in
    * its direct hash slot.
    *
    * @param  key the thread local object
    * @param  i the table index for key's hash code
    * @param  e the entry at table[i]
    * @return the entry associated with key, or null if no such
    */
private Entry ThreadLocalMap.getEntryAfterMiss(ThreadLocal<?> key, int i, Entry e) {
    Entry[] tab = table;
    int len = tab.length;

    while (e != null) {
        ThreadLocal<?> k = e.get();
        if (k == key)
            return e;
        if (k == null)
            expungeStaleEntry(i);
        else
            i = nextIndex(i, len);
        e = tab[i];
    }
    return null;
}
```

`this` 是当前 ThreadLocal 对象实例，使用它作为 key，在 ThreadLocalMap 持有的 `Entry[] table` 中索引相应的 Entry `e`，可以看到，这里便是上面说到的索引 Entry 的两种方式：首先根据 hashcode 快速索引，如果找不到，则调用 `getEntryAfterMiss()` 方法，将当前 ThreadLocal 对象实例 `this` 作为 key，与 Entry 的 referant 作比对，获取相应的 entry。

### 创建过程

接下来看下如果获取不到 Entry，ThreadLocal 如何通过 `setInitialValue()` 进行初始化：

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

首先，调用 `initialValue()` 拿到用户需要存储的值。我们创建 ThreadLocal 对象时可以 override `initialValue()` 方法，并返回需要存储的数据。

获取值后，再通过 `getMap(t)` 获取当前线程所持有的 ThreadLocalMap 实例即这里的变量 `map`，如果 `map` 不为空，则调用 `set(ThreadLocal<?> key, Object value)`将用户数据放入 referant 与当前 threadlocal 实例对象对应的 entry，为空则调用 `createMap(Thread t, T firstValue)` 进行创建，并赋值。

从这里也可以看到，当不同线程访问同一个 threadlocal 对象时，获取 map 也是要根据当前线程来得到，因此不同线程会有各自独享的本地变量。

### 总结

同一个线程中，不同的 ThreadLocal 实例，都会存储在线程成员变量 `ThreadLocal.ThreadLocalMap threadlocals` 的 `Entry[] table` 数组中，不同的 ThreadLocal 实例，通过 ThreadLocal 实例的 hashcode/ Entry 变量的弱引用 referant 进行区分。

同一个 ThreadLocal 类型变量的定义，不同线程去执行时，会在自己的线程中单独创建、获取，从而持有线程独享的本地变量。

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

一种便利函数，相比于不同变量分别去创建 ThreadLocal 类型的变量，在 Entry 中通过 referant 去区分，不如直接自己创建一个 HashMap 放到同一个 ThreadLocal 实例中（即只使用一个 Entry），在业务逻辑层面直接对不同变量通过名字进行映射，这样需要用到时可以随时 `put()`/`get()`，而不需要为每个要使用的全局变量都单独创建一个 ThreadLocal 实例。

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
