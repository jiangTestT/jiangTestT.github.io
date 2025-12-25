# Redis 多数据源读写分离配置




## yml 更新内容

~~~
  redis:
    jedis:
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
        database: ${arb.redis.primary.writer.database}
        hostName: ${arb.redis.primary.writer.url}
        port: ${arb.redis.primary.writer.port}
      reader:
        database: ${arb.redis.primary.reader.database}
        hostName: ${arb.redis.primary.reader.url}
        port: ${arb.redis.primary.reader.port}
    oam:
      writer:
        database: ${arb.redis.oam.writer.database}
        hostName: ${arb.redis.oam.writer.url}
        port: ${arb.redis.oam.writer.port}
      reader:
        database: ${arb.redis.oam.reader.database}
        hostName: ${arb.redis.oam.reader.url}
        port: ${arb.redis.oam.reader.port}
    wsxch:
      writer:
        database: ${arb.redis.wsxch.writer.database}
        hostName: ${arb.redis.wsxch.writer.url}
        port: ${arb.redis.wsxch.writer.port}
      reader:
        database: ${arb.redis.wsxch.reader.database}
        hostName: ${arb.redis.wsxch.reader.url}
        port: ${arb.redis.wsxch.reader.port}
~~~

- 在原有的配置下面增加 writer 和 reader 。 在  profiles 中也要做相应的配置修改

  

## Configuration 修改内容

~~~
@Configuration
public class OAMRedisConfiguration {


    @Bean
    @ConfigurationProperties(prefix = "spring.redis.jedis.pool")
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
    @ConfigurationProperties(prefix = "spring.redis.oam.reader")
    public RedisStandaloneConfiguration oamReaderRedisConfig() {
        RedisStandaloneConfiguration redisStandaloneConfiguration = new RedisStandaloneConfiguration();
        redisStandaloneConfiguration.setDatabase();
        return
    }

    @Bean
    public JedisConnectionFactory oamReaderFactory() {
        JedisClientConfiguration jedisClientConfiguration = JedisClientConfiguration.builder().usePooling()
                .poolConfig(redisPool())
                .build();
        return new JedisConnectionFactory(oamReaderRedisConfig(), jedisClientConfiguration);
    }

    @Bean(name = "oamReaderRedisTemplate")
    public StringRedisTemplate oamReaderRedisTemplate() {
        StringRedisTemplate template = getRedisTemplate();
        template.setConnectionFactory(oamReaderFactory());
        return template;
    }

    @Bean
    @ConfigurationProperties(prefix = "spring.redis.oam.writer")
    public RedisStandaloneConfiguration oamWriterRedisConfig() {
        return new RedisStandaloneConfiguration();
    }

    @Bean
    public JedisConnectionFactory oamWriterFactory() {
        JedisClientConfiguration jedisClientConfiguration = JedisClientConfiguration.builder().usePooling()
                .poolConfig(redisPool())
                .build();
        return new JedisConnectionFactory(oamWriterRedisConfig(), jedisClientConfiguration);
    }

    @Bean(name = "oamWriterRedisTemplate")
    public StringRedisTemplate oamWriterRedisTemplate() {
        StringRedisTemplate template = getRedisTemplate();
        template.setConnectionFactory(oamWriterFactory());
        return template;
    }
}
~~~

- 将原有的单一 authRedisTemplate 配置修改为读写分离的配置模式

## RedisRepository 修改内容

~~~
@Repository
public class OAMRedisRepository extends BaseRedisRepository {

    /**
     * 示例: 使用写连接去执行读的操作
     *
     * @param cacheKey 要获取的cache key
     * @param options  组成 cache key 的参数
     * @return 数组长度
     */
    public Long fetchSetSizeFromWriter(String cacheKey, Object... options) {
        return redisWriterTemplate().opsForSet().size(generateKey(cacheKey, options));
    }

    /**
     * 重写基类Repository中的方法，以切换成slave源，完成多数据源的配置
     * @return StringRedisTemplate
     */
    @Override
    protected StringRedisTemplate redisWriterTemplate() {
        return oamWriterRedisTemplate;
    }

    /**
     * 重写基类Repository中的方法，以切换成slave源，完成多数据源的配置
     * @return StringRedisTemplate
     */
    @Override
    protected StringRedisTemplate redisReaderTemplate() {
        return oamReaderRedisTemplate;
    }

    /**
     * 多数据源的配置。自动注入配置好的一个redis源的模板对象，
     * 此对象必须在配置类中进行配置，并且此处注入的bean name需要与配置一致
     */
    @Autowired
    @Qualifier("oamWriterRedisTemplate")
    private StringRedisTemplate oamWriterRedisTemplate;

    /**
     * 多数据源的配置。自动注入配置好的一个redis源的模板对象，
     * 此对象必须在配置类中进行配置，并且此处注入的bean name需要与配置一致
     */
    @Autowired
    @Qualifier("oamReaderRedisTemplate")
    private StringRedisTemplate oamReaderRedisTemplate;
}
~~~

- 需要将 Configuration 里面读写的StringRedisTemplate注入到 RedisRepository中 并按数据源重写 redisWriterTemplate 和 redisReaderTemplate 接口

- 示例: 使用写连接去执行读的操作

