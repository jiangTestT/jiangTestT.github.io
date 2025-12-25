# Neo4j文档


## 特点

节点
节点和关系都包含属性
关系链接节点
属性是键值对
节点用圆圈表示，关系用方向键表示
关系有方向：单向和双向
每个关系包含"开始节点"和"到节点"或"结束节点"

## 主要元素

节点，属性，关系，标签，数据浏览器

## 基础语法

### 创建

`CREATE`：创建节点、关系和属性

```sql
## create节点
CREATE (:student {name: "小明"} ),(:student {name: "小红"} ),(:student {name: "张三"} )

## 建立关系
match (n:student {name: "小明"}),(m:student {name: "小红"}) create (n) - [r:同学] -> (m)  return n.name, type(r), m.name

MATCH (n:city), (m:city_relation), (t:city)
WHERE n.code = m.code AND m.subCode = t.code
CREATE (n) - [r:`属于`] -> (t)
RETURN n.name, m.subName
```

### 查询

`MATCH`：检索有关节点、关系和属性

`ORDER BY `：排序检索数据

`RETURN`：返回查询结果

`WHERE`：提供条件过滤检索数据

```sql
## 查询
MATCH (n:student) RETURN n LIMIT 25

## 查询关系
MATCH relation = () -[r:`同学`] -> ()  RETURN relation
```

查询两节点的之间的最短路径

```sql
match p = shortestpath((a)-[r*0..4]-(b))
where a.name = '刘备' and b.name='刘禅'
return p
```

查询两节点的之间所有的最短路径

```sql
match p = allshortestpaths((a)-[r*0..4]-(b))
where a.name = '刘备' and b.name='刘禅'
return p
```

### 删除

`DELETE`：删除节点和关系

`REMOVE`：删除节点和关系的属性

```sql
## 移除属性
MATCH (n:student) WHERE n.name = "小明" REMOVE n.sex

## 删除节点，删除节点前必须删除关系
MATCH(n:student)  WHERE id(n) = 3 DELETE n

## 删除关系
MATCH (n:student) -[r] -> (m:student) WHERE n.name = "小明" AND m.name = "小红" DELETE r
```

### 更新

`SET`：添加或更新标签

```sql
## 设置属性
MATCH (n:student) WHERE n.name = "小明" SET n.age = 20
```

### Merge

`MERGE` 语句非常适用于在插入或更新数据时进行模式匹配，它可以确保数据的一致性，同时避免重复创建相同的节点或关系。

`MERGE` 语句用于根据指定的模式进行创建或匹配节点和关系。如果节点或关系不存在时，创建它们，已存在时则进行匹配。

**创建或匹配节点：**

```sql
## 不存在时，创建；存在时，匹配（必须全部属性匹配）
MERGE (p:person {name: '张三', age: '20', phone: '123456789', email:'zhangsan@qq.com'})

## 如果存在数据，删除
MERGE (p:person {name: '张三', age: '20', phone: '18784417369', email:'zhangsan@qq.com'}) DELETE p

## 复杂查询
MATCH relation = (p1: person) - [r:Freid] -> (p2: person) - [k:KNOWS] -> (p3: person)
RETURN relation

## 创建无向链接
MATCH (j:Person { name: '张三' }),(h:Person { name: '王五' })
MERGE (j)-[r:KNOWS]-(h)
RETURN type(r)
```

**创建或匹配带有多个标签的节点：**

```sql
## 这个查询会检查是否已经存在一个带有 "person"和 "man"标签且属性为以下所示的节点。如果不存在，则创建一个新节点。
MERGE (p:person:man {name: '张三', age: '20', phone: '123456789', email:'zhangsan@qq.com'})
```

**创建或匹配关系及相关节点：**

这个查询会首先检查是否已经存在两个具有 "Person" 标签的节点。然后它会检查两者是否有 "FRIEND" 关系，将这两个节点连接起来。

```sql
MERGE (a:person {name: '李四', age: '21', phone: '987654321', email:'lisi@qq.com'});
MERGE (b:person {name: '张三', age: '20', phone: '123456789', email:'zhangsan@qq.com'});
MERGE (a)-[:Freid]->(b);
```

**条件性创建或匹配：**

这个查询会检查是否已经存在一个具有 "Person" 标签且属性满足以下所示的节点。如果不存在，则创建一个新节点并设置 "age" 属性为 18。如果已存在，则更新 "phone" 属性。

```sql
MERGE (p:person {name: '张三', age: '20', phone: '123456789', email:'zhangsan@qq.com'})
ON CREATE SET p.age = '18'
ON MATCH SET p.phone = '147852369'
```

### With

`with`字句定义作用域变量，把with后面结果集当成一个查询结果、在这个结果基础上再做where条件的筛选或者候选的查询

```sql
WITH '刘备' AS a
MATCH (p:Person) WHERE p.name = a RETURN p;

MATCH (p:Person) - [r:兄弟] - (q:Person)
WITH p, COUNT(q) AS people_count
WHERE people_count > 1
RETURN p.name,people_count
```

在with语句后，提前使用limit，限制查询的数据

```sql
MATCH (p:Person)
WHERE p.group = "蜀"
WITH p LIMIT 4
MATCH (p) -- (m:Person) 
WITH p, COLLECT(DISTINCT m.name) AS mName
RETURN p.name, mName
```

### Unwind展开列表

`unwind`为列表的每个元素返回一行，如：某节点有个属性为列表，UNWIND将列表中的元素迭代，每个元素返回一个结果

```sql
WITH ['刘备', '孙权', '曹操'] AS names
UNWIND names AS name
MATCH (p:Person {name: name})
RETURN p
```

### Call执行子查询

子查询是一组在其自己的范围内执行的 Cypher 语句。**子查询通常从外部封闭查询调用。使用子查询，可以限制需要处理的行数。**

```sql
CALL {
	MATCH (p1:Person) WHERE p1.group = "蜀" RETURN p1
}
MATCH (p1) - [r:兄弟] - (p2:Person)
RETURN p2
```

### Foreach

循环操作，常用于批量更新节点的数据

```sql
## 更新/新增子节点的属性
MATCH p = (n:city) - [r:`属于`] -> (m:city)
WHERE n.name = '广东省'
FOREACH (item IN nodes(p) | SET m.marked = '9527') 
```

### Index索引

使用`INDEX`创建或者删除索引

```sql
## 创建索引
CREATE index on : personal (name)

## 删除索引
DROP index on : personal (name)
```

### Unique约束

避免创建重复的数据

```sql
## 创建约束
CREATE constraint on (n:personal) assert n.name is unique

## 删除约束
DROP constraint on (n:personal) assert n.name is unique
```

### 断言函数

`all`，用于判断list中的所有元素，是否满足where条件

```sql
return all(x in [1,2,3,4] where x%2=0)  //返回false
return all(x in [1,2,3,4, null] where x%2=0)  //返回false
```

`any`，用于判断list中，是否存在一个满足where条件的元素

```sql
return any(x in [] where x%2=0)        //返回false
return any(x in [null] where x%2=0)  //返回null
return any(x in [2.0] where x%2=0)   //返回true
```

`exist`，用于判断，是否存在某个属性或者关系

```sql
MATCH (p:person)
WHERE exists(p.name)
RETURN p.name AS name, exists((p)-[:LIVE_IN]->()) AS live
```

`none`，用于判断list中的每一个元素，是否都不满足where条件

```sql
return none(x in [1,3] where x%2=0)  //返回true
return none(x in [1,3, null] where x%2=0)  //返回null
```

`single`，用于判断list中，只有一个元素满足条件

```sql
return single(x in [6,null] where x%2=0) //返回null
```

### 标量函数

`coalesce`，返回参数中第一个不为null的值，如果都为null，则返回null。

```sql
MATCH (p:person) RETURN coalesce(p.name, p.age)
```

`startNode`：用于知道关系的开始节点

`endNode`：用于知道关系的结束节点

```sql
MATCH () -[r:LIVE_IN] -> ()  RETURN startNode(r), endNode(r)
```

`id`：用于知道关系的ID

`type`：用于知道字符串表示种的一个关系的TYPE

`labels`：用于返回节点的标签

```sql
MATCH () -[r:LIVE_IN] -> ()  RETURN id(r), type(r)
```

`head`：返回list的第一个元素

`last`：返回list的最后一个元素

```sql
return head([3,null,1,2,3])   //返回3
return head([])   //返回null
```

`randomUUID`：用于随机生成长度为32的唯一uuid

```sql
return REPLACE(randomuuid(), '-', '')
```

`length`：返回路径的长度

length也可以用于计算string、list、pattern的长度，但是强烈建议只对path使用length，对其它类型长度的计算功能，后续可能被丢弃。

```sql
MATCH relation = (p:person) -[*1..5] -> (m:person)  RETURN relation, length(relation)
```

`size`：用法根据参数的类型确定

size(string)：表示字符串中字符的数量，可以把字符串当作是字符的列表。

size(list)：返回列表中元素的数量。

size(pattern_expression)：也是统计列表中元素的数量

```sql
MATCH relation = (p:person) -[*1..5] -> (m:person)
RETURN m.name, size(m.name), size([1,2,null,4]), size((m) --> ())

MATCH (p1:person), (p2:person)
WHERE p1.name = '王五' AND p2.name = '李四'
RETURN size((p2) --> (p1))

RETURN size([1,2,null,4]) //返回4

MATCH (p:Person) - [r:下属] -> (m:Person)
RETURN size(collect(m))
```

`properties`：用于返回节点或者关系的全部属性

`timestamp`：用于返回当前的时间戳

### 字符串函数

`left`和`right`，用于获取字符串左边/右边，规定长度的子字符串

```sql
MATCH (n:person) 
WHERE n.name = '李四'
RETURN left(n.phone, 2)
```

`ltrim、rtrim、trim`，去除字符串左边、右边、两边的空格

```sql
RETURN ltrim(" Hello Word "), rtrim(" Hello Word "), trim(" Hello Word ")
```

`replace`：replace(original, search, replace)，使用replace替换original中的所有search字符串

```sql
MATCH (n:person) WHERE n.name = '李四'
RETURN replace(n.phone, '2', '1')
```

`reverse`：字符串翻转

`split`：字符串拆分

`substring`：字符串切割

`toLower、toUpper`：用于字符串的大小写转换

```sql
MATCH (n:person) WHERE n.name = '李四'
RETURN reverse(n.phone)

MATCH (n:person) WHERE n.name = '李四'
RETURN split(n.phone, '5')

MATCH (n:person) WHERE n.name = '李四'
RETURN substring(n.phone, 5)

MATCH (n:person) WHERE n.name = '李四'
RETURN toUpper(n.name)

RETURN toUpper('Jerry'), toLower('Jerry')
```

`contains`：判断字符串是否包含另一个字符串，注意区分大小写

```sql
RETURN 'Hello Word' contains 'Hello'
```

`nodes`：用于返回路径上的所有节点

```sql
MATCH relation = (p1: person) -[r:LIVE_IN] -> (l1: location)
RETURN nodes(relation)

MATCH p =(a)-->(b)-->(c) 
WHERE a.name = '李四' RETURN nodes(p)
```

`relationships`：返回路径上的所有关系

```sql
MATCH p =(a)-->(b)-->(c) 
WHERE a.name = '李四' 
RETURN relationships(p), nodes(p)
```

`reduce`：返回每个元素作用于表达式上的结果

```sql
MATCH p =(a)-->(b)-->(c) 
WHERE a.name = '李四' 
RETURN reduce(totalAge = 0, n IN nodes(p)| totalAge + id(n)) AS reduction
```

使用正则表达式进行匹配

```sql
RETURN 'Hello Word' =~ 'Hello.*'
```

### 数值函数

`abs`：取绝对值

`floor`：向下取整

`ceil`：向上取整

`round`：四舍五入

`rand`：返回一个（0, 1）的随机数

`sign`：返回数值的正负，如果值为零，则返回0。如果值为负数，则返回-1。如果值为正数，返回1。

```sql
RETURN abs(-101), floor(3.5), ceil(3.5), round(3.4)
RETURN round(3.4), round(3.5)
RETURN rand(), sign(-1), sign(0), sign(1)
```

### 导入CSV文件

```sql
## 导入csv文件并创建节点
load csv from "file:///city.csv" as row 
create (n:city {code:row[1], name:row[2], administrativeDivision:row[0]})
return count(n)

load csv from "file:///city_relation.csv" as row 
create (n:city_relation {code:row[0], name:row[1], subCode:row[2], subName:row[3]})
return count(n)
```

## 时间函数

date格式：`yyyy-mm-dd`

datetime格式：`yyyy-mm-ddThh:MM:SS:sssZ`

localdatetime格式：`yyyy-mm-ddThh:MM:SS:sss`

localtime格式：`hh:MM:SS.sss`

time格式：`hh:MM:SS.sssZ`

### 获取时间

获取当前时间，可以指定时区

- `date([{timezone}])`
- `datetime([{timezone}])`
- `localdatetime([{timezone}])`
- `localtime([{timezone}])`
- `time([{timezone}])`

```sql
RETURN date()
RETURN date({timezone: 'America/Los Angeles'})
```

使用transaction时返回当前时间。对于同一事务中的每次调用，该值都是相同的 

- `date.transaction([{timezone}])`
- `datetime.transaction([{timezone}])`
- `localdatetime.transaction([{timezone}])`
- `localtime.transaction([{timezone}])`
- `time.transaction([{timezone}])`

```sql
RETURN date.transaction()
```

使用statement返回当前时间。对于同一语句中的每次调用，该值都相同。但是，同一事务中的不同语句可能会产生不同的值 

- `date.statement([{timezone}])`
- `datetime.statement([{timezone}])`
- `localdatetime.statement([{timezone}])`
- `localtime.statement([{timezone}])`
- `time.statement([{timezone}])`

```sql
RETURN date.statement()
```

使用时间返回当前值realtime。该值将是系统的实时时钟

- `date.realtime([{timezone}])`
- `datetime.realtime([{timezone}])`
- `localdatetime.realtime([{timezone}])`
- `localtime.realtime([{timezone}])`
- `time.realtime([{timezone}])`

```sql
RETURN date.realtime()
```

### 创建时间

以**年-月-日**的格式创建时间

- `date({year [, month, day]})`
- `datetime({year [, month, day, hour, minute, second, millisecond, microsecond, nanosecond, timezone]})`
- `localdatetime({year [, month, day, hour, minute, second, millisecond, microsecond, nanosecond]})`

```sql
RETURN date({year: 2024, month: 4, day: 24})
```

以**年-周-日**的格式创建时间

- `date({year [, week, dayOfWeek]})`
- `datetime({year [, week, dayOfWeek, hour, minute, second, millisecond, microsecond, nanosecond, timezone]})`
- `localdatetime({year [, week, dayOfWeek, hour, minute, second, millisecond, microsecond, nanosecond]})`

```sql
RETURN date({year: 2024, week: 17, dayOfWeek: 3})
```

以**年-季度-日**的格式创建时间

- `date({year [, quarter, dayOfQuarter]})`
- `datetime({year [, quarter, dayOfQuarter, hour, minute, second, millisecond, microsecond, nanosecond, timezone]})`
- `localdatetime({year [, quarter, dayOfQuarter, hour, minute, second, millisecond, microsecond, nanosecond]})`

```sql
RETURN date({year: 2024, quarter: 2, dayOfQuarter: 24})
```

以**年-日**的格式创建时间

- `date({year [, ordinalDay]})`
- `datetime({year [, ordinalDay, hour, minute, second, millisecond, microsecond, nanosecond, timezone]})`
- `localdatetime({year [, ordinalDay, hour, minute, second, millisecond, microsecond, nanosecond]})`

```sql
RETURN date({year: 2024, ordinalDay: 115})
```

根据字符串创建时间

- `date(temporalValue)`
- `datetime(temporalValue)`

```sql
RETURN date('2024-04-24'),date('2024-04'),date('202404'),date('2024-W17-3'),date('20240424'),date('2024')

RETURN datetime('2015-07-21T21:40:32.142+0100'),
	   datetime('2015-W30-2T214032.142Z'),
	   datetime('2015T214032-0100'),
	   datetime('20150721T21:40-01:30'),
	   datetime('2015-W30T2140-02'),
	   datetime('2015202T21+18:00'),
	   datetime('2015-07-21T21:40:32.142[Europe/London]'),
	   datetime('2015-07-21T21:40:32.142-04[America/New_York]')
	   
return localdatetime('2015-07-21T21:40:32.142'),
       localdatetime('2015-W30-2T214032.142'),
       localdatetime('2015-202T21:40:32'),
       localdatetime('2015202T21')
```

根据时间戳创建时间

- `datetime({ epochSeconds | epochMillis })`

```sql
## 注意，这里默认的时区是UTC协调世界时
return datetime({epochSeconds: timestamp() / 1000, nanosecond: 23}), datetime({epochMillis: 1713950965318})
```



使用其他时间格式创建另一种时间格式的时间

- `date({date [, year, month, day, week, dayOfWeek, quarter, dayOfQuarter, ordinalDay]})`
- `datetime({datetime [, year, ..., timezone]}) | datetime({date [, year, ..., timezone]}) | datetime({time [, year, ..., timezone]}) | datetime({date, time [, year, ..., timezone]})`
- `localdatetime({datetime [, year, ..., nanosecond]}) | localdatetime({date [, year, ..., nanosecond]}) | localdatetime({time [, year, ..., nanosecond]}) | localdatetime({date, time [, year, ..., nanosecond]})`

```sql
UNWIND [
 date({year: 1984, month: 11, day: 11}), 
 localdatetime({year: 1984, month: 11, day: 11, hour: 12, minute: 31, second: 14}), 
 datetime({year: 1984, month: 11, day: 11, hour: 12, timezone: '+01:00'})
] as dd 
RETURN date({date: dd}) as dateOnly, date({date: dd, day: 28}) as dateDay;
  
WITH date({year: 1984, month: 10, day: 11}) as dd
RETURN datetime({date: dd, hour: 10, minute: 10, second: 10}) as dateHHMMSS,
datetime({date: dd, day: 28, hour: 10, minute: 10, second: 10, timezone:'Pacific/Honolulu'}) as dateDDHHMMSSTimezone;

```

### 分割时间

```sql
WITH datetime({
    year: 2024, month: 4, day: 24,
    hour: 12, minute: 31, second: 14, nanosecond: 645876123,
    timezone: '+01:00'
  }) as d
RETURN
  date.truncate('millennium', d) as truncMillenium,
  datetime.truncate('century', d) as truncCentury,
  date.truncate('decade', d) AS truncDecade,
  date.truncate('year', d, {day: 5}) AS truncYear,
  date.truncate('weekYear', d) as truncWeekYear,
  date.truncate('quarter', d) as truncQuarter,
  date.truncate('month', d) as truncMonth,
  date.truncate('week', d, {dayOfWeek: 2}) as truncWeek,
  date.truncate('day', d) as truncDay,
  datetime.truncate('hour', d) as truncHour,
  datetime.truncate('minute', d) as truncMinute,
  datetime.truncate('second', d) as truncSecond,
  datetime.truncate('millisecond', d) as truncMillisecond,
  datetime.truncate('microsecond', d) as truncMicrosecond
```

## 特殊查询

无向关系的查询，在Neo4J中，关系的创建不能是无向的，但是查询和使用可以，因此可以这样查询和修改关系

```sql
MATCH (m:Person) - [r2:兄弟] - (n:Person)
RETURN DISTINCT m
```


