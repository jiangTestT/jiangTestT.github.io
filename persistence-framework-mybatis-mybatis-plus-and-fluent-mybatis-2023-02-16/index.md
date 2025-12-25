# MyBatis 各个框架集成和性能对比


MyBatis 是一款优秀的持久层框架，它支持自定义 SQL、存储过程以及高级映射。本文章主要是讲述如何集成 MyBatis 及其扩展框架，清楚地讲述 Mybatis、Mybatis Plus 和 Fluent MyBatis 的 CRUD 操作。最后将三者的使用性能做了一个对比，在实际开发过程中，建议使用原生的 MyBatis，具有较强的扩展性，并且其性能较高；当然，如果你追求高效的开发效率，建议使用 MyBatis Plus。



## 多数据源配置(通用)

**数据库读写分离实现方案**

- 通过`Mybatis`配置文件

  通过`MyBatis`配置文件创建读写分离两个`DataSource`，每个`SqlSessionFactoryBean`对象的`mapperLocations`属性制定两个读写数据源的配置文件。将所有读的操作配置在读文件中，所有写的操作配置在写文件中。

- 通过`Spring AOP`

  通过`Spring AOP`在业务层实现读写分离，在`DAO`层调用前定义切面，利用`Spring`的**`AbstractRoutingDataSource`**解决多数据源的问题，实现动态选择数据源

- 通过`Mybatis`的`Plugin`

  通过`Mybatis`的`Plugin`在业务层实现数据库读写分离，在`MyBatis`创建`Statement`对象前通过拦截器选择真正的数据源，在拦截器中根据方法名称不同（`select`、`update`、`insert`、`delete`）选择数据源。

- 通过`Spring`的`AbstractRoutingDataSource`和`Mybatis Plugin`拦截器

  如果你的后台结构是`Spring` + `Mybatis`，可以通过`Spring`的`AbstractRoutingDataSource`和`Mybatis Plugin`拦截器实现非常友好的读写分离，原有代码不需要任何改变

#### 配置bootstrap.yml文件

```yml
spring:
  # 数据库组件
  datasource:
    master:
      writer:
        type: com.alibaba.druid.pool.DruidDataSource # Druid数据库连接池
        driver-class-name: com.mysql.cj.jdbc.Driver
        url: jdbc:mysql://${szt.database.writer.url}/customer_migration?charset=utf8mb4&useSSL=false&rewriteBatchedStatements=true&autoReconnect=true
        username: ${szt.database.username}
        password: ${szt.database.password}
        testOnBorrow: true                          # 指明是否从池中取出连接前进行检验，如果检验失败，则从池中取出连接并尝试取出另一个
        testOnReturn: true                          # 指明连接是否被空闲连接回收器（如果有）进行检验，如果检测失败，则连接将被从池中去除
        testWhileIdle: true
        timeBetweenEvictionRunsMillis: 60000        # 在空闲连接回收器线程运行期间休眠的时间值，姨毫秒为单位。如果设置为非整数，则不运行空闲连接回收器线程
        initialSize: 20
        minIdle: 20
        maxActive: 256
        maxWait: 60000                              # 配置获取连接等待超时的时间
        minEvictableIdleTimeMillis: 300000          # 配置一个连接在池中最小生存的时间，单位是毫秒
        validationQuery: SELECT'x'
        poolPreparedStatements: true                # 打开PSCache，并且指定每个连接上PSCache的大小
        maxPoolPreparedStatementPerConnectionSize: 20
        filters: stat,wall,slf4j                    # 配置监控统计拦截的filters，去掉后监控界面sql无法统计，'wall'用于防火墙，此处是filter修改的地方
      reader:
        type: com.alibaba.druid.pool.DruidDataSource # Druid数据库连接池
        driver-class-name: com.mysql.cj.jdbc.Driver
        url: jdbc:mysql://${szt.database.reader.url}/${szt.database.schema}?charset=utf8mb4&useSSL=false&rewriteBatchedStatements=true&autoReconnect=true
        username: ${szt.database.username}
        password: ${szt.database.password}
        testOnBorrow: true                          # 指明是否从池中取出连接前进行检验，如果检验失败，则从池中取出连接并尝试取出另一个
        testOnReturn: true                          # 指明连接是否被空闲连接回收器（如果有）进行检验，如果检测失败，则连接将被从池中去除
        testWhileIdle: true
        timeBetweenEvictionRunsMillis: 60000        # 在空闲连接回收器线程运行期间休眠的时间值，姨毫秒为单位。如果设置为非整数，则不运行空闲连接回收器线程
        initialSize: 20
        minIdle: 20
        maxActive: 256
        maxWait: 60000                              # 配置获取连接等待超时的时间
        minEvictableIdleTimeMillis: 300000          # 配置一个连接在池中最小生存的时间，单位是毫秒
        validationQuery: SELECT'x'
        poolPreparedStatements: true                # 打开PSCache，并且指定每个连接上PSCache的大小
        maxPoolPreparedStatementPerConnectionSize: 20
        filters: stat,wall,slf4j                    # 配置监控统计拦截的filters，去掉后监控界面sql无法统计，'wall'用于防火墙，此处是filter修改的地方
```

#### 配置多数据源DataSourceConfig

```java
@Configuration
public class DataSourceConfig {

    @Primary
    @Bean(name = "writerDataSource")
    @ConfigurationProperties(prefix = "spring.datasource.master.writer")
    public DataSource writerDataSource() {
        return new DruidDataSource();
    }

    @Bean(name = "readerDataSource")
    @ConfigurationProperties(prefix = "spring.datasource.master.reader")
    public DataSource readerDataSource() {
        return new DruidDataSource();
    }
}
```

#### 全局动态数据源枚举

```java
/**
 * 全局动态数据源实体
 */
public enum DataSourceEnum {
    READ, WRITE;
}
```

#### 配置DynamicDataSourceHolder

```java
/**
 * 动态数据源线程持有者
 */
public final class DynamicDataSourceHolder {

    private static final ThreadLocal<DataSourceEnum> holder = new ThreadLocal<>();

    /**
     * 设置当前线程使用的数据源
     */
    public static void putDataSource(DataSourceEnum dataSource) {
        holder.set(dataSource);
    }

    /**
     * 获取当前线程需要使用的数据源
     */
    public static DataSourceEnum getDataSource() {
        return holder.get();
    }

    /**
     * 清空使用的数据源
     */
    public static void clearDataSource() {
        holder.remove();
    }

}
```

#### 动态数据源DynamicDataSource

```java
/**
 * 动态数据源（继承自spring抽象动态路由数据源）
 */
public class DynamicDataSource extends AbstractRoutingDataSource {

    private DataSource writeDataSource; //写数据源

    private DataSource readDataSource; //读数据源

    /**
     * 在初始化之前被调用，设置默认数据源，以及数据源资源（这里的写法是参考源码中的）
     */
    @Override
    public void afterPropertiesSet() {
        //如果写数据源不存在，则抛出非法异常
        if (this.writeDataSource == null) {
            throw new IllegalArgumentException("Property 'writeDataSource' is required");
        }
        //设置默认目标数据源为主库
        setDefaultTargetDataSource(writeDataSource);
        Map<Object, Object> targetDataSources = new HashMap<>();
        targetDataSources.put(DataSourceEnum.WRITE.name(), writeDataSource);
        if (readDataSource != null) {
            targetDataSources.put(DataSourceEnum.READ.name(), readDataSource);
        }
        setTargetDataSources(targetDataSources);
        super.afterPropertiesSet();
    }

    /**
     * 这是AbstractRoutingDataSource类中的一个抽象方法
     * 返回值是要用的数据源DataSourceEnum的key值
     */
    @Override
    protected Object determineCurrentLookupKey() {
        try {
            //根据当前线程所使用的数据源进行切换
            DataSourceEnum dataSourceEnum = DynamicDataSourceHolder.getDataSource();

            //如果没有被赋值，那么默认使用主库
            if (dataSourceEnum == null) {
                return DataSourceEnum.WRITE.name();
            }
            //使用指定的数据源
            return dataSourceEnum.name();
        } finally {
            DynamicDataSourceHolder.clearDataSource();
        }
    }

    public void setWriteDataSource(DataSource writeDataSource) {
        this.writeDataSource = writeDataSource;
    }

    public void setReadDataSource(DataSource readDataSource) {
        this.readDataSource = readDataSource;
    }
}
```

#### 实现动态数据源插件

```java
/**
 * 动态数据源插件，实现MyBatis拦截器接口
 */
@Intercepts({
        @Signature(type = Executor.class, method = "update",
                args = {MappedStatement.class, Object.class}),
        @Signature(type = Executor.class, method = "query",
                args = {MappedStatement.class, Object.class, RowBounds.class, ResultHandler.class})})
@Component
public class DynamicPlugin implements Interceptor {

    /**
     * 匹配SQL语句的正则表达式
     */
    private static final String REGEX = ".*insert\\u0020.*|.*delete\\u0020.*|.*update\\u0020.*";

    /**
     * 这个map用于存放已经执行过的sql语句所对应的数据源
     */
    private static final Map<String, DataSourceEnum> cacheMap = new ConcurrentHashMap<>();

    @Override
    public Object intercept(Invocation invocation) throws Throwable {
        //获取当前事务同步性进行判断
        boolean synchronizationActive = TransactionSynchronizationManager.isSynchronizationActive();
        //如果当前正在使用事务，则使用默认的库
        if (synchronizationActive) {
            return invocation.proceed();
        }

        // 如果已经指定数据源，则使用指定数据源
        // 可以自定义注解，用AOP的方式，指定数据源
        if (Objects.nonNull(DynamicDataSourceHolder.getDataSource())) {
            return invocation.proceed();
        }

        //从代理类参数中获取参数
        Object[] objects = invocation.getArgs();
        //其中参数的第一个值为执行的sql语句
        MappedStatement mappedStatement = (MappedStatement) objects[0];

        //当前sql语句所应该使用的数据源，通过sql语句的id从map中获取，如果获取到，则之前已经执行过直接取，
        DataSourceEnum dataSourceEnum = cacheMap.get(mappedStatement.getId());
        if (dataSourceEnum != null) {
            DynamicDataSourceHolder.putDataSource(dataSourceEnum);
            return invocation.proceed();
        }

        //如果没有，则重新进行存放
        //ms中获取方法，如果是查询方法
        if (mappedStatement.getSqlCommandType().equals(SqlCommandType.SELECT)) {
            //!selectKey 为自增id查询主键(SELECT LAST_INSERT_ID() )方法，使用主库
            if (mappedStatement.getId().contains(SelectKeyGenerator.SELECT_KEY_SUFFIX)) {
                dataSourceEnum = DataSourceEnum.WRITE;
            } else {
                BoundSql boundSql = mappedStatement.getSqlSource().getBoundSql(objects[1]);
                //通过正则表达式匹配，确定使用那一个数据源
                String sql = boundSql.getSql().replaceAll("[\\t\\n\\r]", " ");
                if (sql.matches(REGEX)) {
                    dataSourceEnum = DataSourceEnum.WRITE;
                } else {
                    dataSourceEnum = DataSourceEnum.READ;
                }
            }
        } else {
            dataSourceEnum = DataSourceEnum.WRITE;
        }
        //将sql对应使用的数据源放进map中存放
        cacheMap.put(mappedStatement.getId(), dataSourceEnum);

        //最后设置使用的数据源
        DynamicDataSourceHolder.putDataSource(dataSourceEnum);

        //执行代理之后的方法
        return invocation.proceed();
    }
}
```

#### Mybatis配置

```java
@Configuration
public class MybatisConfig {

    @Autowired
    @Qualifier("writerDataSource")
    private DataSource writerDataSource;
    
    @Autowired
    @Qualifier("readerDataSource")
    private DataSource readerDataSource;
    
    @Autowired
    private DynamicPlugin dynamicPlugin;

    @Bean(name = "dataSource")
    @ConditionalOnMissingBean
    public DynamicDataSource getDynamicDataSource() {
        DynamicDataSource dynamicDataSource = new DynamicDataSource();
        dynamicDataSource.setReadDataSource(readerDataSource);
        dynamicDataSource.setWriteDataSource(writerDataSource);
        return dynamicDataSource;
    }

    @Bean(name = "sqlSessionFactory")
    @ConditionalOnMissingBean
    public SqlSessionFactory getSqlSessionFactory() throws Exception {
        SqlSessionFactoryBean sqlSessionFactoryBean = new SqlSessionFactoryBean();
        sqlSessionFactoryBean.setPlugins(dynamicPlugin);
        sqlSessionFactoryBean.setDataSource(getDynamicDataSource());
        sqlSessionFactoryBean.setMapperLocations(
                new PathMatchingResourcePatternResolver().getResources("classpath*:/mapper/*Mapper.xml")
        );
        return sqlSessionFactoryBean.getObject();
    }
}
```

备注：

- 为方便增加扩展性，使用`@ConditionalOnMissingBean`注解的地方，可以自己定义

- 创建`SqlSessionFactory `：如使用`Mybatis-plus`创建`SqlSessionFactory`，需要使用`MybatisSqlSessionFactoryBean`；使用原生`Mybatis`和`Filuent-Mybatis`创建`SqlSessionFactoryBean`，需要使用`SqlSessionFactoryBean`

- 创建`SqlSessionFactory`时，`mapperLocations`参数为项目中`*Mapper.xml`文件的路径

- 上述创建`SqlSessionFactory`的方法，不会使用`Mybatis`在`yml`中的配置参数和默认的配置参数，可以通过下面的方式集成`yml`和默认的配置

  ```java
  	@Autowired
      private MybatisPlusAutoConfiguration mybatisAutoConfiguration;
  
  	@Bean(name = "sqlSessionFactory")
      public SqlSessionFactory getSqlSessionFactory() throws Exception {
          return mybatisAutoConfiguration.sqlSessionFactory(getDynamicDataSource());
      }
  ```
  
  

## [Mybatis](https://mybatis.org/mybatis-3/zh/index.html)

### 单表操作
#### Insert
`Method 1`：` XML`文件

```xml
<?xml version="1.0" encoding="UTF-8"?>
<!DOCTYPE mapper PUBLIC "-//mybatis.org//DTD Mapper 3.0//EN" "http://mybatis.org/dtd/mybatis-3-mapper.dtd">
<mapper namespace="com.sztus.mybatis.mapper.CustomerMapper">

    <!-- parameterType为入参的参数类型，开启useGeneratedKeys，返回主键id -->
    <insert id="insertCustomer" parameterType="com.sztus.mybatis.entity.Customer" useGeneratedKeys="true" keyProperty="id">
        insert into customer (status, created_at, updated_at, unique_code)
        values (#{status}, #{createdAt}, #{updatedAt}, #{uniqueCode})
    </insert>
    
</mapper>
```

```java
@Mapper
public interface CustomerMapper {
    int insertCustomer(Customer customer);
}
```

`Method 2`：` SQL`语句构造器 + `JAVA API`

`SQL`语句构造方法

```java
public class InformationSql {
    public String insertCustomerSql(Customer customer) {
        return new SQL()
                .INSERT_INTO("customer")
                .INTO_COLUMNS("status", "created_at", "updated_at", "unique_code")
                .INTO_VALUES(String.valueOf(customer.getStatus()),
                        String.valueOf(customer.getCreatedAt()),
                        String.valueOf(customer.getUpdatedAt()),
                        customer.getUniqueCode()).toString();

    }
}
```

`Mapper`通过注解的形式指向`SQL`语句

```java
@Mapper
public interface CustomerMapper {
    @InsertProvider(type = InformationSql.class, method = "insertCustomerSql")
    @SelectKey(statement = "select LAST_INSERT_ID()", keyProperty = "id", before = false, resultType = Long.class)
    int insertCustomerBySql(Customer customer);
}
```

`Service`调用`Mapper`

```java
@Service
public class InformationService {

    @Autowired
    private CustomerMapper customerMapper;

    public int insertCustomer(Customer customer) {
        return customerMapper.insertCustomerBySql(customer);
    }

}
```

`Method 3`： 注解式

```java
    @Insert("insert into customer (status, created_at, updated_at, unique_code)\n" +
            "        values (#{status}, #{createdAt}, #{updatedAt}, #{uniqueCode})")
	// 返回主键id
	@SelectKey(statement = "select LAST_INSERT_ID()", keyProperty = "id", before = false, resultType = Long.class)
    int insertCustomerByAnnotation(Customer customer);
```

**注意**：`Mybatis`提供3种新增数据的方式，返回结果是影响的行数，如果想要获取新增数据的主键id，需要加上对应的参数或者注解，并从对象中获取主键id

#### Delete

`Method 1`：` XML`文件

```xml
<?xml version="1.0" encoding="UTF-8"?>
<!DOCTYPE mapper PUBLIC "-//mybatis.org//DTD Mapper 3.0//EN" "http://mybatis.org/dtd/mybatis-3-mapper.dtd">
<mapper namespace="com.sztus.mybatis.mapper.CustomerMapper">

    <delete id="deleteCustomerById">
        DELETE FROM customer WHERE id = #{id}
    </delete>
    
</mapper>
```

```java
@Mapper
public interface CustomerMapper {

    int deleteCustomerById(Long customerId);
}
```

`Method2`：` SQL`语句构造器 + `JAVA API`

```java
public String deleteCustomerSql(Long customerId) {
        return new SQL()
                .DELETE_FROM("customer")
                .WHERE("id = " + customerId)
                .toString();
}

@Mapper
public interface CustomerMapper {
    
    @InsertProvider(type = InformationSql.class, method = "deleteCustomerSql")
    int deleteCustomerBySql(Long customerId);
}
```

`Method 3`： 注解式

```java
@Mapper
public interface CustomerMapper {

    @Delete("DELETE FROM customer WHERE id = #{id}")
    int deleteCustomerByAnnotation(Long customerId);
}
```

#### Update

`Method 1`：` XML`文件

```xml
<!-- 单条数据更新 -->
<update id="updateCustomer" parameterType="com.sztus.mybatis.entity.Customer">
    UPDATE customer SET unique_code = #{customer.uniqueCode} WHERE id = #{customer.id}
</update>

<!-- 批量数据更新 -->
<update id="batchUpdateCustomer">
    <foreach collection="list" item="item" open="" close="" separator=";">
        update customer
        <set>
            <if test="item.status != null">status = #{item.status},</if>
            <if test="item.uniqueCode != null">unique_code = #{item.uniqueCode},</if>
            <if test="item.createdAt != null">created_at = #{createdAt},</if>
            <if test="item.updatedAt != null">updated_at = #{item.updatedAt},</if>
        </set>
        where id = #{item.id}
    </foreach>
</update>
```

`Method 2`：` SQL`语句构造器 + `JAVA API`

```java

public String updateCustomerSql(Customer customer) {
    return new SQL()
        .UPDATE("customer")
        .SET("unique_code = " + customer.getUniqueCode())
        .WHERE("id = " + customer.getId())
        .toString();

}

@Mapper
public interface CustomerMapper {
    @Update("UPDATE customer SET unique_code = #{customer.uniqueCode} WHERE id = #{customer.id}")
    int updateCustomerByAnnotation(Customer customer);
}
```

`Method 3`：注解式

```java
@Mapper
public interface CustomerMapper {
    @Update("UPDATE customer SET unique_code = #{customer.uniqueCode} WHERE id = #{customer.id}")
    int updateCustomerByAnnotation(Customer customer);
}
```

#### Query

`Method 1`：` XML`文件

```xml
    <select id="getCustomer" resultType="com.sztus.mybatis.entity.Customer">
        SELECT * FROM customer WHERE unique_code = #{uniqueCode}
    </select>
```

`Method 2`：` SQL`语句构造器 + `JAVA API`

```java
public String getCustomerSql(String uniqueCode) {
    return new SQL()
        .SELECT()
        .FROM("customer")
        .WHERE("unique_code = " + uniqueCode)
        .toString();
}

@Mapper
public interface CustomerMapper {
    @SelectProvider(type = InformationSql.class, method = "getCustomerSql")
    Customer getCustomerBySql(String uniqueCode);
}
```

`Method 3`：注解式

```java
@Mapper
public interface CustomerMapper {
    @Select("SELECT * FROM customer WHERE unique_code = #{uniqueCode}")
    Customer getCustomerByAnnotation(String uniqueCode);
}
```


### 多表操作（动态SQL）
#### XML方式进行多表的CRUD
**动态**`SQL`

```xml
<resultMap id="CustomerAccountMap" type="com.jiafei.object.view.CustomerAccountView">
        <id column="id" property="id"/>
        <result column="customer_id" property="customerId"></result>
        <result column="account_status" property="accountStatus"></result>
        <result column="identifier" property="identifier"></result>
        <result column="portfolio_id" property="portfolioId"></result>
        <result column="open_id" property="openId"></result>
        <result column="customer_no" property="customerNo"></result>
</resultMap>

<select id="listCustomerAccount" resultMap="CustomerAccountMap">
        select * from customer_account_info
        where account_status = #{status}
        and customer_id in
        <foreach collection="customerIds" item="customerId" index="index" separator="," open="(" close=")">
            #{customerId}
        </foreach>
</select>

<select id="getLoan" resultType="com.sztus.mybatis.object.view.GetLoanResult">
    SELECT t1.loan_no, t1.id, t1.account_id, t2.email, t2.zip FROM loan t1
    LEFT JOIn loan_personal_data t2 ON t1.id = t2.loan_id
    WHERE t2.email = #{email}
    <if test="type != null and type != ''">
        AND t1.type = #{type}
    </if>
</select>

<!-- 使用 set+if 标签修改后，如果某项为 null 则不进行更新，而是保持数据库原值 -->
<update id="updateStudent" parameterType="com.sztus.mybatis.entity.Customer">
    UPDATE customer
    <set>
        <if test="status != null and status != ''">
            status = #{customer.status},
        </if>
        <if test="createdAt != null and createdAt != ''">
            created_at = #{customer.createdAt},
        </if>
        <if test="uniqueCode != null and uniqueCode != ''">
            unique_code = #{customer.uniqueCode}
        </if>
    </set>
    WHERE id = #{customer.id};
</update>

<select id="getStudentListWhere" resultType="com.sztus.mybatis.entity.Customer">
    SELECT * from customer
    <where>
        <if test="uniqueCode != null and uniqueCode != ''">
            unique_code = #{uniqueCode}
        </if>
        <if test="id != null and id != ''">
            AND id = #{id}
        </if>
    </where>
</select>

<!-- 当 uniqueCode 值为 null 时，查询语句会出现 “WHERE AND” 的情况，解决该情况除了将"WHERE"改为“WHERE 1=1”之外，还可以利用 where标签。这个“where”标签会知道如果它包含的标签中有返回值的话，它就插入一个‘where’。此外，如果标签返回的内容是以 AND 或 OR 开头的，则它会剔除掉。 -->
<select id="getStudentListWhere" resultType="com.sztus.mybatis.entity.Customer">
    SELECT * from customer
    <where>
        <if test="uniqueCode != null and uniqueCode != ''">
            unique_code = #{uniqueCode}
        </if>
        <if test="id != null and id != ''">
            AND id = #{id}
        </if>
    </where>
</select>
```

**批量插入**
```xml
 <update id="batchUpdateCustomer" parameterType="list">
        <foreach collection="list" item="item" open="" close="" separator=";">
            update customer
            <set>
                <if test="item.status != null">status = #{item.status},</if>
                <if test="item.uniqueCode != null">unique_code = #{item.uniqueCode},</if>
                <if test="item.createdAt != null">created_at = #{createdAt},</if>
                <if test="item.updatedAt != null">updated_at = #{item.updatedAt},</if>
            </set>
            where id = #{item.id}
        </foreach>
</update>

<!-- 
	 separator：分隔符，表示迭代时每个元素之间以什么分隔
  	 open：前缀
	 close：后缀
-->
<insert id="insertBatch" parameterType="java.util.List">
    INSERT INTO customer (status, created_at, updated_at, unique_code) VALUES
    <foreach collection="list" item="item" index="index" separator="," open="(" close=")">
        #{item.status}, #{item.createdAt}, #{item.updatedAt}, #{item.uniqueCode}
    </foreach>
</insert>
```

#### 使用SQL语句构造器进行多表CRUD

不推荐，过于复杂

**关联查询**
```java

```

**批量插入**

```java

```

### 事物支持

需要替换spring事务管理器的数据源

```java
    @Bean
    public DataSourceTransactionManager transactionManager() {
        DataSourceTransactionManager dataSourceTransactionManager = new DataSourceTransactionManager();
        // 替换为自定义的动态数据源
        dataSourceTransactionManager.setDataSource(getDynamicDataSource());
        return dataSourceTransactionManager;
    }
```

在需要加事务的方法，加上注解@Transactional

```java
    @Transactional(rollbackFor = Exception.class)
    public void insertCustomer(Customer customer) {
        customerMapper.insertCustomerBySql(customer);
        Long idA = customer.getId();
        /**
         *  程序运行带这一步，事务还未提交，数据库无法查出idA的数据
         */
        System.out.println(idA);
        customerMapper.insertCustomerBySql(customer);
        Long idB = customer.getId();
        System.out.println(idB);
    }
```



## [Mybatis Plus](https://baomidou.com/)

`MyBatis-Plus`是一个`MyBatis`的增强工具，在`MyBatis`的基础上只做增强不做改变，为简化开发、提高效率而生。

### 使用CRUD接口进行单表操作
#### Insert
- 单条插入

  ```java
  @Mapper
  public interface UserMapper extends BaseMapper<User> {
  }
  
  @Service
  public class UserService {
      
      @Autowired
      private UserMapper userMapper;
      
      public void insertUser(User user) {
          userMapper.insert(user);
      }
  }
  ```
  
  `Mybatis-plus`默认存储完数据后,自动向传进来的实体类塞入主键`ID`，所以理论上无需做多余的配置

- 批量插入 

  ```java
  @Service
  public class UserService {
  
      @Autowired
      private UserMapper userMapper;
  
      public void insertBatch(List<User> list) {
          userMapper.insertBatch(list);
      }
  }
  ```


#### Delete
- 单条删除

  ```java
  @Service
  public class UserService {
  
      @Autowired
      private UserMapper userMapper;
  
      // row为影响行数
      public void deleteUserById(Long id) {
          // 1、通过指定条件删除
          Map<String, Object> condition = new HashMap<>();
          condition.put("id", id);
          int affectRowA = userMapper.deleteByMap(condition);
          // 2、通过主键删除
          int affectRowB = userMapper.deleteById(id);
          // 3、通过实体类删除
          User user = new User();
          user.setId(id);
          int affectRowC = userMapper.deleteById(user);
          // 4、通过QueryWrapper删除
          QueryWrapper<User> userQueryWrapper = new QueryWrapper<>();
          // 此处可以使用 userQueryWrapper.allEq(condition);
          userQueryWrapper.eq("id", id);
          int affectRowD = userMapper.delete(userQueryWrapper);
      }
  }
  ```

- 批量删除

  ```java
  @Service
  public class UserService {
  
      @Autowired
      private UserMapper userMapper;
  
      // row为影响行数
      public void deleteUserBatch(List<User> list) {
          // 1、通过主键id去删除
          List<Long> userIdList = list.parallelStream()
                  .map(User::getId)
                  .collect(Collectors.toList());
          int affectRowA = userMapper.deleteBatchIds(userIdList);
          // 2、通过QueryWrapper批量删除
          QueryWrapper<User> userQueryWrapper = new QueryWrapper<>();
          userQueryWrapper.in("id", userIdList);
          userMapper.delete(userQueryWrapper);
      }
  }
  ```
  
  备注：还可以通过集成`ServiceImpl`类或者调用`Db`类的静态方法（`3.5.3`版本之后），调用如下图所示的方法删除你想删除的数据。
  
  ![image-20230215104352919](C:\Users\Jerry\AppData\Roaming\Typora\typora-user-images\image-20230215104352919.png)

#### Update

```java
@Service
public class UserService extends ServiceImpl<UserMapper, User> {

    @Autowired
    private UserMapper userMapper;

    public void updateUser(User user) {
        // 1、通过id更新，单条数据的更新
        int affectRowA = userMapper.updateById(user);
        // 2、通过条件更新，批量数据的更新
        UpdateWrapper<User> userUpdateWrapper = new UpdateWrapper<>();
        userUpdateWrapper.eq("id", user.getId());
        int affectRowB = userMapper.update(user, userUpdateWrapper);
        // 3、更新或者插入新的数据
        this.saveOrUpdate(user);
    }
}
```



#### Query

```java
@Service
public class UserService extends ServiceImpl<UserMapper, User> {

    @Autowired
    private UserMapper userMapper;

    public void select(Long id, String openId, String email) {
        // 1、通过主键id查询
        User userA = userMapper.selectById(id);

        // 2、通过条件查询单挑数据
        QueryChainWrapper<User> queryChainWrapper = query().eq("open_id", openId);
        User userB = userMapper.selectOne(queryChainWrapper);

        // 3、通过Id批量查询
        List<Long> idList = new ArrayList<>();
        idList.add(id);
        List<User> userListA = userMapper.selectBatchIds(idList);

        QueryWrapper<User> userQueryWrapper = new QueryWrapper<>();
        userQueryWrapper.eq("email", email);

        // 4、查询满足条件的数量
        Long count = userMapper.selectCount(userQueryWrapper);

        // 5、批量查询email
        List<User> userListB = userMapper.selectList(userQueryWrapper);

        // 6、selectObjs只返回第一个字段的值
        // 这里虽然知道了返回两列数据，但是只会返回name字段
        userQueryWrapper.select("name", "age");
        List<Object> list = userMapper.selectObjs(userQueryWrapper);
    }
}
```



### 其他
- 条件构造器

  `Mybatis-Plus`具有丰富的条件构造器，满足各种常见的需求，语法简单易懂，并且支持链式调用。

  ```java
  		Map<String, Object> condition = new HashMap<>();
          condition.put("id", "123");
          condition.put("open_id", "0a4b2fd66e95482cba3071ff4fa0db7b");
  
          QueryWrapper<User> queryWrapper = new QueryWrapper<>();
          queryWrapper
                  // 要查询字段
                  .select("id", "name", "email", "open_id")
                  // 等于，默认用AND链接，id = 123 AND open_id = '0a4b2fd66e95482cba3071ff4fa0db7b'
                  .eq("id", 123).eq("open_id", "0a4b2fd66e95482cba3071ff4fa0db7b")
                  // 全部eq，与上述条件一致，id = 123 AND open_id = '0a4b2fd66e95482cba3071ff4fa0db7b'
                  .allEq(condition)
                  // 不等于， name != 'XXX'
                  .ne("name", "XXX")
                  // 大于，age > 18，大于等于用ge()方法
                  .gt("age", 18)
                  // 小于等于，age <= 50
                  .le("age", 50)
                  // age BETWEEN 18 AND 50
                  .between("age", 18, 50)
                  // 模糊查询，email LIKE %@sztus%
                  // 左右模糊查询调用likeLeft()和likeRight()方法，不匹配用notLikeLeft()和notLikeRight()
                  .like("email", "@sztus")
                  // 条件为NULL，address IS NULL
                  // 不为空用isNotNull
                  .isNull("address")
                  // city IN ('chengdu', 'chongqing')，也可以传List，不在集合中用notIn
                  .in("city", "chengdu", "chongqing")
                  // 或者，下面的条件表示 OR (email = 'test@sztus.com')
                  .or(i -> i.eq("email", "test@sztus.com"))
                  // GROUP BY name, age，排序用orderBy()，orderByDesc()，orderByAsc()
                  .groupBy("name", "age")
                  // 根据判断结果，动态拼接条件
                  .func(
                      i -> {
                          if(true) {
                              i.eq("id", 1);
                          } else {
                              i.ne("id", 1);
                          }
                      }
                  )
                  // 无视优化规则直接拼接到 sql 的最后
                  .last("LIMIT 1");
  ```

  

- 分页插件

  方案一（**不推荐**）：使用`Mybatis-Plus`自带的分页方法

  ```java
      public Page<User> select() {
          QueryWrapper<User> queryWrapper = new QueryWrapper<>();
          queryWrapper.eq("email", "test@sztus.com");
  
          Page<User> userPage = new Page<>(1, 10);
          userMapper.selectPage(userPage, queryWrapper);
          return userPage;
      }
  ```

  该方法先查出所有符合条件的数据，拿到结果集后，通过自带的方法，处理得到具体某一页的数据。在数据量较大的情况下，会严重影响性能。

  方案二（推荐）：通过条件构造器的`last()`方法，在条件最后加上`LIMIT`，拼接分页参数

  ```java
      public List<User> select() {
          QueryWrapper<User> queryWrapper = new QueryWrapper<>();
          queryWrapper.eq("email", "test@sztus.com");
          queryWrapper.last("LIMIT 1, 10");
  
          List<User> list = userMapper.selectList(queryWrapper);
          return list;
      }
  ```

  

## [Fluent Mybatis](https://gitee.com/OAGroup/fluent-mybatis)

基于`mybatis`但青出于蓝

- `No XML`, `No Mapper`, ` No If else`, `No String`魔法值编码
- 只需`Entity`就实现强大的`FluentAPI`: 支持分页, 嵌套查询,` AND OR`组合, 聚合函数...

### 使用逆向工程对服务进行构建

#### 逆向工程生成对应实体

```java
public class Generator {

    static final String url = "jdbc:mysql:xxxxxxxxx";

    public static void main(String[] args) {
        FileGenerator.build(generator.class);
    }

    @Tables(
            /** 数据库连接信息 **/
            url = url, username = "xxxxxxxxx", password = "xxxxxxxxx",
            /** 包名 **/
            basePack = "com.example.demo",
            /** 生成Entity代码目录 如果为聚合项目记得加上子包名 **/
            srcDir = "/demo4/src/main/java",
            /** 生成Dao代码目录 **/
            daoDir = "/demo4/src/main/java",
            /** 定义创建，修改，逻辑删除字段 **/
            gmtCreated = "create", gmtModified = "modified", logicDeleted = "is_del",
            /** 需要生成文件的表, 这里可以配置多个表 **/
            tables = @Table(value = {"customer","customer_account_info"})
    )
    static class generator {
    }
}
```

```java
@Data
@Accessors(
    chain = true
)
@EqualsAndHashCode(
    callSuper = false
)
@FluentMybatis(
    table = "customer",
    schema = "customer"
)
public class CustomerEntity extends RichEntity {
  private static final long serialVersionUID = 1L;

  @TableId("id")
  private Long id;

  @TableField("category")
  private Integer category;

  @TableField("created_at")
  private Long createdAt;

  @TableField("source")
  private Integer source;

  @TableField("status")
  private Integer status;

  @TableField("unique_code")
  private String uniqueCode;

  @TableField("updated_at")
  private Long updatedAt;

  @Override
  public final Class entityClass() {
    return CustomerEntity.class;
  }
}

```

<img src="https://static.dingtalk.com/media/lALPJxRxVw0JYJ7NAjLNAdk_473_562.png_720x720q90g.jpg?bizType=im" alt="img" style="zoom: 80%;" />

注：也可不用逆向工程，但需要在对应`Entity`写好注解，以用于在`target`中生成`mapper`和`wrapper`文件

#### build项目以生成对应mapper和wrapper文件

<img src="https://static.dingtalk.com/media/lALPJxf-1bHRncjNAb3NAeY_486_445.png_720x720q90g.jpg?bizType=im" alt="img" style="zoom:80%;" />

### 使用API进行单表操作

#### Insert

- 单条插入

```java
    void insertTest() {
        long currentTimeMillis = System.currentTimeMillis();
        CustomerEntity customerEntity = new CustomerEntity()
                .setStatus(1)
                .setSource(1)
                .setUniqueCode(UUID.randomUUID().toString().toUpperCase(Locale.ROOT))
                .setCreatedAt(currentTimeMillis)
                .setUpdatedAt(currentTimeMillis);
        //使用save()方法进行插入数据
        int save = customerMapper.save(customerEntity);
        // int save = customerMapper.insertWithPk(customerEntity);
        // int save = customerMapper.insert(customerEntity);
        // Boolean save = customerMapper.saveOrUpdate(customerEntity);
    }
```

*注:`insert()`,` insertWithPk()`, `save()`三种方法用于插入数据，`saveOrUpdate()`方法用于通过判断有无主键来进行插入或保存修改

当使用insert()方法进行插入数据时，实体类中不能有主键数据，使用`insertWithPk()`方法时必须有主键数据，否则会抛异常。

save()方法是insert和`insertWithPk`的简便方法，自动判断Entity是否已经有主键值，但save()不能进行数据修改，会报主键冲突异常

- 批量插入

```java
    void insertBatchTest(List<CustomerEntity> customerList){
        int result = customerMapper.save(customerList);
        // int result = customerMapper.insertBatch(customerList);
        // int result = customerMapper.insertBatchWithPk(customerList);
    }
```

注意：当使用`insertBatch()`方法进行批量插入数据时，实体集合中不能有主键数据，使用`insertBatchWithPk()`方法时必须有主键数据，否则会抛异常。`save(Collection<E> list)`方法是`insertBatch()`和`insertBatchWithPk()`的简便方法，自动判断数据集合是否已经有主键值，但应确保集合中实体的主键数据全有或全无，否则抛出异常。`save(Collection<E> list)`不能进行数据修改，会报主键冲突异常

#### Delete

```java
    void deleteTest(){
        //通过主键删除
        customerMapper.deleteById(2028);
    }
```

```java
    void deleteTest(){
        //通过主键集合删除
        customerMapper.deleteByIds(Arrays.asList(2027,2028));
    }
```

```java
    void deleteTest(){
    	//根据Map<String,Object>设置是键值对删除数据
        customerMapper.deleteByMap(true, new HashMap<String, Object>() {
            {
                this.put("id", 2027);
            }
        });
    }
```

```java
    void deleteTest(){
	    //根据自定义条件物理删除数据
        CustomerQuery update = new CustomerQuery()
                .where.id().in(Arrays.asList(2027,2028)).end();
        customerMapper.delete(update);
        // DELETE FROM customer WHERE id IN (2027,2028)
    }
```

#### Update

```java
    void updateTest() {
    	//按Entity的主键更新Entity中非空字段
        long currentTimeMillis = System.currentTimeMillis();
        CustomerEntity customerEntity = new CustomerEntity()
                .setId(1102L)
                .setStatus(0)
                .setSource(0)
                .setUniqueCode(UUID.randomUUID().toString().toUpperCase(Locale.ROOT))
                .setCreatedAt(currentTimeMillis)
                .setUpdatedAt(currentTimeMillis);
        customerMapper.updateById(customerEntity);
    }	
```

```java
    void updateTest() {
        //自定义更改指定字段数据与筛选条件(会写入空值到库中)
        CustomerUpdate update = new CustomerUpdate()
                .set.status().is(0).updatedAt().is(System.currentTimeMillis()).end() //set 指定修改的字段与数据
                .where.id().eq(1102).end(); // where 指定筛选条件
        // UPDATE customer SET `status` = 0, updated_at = 1676516452634 WHERE id = 1102
        customerMapper.updateBy(update);
    }
```

```java
    void updateTest() throws Exception {
        //批量修改数据
        IUpdate[] updates = this.assembleCustomerUpdate(this.getCustomerList()).toArray(new IUpdate[0]);
        int count = customerMapper.updateBy(updates);
        System.out.println(count);
    }

    private List<IUpdate> assembleCustomerUpdate(List<CustomerEntity> customerEntities) throws Exception {
      //组装批量修改的updater
        ArrayList<IUpdate> result = new ArrayList<>();
        for (CustomerEntity customerEntity : customerEntities) {
            if (Objects.isNull(customerEntity.getId())){
                throw new Exception("PRIMARY Key Can Not Be Null");
            }
            // 将每个实体都组装成通过主键修改的修改器
            IUpdate update = SqlKitFactory.factory(customerMapper).updateById(customerMapper.mapping(), customerEntity);
            result.add(update);
        }
        return result;
    }
    private List<CustomerEntity> getCustomerList() {
        long currentTimeMillis = System.currentTimeMillis();
        CustomerEntity customerEntity1 = new CustomerEntity()
                .setId(2027l)
                .setStatus(0)
                .setSource(0)
                .setUniqueCode(UUID.randomUUID().toString().toUpperCase(Locale.ROOT))
                .setCreatedAt(currentTimeMillis)
                .setUpdatedAt(currentTimeMillis);

        CustomerEntity customerEntity2 = new CustomerEntity()
                .setId(2028l)
                .setStatus(0)
                .setSource(0)
                .setUniqueCode(UUID.randomUUID().toString().toUpperCase(Locale.ROOT))
                .setCreatedAt(currentTimeMillis)
                .setUpdatedAt(currentTimeMillis);
        List<CustomerEntity> list = new ArrayList() {{
            add(customerEntity1);
            add(customerEntity2);
        }};
        return list;
    }
```

#### Query

```java
    void queryTest() {
        //根据主键查询数据
        CustomerEntity customer = customerMapper.findById(2028);
         // SELECT * FROM customer WHERE id = 2028
    }
```

```java
    void queryTest() {
    	//根据自定义条件查询一条记录
        CustomerQuery query = new CustomerQuery()
                .where.id().eq(2028).source().ne(1).end();
        // SELECT * FROM customer WHERE id = 2028 and  source != 1
        CustomerEntity customer = customerMapper.findOne(query);
    }
```

```java
    void queryTest() {
        //根据主键列表查询数据集
        List<CustomerEntity> customer = customerMapper.listByIds(Arrays.asList(482165,482166));
       // SELECT * FROM customer WHERE id in (482165,482166)
    }
```

```java
    void queryTest() {
        //根据Map<Strng,Object>键值对列表查询数据集
        List<CustomerEntity> customerEntities = customerMapper.listByMap(true, new HashMap<String, Object>() {
            {
                this.put(CustomerMapping.id.column, 2020);
            }
        });
        // SELECT * FROM customer WHERE id = 2020
    }
```

```java
    void queryTest() {
    	//根据自定义条件查询Entity数据集
        CustomerQuery query = new CustomerQuery()
                .where.id().in(Arrays.asList(2027,2028)).end();
        // SELECT * FROM customer WHERE id IN (2027,2028)
        List<CustomerEntity> customers = customerMapper.listEntity(query);
    }
```

```java
    void queryTest() {
        //只查询并展示指定字段(id,uniqueCode)
        CustomerQuery query = new CustomerQuery()
                .select.id().uniqueCode().end()
                .where.id().in(Arrays.asList(2027,2028)).end();
        // SELECT id, unique_code FROM customer WHERE id IN (2027,2028)
        List<CustomerEntity> customers = customerMapper.listEntity(query);
    }
```

```java
    void queryTest() {
	    //只查询并展示指定字段(id,以“sta”开头的字段)
        CustomerQuery query = new CustomerQuery()
                .selectId()
                .select.apply(f->f.getProperty().startsWith("sta")).end()
                .where.id().in(Arrays.asList(2027,2028)).end();
        // SELECT id, `status` FROM customer WHERE id IN (2027,2028)
        List<CustomerEntity> customers = customerMapper.listEntity(query);   
    }
```

### 使用API进行多表操作

```java
    void queryTest() {
        //联表查询 最终结果只映射对应调用实体数据(如下只返回Customer对应的数据)
        CustomerQuery leftQuery = new CustomerQuery("a1").selectAll()
                .where.id().in(Arrays.asList(238,283))
                .end();
        CustomerAccountInfoQuery rightQuery = new CustomerAccountInfoQuery("a2")
                .where.portfolioId().eq(60)
                .end();

        IQuery query = leftQuery
                .join(rightQuery)
                .on(l -> l.where.id(), r -> r.where.customerId()).endJoin()
                .build();

        // SELECT * FROM customer a1 LEFT JOIN customer_account_info a2 on a1.id = a2.customer_id 
        // WHERE a1.id in(238,283) and a2.portfolio_id = 60
        List<Map<String, Object>> maps = customerMapper.listMaps(query);
```

```java
    void queryTest() {
        //自定义sql查询
        FreeQuery query = new FreeQuery(null).customizedByQuestion(
                        "SELECT * FROM customer a1 LEFT JOIN customer_account_info a2 on a1.id = a2.customer_id " +
                        "WHERE a1.id = ? and a2.portfolio_id = ?",
                283, 60);
        List<Map<String, Object>> maps = customerMapper.listMaps(query);
    }
```


## 性能对比
使用原生`Mybatis`和`Mybatis-plus`、`Fluent-Mybatis`封装好的方法，对数据库新增一批数据（备注：表字段`20`个，存在大字段，且需要维护索引），在数据量较少情况下，三种方式没有太多的差别，当数据量达到`100`条以上时，原生`Mybatis`通过`foreach`标签组装`Sql`的方式更快，数据量即使达到`5000`条，依然能在`1`秒内完成；而Fluent-Mybatis批量插入数据和Mybatis-Plus批量插入数据，需要花费较长的时间，数据量达到`300`条时，就会花费`1`秒左右，数据据量越大，花费的时间成倍数增长。

```java
    <insert id="insertBatch" parameterType="java.util.List">
        INSERT INTO user (name, age, gender, address,
        month_income, interest, city, open_id,
        password, salt, height, weight,
        careers, telephone, nationality, account_balance,
        email, key_name, ip, level ) VALUES
        <foreach collection="list" item="item" index="index" separator="," >
            (#{item.name}, #{item.age}, #{item.gender}, #{item.address},
            #{item.monthIncome}, #{item.interest}, #{item.city}, #{item.openId},
            #{item.password}, #{item.salt}, #{item.height}, #{item.weight},
            #{item.careers}, #{item.telephone}, #{item.nationality}, #{item.accountBalance},
            #{item.email}, #{item.keyName}, #{item.ip}, #{item.level})
        </foreach>
    </insert>
```



| 数据量（条） | 原生xml方式批量插入 | Fluent-Mybatis批量插入 | Mybatis-Plus批量插入 |
| :----------: | :-----------------: | :--------------------: | :------------------: |
|      5       |          9          |           20           |          24          |
|      10      |         17          |           40           |          37          |
|      30      |         21          |           92           |         123          |
|      50      |         19          |          192           |         186          |
|      80      |         21          |          288           |         294          |
|     100      |         27          |          366           |         347          |
|     300      |         69          |          1026          |         1006         |
|     600      |         123         |          2200          |         2105         |
|     800      |         134         |          2945          |         2771         |
|     1000     |         174         |          3826          |         3244         |
|     2000     |         351         |          7573          |         6857         |
|     3000     |         456         |         10567          |        10557         |
|     5000     |         752         |         16791          |        16986         |
|     6000     |         950         |         20291          |        27171         |




