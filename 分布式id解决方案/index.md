# 分布式id解决方案


## 分布式id概述

在我们业务系统数据量不大的时候，单库单表完全可以支撑现有业务，数据再大一点搞个 MySQL 主从同步读写分离也能对付，这时候我们使用数据库自增id就足够了。

但随着业务数据日渐增长，主从同步也扛不住了，就需要对数据库进行分库分表，但分库分表后需要有一个唯一 ID 来标识一条数据，数据库的自增ID显然不能满足需求；还有就是某些场景需要唯一编号标识，比如订单号，用户编号等都需要有`唯一 ID`做标识。此时一个能够生成`全局唯一 ID`的系统是非常必要的。那么这个`全局唯一 ID`就叫`分布式 ID`。

### 分布式id的要求

- **全局唯一**：必须保证 ID 是全局性唯一的，基本要求
- **高性能**：高可用低延时，ID 生成响应要块，否则反倒会成为业务瓶颈
- **高可用**：100% 的可用性是骗人的，但是也要无限接近于 100% 的可用性
- **好接入**：要秉着拿来即用的设计原则，在系统设计和实现上要尽可能的简单
- **趋势递增**：最好趋势递增，这个要求就得看具体业务场景了，一般不严格要求，如果是使用在业务数据库分库分表的分布式主键 ID，那边递增最后，如果只是用在唯一编号场景，并不太需要连续递增。

## 分布式id的实现方式

### UUID

想要实现获取一个唯一 id 标识会立刻使用 UUID 实现，毕竟实现简单快捷，且满足唯一要求，实现如下：

```java
public static void main(String[] args) {
       String uuid = UUID.randomUUID().toString().replaceAll("-","");
       System.out.println(uuid);
 }
```

但是使用 uuid 生成分布式 id，会有如下问题：

- 无序的字符串，不具备趋势自增特性
- 没有具体的业务含义，像订单号为 uuid 这样毫无意义的字符串，看不出和订单任何关联信息
- 长度过长 16 字节 128 位，36 位长度的字符串，存储以及查询对 MySQL 的性能消耗较大，MySQL 官方明确建议主键要尽量越短越好，作为数据库主键 `UUID` 的无序性会导致数据位置频繁变动，严重影响性能。

### 基于数据库自增ID

基于数据库的`auto_increment`自增 ID 完全可以充当`分布式 ID`，具体实现：需要一个单独的 MySQL 实例用来生成 ID，建表结构如下：

```sql
CREATE DATABASE `SEQ_ID`;
CREATE TABLE SEQID.SEQUENCE_ID (
    id bigint(20) unsigned NOT NULL auto_increment, 
    value char(10) NOT NULL default '',
    PRIMARY KEY (id),
) ENGINE=MyISAM;
insert into SEQUENCE_ID(value)  VALUES ('values');
```

当我们需要一个 ID 的时候，向表中插入一条记录返回`主键 ID`，但这种方式有一个比较致命的缺点，访问量激增时 MySQL 本身就是系统的瓶颈，用它来实现分布式服务风险比较大，存在**支持的并发量不大、数据库单点问题**，不推荐！

### 基于数据库集群模式

前边说了单点数据库方式不可取，那对上边的方式做一些高可用优化，换成主从模式集群。害怕一个主节点挂掉没法用，那就做双主模式集群，也就是两个 Mysql实例都能单独的生产自增 ID。

那这样还会有个问题，两个 MySQL 实例的自增ID都从1开始，**会生成重复的ID怎么办？**

MySQL_1 配置：

```sql
set @@auto_increment_offset = 1;     -- 起始值
set @@auto_increment_increment = 2;  -- 步长
```

MySQL_2 配置：

```sq
set @@auto_increment_offset = 2;     -- 起始值
set @@auto_increment_increment = 2;  -- 步长
```

这样两个MySQL实例的自增ID分别就是：

> 1、3、5、7、9...
> 2、4、6、8、10...

那如果集群后的性能还是扛不住高并发咋办？就要进行 MySQL 扩容增加节点，这是一个比较麻烦的事。

增加第三台`MySQL`实例需要人工修改一、二两台`MySQL实例`的起始值和步长，把`第三台机器的ID`起始生成位置设定在比现有`最大自增ID`的位置远一些，但必须在一、二两台`MySQL实例` ID 还没有增长到`第三台MySQL实例`的`起始ID`值的时候，否则`自增ID`就要出现重复了，**必要时可能还需要停机修改**。

###  基于数据库的号段模式

号段模式是当下分布式ID生成器的主流实现方式之一，号段模式可以理解为从数据库批量的获取自增ID，每次从数据库取出一个号段范围，例如 (1,1000] 代表1000 个 ID，具体的业务服务将本号段，生成 1~1000 的自增 ID 并加载到内存。表结构如下：

```sql
CREATE TABLE id_generator (
  id int(10) NOT NULL,
  max_id bigint(20) NOT NULL COMMENT '当前最大id',
  step int(20) NOT NULL COMMENT '号段的布长',
  biz_type    int(20) NOT NULL COMMENT '业务类型',
  version int(20) NOT NULL COMMENT '版本号',
  PRIMARY KEY (`id`)
)
```

**biz_type** ：代表不同业务类型

**max_id** ：当前最大的可用id

**step** ：代表号段的长度

**version** ：是一个乐观锁，每次都更新 version，保证并发时数据的正确性

等这批号段 ID 用完，再次向数据库申请新号段，对`max_id`字段做一次`update`操作，`update max_id = max_id + step`，update 成功则说明新号段获取成功，新的号段范围是`(max_id ,max_id +step]`。

```sq
update id_generator set max_id = #{max_id + step}, version = version + 1 where version = # {version} and biz_type = XXX
```

由于多业务端可能同时操作，所以采用版本号`version`乐观锁方式更新，这种`分布式ID`生成方式不强依赖于数据库，不会频繁的访问数据库，对数据库的压力小很多。

### 基于redis实现

`Redis`也同样可以实现，原理就是利用`redis`的 `incr`命令实现ID的原子性自增

```sh
127.0.0.1:6379> set seq_id 1     // 初始化自增ID为1
OK
127.0.0.1:6379> incr seq_id      // 增加1，并返回递增后的数值
(integer) 2
```

用`redis`实现需要注意一点，要考虑到 redis 持久化的问题，否则可能 id 重复使用问题

### 基于雪花算法（Snowflake）模式

雪花算法（Snowflake）是 twitter 公司内部分布式项目采用的 ID 生成算法，开源后广受国内大厂的好评，在该算法影响下各大公司相继开发出各具特色的分布式生成器。

![](https://crry-assist.oss-cn-chengdu.aliyuncs.com/Screenshot/Snowflake.png)

`Snowflake`生成的是 Long 类型的ID，一个 Long 类型占8个字节，每个字节占 8 bit，也就是说一个 Long 类型占 64 个bit。

Snowflake ID 组成结构：`正数位`（占 1 bit）+ `时间戳`（占 41 bit）+ `机器ID`（占 5 bit）+ `数据中心`（占 5 bit）+ `自增值`（占 12 bit），总共 64 bit 组成的一个Long 类型。

- 第一个bit位（1bit）：Java 中 long 的最高位是符号位代表正负，正数是 0，负数是 1，一般生成 ID 都为正数，所以默认为 0。
- 时间戳部分（41bit）：毫秒级的时间，不建议存当前时间戳，而是用（当前时间戳 - 固定开始时间戳）的差值，可以使产生的 ID 从更小的值开始；41 位的时间戳可以使用 69 年，(1L << 41) / (1000L \*60\* 60 \*24\* 365) = 69 年
- 工作机器id（10bit）：也被叫做`workId`，这个可以灵活配置，机房或者机器号组合都可以。
- 序列号部分（12bit），自增值支持同一毫秒内同一个节点可以生成4096个ID

根据这个算法的逻辑，只需要将这个算法用 Java 语言实现出来，封装为一个工具方法，那么各个业务应用可以直接使用该工具方法来获取分布式 ID，只需保证每个业务应用有自己的工作机器 id 即可，而不需要单独去搭建一个获取分布式 ID 的应用。

```java
package com.jerry.spring.common.util;

import java.util.Objects;

public class SnowFlakeUtil {
    /**
     * 起始的时间戳
     */
    private static Long START_TIMESTAMP = 1480166465631L;

    /**
     * 每一部分占用的位数
     */
    private final static long SEQUENCE_BIT = 12;   //序列号占用的位数
    private final static long MACHINE_BIT = 5;     //机器标识占用的位数
    private final static long DATA_CENTER_BIT = 5; //数据中心占用的位数

    /**
     * 每一部分的最大值
     */
    private final static long MAX_SEQUENCE = ~(-1L << SEQUENCE_BIT);
    private final static long MAX_MACHINE_NUM = ~(-1L << MACHINE_BIT);
    private final static long MAX_DATA_CENTER_NUM = ~(-1L << DATA_CENTER_BIT);

    /**
     * 每一部分向左的位移
     */
    private final static long MACHINE_LEFT = SEQUENCE_BIT;
    private final static long DATA_CENTER_LEFT = SEQUENCE_BIT + MACHINE_BIT;
    private final static long TIMESTAMP_LEFT = DATA_CENTER_LEFT + DATA_CENTER_BIT;

    private final long dataCenterId;  //数据中心
    private final long machineId;     //机器标识
    private long sequence = 0L; //序列号
    private long lastTimeStamp = -1L;  //上一次时间戳

    private long getNextMill() {
        long mill = getNewTimeStamp();
        while (mill <= lastTimeStamp) {
            mill = getNewTimeStamp();
        }
        return mill;
    }

    private long getNewTimeStamp() {
        return System.currentTimeMillis();
    }

    /**
     * 根据指定的数据中心ID和机器标志ID生成指定的序列号
     */
    public SnowFlakeUtil(long dataCenterId, long machineId, Long startTimestamp) {
        if (dataCenterId > MAX_DATA_CENTER_NUM || dataCenterId < 0) {
            throw new IllegalArgumentException("DtaCenterId can't be greater than MAX_DATA_CENTER_NUM or less than 0！");
        }
        if (machineId > MAX_MACHINE_NUM || machineId < 0) {
            throw new IllegalArgumentException("MachineId can't be greater than MAX_MACHINE_NUM or less than 0！");
        }
        if (Objects.nonNull(startTimestamp)) {
            START_TIMESTAMP = startTimestamp;
        }
        this.dataCenterId = dataCenterId;
        this.machineId = machineId;
    }

    /**
     * 产生下一个ID
     */
    public synchronized long nextId() {
        long currTimeStamp = getNewTimeStamp();
        if (currTimeStamp < lastTimeStamp) {
            throw new RuntimeException("Clock moved backwards.  Refusing to generate id");
        }
        if (currTimeStamp == lastTimeStamp) {
            //相同毫秒内，序列号自增
            sequence = (sequence + 1) & MAX_SEQUENCE;
            //同一毫秒的序列数已经达到最大
            if (sequence == 0L) {
                currTimeStamp = getNextMill();
            }
        } else {
            //不同毫秒内，序列号置为0
            sequence = 0L;
        }

        lastTimeStamp = currTimeStamp;
        return (currTimeStamp - START_TIMESTAMP) << TIMESTAMP_LEFT //时间戳部分
                | dataCenterId << DATA_CENTER_LEFT       //数据中心部分
                | machineId << MACHINE_LEFT             //机器标识部分
                | sequence;                             //序列号部分
    }

    public static void main(String[] args) {
        SnowFlakeUtil snowFlake = new SnowFlakeUtil(2, 3, 1735660800000L);
        for (int i = 0; i < (1 << 4); i++) {
            //10进制
            System.out.println(snowFlake.nextId());
        }
    }
}

```

### 百度（uid-generator）

`uid-generator`是由百度技术部开发，项目GitHub地址 https://github.com/baidu/uid-generator

`uid-generator`是基于`Snowflake`算法实现的，与原始的`snowflake`算法不同在于，`uid-generator`支持自`定义时间戳`、`工作机器ID`和`序列号` 等各部分的位数，而且`uid-generator`中采用用户自定义`workId`的生成策略。

`uid-generator`需要与数据库配合使用，需要新增一个`WORKER_NODE`表。当应用启动时会向数据库表中去插入一条数据，插入成功后返回的自增 ID 就是该机器的`workId`数据由 host，port 组成。

**对于`uid-generator` ID组成结构**：

`workId`，占用了 22 个bit位，时间占用了 28 个bit位，序列化占用了 13 个bit位，需要注意的是，和原始的`snowflake`不太一样，时间的单位是秒，而不是毫秒，`workId`也不一样，而且同一应用每次重启就会消费一个`workId`。

### 美团（Leaf）

leaf为叶子的意思，这名字源自世界上没有两片完全相同的树叶。https://tech.meituan.com/2017/04/21/mt-leaf.html

```properties
mysql://localhost:3306/leaf_test?useUnicode=true&characterEncoding=utf8&characterSetResults=utf8
leaf.jdbc.username=root
leaf.jdbc.password=root

leaf.snowflake.enable=false
#leaf.snowflake.zk.address=
#leaf.snowflake.port=
```

启动`leaf-server` 模块的 `LeafServerApplication`项目就跑起来了

号段模式获取分布式自增 ID 的测试url ：http://localhost:8080/api/segment/get/leaf-segment-test

监控号段模式：http://localhost:8080/cache

### 滴滴（Tinyid）

`Tinyid`由滴滴开发，Github地址：https://github.com/didi/tinyid。

`Tinyid`是基于号段模式原理实现的与`Leaf`如出一辙，每个服务获取一个号段（1000,2000]、（2000,3000]、（3000,4000]
