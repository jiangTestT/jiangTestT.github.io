# 布隆过滤器


## 布隆过滤器

### 什么是布隆过滤器

布隆过滤器（Bloom Filter）是1970年由布隆提出的，它由**二进制向量和一系列随机映射函数（哈希函数）**两部分组成。

优点：

- 可以高效地进行查询，可以用来告诉你“某样东西一定不存在或者可能存在”
- 可以高效的进行插入
- 保密性强，布隆过滤器不存储元素本身
- 相比于传统的List、Set、Map等数据结构，它占用空间更少，因为其本身并不存储任何数据

缺点：

- 其返回的结果是概率性（存在误差）的，布隆过滤器告诉存在的结果，实际上并不一定存在
- 布隆过滤器很难删除数据
- 无法对布隆过滤器扩容，使用布隆过滤器最好是预估大小。如果真要扩容，最好是扩容后把所有需要存放的元素，再重新通过多个哈希函数放入布隆过滤器中，简单的来说就是新建一个布隆过滤器，然后把元素重新遍历到新布隆过滤器

总结：布隆过滤器是一种来检索元素是否在给定大集合中的数据结构，这种数据结构是高效且性能很好的，但缺点是具有一定的错误识别率和删除难度。并且，理论情况下，添加到集合中的元素越多，误报的可能性就越大。

### 使用场景

#### 数据去重

场景描述： 在一些需要对大量数据进行去重的场景，例如用户提交表单、数据同步等，布隆过滤器可以迅速判断某个数据是否已存在，避免重复插入。
应用实例： 在用户提交表单时，使用布隆过滤器判断该用户是否已经提交过相同的数据，从而防止重复提交。

#### 缓存穿透问题

场景描述：当缓存中不存在某个数据，而用户频繁查询该数据时，可能导致缓存穿透问题。布隆过滤器可以在缓存层之前迅速过滤掉不存在的数据，减轻数据库的压力。
应用实例： 在缓存中存储热门商品的 ID 列表，并使用布隆过滤器判断某个商品 ID 是否存在于列表中，从而决定是否查询数据库获取数据。

#### 爬虫数据去重

场景描述： 在爬虫应用中，避免重复抓取相同的数据是一项关键任务。布隆过滤器可以帮助爬虫快速判断某个 URL 是否已经被抓取过。
应用实例： 在爬虫系统中，使用布隆过滤器存储已抓取的URL，以避免重复请求同一 URL。

#### 安全黑名单

场景描述： 在需要防范恶意攻击或恶意请求的场景中，布隆过滤器可以用于快速判断某个 IP 地址或请求是否在黑名单中。
应用实例： 在 Web 应用中，使用布隆过滤器维护一份 IP 黑名单，快速拦截恶意请求。

#### URL访问记录

场景描述： 对于某些需要记录用户访问记录的应用，布隆过滤器可以用于判断某个 URL 是否已经被记录，避免重复记录。
应用实例： 在网站访问日志记录中，使用布隆过滤器判断某个 URL 是否已经被记录，防止访问记录过于庞大。

#### 缓存预热

场景描述： 在系统启动时，通过布隆过滤器判断某些热门数据是否在缓存中，可以加速系统的启动过程。
应用实例： 在 SpringBoot 应用启动时，使用布隆过滤器判断热门商品的 ID 是否在缓存中，并提前加载到缓存中，减少冷启动时的缓存穿透问题。

### 布隆过滤器的原理

#### 数据结构

- 二进制数组

![](https://crry-assist.oss-cn-chengdu.aliyuncs.com/Screenshot/%E5%B8%83%E9%9A%86%E8%BF%87%E6%BB%A4%E5%99%A8%E7%9A%84%E4%BA%8C%E8%BF%9B%E5%88%B6%E6%95%B0%E7%BB%84%E9%9B%86.png)

- 多个无偏 hash 函数：无偏 hash 函数就是能把元素的 hash 值计算的比较均匀的 hash 函数，能使得计算后的元素下标比较均匀的映射到位数组中。

#### 添加元素

往布隆过滤器增加元素，添加的 key 需要根据 k 个无偏 hash 函数计算得到多个 hash 值，然后对数组长度进行**取模**得到数组下标的位置，然后将对应数组下标的位置的值置为 1

- 通过 k 个无偏 hash 函数计算得到 k 个 hash 值
- 依次取模数组长度，得到数组索引
- 将计算得到的数组索引下标位置数据修改为1

例如，key = Jerry，无偏 hash 函数的个数 k=3，分别为 hash1、hash2、hash3。三个 hash 函数计算后得到三个数组下标值，并将其值修改为 1.
如图所示：

![](https://crry-assist.oss-cn-chengdu.aliyuncs.com/Screenshot/%E5%B8%83%E9%9A%86%E8%BF%87%E6%BB%A4%E5%99%A8-hash%E6%98%A0%E5%B0%84%E6%95%B0%E6%8D%AE.png)

#### 查询元素

布隆过滤器最大的用处就在于判断某样东西一定不存在或者可能存在，而这个就是查询元素的结果。其查询元素的过程如下：

- 通过 k 个无偏 hash 函数计算得到 k 个 hash 值
- 依次取模数组长度，得到数组索引
- 判断索引处的值是否全部为 1，如果全部为 1 则存在（这种存在可能是误判），如果存在一个 0 则必定不存在

#### 为什么布隆过滤器会误判

布隆过滤器是通过使用多个哈希函数将集合中的每个元素映射到一个位数组的不同位置上。当查询一个元素是否存在时，该元素会通过相同的哈希函数计算出对应的位数组位置，如果这些位置上的值都是 1，则认为该元素可能存在于集合中；如果有任何一个位置上的值为 0，则可以确定该元素不在集合中。

布隆过滤器之所以有误判率（即假阳性），主要原因是它的设计原理导致的：

- 哈希冲突：由于布隆过滤器使用的是有限大小的位数组，而哈希函数可能会将不同的输入映射到同一个输出位置上，这称为哈希冲突。随着插入到布隆过滤器中的元素数量增加，发生哈希冲突的概率也会增加，从而增加了误判的可能性。
- 不可逆性：一旦一个元素被添加到布隆过滤器中，相应的位就会被设置为1，并且这些位在后续操作中不会被重置为 0（除非进行整个过滤器的重置）。这意味着即使后来发现某些位是因为其他元素的存在而被错误地标记为1，也没有办法纠正这种状态，这进一步增加了误判的可能性。
- 空间限制：为了减少误判率，可以增加位数组的大小或使用的哈希函数的数量。但是，这样做会增加存储成本。因此，在实际应用中，需要在误判率和空间效率之间做出妥协。

布隆过滤器的误判难以避免，但可以通过调整参数来控制误判率在一个可接受的范围内。例如，选择合适的位数组大小和哈希函数数量，可以在需求与误判率间彼此妥协。

### Java 集成布隆过滤器

#### Redis 实现布隆过滤器

Redis中 有一个数据结构叫做 Bitmap，它提供一个最大长度为 512MB（2^32）的位数组。我们可以把它提供给布隆过滤器做位数组。

首先将 guava 实现的本地的布隆过滤器的算法代码拿过来

```java
/**
 * 算法过程：
 * 1. 首先需要k个hash函数，每个函数可以把key散列成为1个整数
 * 2. 初始化时，需要一个长度为n比特的数组，每个比特位初始化为0
 * 3. 某个key加入集合时，用k个hash函数计算出k个散列值，并把数组中对应的比特位置为1
 * 4. 判断某个key是否在集合时，用k个hash函数计算出k个散列值，并查询数组中对应的比特位，如果所有的比特位都是1，认为在集合中。
 **/
public class BloomFilterHelper<T> {
    private int numHashFunctions;
    private int bitSize;
    private Funnel<T> funnel;

    public BloomFilterHelper(Funnel<T> funnel, int expectedInsertions, double fpp) {
        Preconditions.checkArgument(funnel != null, "funnel不能为空");
        this.funnel = funnel;
        // 计算bit数组长度
        bitSize = optimalNumOfBits(expectedInsertions, fpp);
        // 计算hash方法执行次数
        numHashFunctions = optimalNumOfHashFunctions(expectedInsertions, bitSize);
    }

    public int[] murmurHashOffset(T value) {
        int[] offset = new int[numHashFunctions];

        long hash64 = Hashing.murmur3_128().hashObject(value, funnel).asLong();
        int hash1 = (int) hash64;
        int hash2 = (int) (hash64 >>> 32);
        for (int i = 1; i <= numHashFunctions; i++) {
            int nextHash = hash1 + i * hash2;
            if (nextHash < 0) {
                nextHash = ~nextHash;
            }
            offset[i - 1] = nextHash % bitSize;
        }
        return offset;
    }

    /**
     * 计算bit数组长度
     */
    private int optimalNumOfBits(long n, double p) {
        if (p == 0) {
            // 设定最小期望长度
            p = Double.MIN_VALUE;
        }
        return (int) (-n * Math.log(p) / (Math.log(2) * Math.log(2)));
    }

    /**
     * 计算hash方法执行次数
     */
    private int optimalNumOfHashFunctions(long n, long m) {
        return Math.max(1, (int) Math.round((double) m / n * Math.log(2)));
    }
}
```

结合 redis 将数据存到 bitMap 中

```java
public class BloomRedisService {

    private RedisTemplate<String, Object> redisTemplate;
    private BloomFilterHelper bloomFilterHelper;
    
    public void setBloomFilterHelper(BloomFilterHelper bloomFilterHelper) {
        this.bloomFilterHelper = bloomFilterHelper;
    }

    public void setRedisTemplate(RedisTemplate<String, Object> redisTemplate) {
        this.redisTemplate = redisTemplate;
    }

    /**
     * 根据给定的布隆过滤器添加值
     */
    public <T> void addByBloomFilter(String key, T value) {
        int[] offset = bloomFilterHelper.murmurHashOffset(value);
        for (int i : offset) {
            redisTemplate.opsForValue().setBit(key, i, true);
        }
    }

    /**
     * 根据给定的布隆过滤器判断值是否存在
     */
    public <T> boolean includeByBloomFilter(String key, T value) {
        int[] offset = bloomFilterHelper.murmurHashOffset(value);
        for (int i : offset) {
            if (!redisTemplate.opsForValue().getBit(key, i)) {
                return false;
            }
        }
        return true;
    }
}
```

#### Redission 实现布隆过滤器

Redission对Redis做了很多封装，除了常见的分布式锁之外，对布隆过滤器也同样有实现，简单好用

```java
public class RedissonBloomFilter {
 
    public static void main(String[] args) {
        Config config = new Config();
        config.useSingleServer().setAddress("redis://ip:port");
        //构造Redisson
        RedissonClient redisson = Redisson.create(config);
 
        RBloomFilter<String> bloomFilter = redisson.getBloomFilter("phoneList");
        //初始化布隆过滤器：预计元素为100000000L,误差率为3%
        bloomFilter.tryInit(100000000L,0.03);
		//插入数据
        bloomFilter.add("aa");
 
        //判断下面号码是否在布隆过滤器中
        System.out.println(bloomFilter.contains("bb"));//false
        System.out.println(bloomFilter.contains("aa"));//true
    }
```



#### 自定义布隆过滤器

目前 MurmurHash 函数作为布隆过滤器的 hash 函数是使用得比较多的，所以以下内容也会采用这种函数。MurmurHash 是一种高效且具有良好分布性质的哈希函数，通常用于哈希表、布隆过滤器等场景。它的性能较好，且碰撞较少。
Guava 是 Google 提供的一个 Java 开源库，包含了许多常用的工具类和数据结构，包括 集合框架、缓存、并发工具、哈希算法 等，其中就包括了 布隆过滤器（BloomFilter）的实现。可以使用其中的 Hashing.murmur3_128() 方法来创建 MurmurHash 实例。

Maven 配置

```xml
<dependency>
    <groupId>com.google.guava</groupId>
    <artifactId>guava</artifactId>
    <version>33.0.0-jre</version>
</dependency>
```

Java 代码实现布隆过滤器

```java
import com.google.common.hash.HashFunction;
import com.google.common.hash.Hashing;
import com.google.common.hash.Funnels;

import java.nio.charset.Charset;
import java.util.BitSet;
import java.util.function.Function;

public class SimpleBloomFilter<T> {
    private final BitSet bitSet;      // 位数组
    private final int bitSetSize;     // 位数组的大小
    private final int numHashes;      // 哈希函数的数量
    private final HashFunction[] hashFunctions; // 哈希函数

    // 构造函数，初始化布隆过滤器
    public SimpleBloomFilter(int expectedElements, double falsePositiveRate) {
        // 计算位数组的大小
        this.bitSetSize = optimalBitSetSize(expectedElements, falsePositiveRate);
        this.numHashes = optimalNumHashes(expectedElements, bitSetSize);
        this.bitSet = new BitSet(bitSetSize);
        this.hashFunctions = createHashFunctions(numHashes);
    }

    // 向布隆过滤器添加元素
    public void add(T element) {
        for (HashFunction hashFunction : hashFunctions) {
            int hash = hashFunction.newHasher()
                .putObject(element,Funnels.stringFunnel(Charset.defaultCharset()))
                .hash()
                .asInt();
            int bitIndex = Math.abs(hash % bitSetSize);  // 将哈希值映射到位数组的索引位置
            bitSet.set(bitIndex);  // 将该位置设置为1
        }
    }

    // 检查元素是否可能在布隆过滤器中
    public boolean mightContain(T element) {
        for (HashFunction hashFunction : hashFunctions) {
            int hash = hashFunction.newHasher()
                .putObject(element,Funnels.stringFunnel(Charset.defaultCharset()))
                .hash()
                .asInt();
            int bitIndex = Math.abs(hash % bitSetSize);  // 将哈希值映射到位数组的索引位置
            if (!bitSet.get(bitIndex)) {
                return false;  // 如果某个位置是0，说明该元素一定不在布隆过滤器中
            }
        }
        return true;  // 否则，返回可能存在
    }

    // 计算最佳的位数组大小
    private static int optimalBitSetSize(int expectedElements, double falsePositiveRate) {
        return (int) Math.ceil((expectedElements * Math.log(falsePositiveRate)) / Math.log(1.0 / Math.pow(2, Math.log(2))));
    }

    // 计算需要的哈希函数数量
    private static int optimalNumHashes(int expectedElements, int bitSetSize) {
        return (int) Math.round((bitSetSize / expectedElements) * Math.log(2));
    }

    // 创建多个哈希函数
    private HashFunction[] createHashFunctions(int numHashes) {
        HashFunction[] functions = new HashFunction[numHashes];
        for (int i = 0; i < numHashes; i++) {
            // 使用不同的盐值（即种子值）来生成不同的哈希函数
            functions[i] = Hashing.murmur3_128(i);  // 使用 MurmurHash 作为基础
        }
        return functions;
    }

    // 测试布隆过滤器
    public static void main(String[] args) {
        // 创建一个预计存储 1000 个元素，误判率为 0.01 的布隆过滤器
        SimpleBloomFilter<String> bloomFilter = new SimpleBloomFilter<>(1000, 0.01);

        // 向布隆过滤器添加元素
        bloomFilter.add("apple");
        bloomFilter.add("banana");
        bloomFilter.add("orange");

        // 测试是否包含元素
        System.out.println(bloomFilter.mightContain("apple"));  // true
        System.out.println(bloomFilter.mightContain("banana")); // true
        System.out.println(bloomFilter.mightContain("orange")); // true
        System.out.println(bloomFilter.mightContain("grape"));  // false
    }
}

```


