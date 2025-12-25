# Redis 多数据源读写分离配置


## 概速

在现实项目中，我们往往会遇到需要使用多个 Redis 数据源的场景。本文介绍的是一种高度定制化的方案。每个独立的数据源都会使用自己的配置，其中包括针对该数据源的连接池配置。

## yml 更新内容

~~~
  redis:
    lettuce:
      pool:
        maxTotal: 200
        minIdle: 1
        maxWaitMillis: 5000
        maxIdle: 5
        testOnBorrow: true
        testOnReturn: true
        testWhileIdle: true
    primary:
      writer:
        database: ${jerry.redis.primary.writer.database}
        hostName: ${jerry.redis.primary.writer.url}
        port: ${jerry.redis.primary.writer.port}
      reader:
        database: ${jerry.redis.primary.reader.database}
        hostName: ${jerry.redis.primary.reader.url}
        port: ${jerry.redis.primary.reader.port}
    datasourceA:
      writer:
        database: ${jerry.redis.datasourceA.writer.database}
        hostName: ${jerry.redis.datasourceA.writer.url}
        port: ${ajerryrb.redis.datasourceA.writer.port}
      reader:
        database: ${jerry.redis.datasourceA.reader.database}
        hostName: ${jerry.redis.datasourceA.reader.url}
        port: ${jerry.redis.datasourceA.reader.port}
	datasourceB:
      writer:
        database: ${jerry.redis.datasourceB.writer.database}
        hostName: ${jerry.redis.datasourceB.writer.url}
        port: ${ajerryrb.redis.datasourceB.writer.port}
      reader:
        database: ${jerry.redis.datasourceB.reader.database}
        hostName: ${jerry.redis.datasourceB.reader.url}
        port: ${jerry.redis.datasourceB.reader.port}
~~~

- 在原有的配置下面增加 writer 和 reader。 在  profiles 中也要做相应的配置修改

  

## Primary 数据源和 Redis Repository 配置

数据源配置

```java
Configuration
public class PrimaryRedisConfiguration {

    /**
     * 配置lettuce连接池
     */
    @Bean(name = "redisPool")
    @ConfigurationProperties(prefix = "spring.redis.lettuce.pool")
    @Scope(value = "prototype")
    public GenericObjectPoolConfig redisPool() {
        return new GenericObjectPoolConfig();
    }

    @Bean(name = "primaryReaderRedisConfig")
    @ConfigurationProperties(prefix = "spring.redis.primary.reader")
    public RedisStandaloneConfiguration primaryReaderRedisConfig() {
        return new RedisStandaloneConfiguration();
    }

    @Bean(name = "primaryWriterRedisConfig")
    @ConfigurationProperties(prefix = "spring.redis.primary.writer")
    @Primary
    public RedisStandaloneConfiguration primaryWriterRedisConfig() {
        return new RedisStandaloneConfiguration();
    }

    @Bean(name = "redisReaderConnectionFactory")
    public LettuceConnectionFactory redisReaderConnectionFactory() {
        LettucePoolingClientConfiguration poolingClientConfiguration = LettucePoolingClientConfiguration.builder()
                .poolConfig(redisPool())
                .build();
        return new LettuceConnectionFactory(primaryReaderRedisConfig(), poolingClientConfiguration);
    }

    @Bean(name = "primaryReaderRedisTemplate")
    public StringRedisTemplate primaryReaderRedisTemplate() {
        StringRedisTemplate template = new StringRedisTemplate();
        template.setConnectionFactory(redisReaderConnectionFactory());
        return template;
    }

    @Bean(name = "redisWriterConnectionFactory")
    @Primary
    public LettuceConnectionFactory redisWriterConnectionFactory() {
        LettucePoolingClientConfiguration poolingClientConfiguration = LettucePoolingClientConfiguration.builder()
                .poolConfig(redisPool())
                .build();
        return new LettuceConnectionFactory(primaryWriterRedisConfig(), poolingClientConfiguration);
    }

    @Bean(name = "primaryWriterRedisTemplate")
    @Primary
    public StringRedisTemplate primaryWriterRedisTemplate() {
        StringRedisTemplate template = new StringRedisTemplate();
        template.setConnectionFactory(redisWriterConnectionFactory());
        return template;
    }
}
```

Redis Repository 配置，这是 Redis 基础操作类

```java
@Repository
public abstract class BaseRedisRepository {

    @Autowired
    @Qualifier("primaryReaderRedisTemplate")
    private StringRedisTemplate readerRedisTemplate;

    @Autowired
    @Qualifier("primaryWriterRedisTemplate")
    private StringRedisTemplate writerRedisTemplate;

    public StringRedisTemplate getReaderRedisTemplate() {
        return readerRedisTemplate;
    }

    public StringRedisTemplate getWriterRedisTemplate() {
        return writerRedisTemplate;
    }

    public String generateKey(String cacheKey, Object... options) {
        String idStr = connectArray(GlobalConst.STR_POUND, options);
        return String.format("%s@%s", cacheKey, CryptUtil.md5(idStr));
    }


    /**
     * 通过CacheKey 缓存字符串
     *
     * @param cacheKey  缓存Key
     * @param dataStr   缓存数据
     * @param expiredIn 过期时间（秒）
     * @param args      生成最终缓存key的参数
     */
    public void cacheString(String cacheKey, String dataStr, Long expiredIn, Object... args) {
        cacheKey = generateKey(cacheKey, args);
        set(cacheKey, dataStr, expiredIn);
    }

    /**
     * 根据传入Key获取对应String
     *
     * @param cacheKey 缓存建
     * @param args     不定长参数
     */
    public String fetchString(String cacheKey, Object... args) {
        String result = null;
        cacheKey = generateKey(cacheKey, args);
        if (hasKey(cacheKey)) {
            result = get(cacheKey);
        }
        return result;
    }

    public static String connectArray(String separator, Object... args) {
        StringBuilder builder = new StringBuilder();
        String s = GlobalConst.STR_EMPTY;
        if (Objects.nonNull(args)) {
            for (Object arg : args) {
                if (Objects.nonNull(arg)) {
                    builder.append(s).append(arg);
                    if (GlobalConst.STR_EMPTY.equals(s)) {
                        s = separator;
                    }
                }
            }
        }
        return builder.toString();
    }

    public Integer set(String key, String value) {
        try {
            getWriterRedisTemplate().opsForValue().set(key, value);
            return CodeConst.SUCCESS;
        } catch (Exception e) {
            return CodeConst.FAILURE;
        }
    }

    public void set(String key, String value, Long timeout) {
        if (Objects.nonNull(timeout) && timeout > 0) {
            getWriterRedisTemplate().opsForValue().set(key, value, timeout, TimeUnit.SECONDS);
        } else {
            getWriterRedisTemplate().opsForValue().set(key, value);
        }
    }

    public void setExpire(String key, Long timeout) {
        if (timeout > 0) {
            getWriterRedisTemplate().expire(key, timeout, TimeUnit.SECONDS);
        }
    }

    public String get(String key) {
        if (StringUtils.isNotBlank(key)) {
            return getReaderRedisTemplate().opsForValue().get(key);
        } else {
            return null;
        }
    }

    public Long getExpire(String key) {
        return getReaderRedisTemplate().getExpire(key, TimeUnit.SECONDS);
    }

    public Boolean hasKey(String key) {
        try {
            return getReaderRedisTemplate().hasKey(key);
        } catch (Exception e) {
            return false;
        }
    }

    public void delete(String key) {
        if (StringUtils.isNotBlank(key)) {
            getWriterRedisTemplate().delete(key);
        }
    }

    public void cacheListHead(String cacheKey, String value, Object... args) {
        cacheKey = generateKey(cacheKey, args);
        getWriterRedisTemplate().opsForList().leftPush(cacheKey, value);
    }

    public void cacheListTail(String cacheKey, String value, Object... args) {
        cacheKey = generateKey(cacheKey, args);
        getWriterRedisTemplate().opsForList().rightPush(cacheKey, value);
    }

    public String fetchListHead(String cacheKey, Object... args) {
        return fetchListHead(cacheKey, 1, TimeUnit.SECONDS, args);
    }

    public String fetchListHead(String cacheKey, long timeout, TimeUnit unit, Object... args) {
        cacheKey = generateKey(cacheKey, args);
        return getWriterRedisTemplate().opsForList().leftPop(cacheKey, timeout, unit);
    }

    public void removeListValue(String cacheKey, String value, Object... args) {
        cacheKey = generateKey(cacheKey, args);
        getWriterRedisTemplate().opsForList().remove(cacheKey, 0, value);
    }

    public void cacheSetValue(String cacheKey, String... values) {
        getWriterRedisTemplate().opsForSet().add(cacheKey, values);
    }

    public void removeSetValue(String cacheKey, String value) {
        getWriterRedisTemplate().opsForSet().remove(cacheKey, value);
    }

    public Long countSetValue(String cacheKey) {
        return getWriterRedisTemplate().opsForSet().size(cacheKey);
    }

    public Set<String> fetchSetAllValue(String cacheKey) {
        return getReaderRedisTemplate().opsForSet().members(cacheKey);
    }

    public void cacheZSet(String cacheKey, String value) {
        long currentTime = System.currentTimeMillis();
        getWriterRedisTemplate().opsForZSet().add(cacheKey, value, currentTime);
    }

    public void cacheZSet(String cacheKey, String value, double score) {
        getWriterRedisTemplate().opsForZSet().add(cacheKey, value, score);
    }

    public void removeValueFromZSet(String cacheKey, String value) {
        getWriterRedisTemplate().opsForZSet().remove(cacheKey, value);
    }

    public boolean checkValueInSortedSet(String cacheKey, String value) {
        return Objects.nonNull(getWriterRedisTemplate().opsForZSet().rank(cacheKey, value));
    }

    public Set<String> fetchZSetMinValue(String cacheKey) {
        return fetchZSetValue(cacheKey, 0, 1);
    }

    public Set<String> fetchZSetAllValue(String cacheKey) {
        return fetchZSetValue(cacheKey, 0, -1);
    }

    public Set<String> fetchZSetValue(String cacheKey, long start, long end) {
        return getReaderRedisTemplate().opsForZSet().range(cacheKey, start, end);
    }

    public void convertAndSend(String channel, String message) {
        getWriterRedisTemplate().convertAndSend(channel, message);
    }

    public void setHashValue(String cacheKey, String key, String value) {
        getWriterRedisTemplate().opsForHash().put(cacheKey, key, value);
    }

    public Object getHashValue(String cacheKey, String key) {
        return getReaderRedisTemplate().opsForHash().get(cacheKey, key);
    }

    public Map<Object, Object> getHashEntries(String cacheKey) {
        return getReaderRedisTemplate().opsForHash().entries(cacheKey);
    }

    public void updateHashValue(String cacheKey, String key, String newValue) {
        getWriterRedisTemplate().opsForHash().put(cacheKey, key, newValue);
    }

    public void deleteHashField(String cacheKey, String key) {
        getWriterRedisTemplate().opsForHash().delete(cacheKey, key);
    }

    /**
     * 如果不存在，缓存值
     */
    public Boolean setIfAbsent(String cacheKey, String value, Long expire) {
        cacheKey = generateKey(cacheKey, value);
        return getWriterRedisTemplate().opsForValue().setIfAbsent(cacheKey, value, expire, TimeUnit.SECONDS);
    }
}
```



## Configuration 修改内容

如果需要配置多数据源，我们需要增加对应的 Redis 配置类

~~~
@Configuration
public class DatasourceARedisConfiguration {

    @Bean
    @ConfigurationProperties(prefix = "spring.redis.lettuce.pool")
    @Scope(value = "prototype")
    public GenericObjectPoolConfig redisPool() {
        return new GenericObjectPoolConfig();
    }

    private StringRedisTemplate getRedisTemplate() {
        StringRedisTemplate template = new StringRedisTemplate();
        template.setValueSerializer(new GenericFastJsonRedisSerializer());
        template.setValueSerializer(new StringRedisSerializer());
        return template;
    }

    @Bean
    @ConfigurationProperties(prefix = "spring.redis.datasourceA.reader")
    public RedisStandaloneConfiguration datasourceAReaderRedisConfig() {
        RedisStandaloneConfiguration redisStandaloneConfiguration = new RedisStandaloneConfiguration();
        redisStandaloneConfiguration.setDatabase();
        return
    }

    @Bean
    public JedisConnectionFactory datasourceAReaderFactory() {
        LettucePoolingClientConfiguration poolingClientConfiguration = LettucePoolingClientConfiguration.builder()
                .poolConfig(redisPool())
                .build();
        return new LettuceConnectionFactory(primaryReaderRedisConfig(), poolingClientConfiguration);
    }

    @Bean(name = "datasourceAReaderRedisTemplate")
    public StringRedisTemplate datasourceAReaderRedisTemplate() {
        StringRedisTemplate template = getRedisTemplate();
        template.setConnectionFactory(datasourceAReaderFactory());
        return template;
    }

    @Bean
    @ConfigurationProperties(prefix = "spring.redis.datasourceA.writer")
    public RedisStandaloneConfiguration datasourceAWriterRedisConfig() {
        return new RedisStandaloneConfiguration();
    }

    @Bean
    public JedisConnectionFactory datasourceAWriterFactory() {
        LettucePoolingClientConfiguration poolingClientConfiguration = LettucePoolingClientConfiguration.builder()
                .poolConfig(redisPool())
                .build();
        return new LettuceConnectionFactory(primaryWriterRedisConfig(), poolingClientConfiguration);
    }

    @Bean(name = "datasourceAWriterRedisTemplate")
    public StringRedisTemplate datasourceAWriterRedisTemplate() {
        StringRedisTemplate template = getRedisTemplate();
        template.setConnectionFactory(datasourceAWriterFactory());
        return template;
    }
}
~~~

- 将原有的单一 authRedisTemplate 配置修改为读写分离的配置模式

## RedisRepository 修改内容

~~~
@Repository
public class DatasourceARedisRepository extends BaseRedisRepository {

    /**
     * 重写基类Repository中的方法，以切换成slave源，完成多数据源的配置
     * @return StringRedisTemplate
     */
    @Override
    protected StringRedisTemplate redisWriterTemplate() {
        return datasourceAWriterRedisTemplate;
    }

    /**
     * 重写基类Repository中的方法，以切换成slave源，完成多数据源的配置
     * @return StringRedisTemplate
     */
    @Override
    protected StringRedisTemplate redisReaderTemplate() {
        return datasourceAReaderRedisTemplate;
    }

    /**
     * 多数据源的配置。自动注入配置好的一个redis源的模板对象，
     * 此对象必须在配置类中进行配置，并且此处注入的bean name需要与配置一致
     */
    @Autowired
    @Qualifier("datasourceAWriterRedisTemplate")
    private StringRedisTemplate datasourceAWriterRedisTemplate;

    /**
     * 多数据源的配置。自动注入配置好的一个redis源的模板对象，
     * 此对象必须在配置类中进行配置，并且此处注入的bean name需要与配置一致
     */
    @Autowired
    @Qualifier("datasourceAReaderRedisTemplate")
    private StringRedisTemplate datasourceAReaderRedisTemplate;
}
~~~

- 需要将 Configuration 里面读写的 StringRedisTemplate 注入到 RedisRepository 中 并按数据源重写 redisWriterTemplate 和 redisReaderTemplate 接口

- 示例: 使用写连接去执行读的操作

