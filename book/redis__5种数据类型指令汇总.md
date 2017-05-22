# redis

> string \ hash \ list \ set \ zset

### string

* keys *
* set bar 1 / get bar
* exists key
* del key [key ...]
* type key
* incr bar / incrby bar 10
* incrbyfloat bar 2.7
* decr bar / decrby var 10
* append key " xxxx"
* strlen key
* mset k1 v1 k2 v2 ...  /  mget key [key2]
* getbit key offset
* setbit key offset value
* bitcount key start end
* bitop operation destkey key [key ...]
* bitpos key 0/1


### hash
一个散列值最多2*32 -1个字段，适合存储对象

* hset key field value / hget key field
* hmset key field value [key field value] / hmget key field [field]
* hgetall key
* hexists key field
* hsetnx key field value (原子操作，无竟态条件)
* hincrby key field value
* hdel key field [field]
* hkeys key / hvals key

### list
有序字符串列表

* lpush key value [value] / rpush key value [value]
* lpop k / rpop k
* llen k 
* lrange key start end (从左到右)
* lrange key -5 -1 (-1表示最右侧第一个，-5从右往左数第5个)
* lindex key index （index为负值时，从右边开始数）
* lset key index value
* ltrim key start end (删除start-end范围外的数据)
* linsert key before|after pivot value (linsert key before 1 10，在1前面插入10)
* rpoplpush source destination （原子操作）

### set
集合类型，一个集合类型最多存储2*32-1个字符串
无序、唯一性

* sadd key member [member ...] 增加元素
* srem key member [member ...] 删除元素
* smembers key 查看元素
* sismember key member 查看元素是否在集合中
* sdiff key1 key2 (sdiff key2 key1结果不同) 差集
* sinter  key1 key2 key3 交集
* sunion key1 key2 key3 并集
* scard key 获取集合元素个数
* sinterstore destination key1 key2 ... 与sinter类似，只是结果保存在destination
* sdiffstore destination key1 key2 ... 同上
* sunionstore destination key1 key2 ... 同上
* srandmember key [count] 随机获取集合中的元素 
	* 如果count > 0 元素不同
	* count < 0 元素可能相同
* spop key 随机pop一个元素

### zset
有序集合

* zadd key score member [score member ...] 增加元素
* zscore key member 获取分数
* zrange key start stop [withsocres] 获取某个范围的元素列表
* zrangebysocre key [(]min [(]max [withscores] [limit offset count] 
	* zrangebysocre key (80 100 withscores limit 1 3 (获取80（不含）到100（含）分的元素，limit 1 3 表示从结果集中从编号为1（0为第一个元素）开始筛选出3个元素) 结果从小到大排序
* zrevrangebysocre key [(]max [(]min [withscores] [limit offset count] 结果从大到小排序
* zincrby key offset member 为key中的member增加offset分（offset可正可负）
* zcard key 获取集合中元素的数量
* zcount key [(]min [(]max 指定范围内元素个数
* zrem key member [member ...] 删除元素
* zremrangebyscore key [(]min [(]max 删除指定排名范围内的元素
* zremrangebyrank key start stop 删除指定排名范围内的元素
* zrank key member 获取元素的排名(从小到大排序)
* zrevrank key member 获取元素的排名(从大到小排序)
* zinterstore destination numkeys key [key2 ...] [weights weight [weight ...]] [aggregate sum|max|min]
	* 默认情况下 aggregate值为 sum ,交集取和，max取最大值，min取最小值
	*  zadd z1 1 a 2 b
	*  zadd z2 20 a 30 b
	*  zinterstore z3 2 z1 z2 
	* zrange z3 0 -1 withscores
		* 1) "a"
		* 2) "11"
		* 3) "b"
		* 4) "32"
	*  zinterstore z3 2 z1 z2 aggregate min
		* 1) "a"
		* 2) "1"
		* 3) "b"
		* 4) "2"
	* zinterstore z3 2 z1 z2 weights 1 0.1 aggregate max
		* 1) "a"
		* 2) "2"
		* 3) "b"
		* 4) "3"
* zunionstore 同上求并集
 
### 事务

### watch

### 过期时间 expire

* expire foo 20
* persist key 









