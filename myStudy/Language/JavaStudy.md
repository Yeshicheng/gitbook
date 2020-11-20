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

## 多线程学习随记

### ThreadLocal

- ThreadLocal原理：将数据存放在currentThread的ThreadLocalMap【Thread的一个内部类】对象中
  - 执行ThreadLocal.set方法：获取当前线程对象，获取当前线程的ThreadLocalMap对象，首次会使用当前线程和首个值创建该Map对象new ThreadLocalMap(this, firstValue)【table[i] = new Entry(firstKey,firstValue)】，ThreadLocalMap底层数据结构是一个Entry数组（同，将数据放在对应hash值的数组索引位置）
- ThreadLocal设置初始值：重写initValue()方法【默认返回null】
- ThreadLocal不能实现值继承：子线程不能获取父线程的ThreadLocal的值
- 使用InheritableThreadLocal实现值继承
  - 不再通过ThreadLocal.ThreadLocalMap threadLocals中存放数据，而是向ThreadLocal.ThreadLocal-Map inheritableThreadLocals中存放，子线程主动地引用父线程中的inheritableThreadLocals对象的值从而实现继承【将值进行拷贝（浅拷贝）并创建新的ThreadLocalMap对象，因此父线程的TreadLocalMap只是提供了一个初始化的作用】。
  - 由于采用的是浅拷贝，因此对于引用的可变对象，子线程可以感应到引用对象属性值的变化
  - 通过重写childValue方法实现堆积成的值的加工protected Object childValue(Object parentValue)：这里完全可以自定义实现深拷贝

### Lock对象（JDK 1.5）

> Java语言中采用的索为可重入锁

- ReentrantLock
