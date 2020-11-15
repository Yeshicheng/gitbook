# Java学习

## Spring项目中的缓存设计

场景：监控静默配置规则，需根据用户配置的静默规则，定期加载至项目中作为静默处理依据。配置规则缓存特点：1.动态变化，过期的配置规则需要丢弃，同时会有新增的配置规则加入；2.配置项可预见的不会过多，由于报警静默为非日常需求，因此其产生的配置规则记录数不会过多；

实现思路：定期地读取数据库中有效（当前区域未过期）的静默配置记录，并丢弃原有的缓存，60s为一个周期加载配置缓存。

实现方式：Spring定时任务（注解Scheduled）

``` java

    // 缓存对象使用volatile修饰符，保证多线程数据的一致性（写入线程和读取线程）
    private volatile Map<Long, List<MultiValueTag>> tagIgnoreConfigCache;

    // 执行顺序：Constructor >> @Autowired >> @PostConstruct
    // 用于服务器在加载Servlet运行时，被服务器执行一次，确保启动时的周期内加载缓存数据
    @PostConstruct
    private void init() {
        reloadTagIgnoreConfigCache();
    }


    @Scheduled(initialDelay = 60000, fixedDelay = 60000)
    private void reloadTagIgnoreConfigCache() {
        // 使用临时的缓存对象去保存读取的数据，避免对多个线程在读取时共享的缓存对象值发生修改（即保证在一个缓存周期内缓存对象是不可变的）
        Map<Long, List<MultiValueTag>> tmpTagIgnoreConfigCache = new HashMap<>();
        // 省略...
        tagIgnoreConfigCache = tmpTagIgnoreConfigCache;
    }

```
