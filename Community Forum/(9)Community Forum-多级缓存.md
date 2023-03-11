# 多级缓存
->一级缓存（本地缓存) ->二级缓存（分布式缓存) ->DB<br/>
<br/>
本地缓存选用Caffeine这一款基于Java 8的高性能，接近最佳的缓存库。<br/>
<br/>
Caffeine的使用也比较的简便，如下所示

```Java
    //帖子列表缓存
    private LoadingCache<String, List<DiscussPost>> postListCache;

    //帖子总数缓存
    private LoadingCache<Integer, Integer> postRowsCache;

    @PostConstruct
    public void init() {
        //初始化帖子列表缓存
        postListCache = Caffeine.newBuilder()
                .maximumSize(maxSize)
                .expireAfterWrite(expireSeconds, TimeUnit.SECONDS)
                .build(new CacheLoader<String, List<DiscussPost>>() {
                    @Override
                    public @Nullable List<DiscussPost> load(@NonNull String key) throws Exception {
                        if (key == null || key.length() == 0) {
                            throw new IllegalArgumentException("参数错误");
                        }
                        String[] params = key.split(":");
                        if (params == null || params.length != 2) {
                            throw new IllegalArgumentException("参数错误");
                        }
                        int offset =  Integer.parseInt(params[0]);
                        int limit = Integer.parseInt(params[1]);
                        //二级缓存： Redis -> MySQL
                        String redisKey = RedisKeyUtil.getPostKey(offset, limit);
                        Set<DiscussPost> set = redisTemplate.opsForZSet().range(redisKey, offset, offset + limit - 1);
                        List<DiscussPost> list = new ArrayList<>();
                        list.addAll(set);
                        if (list != null && list.size() > 0) {
                            logger.info("从Redis获取数据");
                            return list;
                        }
                        list = discussPostMapper.selectDiscussPosts(0, offset, limit, 1);
                        //更新Redis缓存
                        for (int i = 0; i < list.size(); i++) {
                            redisTemplate.opsForZSet().add(redisKey, list.get(i), i);
                        }
                        logger.info("从数据库获取数据");
                        return list;
                    }
                });
        //初始化帖子总数缓存
        postRowsCache = Caffeine.newBuilder()
                .maximumSize(maxSize)
                .expireAfterWrite(expireSeconds, TimeUnit.SECONDS)
                .build(new CacheLoader<Integer, Integer>() {
                    @Override
                    public @Nullable Integer load(@NonNull Integer integer) throws Exception {
                        //二级缓存： Redis -> MySQL

                        return discussPostMapper.selectDiscussPostRows(0);
                    }
                });
    }
```

初始化时配好参数，重写load方法，此方法是在Caffeine未存储所需数据时，所要采取的导入操作，方法最终的返回就是要导入本地缓存的数据。<br/>
参数key是由起始帖子位置和截至帖子位置拼接而成。<br/>
一级缓存和二级缓存的key代表了某一页，value也就是那一页的帖子列表的数据。<br/>
既然一级Caffeine缓存中未查询到指定数据，那就查询二级Redis缓存。这里Redis中采用ZSet数据结构，是为了保证帖子的有序性。因此，score的值可以直接设置为从0到size-1，只需保证当前页的顺序性即可。<br/>
如果二级Redis缓存也未查询到，那就只能到MySQL中查询，并更新二级缓存和一级缓存。<br/>
<br/>
当然了，当发生数据的新增和更新时，就需要删除缓存。

```Java
//清空缓存
postListCache.invalidateAll();
postRowsCache.invalidateAll();
Set<String> set = redisTemplate.keys("post:page*");
redisTemplate.delete(set);
```

Caffeine缓存直接清空。<br/>
Redis中则是模糊匹配key，然后再全部删除。