# 简介
官方介绍Caffeine是基于JDK8的高性能本地缓存库，提供了几乎完美的命中率。它有点类似JDK中的ConcurrentMap，实际上，Caffeine中的LocalCache接口就是实现了JDK中的ConcurrentMap接口，但两者并不完全一样。最根本的区别就是，ConcurrentMap保存所有添加的元素，除非显示删除之（比如调用remove方法）。而本地缓存一般会配置自动剔除策略，为了保护应用程序，限制内存占用情况，防止内存溢出。

Caffeine提供了灵活的构造方法，从而创建可以满足如下特性的本地缓存：

1. 自动把数据加载到本地缓存中，并且可以配置异步；<br/>
2. 基于数量剔除策略；<br/>
3. 基于失效时间剔除策略，这个时间是从最后一次访问或者写入算起；<br/>
4. 异步刷新；<br/>
5. Key会被包装成Weak引用；<br/>
6. Value会被包装成Weak或者Soft引用，从而能被GC掉，而不至于内存泄漏；<br/>
7. 数据剔除提醒；<br/>
8. 写入广播机制；<br/>
9. 缓存访问可以统计；<br/>

# 使用
Caffeine使用还是非常简单的，如果你用过GuavaCache，那就更简单了，因为Caffeine的API设计大量借鉴了GuavaCache。首先，引入Maven依赖：

```XML
<dependency>
    <groupId>com.github.ben-manes.caffeine</groupId>
    <artifactId>caffeine</artifactId>
    <version>2.8.4</version>
</dependency>
```

然后构造Cache使用即可：

```Java
Cache<String, String> cache = Caffeine.newBuilder()
        // 数量上限
        .maximumSize(1024)
        // 过期机制
        .expireAfterWrite(5, TimeUnit.MINUTES)
        // 弱引用key
        .weakKeys()
        // 弱引用value
        .weakValues()
        // 剔除监听
        .removalListener((RemovalListener<String, String>) (key, value, cause) -> 
                System.out.println("key:" + key + ", value:" + value + ", 删除原因:" + cause.toString()))
        .build();
// 将数据放入本地缓存中
cache.put("username", "afei");
cache.put("password", "123456");
// 从本地缓存中取出数据
System.out.println(cache.getIfPresent("username"));
System.out.println(cache.getIfPresent("password"));
System.out.println(cache.get("blog", key -> {
    // 本地缓存没有的话，从数据库或者Redis中获取
    return getValue(key);
}));
```

当然，使用本地缓存时，我们也可以使用异步加载机制：

```Java
AsyncLoadingCache<String, String> cache = Caffeine.newBuilder()
        // 数量上限
        .maximumSize(2)
        // 失效时间
        .expireAfterWrite(5, TimeUnit.MINUTES)
        .refreshAfterWrite(1, TimeUnit.MINUTES)
        // 异步加载机制
        .buildAsync(new CacheLoader<String, String>() {
            @Nullable
            @Override
            public String load(@NonNull String key) throws Exception {
                return getValue(key);
            }
        });
System.out.println(cache.get("username").get());
System.out.println(cache.get("password").get(10, TimeUnit.MINUTES));
System.out.println(cache.get("username").get(10, TimeUnit.MINUTES));
System.out.println(cache.get("blog").get());
```

接下来，我们对一些重要特性进行更加深入的分析。

# 过期机制
本地缓存的过期机制是非常重要的，因为本地缓存中的数据并不像业务数据那样需要保证不丢失。本地缓存的数据一般都会要求保证命中率的前提下，尽可能的占用更少的内存，并可在极端情况下，可以被GC掉。

Caffeine的过期机制都是在构造Cache的时候申明，主要有如下几种：

1.expireAfterWrite：表示自从最后一次写入后多久就会过期；<br/>
2.expireAfterAccess：表示自从最后一次访问（写入或者读取）后多久就会过期；<br/>
3.expireAfter：自定义过期策略；<br/>

# 刷新机制
在构造Cache时通过refreshAfterWrite方法指定刷新周期，例如refreshAfterWrite(10, TimeUnit.SECONDS)表示10秒钟刷新一次：

```Java
.build(new CacheLoader<String, String>() {
    @Override
    public String load(String k) {
        // 这里我们就可以从数据库或者其他地方查询最新的数据
        return getValue(k);
    }
});
```

需要注意的是，Caffeine的刷新机制是「被动」的。举个例子，假如我们申明了10秒刷新一次。我们在时间T访问并获取到值v1，在T+5秒的时候，数据库中这个值已经更新为v2。但是在T+12秒，即已经过了10秒我们通过Caffeine从本地缓存中获取到的「还是v1」，并不是v2。在这个获取过程中，Caffeine发现时间已经过了10秒，然后会将v2加载到本地缓存中，下一次获取时才能拿到v2。即它的实现原理是在get方法中，调用afterRead的时候，调用refreshIfNeeded方法判断是否需要刷新数据。这就意味着，如果不读取本地缓存中的数据的话，无论刷新时间间隔是多少，本地缓存中的数据永远是旧的数据！

# 剔除机制
在构造Cache时可以通过removalListener方法申明剔除监听器，从而可以跟踪本地缓存中被剔除的数据历史信息。根据RemovalCause.java枚举值可知，剔除策略有如下5种：
+ 「EXPLICIT」：调用方法（例如：cache.invalidate(key)、cache.invalidateAll）显示剔除数据；
+ 「REPLACED」：不是真正被剔除，而是用户调用一些方法（例如：put()，putAll()等）改了之前的值；
+ 「COLLECTED」：表示缓存中的Key或者Value被垃圾回收掉了；
+ 「EXPIRED」: expireAfterWrite/expireAfterAccess约定时间内没有任何访问导致被剔除；
+ 「SIZE」：超过maximumSize限制的元素个数被剔除的原因；

# GuavaCache和Caffeine差异
1. 剔除算法方面，GuavaCache采用的是「LRU」算法，而Caffeine采用的是「Window TinyLFU」算法，这是两者之间最大，也是根本的区别。<br/>
2. 立即失效方面，Guava会把立即失效 (例如：expireAfterAccess(0) and expireAfterWrite(0)) 转成设置最大Size为0。这就会导致剔除提醒的原因是SIZE而不是EXPIRED。Caffiene能正确识别这种剔除原因。<br/>
3. 取代提醒方面，Guava只要数据被替换，不管什么原因，都会触发剔除监听器。而Caffiene在取代值和先前值的引用完全一样时不会触发监听器。<br/>
4. 异步化方方面，Caffiene的很多工作都是交给线程池去做的（默认：ForkJoinPool.commonPool()），例如：剔除监听器，刷新机制，维护工作等。<br/>
