# Redis 命令参考

本文档是 [Redis Command Reference](http://redis.io/commands) 和 [Redis Documentation](http://redis.io/documentation) 的中文翻译版： 所有 Redis 命令文档均已翻译完毕， Redis 最重要的一部分主题（topic）文档， 比如事务、持久化、复制、Sentinel、集群等文章也已翻译完毕。

文档目前描述的内容以 Redis 2.8 版本为准， 查看[*更新日志(change log)*](http://doc.redisfans.com/change_log.html#change-log)可以了解本文档对 Redis 2.8 所做的更新。

你可以通过网址 [doc.redisfans.com](http://doc.redisfans.com/) 在线阅览本文档， 也可以下载 [PDF 格式](https://raw.github.com/huangz1990/redis/master/download/redis.pdf) 或者 [HTML 格式](http://media.readthedocs.org/htmlzip/redis/latest/redis.zip) 的离线版本。

# Key（键）

## DEL

**DEL key [key ...]**

删除给定的一个或多个 `key` 。

不存在的 `key` 会被忽略。

- **可用版本：**

  >= 1.0.0

- **时间复杂度：**

  O(N)， `N` 为被删除的 `key` 的数量。删除单个字符串类型的 `key` ，时间复杂度为O(1)。删除单个列表、集合、有序集合或哈希表类型的 `key` ，时间复杂度为O(M)， `M` 为以上数据结构内的元素数量。

- **返回值：**

  被删除 `key` 的数量。

```bash
#  删除单个 key
redis> SET name huangz
OK
redis> DEL name
(integer) 1

# 删除一个不存在的 key
redis> EXISTS phone
(integer) 0
redis> DEL phone # 失败，没有 key 被删除
(integer) 0

# 同时删除多个 key
redis> SET name "redis"
OK
redis> SET type "key-value store"
OK
redis> SET website "redis.com"
OK
redis> DEL name type website
(integer) 3
```

## DUMP

**DUMP key**

序列化给定 `key` ，并返回被序列化的值，使用 [*RESTORE*](http://doc.redisfans.com/key/restore.html) 命令可以将这个值反序列化为 Redis 键。

序列化生成的值有以下几个特点：

- 它带有 64 位的校验和，用于检测错误， [*RESTORE*](http://doc.redisfans.com/key/restore.html) 在进行反序列化之前会先检查校验和。
- 值的编码格式和 RDB 文件保持一致。
- RDB 版本会被编码在序列化值当中，如果因为 Redis 的版本不同造成 RDB 格式不兼容，那么 Redis 会拒绝对这个值进行反序列化操作。

序列化的值不包括任何生存时间信息。

- **可用版本：**

  >= 2.6.0

- **时间复杂度：**

  查找给定键的复杂度为 O(1) ，对键进行序列化的复杂度为 O(N*M) ，其中 N 是构成 `key` 的 Redis 对象的数量，而 M 则是这些对象的平均大小。如果序列化的对象是比较小的字符串，那么复杂度为 O(1) 。

- **返回值：**

  如果 `key` 不存在，那么返回 `nil` 。否则，返回序列化之后的值。

```bash
redis> SET greeting "hello, dumping world!"
OK

redis> DUMP greeting
"\x00\x15hello, dumping world!\x06\x00E\xa0Z\x82\xd8r\xc1\xde"

redis> DUMP not-exists-key
(nil)
```

## EXISTS

**EXISTS key**

检查给定 `key` 是否存在。

- **可用版本：**

  >= 1.0.0

- **时间复杂度：**

  O(1)

- **返回值：**

  若 `key` 存在，返回 `1` ，否则返回 `0` 。

```bash
redis> SET db "redis"
OK

redis> EXISTS db
(integer) 1

redis> DEL db
(integer) 1

redis> EXISTS db
(integer) 0
```

## EXPIRE

**EXPIRE key seconds**

为给定 `key` 设置生存时间，当 `key` 过期时(生存时间为 `0` )，它会被自动删除。

在 Redis 中，带有生存时间的 `key` 被称为『易失的』(volatile)。

生存时间可以通过使用 [*DEL*](http://doc.redisfans.com/key/del.html#del) 命令来删除整个 `key` 来移除，或者被 [*SET*](http://doc.redisfans.com/string/set.html#set) 和 [*GETSET*](http://doc.redisfans.com/string/getset.html#getset) 命令覆写(overwrite)，这意味着，如果一个命令只是修改(alter)一个带生存时间的 `key` 的值而不是用一个新的 `key` 值来代替(replace)它的话，那么生存时间不会被改变。

比如说，对一个 `key` 执行 [*INCR*](http://doc.redisfans.com/string/incr.html#incr) 命令，对一个列表进行 [*LPUSH*](http://doc.redisfans.com/list/lpush.html#lpush) 命令，或者对一个哈希表执行 [*HSET*](http://doc.redisfans.com/hash/hset.html#hset) 命令，这类操作都不会修改 `key` 本身的生存时间。

另一方面，如果使用 [*RENAME*](http://doc.redisfans.com/key/rename.html) 对一个 `key` 进行改名，那么改名后的 `key` 的生存时间和改名前一样。

[*RENAME*](http://doc.redisfans.com/key/rename.html) 命令的另一种可能是，尝试将一个带生存时间的 `key` 改名成另一个带生存时间的 `another_key` ，这时旧的 `another_key` (以及它的生存时间)会被删除，然后旧的 `key` 会改名为 `another_key` ，因此，新的 `another_key` 的生存时间也和原本的 `key` 一样。

使用 [*PERSIST*](http://doc.redisfans.com/key/persist.html) 命令可以在不删除 `key` 的情况下，移除 `key` 的生存时间，让 `key` 重新成为一个『持久的』(persistent) `key` 。

**更新生存时间**

可以对一个已经带有生存时间的 `key` 执行 [*EXPIRE*](http://doc.redisfans.com/key/expire.html#expire) 命令，新指定的生存时间会取代旧的生存时间。

**过期时间的精确度**

在 Redis 2.4 版本中，过期时间的延迟在 1 秒钟之内 —— 也即是，就算 `key` 已经过期，但它还是可能在过期之后一秒钟之内被访问到，而在新的 Redis 2.6 版本中，延迟被降低到 1 毫秒之内。

**Redis 2.1.3 之前的不同之处**

在 Redis 2.1.3 之前的版本中，修改一个带有生存时间的 `key` 会导致整个 `key` 被删除，这一行为是受当时复制(replication)层的限制而作出的，现在这一限制已经被修复。

- **可用版本：**

  >= 1.0.0

- **时间复杂度：**

  O(1)

- **返回值：**

  设置成功返回 `1` 。当 `key` 不存在或者不能为 `key` 设置生存时间时(比如在低于 2.1.3 版本的 Redis 中你尝试更新 `key` 的生存时间)，返回 `0` 。

```
redis> SET cache_page "www.google.com"
OK

redis> EXPIRE cache_page 30  # 设置过期时间为 30 秒
(integer) 1

redis> TTL cache_page    # 查看剩余生存时间
(integer) 23

redis> EXPIRE cache_page 30000   # 更新过期时间
(integer) 1

redis> TTL cache_page
(integer) 29996
```

### 模式：导航会话

假设你有一项 web 服务，打算根据用户最近访问的 N 个页面来进行物品推荐，并且假设用户停止阅览超过 60 秒，那么就清空阅览记录(为了减少物品推荐的计算量，并且保持推荐物品的新鲜度)。

这些最近访问的页面记录，我们称之为『导航会话』(Navigation session)，可以用 [*INCR*](http://doc.redisfans.com/string/incr.html#incr) 和 [*RPUSH*](http://doc.redisfans.com/list/rpush.html#rpush) 命令在 Redis 中实现它：每当用户阅览一个网页的时候，执行以下代码：

```
MULTI
    RPUSH pagewviews.user:<userid> http://.....
    EXPIRE pagewviews.user:<userid> 60
EXEC
```

如果用户停止阅览超过 60 秒，那么它的导航会话就会被清空，当用户重新开始阅览的时候，系统又会重新记录导航会话，继续进行物品推荐。

## EXPIREAT

**EXPIREAT key timestamp**

[EXPIREAT](http://doc.redisfans.com/key/expireat.html#expireat) 的作用和 [*EXPIRE*](http://doc.redisfans.com/key/expire.html) 类似，都用于为 `key` 设置生存时间。

不同在于 [EXPIREAT](http://doc.redisfans.com/key/expireat.html#expireat) 命令接受的时间参数是 UNIX 时间戳(unix timestamp)。

- **可用版本：**

  >= 1.2.0

- **时间复杂度：**

  O(1)

- **返回值：**

  如果生存时间设置成功，返回 `1` 。当 `key` 不存在或没办法设置生存时间，返回 `0` 。

```
redis> SET cache www.google.com
OK

redis> EXPIREAT cache 1355292000     # 这个 key 将在 2012.12.12 过期
(integer) 1

redis> TTL cache
(integer) 45081860
```

## KEYS

**KEYS pattern**

查找所有符合给定模式 `pattern` 的 `key` 。

`KEYS *` 匹配数据库中所有 `key` 。

`KEYS h?llo` 匹配 `hello` ， `hallo` 和 `hxllo` 等。

`KEYS h*llo` 匹配 `hllo` 和 `heeeeello` 等。

`KEYS h[ae]llo` 匹配 `hello` 和 `hallo` ，但不匹配 `hillo` 。

特殊符号用 `\` 隔开

> [KEYS](http://doc.redisfans.com/key/keys.html#keys) 的速度非常快，但在一个大的数据库中使用它仍然可能造成性能问题，如果你需要从一个数据集中查找特定的 `key` ，你最好还是用 Redis 的集合结构(set)来代替。

- **可用版本：**

  >= 1.0.0

- **时间复杂度：**

  O(N)， `N` 为数据库中 `key` 的数量。

- **返回值：**

  符合给定模式的 `key` 列表。

### 示例

```
redis> MSET one 1 two 2 three 3 four 4  # 一次设置 4 个 key
OK

redis> KEYS *o*
1) "four"
2) "two"
3) "one"

redis> KEYS t??
1) "two"

redis> KEYS t[w]*
1) "two"

redis> KEYS *  # 匹配数据库内所有 key
1) "four"
2) "three"
3) "two"
4) "one"
```

## MIGRATE

**MIGRATE host port key destination-db timeout [COPY] [REPLACE]**

将 `key` 原子性地从当前实例传送到目标实例的指定数据库上，一旦传送成功， `key` 保证会出现在目标实例上，而当前实例上的 `key` 会被删除。

这个命令是一个原子操作，它在执行的时候会阻塞进行迁移的两个实例，直到以下任意结果发生：迁移成功，迁移失败，等到超时。

命令的内部实现是这样的：它在当前实例对给定 `key` 执行 [*DUMP*](http://doc.redisfans.com/key/dump.html) 命令 ，将它序列化，然后传送到目标实例，目标实例再使用 [*RESTORE*](http://doc.redisfans.com/key/restore.html) 对数据进行反序列化，并将反序列化所得的数据添加到数据库中；当前实例就像目标实例的客户端那样，只要看到 [*RESTORE*](http://doc.redisfans.com/key/restore.html) 命令返回 `OK` ，它就会调用 [*DEL*](http://doc.redisfans.com/key/del.html) 删除自己数据库上的 `key` 。

`timeout` 参数以毫秒为格式，指定当前实例和目标实例进行沟通的**最大间隔时间**。这说明操作并不一定要在 `timeout` 毫秒内完成，只是说数据传送的时间不能超过这个 `timeout` 数。

[MIGRATE](http://doc.redisfans.com/key/migrate.html#migrate) 命令需要在给定的时间规定内完成 IO 操作。如果在传送数据时发生 IO 错误，或者达到了超时时间，那么命令会停止执行，并返回一个特殊的错误： `IOERR` 。

当 `IOERR` 出现时，有以下两种可能：

- `key` 可能存在于两个实例
- `key` 可能只存在于当前实例

唯一不可能发生的情况就是丢失 `key` ，因此，如果一个客户端执行 [MIGRATE](http://doc.redisfans.com/key/migrate.html#migrate) 命令，并且不幸遇上 `IOERR` 错误，那么这个客户端唯一要做的就是检查自己数据库上的 `key` 是否已经被正确地删除。

如果有其他错误发生，那么 [MIGRATE](http://doc.redisfans.com/key/migrate.html#migrate) 保证 `key` 只会出现在当前实例中。（当然，目标实例的给定数据库上可能有和 `key` 同名的键，不过这和 [MIGRATE](http://doc.redisfans.com/key/migrate.html#migrate) 命令没有关系）。

**可选项：**

- `COPY` ：不移除源实例上的 `key` 。
- `REPLACE` ：替换目标实例上已存在的 `key` 。

- **可用版本：**

  > \>= 2.6.0

- **时间复杂度：**

  这个命令在源实例上实际执行 [*DUMP*](http://doc.redisfans.com/key/dump.html) 命令和 [*DEL*](http://doc.redisfans.com/key/del.html) 命令，在目标实例执行 [*RESTORE*](http://doc.redisfans.com/key/restore.html) 命令，查看以上命令的文档可以看到详细的复杂度说明。`key` 数据在两个实例之间传输的复杂度为 O(N) 。

- **返回值：**

  迁移成功时返回 `OK` ，否则返回相应的错误。

### 示例

先启动两个 Redis 实例，一个使用默认的 6379 端口，一个使用 7777 端口。

```
$ ./redis-server &
[1] 3557

...

$ ./redis-server --port 7777 &
[2] 3560

...
```

然后用客户端连上 6379 端口的实例，设置一个键，然后将它迁移到 7777 端口的实例上：

```
$ ./redis-cli

redis 127.0.0.1:6379> flushdb
OK

redis 127.0.0.1:6379> SET greeting "Hello from 6379 instance"
OK

redis 127.0.0.1:6379> MIGRATE 127.0.0.1 7777 greeting 0 1000
OK

redis 127.0.0.1:6379> EXISTS greeting                           # 迁移成功后 key 被删除
(integer) 0
```

使用另一个客户端，查看 7777 端口上的实例：

```
$ ./redis-cli -p 7777

redis 127.0.0.1:7777> GET greeting
"Hello from 6379 instance"
```

## MOVE

**MOVE key db**

将当前数据库的 `key` 移动到给定的数据库 `db` 当中。

如果当前数据库(源数据库)和给定数据库(目标数据库)有相同名字的给定 `key` ，或者 `key` 不存在于当前数据库，那么 `MOVE` 没有任何效果。

因此，也可以利用这一特性，将 [MOVE](http://doc.redisfans.com/key/move.html#move) 当作锁(locking)原语(primitive)。

- **可用版本：**

  >= 1.0.0

- **时间复杂度：**

  O(1)

- **返回值：**

  移动成功返回 `1` ，失败则返回 `0` 。

```
# key 存在于当前数据库

redis> SELECT 0                             # redis默认使用数据库 0，为了清晰起见，这里再显式指定一次。
OK

redis> SET song "secret base - Zone"
OK

redis> MOVE song 1                          # 将 song 移动到数据库 1
(integer) 1

redis> EXISTS song                          # song 已经被移走
(integer) 0

redis> SELECT 1                             # 使用数据库 1
OK

redis:1> EXISTS song                        # 证实 song 被移到了数据库 1 (注意命令提示符变成了"redis:1"，表明正在使用数据库 1)
(integer) 1


# 当 key 不存在的时候

redis:1> EXISTS fake_key
(integer) 0

redis:1> MOVE fake_key 0                    # 试图从数据库 1 移动一个不存在的 key 到数据库 0，失败
(integer) 0

redis:1> select 0                           # 使用数据库0
OK

redis> EXISTS fake_key                      # 证实 fake_key 不存在
(integer) 0


# 当源数据库和目标数据库有相同的 key 时

redis> SELECT 0                             # 使用数据库0
OK
redis> SET favorite_fruit "banana"
OK

redis> SELECT 1                             # 使用数据库1
OK
redis:1> SET favorite_fruit "apple"
OK

redis:1> SELECT 0                           # 使用数据库0，并试图将 favorite_fruit 移动到数据库 1
OK

redis> MOVE favorite_fruit 1                # 因为两个数据库有相同的 key，MOVE 失败
(integer) 0

redis> GET favorite_fruit                   # 数据库 0 的 favorite_fruit 没变
"banana"

redis> SELECT 1
OK

redis:1> GET favorite_fruit                 # 数据库 1 的 favorite_fruit 也是
"apple"
```

## OBJECT

**OBJECT subcommand [arguments [arguments]]**

[OBJECT](http://doc.redisfans.com/key/object.html#object) 命令允许从内部察看给定 `key` 的 Redis 对象。

它通常用在除错(debugging)或者了解为了节省空间而对 `key` 使用特殊编码的情况。

当将Redis用作缓存程序时，你也可以通过 [OBJECT](http://doc.redisfans.com/key/object.html#object) 命令中的信息，决定 `key` 的驱逐策略(eviction policies)。

OBJECT 命令有多个子命令：

- `OBJECT REFCOUNT ` 返回给定 `key` 引用所储存的值的次数。此命令主要用于除错。
- `OBJECT ENCODING ` 返回给定 `key` 锁储存的值所使用的内部表示(representation)。
- `OBJECT IDLETIME ` 返回给定 `key` 自储存以来的空转时间(idle， 没有被读取也没有被写入)，以秒为单位。

对象可以以多种方式编码：

- 字符串可以被编码为 `raw` (一般字符串)或 `int` (用字符串表示64位数字是为了节约空间)。
- 列表可以被编码为 `ziplist` 或 `linkedlist` 。 `ziplist` 是为节约大小较小的列表空间而作的特殊表示。
- 集合可以被编码为 `intset` 或者 `hashtable` 。 `intset` 是只储存数字的小集合的特殊表示。
- 哈希表可以编码为 `zipmap` 或者 `hashtable` 。 `zipmap` 是小哈希表的特殊表示。
- 有序集合可以被编码为 `ziplist` 或者 `skiplist` 格式。 `ziplist` 用于表示小的有序集合，而 `skiplist` 则用于表示任何大小的有序集合。

假如你做了什么让 Redis 没办法再使用节省空间的编码时(比如将一个只有 1 个元素的集合扩展为一个有 100 万个元素的集合)，特殊编码类型(specially encoded types)会自动转换成通用类型(general type)。

- **可用版本：**

  >= 2.2.3

- **时间复杂度：**

  O(1)

- **返回值：**

  `REFCOUNT` 和 `IDLETIME` 返回数字。`ENCODING` 返回相应的编码类型。

```
redis> SET game "COD"           # 设置一个字符串
OK

redis> OBJECT REFCOUNT game     # 只有一个引用
(integer) 1

redis> OBJECT IDLETIME game     # 等待一阵。。。然后查看空转时间
(integer) 90

redis> GET game                 # 提取game， 让它处于活跃(active)状态
"COD"

redis> OBJECT IDLETIME game     # 不再处于空转
(integer) 0

redis> OBJECT ENCODING game     # 字符串的编码方式
"raw"

redis> SET phone 15820123123    # 大的数字也被编码为字符串
OK

redis> OBJECT ENCODING phone
"raw"

redis> SET age 20               # 短数字被编码为 int
OK

redis> OBJECT ENCODING age
"int"
```

## PERSIST

**PERSIST key**

移除给定 `key` 的生存时间，将这个 `key` 从『易失的』(带生存时间 `key` )转换成『持久的』(一个不带生存时间、永不过期的 `key` )。

- **可用版本：**

  >= 2.2.0

- **时间复杂度：**

  O(1)

- **返回值：**

  当生存时间移除成功时，返回 `1` .如果 `key` 不存在或 `key` 没有设置生存时间，返回 `0` 。

```
redis> SET mykey "Hello"
OK

redis> EXPIRE mykey 10  # 为 key 设置生存时间
(integer) 1

redis> TTL mykey
(integer) 10

redis> PERSIST mykey    # 移除 key 的生存时间
(integer) 1

redis> TTL mykey
(integer) -1
```

## PEXPIRE

**PEXPIRE key milliseconds**

这个命令和 [*EXPIRE*](http://doc.redisfans.com/key/expire.html) 命令的作用类似，但是它以毫秒为单位设置 `key` 的生存时间，而不像 [*EXPIRE*](http://doc.redisfans.com/key/expire.html) 命令那样，以秒为单位。

- **可用版本：**

  >= 2.6.0

- **时间复杂度：**

  O(1)

- **返回值：**

  设置成功，返回 `1``key` 不存在或设置失败，返回 `0`

```
redis> SET mykey "Hello"
OK

redis> PEXPIRE mykey 1500
(integer) 1

redis> TTL mykey    # TTL 的返回值以秒为单位
(integer) 2

redis> PTTL mykey   # PTTL 可以给出准确的毫秒数
(integer) 1499
```

## PEXPIREAT

**PEXPIREAT key milliseconds-timestamp**

这个命令和 [*EXPIREAT*](http://doc.redisfans.com/key/expireat.html) 命令类似，但它以毫秒为单位设置 `key` 的过期 unix 时间戳，而不是像 [*EXPIREAT*](http://doc.redisfans.com/key/expireat.html) 那样，以秒为单位。

- **可用版本：**

  >= 2.6.0

- **时间复杂度：**

  O(1)

- **返回值：**

  如果生存时间设置成功，返回 `1` 。当 `key` 不存在或没办法设置生存时间时，返回 `0` 。(查看 [*EXPIRE*](http://doc.redisfans.com/key/expire.html) 命令获取更多信息)

```
redis> SET mykey "Hello"
OK

redis> PEXPIREAT mykey 1555555555005
(integer) 1

redis> TTL mykey           # TTL 返回秒
(integer) 223157079

redis> PTTL mykey          # PTTL 返回毫秒
(integer) 223157079318
```

## PTTL

**PTTL key**

这个命令类似于 [*TTL*](http://doc.redisfans.com/key/ttl.html#ttl) 命令，但它以毫秒为单位返回 `key` 的剩余生存时间，而不是像 [*TTL*](http://doc.redisfans.com/key/ttl.html#ttl) 命令那样，以秒为单位。

- **可用版本：**

  >= 2.6.0

- **复杂度：**

  O(1)

- **返回值：**

  当 `key` 不存在时，返回 `-2` 。当 `key` 存在但没有设置剩余生存时间时，返回 `-1` 。否则，以毫秒为单位，返回 `key` 的剩余生存时间。

在 Redis 2.8 以前，当 `key` 不存在，或者 `key` 没有设置剩余生存时间时，命令都返回 `-1` 。

```
# 不存在的 key

redis> FLUSHDB
OK

redis> PTTL key
(integer) -2


# key 存在，但没有设置剩余生存时间

redis> SET key value
OK

redis> PTTL key
(integer) -1


# 有剩余生存时间的 key

redis> PEXPIRE key 10086
(integer) 1

redis> PTTL key
(integer) 6179
```

## RANDOMKEY

**RANDOMKEY**

从当前数据库中随机返回(不删除)一个 `key` 。

- **可用版本：**

  >= 1.0.0

- **时间复杂度：**

  O(1)

- **返回值：**

  当数据库不为空时，返回一个 `key` 。当数据库为空时，返回 `nil` 。

```
# 数据库不为空

redis> MSET fruit "apple" drink "beer" food "cookies"   # 设置多个 key
OK

redis> RANDOMKEY
"fruit"

redis> RANDOMKEY
"food"

redis> KEYS *    # 查看数据库内所有key，证明 RANDOMKEY 并不删除 key
1) "food"
2) "drink"
3) "fruit"


# 数据库为空

redis> FLUSHDB  # 删除当前数据库所有 key
OK

redis> RANDOMKEY
(nil)
```

## RENAME

**RENAME key newkey**

将 `key` 改名为 `newkey` 。

当 `key` 和 `newkey` 相同，或者 `key` 不存在时，返回一个错误。

当 `newkey` 已经存在时， [RENAME](http://doc.redisfans.com/key/rename.html#rename) 命令将覆盖旧值。

- **可用版本：**

  >= 1.0.0

- **时间复杂度：**

  O(1)

- **返回值：**

  改名成功时提示 `OK` ，失败时候返回一个错误。

```
# key 存在且 newkey 不存在

redis> SET message "hello world"
OK

redis> RENAME message greeting
OK

redis> EXISTS message               # message 不复存在
(integer) 0

redis> EXISTS greeting              # greeting 取而代之
(integer) 1


# 当 key 不存在时，返回错误

redis> RENAME fake_key never_exists
(error) ERR no such key


# newkey 已存在时， RENAME 会覆盖旧 newkey

redis> SET pc "lenovo"
OK

redis> SET personal_computer "dell"
OK

redis> RENAME pc personal_computer
OK

redis> GET pc
(nil)

redis:1> GET personal_computer      # 原来的值 dell 被覆盖了
"lenovo"
```

## RENAMENX

**RENAMENX key newkey**

当且仅当 `newkey` 不存在时，将 `key` 改名为 `newkey` 。

当 `key` 不存在时，返回一个错误。

- **可用版本：**

  >= 1.0.0

- **时间复杂度：**

  O(1)

- **返回值：**

  修改成功时，返回 `1` 。如果 `newkey` 已经存在，返回 `0` 。

```
# newkey 不存在，改名成功

redis> SET player "MPlyaer"
OK

redis> EXISTS best_player
(integer) 0

redis> RENAMENX player best_player
(integer) 1


# newkey存在时，失败

redis> SET animal "bear"
OK

redis> SET favorite_animal "butterfly"
OK

redis> RENAMENX animal favorite_animal
(integer) 0

redis> get animal
"bear"

redis> get favorite_animal
"butterfly"
```

## RESTORE

**RESTORE key ttl serialized-value**

反序列化给定的序列化值，并将它和给定的 `key` 关联。

参数 `ttl` 以毫秒为单位为 `key` 设置生存时间；如果 `ttl` 为 `0` ，那么不设置生存时间。

[RESTORE](http://doc.redisfans.com/key/restore.html#restore) 在执行反序列化之前会先对序列化值的 RDB 版本和数据校验和进行检查，如果 RDB 版本不相同或者数据不完整的话，那么 [RESTORE](http://doc.redisfans.com/key/restore.html#restore) 会拒绝进行反序列化，并返回一个错误。

更多信息可以参考 [*DUMP*](http://doc.redisfans.com/key/dump.html) 命令。

- **可用版本：**

  >= 2.6.0

- **时间复杂度：**

  查找给定键的复杂度为 O(1) ，对键进行反序列化的复杂度为 O(N*M) ，其中 N 是构成 `key` 的 Redis 对象的数量，而 M 则是这些对象的平均大小。有序集合(sorted set)的反序列化复杂度为 O(N*M*log(N)) ，因为有序集合每次插入的复杂度为 O(log(N)) 。如果反序列化的对象是比较小的字符串，那么复杂度为 O(1) 。

- **返回值：**

  如果反序列化成功那么返回 `OK` ，否则返回一个错误。

```
redis> SET greeting "hello, dumping world!"
OK

redis> DUMP greeting
"\x00\x15hello, dumping world!\x06\x00E\xa0Z\x82\xd8r\xc1\xde"

redis> RESTORE greeting-again 0 "\x00\x15hello, dumping world!\x06\x00E\xa0Z\x82\xd8r\xc1\xde"
OK

redis> GET greeting-again
"hello, dumping world!"

redis> RESTORE fake-message 0 "hello moto moto blah blah"   ; 使用错误的值进行反序列化
(error) ERR DUMP payload version or checksum are wrong
```

## SORT

**SORT key [BY pattern] [LIMIT offset count] [GET pattern [GET pattern ...]] [ASC | DESC] [ALPHA] [STORE destination]**

返回或保存给定列表、集合、有序集合 `key` 中经过排序的元素。

排序默认以数字作为对象，值被解释为双精度浮点数，然后进行比较。

### 一般 SORT 用法

最简单的 [SORT](http://doc.redisfans.com/key/sort.html#sort) 使用方法是 `SORT key` 和 `SORT key DESC` ：

- `SORT key` 返回键值从小到大排序的结果。
- `SORT key DESC` 返回键值从大到小排序的结果。

假设 `today_cost` 列表保存了今日的开销金额， 那么可以用 [SORT](http://doc.redisfans.com/key/sort.html#sort) 命令对它进行排序：

```
# 开销金额列表

redis> LPUSH today_cost 30 1.5 10 8
(integer) 4

# 排序

redis> SORT today_cost
1) "1.5"
2) "8"
3) "10"
4) "30"

# 逆序排序

redis 127.0.0.1:6379> SORT today_cost DESC
1) "30"
2) "10"
3) "8"
4) "1.5"
```

### 使用 ALPHA 修饰符对字符串进行排序

因为 [SORT](http://doc.redisfans.com/key/sort.html#sort) 命令默认排序对象为数字， 当需要对字符串进行排序时， 需要显式地在 [SORT](http://doc.redisfans.com/key/sort.html#sort) 命令之后添加 `ALPHA` 修饰符：

```
# 网址

redis> LPUSH website "www.reddit.com"
(integer) 1

redis> LPUSH website "www.slashdot.com"
(integer) 2

redis> LPUSH website "www.infoq.com"
(integer) 3

# 默认（按数字）排序

redis> SORT website
1) "www.infoq.com"
2) "www.slashdot.com"
3) "www.reddit.com"

# 按字符排序

redis> SORT website ALPHA
1) "www.infoq.com"
2) "www.reddit.com"
3) "www.slashdot.com"
```

如果系统正确地设置了 `LC_COLLATE` 环境变量的话，Redis能识别 `UTF-8` 编码。

### 使用 LIMIT 修饰符限制返回结果

排序之后返回元素的数量可以通过 `LIMIT` 修饰符进行限制， 修饰符接受 `offset` 和 `count` 两个参数：

- `offset` 指定要跳过的元素数量。
- `count` 指定跳过 `offset` 个指定的元素之后，要返回多少个对象。

以下例子返回排序结果的前 5 个对象( `offset` 为 `0` 表示没有元素被跳过)。

```
# 添加测试数据，列表值为 1 指 10

redis 127.0.0.1:6379> RPUSH rank 1 3 5 7 9
(integer) 5

redis 127.0.0.1:6379> RPUSH rank 2 4 6 8 10
(integer) 10

# 返回列表中最小的 5 个值

redis 127.0.0.1:6379> SORT rank LIMIT 0 5
1) "1"
2) "2"
3) "3"
4) "4"
5) "5"
```

可以组合使用多个修饰符。以下例子返回从大到小排序的前 5 个对象。

```
redis 127.0.0.1:6379> SORT rank LIMIT 0 5 DESC
1) "10"
2) "9"
3) "8"
4) "7"
5) "6"
```

### 使用外部 key 进行排序

可以使用外部 `key` 的数据作为权重，代替默认的直接对比键值的方式来进行排序。

假设现在有用户数据如下：

| uid  | user_name_{uid} | user_level_{uid} |
| :--- | :-------------- | :--------------- |
| 1    | admin           | 9999             |
| 2    | jack            | 10               |
| 3    | peter           | 25               |
| 4    | mary            | 70               |

以下代码将数据输入到 Redis 中：

```
# admin

redis 127.0.0.1:6379> LPUSH uid 1
(integer) 1

redis 127.0.0.1:6379> SET user_name_1 admin
OK

redis 127.0.0.1:6379> SET user_level_1 9999
OK

# jack

redis 127.0.0.1:6379> LPUSH uid 2
(integer) 2

redis 127.0.0.1:6379> SET user_name_2 jack
OK

redis 127.0.0.1:6379> SET user_level_2 10
OK

# peter

redis 127.0.0.1:6379> LPUSH uid 3
(integer) 3

redis 127.0.0.1:6379> SET user_name_3 peter
OK

redis 127.0.0.1:6379> SET user_level_3 25
OK

# mary

redis 127.0.0.1:6379> LPUSH uid 4
(integer) 4

redis 127.0.0.1:6379> SET user_name_4 mary
OK

redis 127.0.0.1:6379> SET user_level_4 70
OK
```

#### BY 选项

默认情况下， `SORT uid` 直接按 `uid` 中的值排序：

```
redis 127.0.0.1:6379> SORT uid
1) "1"      # admin
2) "2"      # jack
3) "3"      # peter
4) "4"      # mary
```

通过使用 `BY` 选项，可以让 `uid` 按其他键的元素来排序。

比如说， 以下代码让 `uid` 键按照 `user_level_{uid}` 的大小来排序：

```
redis 127.0.0.1:6379> SORT uid BY user_level_*
1) "2"      # jack , level = 10
2) "3"      # peter, level = 25
3) "4"      # mary, level = 70
4) "1"      # admin, level = 9999
```

`user_level_*` 是一个占位符， 它先取出 `uid` 中的值， 然后再用这个值来查找相应的键。

比如在对 `uid` 列表进行排序时， 程序就会先取出 `uid` 的值 `1` 、 `2` 、 `3` 、 `4` ， 然后使用 `user_level_1` 、 `user_level_2` 、 `user_level_3` 和 `user_level_4` 的值作为排序 `uid` 的权重。

#### GET 选项

使用 `GET` 选项， 可以根据排序的结果来取出相应的键值。

比如说， 以下代码先排序 `uid` ， 再取出键 `user_name_{uid}` 的值：

```
redis 127.0.0.1:6379> SORT uid GET user_name_*
1) "admin"
2) "jack"
3) "peter"
4) "mary"
```

#### 组合使用 BY 和 GET

通过组合使用 `BY` 和 `GET` ， 可以让排序结果以更直观的方式显示出来。

比如说， 以下代码先按 `user_level_{uid}` 来排序 `uid` 列表， 再取出相应的 `user_name_{uid}` 的值：

```
redis 127.0.0.1:6379> SORT uid BY user_level_* GET user_name_*
1) "jack"       # level = 10
2) "peter"      # level = 25
3) "mary"       # level = 70
4) "admin"      # level = 9999
```

现在的排序结果要比只使用 `SORT uid BY user_level_*` 要直观得多。

#### 获取多个外部键

可以同时使用多个 `GET` 选项， 获取多个外部键的值。

以下代码就按 `uid` 分别获取 `user_level_{uid}` 和 `user_name_{uid}` ：

```
redis 127.0.0.1:6379> SORT uid GET user_level_* GET user_name_*
1) "9999"       # level
2) "admin"      # name
3) "10"
4) "jack"
5) "25"
6) "peter"
7) "70"
8) "mary"
```

`GET` 有一个额外的参数规则，那就是 —— 可以用 `#` 获取被排序键的值。

以下代码就将 `uid` 的值、及其相应的 `user_level_*` 和 `user_name_*` 都返回为结果：

```
redis 127.0.0.1:6379> SORT uid GET # GET user_level_* GET user_name_*
1) "1"          # uid
2) "9999"       # level
3) "admin"      # name
4) "2"
5) "10"
6) "jack"
7) "3"
8) "25"
9) "peter"
10) "4"
11) "70"
12) "mary"
```

#### 获取外部键，但不进行排序

通过将一个不存在的键作为参数传给 `BY` 选项， 可以让 `SORT` 跳过排序操作， 直接返回结果：

```
redis 127.0.0.1:6379> SORT uid BY not-exists-key
1) "4"
2) "3"
3) "2"
4) "1"
```

这种用法在单独使用时，没什么实际用处。

不过，通过将这种用法和 `GET` 选项配合， 就可以在不排序的情况下， 获取多个外部键， 相当于执行一个整合的获取操作（类似于 SQL 数据库的 `join` 关键字）。

以下代码演示了，如何在不引起排序的情况下，使用 `SORT` 、 `BY` 和 `GET` 获取多个外部键：

```
redis 127.0.0.1:6379> SORT uid BY not-exists-key GET # GET user_level_* GET user_name_*
1) "4"      # id
2) "70"     # level
3) "mary"   # name
4) "3"
5) "25"
6) "peter"
7) "2"
8) "10"
9) "jack"
10) "1"
11) "9999"
12) "admin"
```

#### 将哈希表作为 GET 或 BY 的参数

除了可以将字符串键之外， 哈希表也可以作为 `GET` 或 `BY` 选项的参数来使用。

比如说，对于前面给出的用户信息表：

| uid  | user_name_{uid} | user_level_{uid} |
| :--- | :-------------- | :--------------- |
| 1    | admin           | 9999             |
| 2    | jack            | 10               |
| 3    | peter           | 25               |
| 4    | mary            | 70               |

我们可以不将用户的名字和级别保存在 `user_name_{uid}` 和 `user_level_{uid}` 两个字符串键中， 而是用一个带有 `name` 域和 `level` 域的哈希表 `user_info_{uid}` 来保存用户的名字和级别信息：

```
redis 127.0.0.1:6379> HMSET user_info_1 name admin level 9999
OK

redis 127.0.0.1:6379> HMSET user_info_2 name jack level 10
OK

redis 127.0.0.1:6379> HMSET user_info_3 name peter level 25
OK

redis 127.0.0.1:6379> HMSET user_info_4 name mary level 70
OK
```

之后， `BY` 和 `GET` 选项都可以用 `key->field` 的格式来获取哈希表中的域的值， 其中 `key` 表示哈希表键， 而 `field` 则表示哈希表的域：

```
redis 127.0.0.1:6379> SORT uid BY user_info_*->level
1) "2"
2) "3"
3) "4"
4) "1"

redis 127.0.0.1:6379> SORT uid BY user_info_*->level GET user_info_*->name
1) "jack"
2) "peter"
3) "mary"
4) "admin"
```

### 保存排序结果

默认情况下， [SORT](http://doc.redisfans.com/key/sort.html#sort) 操作只是简单地返回排序结果，并不进行任何保存操作。

通过给 `STORE` 选项指定一个 `key` 参数，可以将排序结果保存到给定的键上。

如果被指定的 `key` 已存在，那么原有的值将被排序结果覆盖。

```
# 测试数据

redis 127.0.0.1:6379> RPUSH numbers 1 3 5 7 9
(integer) 5

redis 127.0.0.1:6379> RPUSH numbers 2 4 6 8 10
(integer) 10

redis 127.0.0.1:6379> LRANGE numbers 0 -1
1) "1"
2) "3"
3) "5"
4) "7"
5) "9"
6) "2"
7) "4"
8) "6"
9) "8"
10) "10"

redis 127.0.0.1:6379> SORT numbers STORE sorted-numbers
(integer) 10

# 排序后的结果

redis 127.0.0.1:6379> LRANGE sorted-numbers 0 -1
1) "1"
2) "2"
3) "3"
4) "4"
5) "5"
6) "6"
7) "7"
8) "8"
9) "9"
10) "10"
```

可以通过将 [SORT](http://doc.redisfans.com/key/sort.html#sort) 命令的执行结果保存，并用 [*EXPIRE*](http://doc.redisfans.com/key/expire.html#expire) 为结果设置生存时间，以此来产生一个 [SORT](http://doc.redisfans.com/key/sort.html#sort) 操作的结果缓存。

这样就可以避免对 [SORT](http://doc.redisfans.com/key/sort.html#sort) 操作的频繁调用：只有当结果集过期时，才需要再调用一次 [SORT](http://doc.redisfans.com/key/sort.html#sort) 操作。

另外，为了正确实现这一用法，你可能需要加锁以避免多个客户端同时进行缓存重建(也就是多个客户端，同一时间进行 [SORT](http://doc.redisfans.com/key/sort.html#sort) 操作，并保存为结果集)，具体参见 [*SETNX*](http://doc.redisfans.com/string/setnx.html#setnx) 命令。

- **可用版本：**

  >= 1.0.0

- **时间复杂度：**

  O(N+M*log(M))， `N` 为要排序的列表或集合内的元素数量， `M` 为要返回的元素数量。如果只是使用 [SORT](http://doc.redisfans.com/key/sort.html#sort) 命令的 `GET` 选项获取数据而没有进行排序，时间复杂度 O(N)。

- **返回值：**

  没有使用 `STORE` 参数，返回列表形式的排序结果。使用 `STORE` 参数，返回排序结果的元素数量。

## TTL

**TTL key**

以秒为单位，返回给定 `key` 的剩余生存时间(TTL, time to live)。

- **可用版本：**

  >= 1.0.0

- **时间复杂度：**

  O(1)

- **返回值：**

  当 `key` 不存在时，返回 `-2` 。当 `key` 存在但没有设置剩余生存时间时，返回 `-1` 。否则，以秒为单位，返回 `key` 的剩余生存时间。

在 Redis 2.8 以前，当 `key` 不存在，或者 `key` 没有设置剩余生存时间时，命令都返回 `-1` 。

```
# 不存在的 key

redis> FLUSHDB
OK

redis> TTL key
(integer) -2


# key 存在，但没有设置剩余生存时间

redis> SET key value
OK

redis> TTL key
(integer) -1


# 有剩余生存时间的 key

redis> EXPIRE key 10086
(integer) 1

redis> TTL key
(integer) 10084
```

## TYPE

**TYPE key**

返回 `key` 所储存的值的类型。

- **可用版本：**

  >= 1.0.0

- **时间复杂度：**

  O(1)

- **返回值：**

  `none` (key不存在)`string` (字符串)`list` (列表)`set` (集合)`zset` (有序集)`hash` (哈希表)

```
# 字符串
redis> SET weather "sunny"
OK
redis> TYPE weather
string

# 列表
redis> LPUSH book_list "programming in scala"
(integer) 1
redis> TYPE book_list
list

# 集合
redis> SADD pat "dog"
(integer) 1
redis> TYPE pat
set
```

## SCAN

**SCAN cursor [MATCH pattern] [COUNT count]**

[*SCAN*](http://doc.redisfans.com/key/scan.html#scan) 命令及其相关的 [*SSCAN*](http://doc.redisfans.com/set/sscan.html#sscan) 命令、 [*HSCAN*](http://doc.redisfans.com/hash/hscan.html#hscan) 命令和 [*ZSCAN*](http://doc.redisfans.com/sorted_set/zscan.html#zscan) 命令都用于增量地迭代（incrementally iterate）一集元素（a collection of elements）：

- [*SCAN*](http://doc.redisfans.com/key/scan.html#scan) 命令用于迭代当前数据库中的数据库键。
- [*SSCAN*](http://doc.redisfans.com/set/sscan.html#sscan) 命令用于迭代集合键中的元素。
- [*HSCAN*](http://doc.redisfans.com/hash/hscan.html#hscan) 命令用于迭代哈希键中的键值对。
- [*ZSCAN*](http://doc.redisfans.com/sorted_set/zscan.html#zscan) 命令用于迭代有序集合中的元素（包括元素成员和元素分值）。

以上列出的四个命令都支持增量式迭代， 它们每次执行都只会返回少量元素， 所以这些命令可以用于生产环境， 而不会出现像 [*KEYS*](http://doc.redisfans.com/key/keys.html#keys) 命令、 [*SMEMBERS*](http://doc.redisfans.com/set/smembers.html#smembers) 命令带来的问题 —— 当 [*KEYS*](http://doc.redisfans.com/key/keys.html#keys) 命令被用于处理一个大的数据库时， 又或者 [*SMEMBERS*](http://doc.redisfans.com/set/smembers.html#smembers) 命令被用于处理一个大的集合键时， 它们可能会阻塞服务器达数秒之久。

不过， 增量式迭代命令也不是没有缺点的： 举个例子， 使用 [*SMEMBERS*](http://doc.redisfans.com/set/smembers.html#smembers) 命令可以返回集合键当前包含的所有元素， 但是对于 [*SCAN*](http://doc.redisfans.com/key/scan.html#scan) 这类增量式迭代命令来说， 因为在对键进行增量式迭代的过程中， 键可能会被修改， 所以增量式迭代命令只能对被返回的元素提供有限的保证 （offer limited guarantees about the returned elements）。

因为 [*SCAN*](http://doc.redisfans.com/key/scan.html#scan) 、 [*SSCAN*](http://doc.redisfans.com/set/sscan.html#sscan) 、 [*HSCAN*](http://doc.redisfans.com/hash/hscan.html#hscan) 和 [*ZSCAN*](http://doc.redisfans.com/sorted_set/zscan.html#zscan) 四个命令的工作方式都非常相似， 所以这个文档会一并介绍这四个命令， 但是要记住：

- [*SSCAN*](http://doc.redisfans.com/set/sscan.html#sscan) 命令、 [*HSCAN*](http://doc.redisfans.com/hash/hscan.html#hscan) 命令和 [*ZSCAN*](http://doc.redisfans.com/sorted_set/zscan.html#zscan) 命令的第一个参数总是一个数据库键。
- 而 [*SCAN*](http://doc.redisfans.com/key/scan.html#scan) 命令则不需要在第一个参数提供任何数据库键 —— 因为它迭代的是当前数据库中的所有数据库键。

### SCAN 命令的基本用法

[*SCAN*](http://doc.redisfans.com/key/scan.html#scan) 命令是一个基于游标的迭代器（cursor based iterator）： [*SCAN*](http://doc.redisfans.com/key/scan.html#scan) 命令每次被调用之后， 都会向用户返回一个新的游标， 用户在下次迭代时需要使用这个新游标作为 [*SCAN*](http://doc.redisfans.com/key/scan.html#scan) 命令的游标参数， 以此来延续之前的迭代过程。

当 [*SCAN*](http://doc.redisfans.com/key/scan.html#scan) 命令的游标参数被设置为 `0` 时， 服务器将开始一次新的迭代， 而当服务器向用户返回值为 `0` 的游标时， 表示迭代已结束。

以下是一个 [*SCAN*](http://doc.redisfans.com/key/scan.html#scan) 命令的迭代过程示例：

```
redis 127.0.0.1:6379> scan 0
1) "17"
2)  1) "key:12"
    2) "key:8"
    3) "key:4"
    4) "key:14"
    5) "key:16"
    6) "key:17"
    7) "key:15"
    8) "key:10"
    9) "key:3"
    10) "key:7"
    11) "key:1"

redis 127.0.0.1:6379> scan 17
1) "0"
2) 1) "key:5"
   2) "key:18"
   3) "key:0"
   4) "key:2"
   5) "key:19"
   6) "key:13"
   7) "key:6"
   8) "key:9"
   9) "key:11"
```

在上面这个例子中， 第一次迭代使用 `0` 作为游标， 表示开始一次新的迭代。

第二次迭代使用的是第一次迭代时返回的游标， 也即是命令回复第一个元素的值 —— `17` 。

从上面的示例可以看到， [*SCAN*](http://doc.redisfans.com/key/scan.html#scan) 命令的回复是一个包含两个元素的数组， 第一个数组元素是用于进行下一次迭代的新游标， 而第二个数组元素则是一个数组， 这个数组中包含了所有被迭代的元素。

在第二次调用 [*SCAN*](http://doc.redisfans.com/key/scan.html#scan) 命令时， 命令返回了游标 `0` ， 这表示迭代已经结束， 整个数据集（collection）已经被完整遍历过了。

以 `0` 作为游标开始一次新的迭代， 一直调用 [*SCAN*](http://doc.redisfans.com/key/scan.html#scan) 命令， 直到命令返回游标 `0` ， 我们称这个过程为一次**完整遍历**（full iteration）。

### SCAN 命令的保证（guarantees）

[*SCAN*](http://doc.redisfans.com/key/scan.html#scan) 命令， 以及其他增量式迭代命令， 在进行完整遍历的情况下可以为用户带来以下保证： 从完整遍历开始直到完整遍历结束期间， 一直存在于数据集内的所有元素都会被完整遍历返回； 这意味着， 如果有一个元素， 它从遍历开始直到遍历结束期间都存在于被遍历的数据集当中， 那么 [*SCAN*](http://doc.redisfans.com/key/scan.html#scan) 命令总会在某次迭代中将这个元素返回给用户。

然而因为增量式命令仅仅使用游标来记录迭代状态， 所以这些命令带有以下缺点：

- 同一个元素可能会被返回多次。 处理重复元素的工作交由应用程序负责， 比如说， 可以考虑将迭代返回的元素仅仅用于可以安全地重复执行多次的操作上。
- 如果一个元素是在迭代过程中被添加到数据集的， 又或者是在迭代过程中从数据集中被删除的， 那么这个元素可能会被返回， 也可能不会， 这是未定义的（undefined）。

### SCAN 命令每次执行返回的元素数量

增量式迭代命令并不保证每次执行都返回某个给定数量的元素。

增量式命令甚至可能会返回零个元素， 但只要命令返回的游标不是 `0` ， 应用程序就不应该将迭代视作结束。

不过命令返回的元素数量总是符合一定规则的， 在实际中：

- 对于一个大数据集来说， 增量式迭代命令每次最多可能会返回数十个元素；
- 而对于一个足够小的数据集来说， 如果这个数据集的底层表示为编码数据结构（encoded data structure，适用于是小集合键、小哈希键和小有序集合键）， 那么增量迭代命令将在一次调用中返回数据集中的所有元素。

最后， 用户可以通过增量式迭代命令提供的 `COUNT` 选项来指定每次迭代返回元素的最大值。

### COUNT 选项

虽然增量式迭代命令不保证每次迭代所返回的元素数量， 但我们可以使用 `COUNT` 选项， 对命令的行为进行一定程度上的调整。

基本上， `COUNT` 选项的作用就是让用户告知迭代命令， 在每次迭代中应该从数据集里返回多少元素。

虽然 `COUNT` 选项**只是对增量式迭代命令的一种提示**（hint）， 但是在大多数情况下， 这种提示都是有效的。

- `COUNT` 参数的默认值为 `10` 。
- 在迭代一个足够大的、由哈希表实现的数据库、集合键、哈希键或者有序集合键时， 如果用户没有使用 `MATCH` 选项， 那么命令返回的元素数量通常和 `COUNT` 选项指定的一样， 或者比 `COUNT` 选项指定的数量稍多一些。
- 在迭代一个编码为整数集合（intset，一个只由整数值构成的小集合）、 或者编码为压缩列表（ziplist，由不同值构成的一个小哈希或者一个小有序集合）时， 增量式迭代命令通常会无视 `COUNT` 选项指定的值， 在第一次迭代就将数据集包含的所有元素都返回给用户。

**并非每次迭代都要使用相同的** `COUNT` **值。**

用户可以在每次迭代中按自己的需要随意改变 `COUNT` 值， 只要记得将上次迭代返回的游标用到下次迭代里面就可以了。

### MATCH 选项

和 [*KEYS*](http://doc.redisfans.com/key/keys.html#keys) 命令一样， 增量式迭代命令也可以通过提供一个 glob 风格的模式参数， 让命令只返回和给定模式相匹配的元素， 这一点可以通过在执行增量式迭代命令时， 通过给定 `MATCH ` 参数来实现。

以下是一个使用 `MATCH` 选项进行迭代的示例：

```
redis 127.0.0.1:6379> sadd myset 1 2 3 foo foobar feelsgood
(integer) 6

redis 127.0.0.1:6379> sscan myset 0 match f*
1) "0"
2) 1) "foo"
   2) "feelsgood"
   3) "foobar"
```

需要注意的是， 对元素的模式匹配工作是在命令从数据集中取出元素之后， 向客户端返回元素之前的这段时间内进行的， 所以如果被迭代的数据集中只有少量元素和模式相匹配， 那么迭代命令或许会在多次执行中都不返回任何元素。

以下是这种情况的一个例子：

```
redis 127.0.0.1:6379> scan 0 MATCH *11*
1) "288"
2) 1) "key:911"

redis 127.0.0.1:6379> scan 288 MATCH *11*
1) "224"
2) (empty list or set)

redis 127.0.0.1:6379> scan 224 MATCH *11*
1) "80"
2) (empty list or set)

redis 127.0.0.1:6379> scan 80 MATCH *11*
1) "176"
2) (empty list or set)

redis 127.0.0.1:6379> scan 176 MATCH *11* COUNT 1000
1) "0"
2)  1) "key:611"
    2) "key:711"
    3) "key:118"
    4) "key:117"
    5) "key:311"
    6) "key:112"
    7) "key:111"
    8) "key:110"
    9) "key:113"
   10) "key:211"
   11) "key:411"
   12) "key:115"
   13) "key:116"
   14) "key:114"
   15) "key:119"
   16) "key:811"
   17) "key:511"
   18) "key:11"
```

如你所见， 以上的大部分迭代都不返回任何元素。

在最后一次迭代， 我们通过将 `COUNT` 选项的参数设置为 `1000` ， 强制命令为本次迭代扫描更多元素， 从而使得命令返回的元素也变多了。

### 并发执行多个迭代

在同一时间， 可以有任意多个客户端对同一数据集进行迭代， 客户端每次执行迭代都需要传入一个游标， 并在迭代执行之后获得一个新的游标， 而这个游标就包含了迭代的所有状态， 因此， 服务器无须为迭代记录任何状态。

### 中途停止迭代

因为迭代的所有状态都保存在游标里面， 而服务器无须为迭代保存任何状态， 所以客户端可以在中途停止一个迭代， 而无须对服务器进行任何通知。

即使有任意数量的迭代在中途停止， 也不会产生任何问题。

### 使用错误的游标进行增量式迭代

使用间断的（broken）、负数、超出范围或者其他非正常的游标来执行增量式迭代并不会造成服务器崩溃， 但可能会让命令产生未定义的行为。

未定义行为指的是， 增量式命令对返回值所做的保证可能会不再为真。

只有两种游标是合法的：

1. 在开始一个新的迭代时， 游标必须为 `0` 。
2. 增量式迭代命令在执行之后返回的， 用于延续（continue）迭代过程的游标。

### 迭代终结的保证

增量式迭代命令所使用的算法只保证在数据集的大小有界（bounded）的情况下， 迭代才会停止， 换句话说， 如果被迭代数据集的大小不断地增长的话， 增量式迭代命令可能永远也无法完成一次完整迭代。

从直觉上可以看出， 当一个数据集不断地变大时， 想要访问这个数据集中的所有元素就需要做越来越多的工作， 能否结束一个迭代取决于用户执行迭代的速度是否比数据集增长的速度更快。

**可用版本：**

> \>= 2.8.0

**时间复杂度：**

> 增量式迭代命令每次执行的复杂度为 O(1) ， 对数据集进行一次完整迭代的复杂度为 O(N) ， 其中 N 为数据集中的元素数量。

**返回值：**

> [*SCAN*](http://doc.redisfans.com/key/scan.html#scan) 命令、 [*SSCAN*](http://doc.redisfans.com/set/sscan.html#sscan) 命令、 [*HSCAN*](http://doc.redisfans.com/hash/hscan.html#hscan) 命令和 [*ZSCAN*](http://doc.redisfans.com/sorted_set/zscan.html#zscan) 命令都返回一个包含两个元素的 multi-bulk 回复： 回复的第一个元素是字符串表示的无符号 64 位整数（游标）， 回复的第二个元素是另一个 multi-bulk 回复， 这个 multi-bulk 回复包含了本次被迭代的元素。
>
> [*SCAN*](http://doc.redisfans.com/key/scan.html#scan) 命令返回的每个元素都是一个数据库键。
>
> [*SSCAN*](http://doc.redisfans.com/set/sscan.html#sscan) 命令返回的每个元素都是一个集合成员。
>
> [*HSCAN*](http://doc.redisfans.com/hash/hscan.html#hscan) 命令返回的每个元素都是一个键值对，一个键值对由一个键和一个值组成。
>
> [*ZSCAN*](http://doc.redisfans.com/sorted_set/zscan.html#zscan) 命令返回的每个元素都是一个有序集合元素，一个有序集合元素由一个成员（member）和一个分值（score）组成。

# String（字符串）

## APPEND

**APPEND key value**

如果 `key` 已经存在并且是一个字符串， [APPEND](http://doc.redisfans.com/string/append.html#append) 命令将 `value` 追加到 `key` 原来的值的末尾。

如果 `key` 不存在， [APPEND](http://doc.redisfans.com/string/append.html#append) 就简单地将给定 `key` 设为 `value` ，就像执行 `SET key value` 一样。

- **可用版本：**

  >= 2.0.0

- **时间复杂度：**

  平摊O(1)

- **返回值：**

  追加 `value` 之后， `key` 中字符串的长度。

```
# 对不存在的 key 执行 APPEND
redis> EXISTS myphone               # 确保 myphone 不存在
(integer) 0

redis> APPEND myphone "nokia"       # 对不存在的 key 进行 APPEND ，等同于 SET myphone "nokia"
(integer) 5                         # 字符长度

# 对已存在的字符串进行 APPEND
redis> APPEND myphone " - 1110"     # 长度从 5 个字符增加到 12 个字符
(integer) 12

redis> GET myphone
"nokia - 1110"
```

### 模式：时间序列(Time series)

[APPEND](http://doc.redisfans.com/string/append.html#append) 可以为一系列定长(fixed-size)数据(sample)提供一种紧凑的表示方式，通常称之为时间序列。

每当一个新数据到达的时候，执行以下命令：

```
APPEND timeseries "fixed-size sample"
```

然后可以通过以下的方式访问时间序列的各项属性：

- [*STRLEN*](http://doc.redisfans.com/string/strlen.html#strlen) 给出时间序列中数据的数量
- [*GETRANGE*](http://doc.redisfans.com/string/getrange.html#getrange) 可以用于随机访问。只要有相关的时间信息的话，我们就可以在 Redis 2.6 中使用 Lua 脚本和 [*GETRANGE*](http://doc.redisfans.com/string/getrange.html#getrange) 命令实现二分查找。
- [*SETRANGE*](http://doc.redisfans.com/string/setrange.html#setrange) 可以用于覆盖或修改已存在的的时间序列。

这个模式的唯一缺陷是我们只能增长时间序列，而不能对时间序列进行缩短，因为 Redis 目前还没有对字符串进行修剪(tirm)的命令，但是，不管怎么说，这个模式的储存方式还是可以节省下大量的空间。

可以考虑使用 UNIX 时间戳作为时间序列的键名，这样一来，可以避免单个 `key` 因为保存过大的时间序列而占用大量内存，另一方面，也可以节省下大量命名空间。

下面是一个时间序列的例子：

```
redis> APPEND ts "0043"
(integer) 4

redis> APPEND ts "0035"
(integer) 8

redis> GETRANGE ts 0 3
"0043"

redis> GETRANGE ts 4 7
"0035"
```

## BITCOUNT

**BITCOUNT key [start] [end]**

计算给定字符串中，被设置为 `1` 的比特位的数量。

一般情况下，给定的整个字符串都会被进行计数，通过指定额外的 `start` 或 `end` 参数，可以让计数只在特定的位上进行。

`start` 和 `end` 参数的设置和 [*GETRANGE*](http://doc.redisfans.com/string/getrange.html#getrange) 命令类似，都可以使用负数值：比如 `-1` 表示最后一个位，而 `-2` 表示倒数第二个位，以此类推。

不存在的 `key` 被当成是空字符串来处理，因此对一个不存在的 `key` 进行 `BITCOUNT` 操作，结果为 `0` 。

- **可用版本：**

  >= 2.6.0

- **时间复杂度：**

  O(N)

- **返回值：**

  被设置为 `1` 的位的数量。

```
redis> BITCOUNT bits
(integer) 0

redis> SETBIT bits 0 1          # 0001
(integer) 0

redis> BITCOUNT bits
(integer) 1

redis> SETBIT bits 3 1          # 1001
(integer) 0

redis> BITCOUNT bits
(integer) 2
```

### 模式：使用 bitmap 实现用户上线次数统计

Bitmap 对于一些特定类型的计算非常有效。

假设现在我们希望记录自己网站上的用户的上线频率，比如说，计算用户 A 上线了多少天，用户 B 上线了多少天，诸如此类，以此作为数据，从而决定让哪些用户参加 beta 测试等活动 —— 这个模式可以使用 [*SETBIT*](http://doc.redisfans.com/string/setbit.html#setbit) 和 [*BITCOUNT*](http://doc.redisfans.com/string/bitcount.html#bitcount) 来实现。

比如说，每当用户在某一天上线的时候，我们就使用 [*SETBIT*](http://doc.redisfans.com/string/setbit.html#setbit) ，以用户名作为 `key` ，将那天所代表的网站的上线日作为 `offset` 参数，并将这个 `offset` 上的为设置为 `1` 。

举个例子，如果今天是网站上线的第 100 天，而用户 peter 在今天阅览过网站，那么执行命令 `SETBIT peter 100 1` ；如果明天 peter 也继续阅览网站，那么执行命令 `SETBIT peter 101 1` ，以此类推。

当要计算 peter 总共以来的上线次数时，就使用 [*BITCOUNT*](http://doc.redisfans.com/string/bitcount.html#bitcount) 命令：执行 `BITCOUNT peter` ，得出的结果就是 peter 上线的总天数。

更详细的实现可以参考博文(墙外) [Fast, easy, realtime metrics using Redis bitmaps](http://blog.getspool.com/2011/11/29/fast-easy-realtime-metrics-using-redis-bitmaps/) 。

### 性能

前面的上线次数统计例子，即使运行 10 年，占用的空间也只是每个用户 10*365 比特位(bit)，也即是每个用户 456 字节。对于这种大小的数据来说， [*BITCOUNT*](http://doc.redisfans.com/string/bitcount.html#bitcount) 的处理速度就像 [*GET*](http://doc.redisfans.com/string/get.html#get) 和 [*INCR*](http://doc.redisfans.com/string/incr.html#incr) 这种 O(1) 复杂度的操作一样快。

如果你的 bitmap 数据非常大，那么可以考虑使用以下两种方法：

1. 将一个大的 bitmap 分散到不同的 key 中，作为小的 bitmap 来处理。使用 Lua 脚本可以很方便地完成这一工作。
2. 使用 [*BITCOUNT*](http://doc.redisfans.com/string/bitcount.html#bitcount) 的 `start` 和 `end` 参数，每次只对所需的部分位进行计算，将位的累积工作(accumulating)放到客户端进行，并且对结果进行缓存 (caching)。

## BITOP

**BITOP operation destkey key [key ...]**

对一个或多个保存二进制位的字符串 `key` 进行位元操作，并将结果保存到 `destkey` 上。

`operation` 可以是 `AND` 、 `OR` 、 `NOT` 、 `XOR` 这四种操作中的任意一种：

- `BITOP AND destkey key [key ...]` ，对一个或多个 `key` 求逻辑并，并将结果保存到 `destkey` 。
- `BITOP OR destkey key [key ...]` ，对一个或多个 `key` 求逻辑或，并将结果保存到 `destkey` 。
- `BITOP XOR destkey key [key ...]` ，对一个或多个 `key` 求逻辑异或，并将结果保存到 `destkey` 。
- `BITOP NOT destkey key` ，对给定 `key` 求逻辑非，并将结果保存到 `destkey` 。

除了 `NOT` 操作之外，其他操作都可以接受一个或多个 `key` 作为输入。

**处理不同长度的字符串**

当 [BITOP](http://doc.redisfans.com/string/bitop.html#bitop) 处理不同长度的字符串时，较短的那个字符串所缺少的部分会被看作 `0` 。

空的 `key` 也被看作是包含 `0` 的字符串序列。

- **可用版本：**

  >= 2.6.0

- **时间复杂度：**

  O(N)

- **返回值：**

  保存到 `destkey` 的字符串的长度，和输入 `key` 中最长的字符串长度相等。

[BITOP](http://doc.redisfans.com/string/bitop.html#bitop) 的复杂度为 O(N) ，当处理大型矩阵(matrix)或者进行大数据量的统计时，最好将任务指派到附属节点(slave)进行，避免阻塞主节点。

```
redis> SETBIT bits-1 0 1        # bits-1 = 1001
(integer) 0

redis> SETBIT bits-1 3 1
(integer) 0

redis> SETBIT bits-2 0 1        # bits-2 = 1011
(integer) 0

redis> SETBIT bits-2 1 1
(integer) 0

redis> SETBIT bits-2 3 1
(integer) 0

redis> BITOP AND and-result bits-1 bits-2
(integer) 1

redis> GETBIT and-result 0      # and-result = 1001
(integer) 1

redis> GETBIT and-result 1
(integer) 0

redis> GETBIT and-result 2
(integer) 0

redis> GETBIT and-result 3
(integer) 1
```

## DECR

**DECR key**

将 `key` 中储存的数字值减一。

如果 `key` 不存在，那么 `key` 的值会先被初始化为 `0` ，然后再执行 [DECR](http://doc.redisfans.com/string/decr.html#decr) 操作。

如果值包含错误的类型，或字符串类型的值不能表示为数字，那么返回一个错误。

本操作的值限制在 64 位(bit)有符号数字表示之内。

关于递增(increment) / 递减(decrement)操作的更多信息，请参见 [*INCR*](http://doc.redisfans.com/string/incr.html#incr) 命令。

- **可用版本：**

  >= 1.0.0

- **时间复杂度：**

  O(1)

- **返回值：**

  执行 [DECR](http://doc.redisfans.com/string/decr.html#decr) 命令之后 `key` 的值。

```
# 对存在的数字值 key 进行 DECR
redis> SET failure_times 10
OK

redis> DECR failure_times
(integer) 9

# 对不存在的 key 值进行 DECR
redis> EXISTS count
(integer) 0

redis> DECR count
(integer) -1


# 对存在但不是数值的 key 进行 DECR
redis> SET company YOUR_CODE_SUCKS.LLC
OK

redis> DECR company
(error) ERR value is not an integer or out of range
```

## DECRBY

**DECRBY key decrement**

将 `key` 所储存的值减去减量 `decrement` 。

如果 `key` 不存在，那么 `key` 的值会先被初始化为 `0` ，然后再执行 [DECRBY](http://doc.redisfans.com/string/decrby.html#decrby) 操作。

如果值包含错误的类型，或字符串类型的值不能表示为数字，那么返回一个错误。

本操作的值限制在 64 位(bit)有符号数字表示之内。

关于更多递增(increment) / 递减(decrement)操作的更多信息，请参见 [*INCR*](http://doc.redisfans.com/string/incr.html#incr) 命令。

- **可用版本：**

  >= 1.0.0

- **时间复杂度：**

  O(1)

- **返回值：**

  减去 `decrement` 之后， `key` 的值。

```
# 对已存在的 key 进行 DECRBY
redis> SET count 100
OK

redis> DECRBY count 20
(integer) 80

# 对不存在的 key 进行DECRBY
redis> EXISTS pages
(integer) 0

redis> DECRBY pages 10
(integer) -10
```

## GET

**GET key**

返回 `key` 所关联的字符串值。

如果 `key` 不存在那么返回特殊值 `nil` 。

假如 `key` 储存的值不是字符串类型，返回一个错误，因为 [GET](http://doc.redisfans.com/string/get.html#get) 只能用于处理字符串值。

- **可用版本：**

  >= 1.0.0

- **时间复杂度：**

  O(1)

- **返回值：**

  当 `key` 不存在时，返回 `nil` ，否则，返回 `key` 的值。如果 `key` 不是字符串类型，那么返回一个错误。

```
# 对不存在的 key 或字符串类型 key 进行 GET
redis> GET db
(nil)

redis> SET db redis
OK

redis> GET db
"redis"

# 对不是字符串类型的 key 进行 GET
redis> DEL db
(integer) 1

redis> LPUSH db redis mongodb mysql
(integer) 3

redis> GET db
(error) ERR Operation against a key holding the wrong kind of value
```

## GETBIT

**GETBIT key offset**

对 `key` 所储存的字符串值，获取指定偏移量上的位(bit)。

当 `offset` 比字符串值的长度大，或者 `key` 不存在时，返回 `0` 。

- **可用版本：**

  >= 2.2.0

- **时间复杂度：**

  O(1)

- **返回值：**

  字符串值指定偏移量上的位(bit)。

```
# 对不存在的 key 或者不存在的 offset 进行 GETBIT， 返回 0
redis> EXISTS bit
(integer) 0

redis> GETBIT bit 10086
(integer) 0

# 对已存在的 offset 进行 GETBIT
redis> SETBIT bit 10086 1
(integer) 0

redis> GETBIT bit 10086
(integer) 1
```

## GETRANGE

**GETRANGE key start end**

返回 `key` 中字符串值的子字符串，字符串的截取范围由 `start` 和 `end` 两个偏移量决定(包括 `start` 和 `end` 在内)。

负数偏移量表示从字符串最后开始计数， `-1` 表示最后一个字符， `-2` 表示倒数第二个，以此类推。

[GETRANGE](http://doc.redisfans.com/string/getrange.html#getrange) 通过保证子字符串的值域(range)不超过实际字符串的值域来处理超出范围的值域请求。

在 <= 2.0 的版本里，GETRANGE 被叫作 SUBSTR。

- **可用版本：**

  >= 2.4.0

- **时间复杂度：**

  O(N)， `N` 为要返回的字符串的长度。复杂度最终由字符串的返回值长度决定，但因为从已有字符串中取出子字符串的操作非常廉价(cheap)，所以对于长度不大的字符串，该操作的复杂度也可看作O(1)。

- **返回值：**

  截取得出的子字符串。

```
redis> SET greeting "hello, my friend"
OK

redis> GETRANGE greeting 0 4          # 返回索引0-4的字符，包括4。
"hello"

redis> GETRANGE greeting -1 -5        # 不支持回绕操作
""

redis> GETRANGE greeting -3 -1        # 负数索引
"end"

redis> GETRANGE greeting 0 -1         # 从第一个到最后一个
"hello, my friend"

redis> GETRANGE greeting 0 1008611    # 值域范围不超过实际字符串，超过部分自动被符略
"hello, my friend"
```

## GETSET

**GETSET key value**

将给定 `key` 的值设为 `value` ，并返回 `key` 的旧值(old value)。

当 `key` 存在但不是字符串类型时，返回一个错误。

- **可用版本：**

  >= 1.0.0

- **时间复杂度：**

  O(1)

- **返回值：**

  返回给定 `key` 的旧值。当 `key` 没有旧值时，也即是， `key` 不存在时，返回 `nil` 。

```
redis> GETSET db mongodb    # 没有旧值，返回 nil
(nil)

redis> GET db
"mongodb"

redis> GETSET db redis      # 返回旧值 mongodb
"mongodb"

redis> GET db
"redis"
```

### 模式

[GETSET](http://doc.redisfans.com/string/getset.html#getset) 可以和 [*INCR*](http://doc.redisfans.com/string/incr.html#incr) 组合使用，实现一个有原子性(atomic)复位操作的计数器(counter)。

举例来说，每次当某个事件发生时，进程可能对一个名为 `mycount` 的 `key` 调用 [*INCR*](http://doc.redisfans.com/string/incr.html#incr) 操作，通常我们还要在一个原子时间内同时完成获得计数器的值和将计数器值复位为 `0` 两个操作。

可以用命令 `GETSET mycounter 0` 来实现这一目标。

```
redis> INCR mycount
(integer) 11

redis> GETSET mycount 0  # 一个原子内完成 GET mycount 和 SET mycount 0 操作
"11"

redis> GET mycount       # 计数器被重置
"0"
```

## INCR

**INCR key**

将 `key` 中储存的数字值增一。

如果 `key` 不存在，那么 `key` 的值会先被初始化为 `0` ，然后再执行 [INCR](http://doc.redisfans.com/string/incr.html#incr) 操作。

如果值包含错误的类型，或字符串类型的值不能表示为数字，那么返回一个错误。

本操作的值限制在 64 位(bit)有符号数字表示之内。

这是一个针对字符串的操作，因为 Redis 没有专用的整数类型，所以 key 内储存的字符串被解释为十进制 64 位有符号整数来执行 INCR 操作。

- **可用版本：**

  >= 1.0.0

- **时间复杂度：**

  O(1)

- **返回值：**

  执行 [INCR](http://doc.redisfans.com/string/incr.html#incr) 命令之后 `key` 的值。

```
redis> SET page_view 20
OK

redis> INCR page_view
(integer) 21

redis> GET page_view    # 数字值在 Redis 中以字符串的形式保存
"21"
```

### 模式：计数器

计数器是 Redis 的原子性自增操作可实现的最直观的模式了，它的想法相当简单：每当某个操作发生时，向 Redis 发送一个 [INCR](http://doc.redisfans.com/string/incr.html#incr) 命令。

比如在一个 web 应用程序中，如果想知道用户在一年中每天的点击量，那么只要将用户 ID 以及相关的日期信息作为键，并在每次用户点击页面时，执行一次自增操作即可。

比如用户名是 `peter` ，点击时间是 2012 年 3 月 22 日，那么执行命令 `INCR peter::2012.3.22` 。

可以用以下几种方式扩展这个简单的模式：

- 可以通过组合使用 [INCR](http://doc.redisfans.com/string/incr.html#incr) 和 [*EXPIRE*](http://doc.redisfans.com/key/expire.html#expire) ，来达到只在规定的生存时间内进行计数(counting)的目的。
- 客户端可以通过使用 [*GETSET*](http://doc.redisfans.com/string/getset.html#getset) 命令原子性地获取计数器的当前值并将计数器清零，更多信息请参考 [*GETSET*](http://doc.redisfans.com/string/getset.html#getset) 命令。
- 使用其他自增/自减操作，比如 [*DECR*](http://doc.redisfans.com/string/decr.html#decr) 和 [*INCRBY*](http://doc.redisfans.com/string/incrby.html#incrby) ，用户可以通过执行不同的操作增加或减少计数器的值，比如在游戏中的记分器就可能用到这些命令。

### 模式：限速器

限速器是特殊化的计算器，它用于限制一个操作可以被执行的速率(rate)。

限速器的典型用法是限制公开 API 的请求次数，以下是一个限速器实现示例，它将 API 的最大请求数限制在每个 IP 地址每秒钟十个之内：

```
FUNCTION LIMIT_API_CALL(ip)
ts = CURRENT_UNIX_TIME()
keyname = ip+":"+ts
current = GET(keyname)

IF current != NULL AND current > 10 THEN
    ERROR "too many requests per second"
END

IF current == NULL THEN
    MULTI
        INCR(keyname, 1)
        EXPIRE(keyname, 1)
    EXEC
ELSE
    INCR(keyname, 1)
END

PERFORM_API_CALL()
```

这个实现每秒钟为每个 IP 地址使用一个不同的计数器，并用 [*EXPIRE*](http://doc.redisfans.com/key/expire.html#expire) 命令设置生存时间(这样 Redis 就会负责自动删除过期的计数器)。

注意，我们使用事务打包执行 [*INCR*](http://doc.redisfans.com/string/incr.html#incr) 命令和 [*EXPIRE*](http://doc.redisfans.com/key/expire.html#expire) 命令，避免引入竞争条件，保证每次调用 API 时都可以正确地对计数器进行自增操作并设置生存时间。

以下是另一个限速器实现：

```
FUNCTION LIMIT_API_CALL(ip):
current = GET(ip)
IF current != NULL AND current > 10 THEN
    ERROR "too many requests per second"
ELSE
    value = INCR(ip)
    IF value == 1 THEN
        EXPIRE(ip,1)
    END
    PERFORM_API_CALL()
END
```

这个限速器只使用单个计数器，它的生存时间为一秒钟，如果在一秒钟内，这个计数器的值大于 `10` 的话，那么访问就会被禁止。

这个新的限速器在思路方面是没有问题的，但它在实现方面不够严谨，如果我们仔细观察一下的话，就会发现在 [*INCR*](http://doc.redisfans.com/string/incr.html#incr) 和 [*EXPIRE*](http://doc.redisfans.com/key/expire.html#expire) 之间存在着一个竞争条件，假如客户端在执行 [*INCR*](http://doc.redisfans.com/string/incr.html#incr) 之后，因为某些原因(比如客户端失败)而忘记设置 [*EXPIRE*](http://doc.redisfans.com/key/expire.html#expire) 的话，那么这个计数器就会一直存在下去，造成每个用户只能访问 `10` 次，噢，这简直是个灾难！

要消灭这个实现中的竞争条件，我们可以将它转化为一个 Lua 脚本，并放到 Redis 中运行(这个方法仅限于 Redis 2.6 及以上的版本)：

```
local current
current = redis.call("incr",KEYS[1])
if tonumber(current) == 1 then
    redis.call("expire",KEYS[1],1)
end
```

通过将计数器作为脚本放到 Redis 上运行，我们保证了 [*INCR*](http://doc.redisfans.com/string/incr.html#incr) 和 [*EXPIRE*](http://doc.redisfans.com/key/expire.html#expire) 两个操作的原子性，现在这个脚本实现不会引入竞争条件，它可以运作的很好。

关于在 Redis 中运行 Lua 脚本的更多信息，请参考 [*EVAL*](http://doc.redisfans.com/script/eval.html#eval) 命令。

还有另一种消灭竞争条件的方法，就是使用 Redis 的列表结构来代替 [*INCR*](http://doc.redisfans.com/string/incr.html#incr) 命令，这个方法无须脚本支持，因此它在 Redis 2.6 以下的版本也可以运行得很好：

```
FUNCTION LIMIT_API_CALL(ip)
current = LLEN(ip)
IF current > 10 THEN
    ERROR "too many requests per second"
ELSE
    IF EXISTS(ip) == FALSE
        MULTI
            RPUSH(ip,ip)
            EXPIRE(ip,1)
        EXEC
    ELSE
        RPUSHX(ip,ip)
    END
    PERFORM_API_CALL()
END
```

新的限速器使用了列表结构作为容器， [*LLEN*](http://doc.redisfans.com/list/llen.html#llen) 用于对访问次数进行检查，一个事务包裹着 [*RPUSH*](http://doc.redisfans.com/list/rpush.html#rpush) 和 [*EXPIRE*](http://doc.redisfans.com/key/expire.html#expire) 两个命令，用于在第一次执行计数时创建列表，并正确设置地设置过期时间，最后， [*RPUSHX*](http://doc.redisfans.com/list/rpushx.html#rpushx) 在后续的计数操作中进行增加操作。

## INCRBY

**INCRBY key increment**

将 `key` 所储存的值加上增量 `increment` 。

如果 `key` 不存在，那么 `key` 的值会先被初始化为 `0` ，然后再执行 [INCRBY](http://doc.redisfans.com/string/incrby.html#incrby) 命令。

如果值包含错误的类型，或字符串类型的值不能表示为数字，那么返回一个错误。

本操作的值限制在 64 位(bit)有符号数字表示之内。

关于递增(increment) / 递减(decrement)操作的更多信息，参见 [*INCR*](http://doc.redisfans.com/string/incr.html#incr) 命令。

- **可用版本：**

  >= 1.0.0

- **时间复杂度：**

  O(1)

- **返回值：**

  加上 `increment` 之后， `key` 的值。

```
# key 存在且是数字值
redis> SET rank 50
OK

redis> INCRBY rank 20
(integer) 70

redis> GET rank
"70"

# key 不存在时
redis> EXISTS counter
(integer) 0

redis> INCRBY counter 30
(integer) 30

redis> GET counter
"30"

# key 不是数字值时
redis> SET book "long long ago..."
OK

redis> INCRBY book 200
(error) ERR value is not an integer or out of range
```

## INCRBYFLOAT

**INCRBYFLOAT key increment**

为 `key` 中所储存的值加上浮点数增量 `increment` 。

如果 `key` 不存在，那么 [INCRBYFLOAT](http://doc.redisfans.com/string/incrbyfloat.html#incrbyfloat) 会先将 `key` 的值设为 `0` ，再执行加法操作。

如果命令执行成功，那么 `key` 的值会被更新为（执行加法之后的）新值，并且新值会以字符串的形式返回给调用者。

无论是 `key` 的值，还是增量 `increment` ，都可以使用像 `2.0e7` 、 `3e5` 、 `90e-2` 那样的指数符号(exponential notation)来表示，但是，**执行 INCRBYFLOAT 命令之后的值**总是以同样的形式储存，也即是，它们总是由一个数字，一个（可选的）小数点和一个任意位的小数部分组成（比如 `3.14` 、 `69.768` ，诸如此类)，小数部分尾随的 `0` 会被移除，如果有需要的话，还会将浮点数改为整数（比如 `3.0` 会被保存成 `3` ）。

除此之外，无论加法计算所得的浮点数的实际精度有多长， [INCRBYFLOAT](http://doc.redisfans.com/string/incrbyfloat.html#incrbyfloat) 的计算结果也最多只能表示小数点的后十七位。

当以下任意一个条件发生时，返回一个错误：

- `key` 的值不是字符串类型(因为 Redis 中的数字和浮点数都以字符串的形式保存，所以它们都属于字符串类型）
- `key` 当前的值或者给定的增量 `increment` 不能解释(parse)为双精度浮点数(double precision floating point number）

- **可用版本：**

  >= 2.6.0

- **时间复杂度：**

  O(1)

- **返回值：**

  执行命令之后 `key` 的值。

```
# 值和增量都不是指数符号
redis> SET mykey 10.50
OK

redis> INCRBYFLOAT mykey 0.1
"10.6"

# 值和增量都是指数符号
redis> SET mykey 314e-2
OK

redis> GET mykey                # 用 SET 设置的值可以是指数符号
"314e-2"

redis> INCRBYFLOAT mykey 0      # 但执行 INCRBYFLOAT 之后格式会被改成非指数符号
"3.14"

# 可以对整数类型执行
redis> SET mykey 3
OK

redis> INCRBYFLOAT mykey 1.1
"4.1"

# 后跟的 0 会被移除
redis> SET mykey 3.0
OK

redis> GET mykey                                    # SET 设置的值小数部分可以是 0
"3.0"

redis> INCRBYFLOAT mykey 1.000000000000000000000    # 但 INCRBYFLOAT 会将无用的 0 忽略掉，有需要的话，将浮点变为整数
"4"

redis> GET mykey
"4"
```

## MGET

**MGET key [key ...]**

返回所有(一个或多个)给定 `key` 的值。

如果给定的 `key` 里面，有某个 `key` 不存在，那么这个 `key` 返回特殊值 `nil` 。因此，该命令永不失败。

- **可用版本：**

  >= 1.0.0

- **时间复杂度:**

  O(N) , `N` 为给定 `key` 的数量。

- **返回值：**

  一个包含所有给定 `key` 的值的列表。

```
redis> SET redis redis.com
OK

redis> SET mongodb mongodb.org
OK

redis> MGET redis mongodb
1) "redis.com"
2) "mongodb.org"

redis> MGET redis mongodb mysql     # 不存在的 mysql 返回 nil
1) "redis.com"
2) "mongodb.org"
3) (nil)
```

## MSET

**MSET key value [key value ...]**

同时设置一个或多个 `key-value` 对。

如果某个给定 `key` 已经存在，那么 [MSET](http://doc.redisfans.com/string/mset.html#mset) 会用新值覆盖原来的旧值，如果这不是你所希望的效果，请考虑使用 [*MSETNX*](http://doc.redisfans.com/string/msetnx.html#msetnx) 命令：它只会在所有给定 `key` 都不存在的情况下进行设置操作。

[MSET](http://doc.redisfans.com/string/mset.html#mset) 是一个原子性(atomic)操作，所有给定 `key` 都会在同一时间内被设置，某些给定 `key` 被更新而另一些给定 `key` 没有改变的情况，不可能发生。

- **可用版本：**

  >= 1.0.1

- **时间复杂度：**

  O(N)， `N` 为要设置的 `key` 数量。

- **返回值：**

  总是返回 `OK` (因为 `MSET` 不可能失败)

```
redis> MSET date "2012.3.30" time "11:00 a.m." weather "sunny"
OK

redis> MGET date time weather
1) "2012.3.30"
2) "11:00 a.m."
3) "sunny"

# MSET 覆盖旧值例子
redis> SET google "google.hk"
OK

redis> MSET google "google.com"
OK

redis> GET google
"google.com"
```

## MSETNX

**MSETNX key value [key value ...]**

同时设置一个或多个 `key-value` 对，当且仅当所有给定 `key` 都不存在。

即使只有一个给定 `key` 已存在， [MSETNX](http://doc.redisfans.com/string/msetnx.html#msetnx) 也会拒绝执行所有给定 `key` 的设置操作。

[MSETNX](http://doc.redisfans.com/string/msetnx.html#msetnx) 是原子性的，因此它可以用作设置多个不同 `key` 表示不同字段(field)的唯一性逻辑对象(unique logic object)，所有字段要么全被设置，要么全不被设置。

- **可用版本：**

  >= 1.0.1

- **时间复杂度：**

  O(N)， `N` 为要设置的 `key` 的数量。

- **返回值：**

  当所有 `key` 都成功设置，返回 `1` 。如果所有给定 `key` 都设置失败(至少有一个 `key` 已经存在)，那么返回 `0` 。

```
# 对不存在的 key 进行 MSETNX
redis> MSETNX rmdbs "MySQL" nosql "MongoDB" key-value-store "redis"
(integer) 1

redis> MGET rmdbs nosql key-value-store
1) "MySQL"
2) "MongoDB"
3) "redis"

# MSET 的给定 key 当中有已存在的 key
redis> MSETNX rmdbs "Sqlite" language "python"  # rmdbs 键已经存在，操作失败
(integer) 0

redis> EXISTS language                          # 因为 MSET 是原子性操作，language 没有被设置
(integer) 0

redis> GET rmdbs                                # rmdbs 也没有被修改
"MySQL"
```

## PSETEX

**PSETEX key milliseconds value**

这个命令和 [*SETEX*](http://doc.redisfans.com/string/setex.html#setex) 命令相似，但它以毫秒为单位设置 `key` 的生存时间，而不是像 [*SETEX*](http://doc.redisfans.com/string/setex.html#setex) 命令那样，以秒为单位。

- **可用版本：**

  >= 2.6.0

- **时间复杂度：**

  O(1)

- **返回值：**

  设置成功时返回 `OK` 。

```
redis> PSETEX mykey 1000 "Hello"
OK

redis> PTTL mykey
(integer) 999

redis> GET mykey
"Hello"
```

## SET

**SET key value [EX seconds] [PX milliseconds] [NX|XX]**

将字符串值 `value` 关联到 `key` 。

如果 `key` 已经持有其他值， [SET](http://doc.redisfans.com/string/set.html#set) 就覆写旧值，无视类型。

对于某个原本带有生存时间（TTL）的键来说， 当 [*SET*](http://doc.redisfans.com/string/set.html#set) 命令成功在这个键上执行时， 这个键原有的 TTL 将被清除。

**可选参数**

从 Redis 2.6.12 版本开始， [*SET*](http://doc.redisfans.com/string/set.html#set) 命令的行为可以通过一系列参数来修改：

- `EX second` ：设置键的过期时间为 `second` 秒。 `SET key value EX second` 效果等同于 `SETEX key second value` 。
- `PX millisecond` ：设置键的过期时间为 `millisecond` 毫秒。 `SET key value PX millisecond` 效果等同于 `PSETEX key millisecond value` 。
- `NX` ：只在键不存在时，才对键进行设置操作。 `SET key value NX` 效果等同于 `SETNX key value` 。
- `XX` ：只在键已经存在时，才对键进行设置操作。

因为 [*SET*](http://doc.redisfans.com/string/set.html#set) 命令可以通过参数来实现和 [*SETNX*](http://doc.redisfans.com/string/setnx.html#setnx) 、 [*SETEX*](http://doc.redisfans.com/string/setex.html#setex) 和 [*PSETEX*](http://doc.redisfans.com/string/psetex.html#psetex) 三个命令的效果，所以将来的 Redis 版本可能会废弃并最终移除 [*SETNX*](http://doc.redisfans.com/string/setnx.html#setnx) 、 [*SETEX*](http://doc.redisfans.com/string/setex.html#setex) 和 [*PSETEX*](http://doc.redisfans.com/string/psetex.html#psetex) 这三个命令。

- **可用版本：**

  >= 1.0.0

- **时间复杂度：**

  O(1)

- **返回值：**

  在 Redis 2.6.12 版本以前， [*SET*](http://doc.redisfans.com/string/set.html#set) 命令总是返回 `OK` 。从 Redis 2.6.12 版本开始， [*SET*](http://doc.redisfans.com/string/set.html#set) 在设置操作成功完成时，才返回 `OK` 。如果设置了 `NX` 或者 `XX` ，但因为条件没达到而造成设置操作未执行，那么命令返回空批量回复（NULL Bulk Reply）。

```
# 对不存在的键进行设置
redis 127.0.0.1:6379> SET key "value"
OK

redis 127.0.0.1:6379> GET key
"value"

# 对已存在的键进行设置
redis 127.0.0.1:6379> SET key "new-value"
OK

redis 127.0.0.1:6379> GET key
"new-value"

# 使用 EX 选项
redis 127.0.0.1:6379> SET key-with-expire-time "hello" EX 10086
OK

redis 127.0.0.1:6379> GET key-with-expire-time
"hello"

redis 127.0.0.1:6379> TTL key-with-expire-time
(integer) 10069

# 使用 PX 选项
redis 127.0.0.1:6379> SET key-with-pexpire-time "moto" PX 123321
OK

redis 127.0.0.1:6379> GET key-with-pexpire-time
"moto"

redis 127.0.0.1:6379> PTTL key-with-pexpire-time
(integer) 111939

# 使用 NX 选项
redis 127.0.0.1:6379> SET not-exists-key "value" NX
OK      # 键不存在，设置成功

redis 127.0.0.1:6379> GET not-exists-key
"value"

redis 127.0.0.1:6379> SET not-exists-key "new-value" NX
(nil)   # 键已经存在，设置失败

redis 127.0.0.1:6379> GEt not-exists-key
"value" # 维持原值不变

# 使用 XX 选项
redis 127.0.0.1:6379> EXISTS exists-key
(integer) 0

redis 127.0.0.1:6379> SET exists-key "value" XX
(nil)   # 因为键不存在，设置失败

redis 127.0.0.1:6379> SET exists-key "value"
OK      # 先给键设置一个值

redis 127.0.0.1:6379> SET exists-key "new-value" XX
OK      # 设置新值成功

redis 127.0.0.1:6379> GET exists-key
"new-value"

# NX 或 XX 可以和 EX 或者 PX 组合使用
redis 127.0.0.1:6379> SET key-with-expire-and-NX "hello" EX 10086 NX
OK

redis 127.0.0.1:6379> GET key-with-expire-and-NX
"hello"

redis 127.0.0.1:6379> TTL key-with-expire-and-NX
(integer) 10063

redis 127.0.0.1:6379> SET key-with-pexpire-and-XX "old value"
OK

redis 127.0.0.1:6379> SET key-with-pexpire-and-XX "new value" PX 123321
OK

redis 127.0.0.1:6379> GET key-with-pexpire-and-XX
"new value"

redis 127.0.0.1:6379> PTTL key-with-pexpire-and-XX
(integer) 112999

# EX 和 PX 可以同时出现，但后面给出的选项会覆盖前面给出的选项
redis 127.0.0.1:6379> SET key "value" EX 1000 PX 5000000
OK

redis 127.0.0.1:6379> TTL key
(integer) 4993  # 这是 PX 参数设置的值

redis 127.0.0.1:6379> SET another-key "value" PX 5000000 EX 1000
OK

redis 127.0.0.1:6379> TTL another-key
(integer) 997   # 这是 EX 参数设置的值
```

### 使用模式

命令 `SET resource-name anystring NX EX max-lock-time` 是一种在 Redis 中实现锁的简单方法。

客户端执行以上的命令：

- 如果服务器返回 `OK` ，那么这个客户端获得锁。
- 如果服务器返回 `NIL` ，那么客户端获取锁失败，可以在稍后再重试。

设置的过期时间到达之后，锁将自动释放。

可以通过以下修改，让这个锁实现更健壮：

- 不使用固定的字符串作为键的值，而是设置一个不可猜测（non-guessable）的长随机字符串，作为口令串（token）。
- 不使用 [*DEL*](http://doc.redisfans.com/key/del.html#del) 命令来释放锁，而是发送一个 Lua 脚本，这个脚本只在客户端传入的值和键的口令串相匹配时，才对键进行删除。

这两个改动可以防止持有过期锁的客户端误删现有锁的情况出现。

以下是一个简单的解锁脚本示例：

```
if redis.call("get",KEYS[1]) == ARGV[1]
then
    return redis.call("del",KEYS[1])
else
    return 0
end
```

这个脚本可以通过 `EVAL ...script... 1 resource-name token-value` 命令来调用。

## SETBIT

**SETBIT key offset value**

对 `key` 所储存的字符串值，设置或清除指定偏移量上的位(bit)。

位的设置或清除取决于 `value` 参数，可以是 `0` 也可以是 `1` 。

当 `key` 不存在时，自动生成一个新的字符串值。

字符串会进行伸展(grown)以确保它可以将 `value` 保存在指定的偏移量上。当字符串值进行伸展时，空白位置以 `0` 填充。

`offset` 参数必须大于或等于 `0` ，小于 2^32 (bit 映射被限制在 512 MB 之内)。

对使用大的 `offset` 的 [SETBIT](http://doc.redisfans.com/string/setbit.html#setbit) 操作来说，内存分配可能造成 Redis 服务器被阻塞。具体参考 [*SETRANGE*](http://doc.redisfans.com/string/setrange.html#setrange) 命令，warning(警告)部分。

- **可用版本：**

  >= 2.2.0

- **时间复杂度:**

  O(1)

- **返回值：**

  指定偏移量原来储存的位。

```
redis> SETBIT bit 10086 1
(integer) 0

redis> GETBIT bit 10086
(integer) 1

redis> GETBIT bit 100   # bit 默认被初始化为 0
(integer) 0
```

## SETEX

**SETEX key seconds value**

将值 `value` 关联到 `key` ，并将 `key` 的生存时间设为 `seconds` (以秒为单位)。

如果 `key` 已经存在， [SETEX](http://doc.redisfans.com/string/setex.html#setex) 命令将覆写旧值。

这个命令类似于以下两个命令：

```
SET key value
EXPIRE key seconds  # 设置生存时间
```

不同之处是， [SETEX](http://doc.redisfans.com/string/setex.html#setex) 是一个原子性(atomic)操作，关联值和设置生存时间两个动作会在同一时间内完成，该命令在 Redis 用作缓存时，非常实用。

- **可用版本：**

  >= 2.0.0

- **时间复杂度：**

  O(1)

- **返回值：**

  设置成功时返回 `OK` 。当 `seconds` 参数不合法时，返回一个错误。

```
# 在 key 不存在时进行 SETEX
redis> SETEX cache_user_id 60 10086
OK

redis> GET cache_user_id  # 值
"10086"

redis> TTL cache_user_id  # 剩余生存时间
(integer) 49

# key 已经存在时，SETEX 覆盖旧值
redis> SET cd "timeless"
OK

redis> SETEX cd 3000 "goodbye my love"
OK

redis> GET cd
"goodbye my love"

redis> TTL cd
(integer) 2997
```

## SETNX

**SETNX key value**

将 `key` 的值设为 `value` ，当且仅当 `key` 不存在。

若给定的 `key` 已经存在，则 [SETNX](http://doc.redisfans.com/string/setnx.html#setnx) 不做任何动作。

[SETNX](http://doc.redisfans.com/string/setnx.html#setnx) 是『SET if Not eXists』(如果不存在，则 SET)的简写。

- **可用版本：**

  >= 1.0.0

- **时间复杂度：**

  O(1)

- **返回值：**

  设置成功，返回 `1` 。设置失败，返回 `0` 。

```
redis> EXISTS job                # job 不存在
(integer) 0

redis> SETNX job "programmer"    # job 设置成功
(integer) 1

redis> SETNX job "code-farmer"   # 尝试覆盖 job ，失败
(integer) 0

redis> GET job                   # 没有被覆盖
"programmer"
```

## SETRANGE

**SETRANGE key offset value**

用 `value` 参数覆写(overwrite)给定 `key` 所储存的字符串值，从偏移量 `offset` 开始。

不存在的 `key` 当作空白字符串处理。

[SETRANGE](http://doc.redisfans.com/string/setrange.html#setrange) 命令会确保字符串足够长以便将 `value` 设置在指定的偏移量上，如果给定 `key` 原来储存的字符串长度比偏移量小(比如字符串只有 `5` 个字符长，但你设置的 `offset` 是 `10` )，那么原字符和偏移量之间的空白将用零字节(zerobytes, `"\x00"` )来填充。

注意你能使用的最大偏移量是 2^29-1(536870911) ，因为 Redis 字符串的大小被限制在 512 兆(megabytes)以内。如果你需要使用比这更大的空间，你可以使用多个 `key` 。

当生成一个很长的字符串时，Redis 需要分配内存空间，该操作有时候可能会造成服务器阻塞(block)。在2010年的Macbook Pro上，设置偏移量为 536870911(512MB 内存分配)，耗费约 300 毫秒， 设置偏移量为 134217728(128MB 内存分配)，耗费约 80 毫秒，设置偏移量 33554432(32MB 内存分配)，耗费约 30 毫秒，设置偏移量为 8388608(8MB 内存分配)，耗费约 8 毫秒。 注意若首次内存分配成功之后，再对同一个 `key` 调用 [SETRANGE](http://doc.redisfans.com/string/setrange.html#setrange) 操作，无须再重新内存。

- **可用版本：**

  >= 2.2.0

- **时间复杂度：**

  对小(small)的字符串，平摊复杂度O(1)。(关于什么字符串是”小”的，请参考 [*APPEND*](http://doc.redisfans.com/string/append.html#append) 命令)否则为O(M)， `M` 为 `value` 参数的长度。

- **返回值：**

  被 [SETRANGE](http://doc.redisfans.com/string/setrange.html#setrange) 修改之后，字符串的长度。

```
# 对非空字符串进行 SETRANGE
redis> SET greeting "hello world"
OK

redis> SETRANGE greeting 6 "Redis"
(integer) 11

redis> GET greeting
"hello Redis"

# 对空字符串/不存在的 key 进行 SETRANGE
redis> EXISTS empty_string
(integer) 0

redis> SETRANGE empty_string 5 "Redis!"   # 对不存在的 key 使用 SETRANGE
(integer) 11

redis> GET empty_string                   # 空白处被"\x00"填充
"\x00\x00\x00\x00\x00Redis!"
```

### 模式

因为有了 [SETRANGE](http://doc.redisfans.com/string/setrange.html#setrange) 和 [*GETRANGE*](http://doc.redisfans.com/string/getrange.html#getrange) 命令，你可以将 Redis 字符串用作具有O(1)随机访问时间的线性数组，这在很多真实用例中都是非常快速且高效的储存方式，具体请参考 [*APPEND*](http://doc.redisfans.com/string/append.html#append) 命令的『模式：时间序列』部分。

## STRLEN

**STRLEN key**

返回 `key` 所储存的字符串值的长度。

当 `key` 储存的不是字符串值时，返回一个错误。

- **可用版本：**

  >= 2.2.0

- **复杂度：**

  O(1)

- **返回值：**

  字符串值的长度。当 `key` 不存在时，返回 `0` 。

```
# 获取字符串的长度
redis> SET mykey "Hello world"
OK

redis> STRLEN mykey
(integer) 11

# 不存在的 key 长度为 0
redis> STRLEN nonexisting
(integer) 0
```

# Hash（哈希表）

## HDEL

**HDEL key field [field ...]**

删除哈希表 `key` 中的一个或多个指定域，不存在的域将被忽略。

在Redis2.4以下的版本里， [HDEL](http://doc.redisfans.com/hash/hdel.html#hdel) 每次只能删除单个域，如果你需要在一个原子时间内删除多个域，请将命令包含在 [*MULTI*](http://doc.redisfans.com/transaction/multi.html#multi) / [*EXEC*](http://doc.redisfans.com/transaction/exec.html#exec) 块内。

- **可用版本：**

  >= 2.0.0

- **时间复杂度:**

  O(N)， `N` 为要删除的域的数量。

- **返回值:**

  被成功移除的域的数量，不包括被忽略的域。

```
# 测试数据
redis> HGETALL abbr
1) "a"
2) "apple"
3) "b"
4) "banana"
5) "c"
6) "cat"
7) "d"
8) "dog"


# 删除单个域
redis> HDEL abbr a
(integer) 1


# 删除不存在的域
redis> HDEL abbr not-exists-field
(integer) 0


# 删除多个域
redis> HDEL abbr b c
(integer) 2

redis> HGETALL abbr
1) "d"
2) "dog"
```

## HEXISTS

**HEXISTS key field**

查看哈希表 `key` 中，给定域 `field` 是否存在。

- **可用版本：**

  >= 2.0.0

- **时间复杂度：**

  O(1)

- **返回值：**

  如果哈希表含有给定域，返回 `1` 。如果哈希表不含有给定域，或 `key` 不存在，返回 `0` 。

```
redis> HEXISTS phone myphone
(integer) 0

redis> HSET phone myphone nokia-1110
(integer) 1

redis> HEXISTS phone myphone
(integer) 1
```

## HGET

**HGET key field**

返回哈希表 `key` 中给定域 `field` 的值。

- **可用版本：**

  >= 2.0.0

- **时间复杂度：**

  O(1)

- **返回值：**

  给定域的值。当给定域不存在或是给定 `key` 不存在时，返回 `nil` 。

```
# 域存在
redis> HSET site redis redis.com
(integer) 1

redis> HGET site redis
"redis.com"

# 域不存在
redis> HGET site mysql
(nil)
```

## HGETALL

**HGETALL key**

返回哈希表 `key` 中，所有的域和值。

在返回值里，紧跟每个域名(field name)之后是域的值(value)，所以返回值的长度是哈希表大小的两倍。

- **可用版本：**

  >= 2.0.0

- **时间复杂度：**

  O(N)， `N` 为哈希表的大小。

- **返回值：**

  以列表形式返回哈希表的域和域的值。若 `key` 不存在，返回空列表。

```
redis> HSET people jack "Jack Sparrow"
(integer) 1

redis> HSET people gump "Forrest Gump"
(integer) 1

redis> HGETALL people
1) "jack"          # 域
2) "Jack Sparrow"  # 值
3) "gump"
4) "Forrest Gump"
```

## HINCRBY

**HINCRBY key field increment**

为哈希表 `key` 中的域 `field` 的值加上增量 `increment` 。

增量也可以为负数，相当于对给定域进行减法操作。

如果 `key` 不存在，一个新的哈希表被创建并执行 [HINCRBY](http://doc.redisfans.com/hash/hincrby.html#hincrby) 命令。

如果域 `field` 不存在，那么在执行命令前，域的值被初始化为 `0` 。

对一个储存字符串值的域 `field` 执行 [HINCRBY](http://doc.redisfans.com/hash/hincrby.html#hincrby) 命令将造成一个错误。

本操作的值被限制在 64 位(bit)有符号数字表示之内。

- **可用版本：**

  >= 2.0.0

- **时间复杂度：**

  O(1)

- **返回值：**

  执行 [HINCRBY](http://doc.redisfans.com/hash/hincrby.html#hincrby) 命令之后，哈希表 `key` 中域 `field` 的值。

```
# increment 为正数
redis> HEXISTS counter page_view    # 对空域进行设置
(integer) 0
redis> HINCRBY counter page_view 200
(integer) 200
redis> HGET counter page_view
"200"

# increment 为负数
redis> HGET counter page_view
"200"
redis> HINCRBY counter page_view -50
(integer) 150
redis> HGET counter page_view
"150"

# 尝试对字符串值的域执行HINCRBY命令
redis> HSET myhash string hello,world       # 设定一个字符串值
(integer) 1
redis> HGET myhash string
"hello,world"
redis> HINCRBY myhash string 1              # 命令执行失败，错误。
(error) ERR hash value is not an integer
redis> HGET myhash string                   # 原值不变
"hello,world"
```

## HINCRBYFLOAT

**HINCRBYFLOAT key field increment**

为哈希表 `key` 中的域 `field` 加上浮点数增量 `increment` 。

如果哈希表中没有域 `field` ，那么 [HINCRBYFLOAT](http://doc.redisfans.com/hash/hincrbyfloat.html#hincrbyfloat) 会先将域 `field` 的值设为 `0` ，然后再执行加法操作。

如果键 `key` 不存在，那么 [HINCRBYFLOAT](http://doc.redisfans.com/hash/hincrbyfloat.html#hincrbyfloat) 会先创建一个哈希表，再创建域 `field` ，最后再执行加法操作。

当以下任意一个条件发生时，返回一个错误：

- 域 `field` 的值不是字符串类型(因为 redis 中的数字和浮点数都以字符串的形式保存，所以它们都属于字符串类型）
- 域 `field` 当前的值或给定的增量 `increment` 不能解释(parse)为双精度浮点数(double precision floating point number)

[HINCRBYFLOAT](http://doc.redisfans.com/hash/hincrbyfloat.html#hincrbyfloat) 命令的详细功能和 [*INCRBYFLOAT*](http://doc.redisfans.com/string/incrbyfloat.html#incrbyfloat) 命令类似，请查看 [*INCRBYFLOAT*](http://doc.redisfans.com/string/incrbyfloat.html#incrbyfloat) 命令获取更多相关信息。

- **可用版本：**

  >= 2.6.0

- **时间复杂度：**

  O(1)

- **返回值：**

  执行加法操作之后 `field` 域的值。

```
# 值和增量都是普通小数
redis> HSET mykey field 10.50
(integer) 1
redis> HINCRBYFLOAT mykey field 0.1
"10.6"

# 值和增量都是指数符号
redis> HSET mykey field 5.0e3
(integer) 0
redis> HINCRBYFLOAT mykey field 2.0e2
"5200"

# 对不存在的键执行 HINCRBYFLOAT
redis> EXISTS price
(integer) 0
redis> HINCRBYFLOAT price milk 3.5
"3.5"
redis> HGETALL price
1) "milk"
2) "3.5"

# 对不存在的域进行 HINCRBYFLOAT
redis> HGETALL price
1) "milk"
2) "3.5"
redis> HINCRBYFLOAT price coffee 4.5   # 新增 coffee 域
"4.5"
redis> HGETALL price
1) "milk"
2) "3.5"
3) "coffee"
4) "4.5"
```

## HKEYS

**HKEYS key**

返回哈希表 `key` 中的所有域。

- **可用版本：**

  >= 2.0.0

- **时间复杂度：**

  O(N)， `N` 为哈希表的大小。

- **返回值：**

  一个包含哈希表中所有域的表。当 `key` 不存在时，返回一个空表。

```
# 哈希表非空
redis> HMSET website google www.google.com yahoo www.yahoo.com
OK
redis> HKEYS website
1) "google"
2) "yahoo"

# 空哈希表/key不存在
redis> EXISTS fake_key
(integer) 0
redis> HKEYS fake_key
(empty list or set)
```

## HLEN

**HLEN key**

返回哈希表 `key` 中域的数量。

- **时间复杂度：**

  O(1)

- **返回值：**

  哈希表中域的数量。当 `key` 不存在时，返回 `0` 。

```
redis> HSET db redis redis.com
(integer) 1

redis> HSET db mysql mysql.com
(integer) 1

redis> HLEN db
(integer) 2

redis> HSET db mongodb mongodb.org
(integer) 1

redis> HLEN db
(integer) 3
```

## HMGET

**HMGET key field [field ...]**

返回哈希表 `key` 中，一个或多个给定域的值。

如果给定的域不存在于哈希表，那么返回一个 `nil` 值。

因为不存在的 `key` 被当作一个空哈希表来处理，所以对一个不存在的 `key` 进行 [HMGET](http://doc.redisfans.com/hash/hmget.html#hmget) 操作将返回一个只带有 `nil` 值的表。

- **可用版本：**

  >= 2.0.0

- **时间复杂度：**

  O(N)， `N` 为给定域的数量。

- **返回值：**

  一个包含多个给定域的关联值的表，表值的排列顺序和给定域参数的请求顺序一样。

```
redis> HMSET pet dog "doudou" cat "nounou"    # 一次设置多个域
OK

redis> HMGET pet dog cat fake_pet             # 返回值的顺序和传入参数的顺序一样
1) "doudou"
2) "nounou"
3) (nil)                                      # 不存在的域返回nil值
```

## HMSET

**HMSET key field value [field value ...]**

同时将多个 `field-value` (域-值)对设置到哈希表 `key` 中。

此命令会覆盖哈希表中已存在的域。

如果 `key` 不存在，一个空哈希表被创建并执行 [HMSET](http://doc.redisfans.com/hash/hmset.html#hmset) 操作。

- **可用版本：**

  >= 2.0.0

- **时间复杂度：**

  O(N)， `N` 为 `field-value` 对的数量。

- **返回值：**

  如果命令执行成功，返回 `OK` 。当 `key` 不是哈希表(hash)类型时，返回一个错误。

```
redis> HMSET website google www.google.com yahoo www.yahoo.com
OK

redis> HGET website google
"www.google.com"

redis> HGET website yahoo
"www.yahoo.com"
```

## HSET

**HSET key field value**

将哈希表 `key` 中的域 `field` 的值设为 `value` 。

如果 `key` 不存在，一个新的哈希表被创建并进行 [HSET](http://doc.redisfans.com/hash/hset.html#hset) 操作。

如果域 `field` 已经存在于哈希表中，旧值将被覆盖。

- **可用版本：**

  >= 2.0.0

- **时间复杂度：**

  O(1)

- **返回值：**

  如果 `field` 是哈希表中的一个新建域，并且值设置成功，返回 `1` 。如果哈希表中域 `field` 已经存在且旧值已被新值覆盖，返回 `0` 。

```
redis> HSET website google "www.g.cn"       # 设置一个新域
(integer) 1

redis> HSET website google "www.google.com" # 覆盖一个旧域
(integer) 0
```

## HSETNX

**HSETNX key field value**

将哈希表 `key` 中的域 `field` 的值设置为 `value` ，当且仅当域 `field` 不存在。

若域 `field` 已经存在，该操作无效。

如果 `key` 不存在，一个新哈希表被创建并执行 [HSETNX](http://doc.redisfans.com/hash/hsetnx.html#hsetnx) 命令。

- **可用版本：**

  >= 2.0.0

- **时间复杂度：**

  O(1)

- **返回值：**

  设置成功，返回 `1` 。如果给定域已经存在且没有操作被执行，返回 `0` 。

```
redis> HSETNX nosql key-value-store redis
(integer) 1

redis> HSETNX nosql key-value-store redis       # 操作无效，域 key-value-store 已存在
(integer) 0
```

## HVALS

**HVALS key**

返回哈希表 `key` 中所有域的值。

- **可用版本：**

  >= 2.0.0

- **时间复杂度：**

  O(N)， `N` 为哈希表的大小。

- **返回值：**

  一个包含哈希表中所有值的表。当 `key` 不存在时，返回一个空表。

```
# 非空哈希表
redis> HMSET website google www.google.com yahoo www.yahoo.com
OK

redis> HVALS website
1) "www.google.com"
2) "www.yahoo.com"


# 空哈希表/不存在的key

redis> EXISTS not_exists
(integer) 0

redis> HVALS not_exists
(empty list or set)
```

## HSCAN

**HSCAN key cursor [MATCH pattern] [COUNT count]**

具体信息请参考 [*SCAN*](http://doc.redisfans.com/key/scan.html#scan) 命令。

# List（列表）

## BLPOP

**BLPOP key [key ...] timeout**

[BLPOP](http://doc.redisfans.com/list/blpop.html#blpop) 是列表的阻塞式(blocking)弹出原语。

它是 [*LPOP*](http://doc.redisfans.com/list/lpop.html#lpop) 命令的阻塞版本，当给定列表内没有任何元素可供弹出的时候，连接将被 [BLPOP](http://doc.redisfans.com/list/blpop.html#blpop) 命令阻塞，直到等待超时或发现可弹出元素为止。

当给定多个 `key` 参数时，按参数 `key` 的先后顺序依次检查各个列表，弹出第一个非空列表的头元素。

**非阻塞行为**

当 [BLPOP](http://doc.redisfans.com/list/blpop.html#blpop) 被调用时，如果给定 `key` 内至少有一个非空列表，那么弹出遇到的第一个非空列表的头元素，并和被弹出元素所属的列表的名字一起，组成结果返回给调用者。

当存在多个给定 `key` 时， [BLPOP](http://doc.redisfans.com/list/blpop.html#blpop) 按给定 `key` 参数排列的先后顺序，依次检查各个列表。

假设现在有 `job` 、 `command` 和 `request` 三个列表，其中 `job` 不存在， `command` 和 `request` 都持有非空列表。考虑以下命令：

```
BLPOP job command request 0
```

[BLPOP](http://doc.redisfans.com/list/blpop.html#blpop) 保证返回的元素来自 `command` ，因为它是按”查找 `job` -> 查找 `command` -> 查找 `request` “这样的顺序，第一个找到的非空列表。

```
redis> DEL job command request           # 确保key都被删除
(integer) 0

redis> LPUSH command "update system..."  # 为command列表增加一个值
(integer) 1

redis> LPUSH request "visit page"        # 为request列表增加一个值
(integer) 1

redis> BLPOP job command request 0       # job 列表为空，被跳过，紧接着 command 列表的第一个元素被弹出。
1) "command"                             # 弹出元素所属的列表
2) "update system..."                    # 弹出元素所属的值
```

**阻塞行为**

如果所有给定 `key` 都不存在或包含空列表，那么 [BLPOP](http://doc.redisfans.com/list/blpop.html#blpop) 命令将阻塞连接，直到等待超时，或有另一个客户端对给定 `key` 的任意一个执行 [*LPUSH*](http://doc.redisfans.com/list/lpush.html#lpush) 或 [*RPUSH*](http://doc.redisfans.com/list/rpush.html#rpush) 命令为止。

超时参数 `timeout` 接受一个以秒为单位的数字作为值。超时参数设为 `0` 表示阻塞时间可以无限期延长(block indefinitely) 。

```
redis> EXISTS job                # 确保两个 key 都不存在
(integer) 0
redis> EXISTS command
(integer) 0

redis> BLPOP job command 300     # 因为key一开始不存在，所以操作会被阻塞，直到另一客户端对 job 或者 command 列表进行 PUSH 操作。
1) "job"                         # 这里被 push 的是 job
2) "do my home work"             # 被弹出的值
(26.26s)                         # 等待的秒数

redis> BLPOP job command 5       # 等待超时的情况
(nil)
(5.66s)                          # 等待的秒数
```

**相同的key被多个客户端同时阻塞**

相同的 `key` 可以被多个客户端同时阻塞。

不同的客户端被放进一个队列中，按『先阻塞先服务』(first-BLPOP，first-served)的顺序为 `key` 执行 [BLPOP](http://doc.redisfans.com/list/blpop.html#blpop) 命令。

**在MULTI/EXEC事务中的BLPOP**

[BLPOP](http://doc.redisfans.com/list/blpop.html#blpop) 可以用于流水线(pipline,批量地发送多个命令并读入多个回复)，但把它用在 [*MULTI*](http://doc.redisfans.com/transaction/multi.html#multi) / [*EXEC*](http://doc.redisfans.com/transaction/exec.html#exec) 块当中没有意义。因为这要求整个服务器被阻塞以保证块执行时的原子性，该行为阻止了其他客户端执行 [*LPUSH*](http://doc.redisfans.com/list/lpush.html#lpush) 或 [*RPUSH*](http://doc.redisfans.com/list/rpush.html#rpush) 命令。

因此，一个被包裹在 [*MULTI*](http://doc.redisfans.com/transaction/multi.html#multi) / [*EXEC*](http://doc.redisfans.com/transaction/exec.html#exec) 块内的 [BLPOP](http://doc.redisfans.com/list/blpop.html#blpop) 命令，行为表现得就像 [*LPOP*](http://doc.redisfans.com/list/lpop.html#lpop) 一样，对空列表返回 `nil` ，对非空列表弹出列表元素，不进行任何阻塞操作。

```
# 对非空列表进行操作

redis> RPUSH job programming
(integer) 1

redis> MULTI
OK

redis> BLPOP job 30
QUEUED

redis> EXEC           # 不阻塞，立即返回
1) 1) "job"
   2) "programming"

# 对空列表进行操作
redis> LLEN job      # 空列表
(integer) 0

redis> MULTI
OK

redis> BLPOP job 30
QUEUED

redis> EXEC         # 不阻塞，立即返回
1) (nil)
```

- **可用版本：**

  >= 2.0.0

- **时间复杂度：**

  O(1)

- **返回值：**

  如果列表为空，返回一个 `nil` 。否则，返回一个含有两个元素的列表，第一个元素是被弹出元素所属的 `key` ，第二个元素是被弹出元素的值。

### 模式：事件提醒

有时候，为了等待一个新元素到达数据中，需要使用轮询的方式对数据进行探查。

另一种更好的方式是，使用系统提供的阻塞原语，在新元素到达时立即进行处理，而新元素还没到达时，就一直阻塞住，避免轮询占用资源。

对于 Redis ，我们似乎需要一个阻塞版的 [*SPOP*](http://doc.redisfans.com/set/spop.html#spop) 命令，但实际上，使用 [BLPOP](http://doc.redisfans.com/list/blpop.html#blpop) 或者 [*BRPOP*](http://doc.redisfans.com/list/brpop.html#brpop) 就能很好地解决这个问题。

使用元素的客户端(消费者)可以执行类似以下的代码：

```
LOOP forever
    WHILE SPOP(key) returns elements
        ... process elements ...
    END
    BRPOP helper_key
END
```

添加元素的客户端(消费者)则执行以下代码：

```
MULTI
    SADD key element
    LPUSH helper_key x
EXEC
```

## BRPOP

**BRPOP key [key ...] timeout**

[BRPOP](http://doc.redisfans.com/list/brpop.html#brpop) 是列表的阻塞式(blocking)弹出原语。

它是 [*RPOP*](http://doc.redisfans.com/list/rpop.html#rpop) 命令的阻塞版本，当给定列表内没有任何元素可供弹出的时候，连接将被 [BRPOP](http://doc.redisfans.com/list/brpop.html#brpop) 命令阻塞，直到等待超时或发现可弹出元素为止。

当给定多个 `key` 参数时，按参数 `key` 的先后顺序依次检查各个列表，弹出第一个非空列表的尾部元素。

关于阻塞操作的更多信息，请查看 [*BLPOP*](http://doc.redisfans.com/list/blpop.html#blpop) 命令， [BRPOP](http://doc.redisfans.com/list/brpop.html#brpop) 除了弹出元素的位置和 [*BLPOP*](http://doc.redisfans.com/list/blpop.html#blpop) 不同之外，其他表现一致。

- **可用版本：**

  >= 2.0.0

- **时间复杂度：**

  O(1)

- **返回值：**

  假如在指定时间内没有任何元素被弹出，则返回一个 `nil` 和等待时长。反之，返回一个含有两个元素的列表，第一个元素是被弹出元素所属的 `key` ，第二个元素是被弹出元素的值。

```
redis> LLEN course
(integer) 0

redis> RPUSH course algorithm001
(integer) 1

redis> RPUSH course c++101
(integer) 2

redis> BRPOP course 30
1) "course"             # 弹出元素的 key
2) "c++101"             # 弹出元素的值
```

## BRPOPLPUSH

**BRPOPLPUSH source destination timeout**

[BRPOPLPUSH](http://doc.redisfans.com/list/brpoplpush.html#brpoplpush) 是 [*RPOPLPUSH*](http://doc.redisfans.com/list/rpoplpush.html#rpoplpush) 的阻塞版本，当给定列表 `source` 不为空时， [BRPOPLPUSH](http://doc.redisfans.com/list/brpoplpush.html#brpoplpush) 的表现和 [*RPOPLPUSH*](http://doc.redisfans.com/list/rpoplpush.html#rpoplpush) 一样。

当列表 `source` 为空时， [BRPOPLPUSH](http://doc.redisfans.com/list/brpoplpush.html#brpoplpush) 命令将阻塞连接，直到等待超时，或有另一个客户端对 `source` 执行 [*LPUSH*](http://doc.redisfans.com/list/lpush.html#lpush) 或 [*RPUSH*](http://doc.redisfans.com/list/rpush.html#rpush) 命令为止。

超时参数 `timeout` 接受一个以秒为单位的数字作为值。超时参数设为 `0` 表示阻塞时间可以无限期延长(block indefinitely) 。

更多相关信息，请参考 [*RPOPLPUSH*](http://doc.redisfans.com/list/rpoplpush.html#rpoplpush) 命令。

- **可用版本：**

  >= 2.2.0

- **时间复杂度：**

  O(1)

- **返回值：**

  假如在指定时间内没有任何元素被弹出，则返回一个 `nil` 和等待时长。反之，返回一个含有两个元素的列表，第一个元素是被弹出元素的值，第二个元素是等待时长。

```
# 非空列表

redis> BRPOPLPUSH msg reciver 500
"hello moto"                        # 弹出元素的值
(3.38s)                             # 等待时长

redis> LLEN reciver
(integer) 1

redis> LRANGE reciver 0 0
1) "hello moto"


# 空列表

redis> BRPOPLPUSH msg reciver 1
(nil)
(1.34s)
```

### 模式：安全队列

参考 [*RPOPLPUSH*](http://doc.redisfans.com/list/rpoplpush.html#rpoplpush) 命令的『安全队列』模式。

### 模式：循环列表

参考 [*RPOPLPUSH*](http://doc.redisfans.com/list/rpoplpush.html#rpoplpush) 命令的『循环列表』模式。

## LINDEX

**LINDEX key index**

返回列表 `key` 中，下标为 `index` 的元素。

下标(index)参数 `start` 和 `stop` 都以 `0` 为底，也就是说，以 `0` 表示列表的第一个元素，以 `1` 表示列表的第二个元素，以此类推。

你也可以使用负数下标，以 `-1` 表示列表的最后一个元素， `-2` 表示列表的倒数第二个元素，以此类推。

如果 `key` 不是列表类型，返回一个错误。

- **可用版本：**

  >= 1.0.0

- **时间复杂度：**

  O(N)， `N` 为到达下标 `index` 过程中经过的元素数量。因此，对列表的头元素和尾元素执行 [LINDEX](http://doc.redisfans.com/list/lindex.html#lindex) 命令，复杂度为O(1)。

- **返回值:**

  列表中下标为 `index` 的元素。如果 `index` 参数的值不在列表的区间范围内(out of range)，返回 `nil` 。

```
redis> LPUSH mylist "World"
(integer) 1

redis> LPUSH mylist "Hello"
(integer) 2

redis> LINDEX mylist 0
"Hello"

redis> LINDEX mylist -1
"World"

redis> LINDEX mylist 3        # index不在 mylist 的区间范围内
(nil)
```

## LINSERT

**LINSERT key BEFORE|AFTER pivot value**

将值 `value` 插入到列表 `key` 当中，位于值 `pivot` 之前或之后。

当 `pivot` 不存在于列表 `key` 时，不执行任何操作。

当 `key` 不存在时， `key` 被视为空列表，不执行任何操作。

如果 `key` 不是列表类型，返回一个错误。

- **可用版本：**

  >= 2.2.0

- **时间复杂度:**

  O(N)， `N` 为寻找 `pivot` 过程中经过的元素数量。

- **返回值:**

  如果命令执行成功，返回插入操作完成之后，列表的长度。如果没有找到 `pivot` ，返回 `-1` 。如果 `key` 不存在或为空列表，返回 `0` 。

```
redis> RPUSH mylist "Hello"
(integer) 1

redis> RPUSH mylist "World"
(integer) 2

redis> LINSERT mylist BEFORE "World" "There"
(integer) 3

redis> LRANGE mylist 0 -1
1) "Hello"
2) "There"
3) "World"

# 对一个非空列表插入，查找一个不存在的 pivot
redis> LINSERT mylist BEFORE "go" "let's"
(integer) -1                                    # 失败

# 对一个空列表执行 LINSERT 命令
redis> EXISTS fake_list
(integer) 0

redis> LINSERT fake_list BEFORE "nono" "gogogog"
(integer) 0                                      # 失败
```

## LLEN

**LLEN key**

返回列表 `key` 的长度。

如果 `key` 不存在，则 `key` 被解释为一个空列表，返回 `0` .

如果 `key` 不是列表类型，返回一个错误。

- **可用版本：**

  >= 1.0.0

- **时间复杂度：**

  O(1)

- **返回值：**

  列表 `key` 的长度。

```
# 空列表
redis> LLEN job
(integer) 0

# 非空列表
redis> LPUSH job "cook food"
(integer) 1

redis> LPUSH job "have lunch"
(integer) 2

redis> LLEN job
(integer) 2
```

## LPOP

**LPOP key**

移除并返回列表 `key` 的头元素。

- **可用版本：**

  >= 1.0.0

- **时间复杂度：**

  O(1)

- **返回值：**

  列表的头元素。当 `key` 不存在时，返回 `nil` 。

```
redis> LLEN course
(integer) 0

redis> RPUSH course algorithm001
(integer) 1

redis> RPUSH course c++101
(integer) 2

redis> LPOP course  # 移除头元素
"algorithm001"
```

## LPUSH

**LPUSH key value [value ...]**

将一个或多个值 `value` 插入到列表 `key` 的表头

如果有多个 `value` 值，那么各个 `value` 值按从左到右的顺序依次插入到表头： 比如说，对空列表 `mylist` 执行命令 `LPUSH mylist a b c` ，列表的值将是 `c b a` ，这等同于原子性地执行 `LPUSH mylist a` 、 `LPUSH mylist b` 和 `LPUSH mylist c` 三个命令。

如果 `key` 不存在，一个空列表会被创建并执行 [LPUSH](http://doc.redisfans.com/list/lpush.html#lpush) 操作。

当 `key` 存在但不是列表类型时，返回一个错误。

在Redis 2.4版本以前的 [LPUSH](http://doc.redisfans.com/list/lpush.html#lpush) 命令，都只接受单个 `value` 值。

- **可用版本：**

  >= 1.0.0

- **时间复杂度：**

  O(1)

- **返回值：**

  执行 [LPUSH](http://doc.redisfans.com/list/lpush.html#lpush) 命令后，列表的长度。

```
# 加入单个元素
redis> LPUSH languages python
(integer) 1

# 加入重复元素
redis> LPUSH languages python
(integer) 2

redis> LRANGE languages 0 -1     # 列表允许重复元素
1) "python"
2) "python"

# 加入多个元素
redis> LPUSH mylist a b c
(integer) 3

redis> LRANGE mylist 0 -1
1) "c"
2) "b"
3) "a"
```

## LPUSHX

**LPUSHX key value**

将值 `value` 插入到列表 `key` 的表头，当且仅当 `key` 存在并且是一个列表。

和 [*LPUSH*](http://doc.redisfans.com/list/lpush.html#lpush) 命令相反，当 `key` 不存在时， [LPUSHX](http://doc.redisfans.com/list/lpushx.html#lpushx) 命令什么也不做。

- **可用版本：**

  >= 2.2.0

- **时间复杂度：**

  O(1)

- **返回值：**

  [LPUSHX](http://doc.redisfans.com/list/lpushx.html#lpushx) 命令执行之后，表的长度。

```
# 对空列表执行 LPUSHX
redis> LLEN greet                       # greet 是一个空列表
(integer) 0

redis> LPUSHX greet "hello"             # 尝试 LPUSHX，失败，因为列表为空
(integer) 0

# 对非空列表执行 LPUSHX
redis> LPUSH greet "hello"              # 先用 LPUSH 创建一个有一个元素的列表
(integer) 1

redis> LPUSHX greet "good morning"      # 这次 LPUSHX 执行成功
(integer) 2

redis> LRANGE greet 0 -1
1) "good morning"
2) "hello"
```

## LRANGE

**LRANGE key start stop**

返回列表 `key` 中指定区间内的元素，区间以偏移量 `start` 和 `stop` 指定。

下标(index)参数 `start` 和 `stop` 都以 `0` 为底，也就是说，以 `0` 表示列表的第一个元素，以 `1` 表示列表的第二个元素，以此类推。

你也可以使用负数下标，以 `-1` 表示列表的最后一个元素， `-2` 表示列表的倒数第二个元素，以此类推。

**注意LRANGE命令和编程语言区间函数的区别**

假如你有一个包含一百个元素的列表，对该列表执行 `LRANGE list 0 10` ，结果是一个包含11个元素的列表，这表明 `stop` 下标也在 [LRANGE](http://doc.redisfans.com/list/lrange.html#lrange) 命令的取值范围之内(闭区间)，这和某些语言的区间函数可能不一致，比如Ruby的 `Range.new` 、 `Array#slice` 和Python的 `range()` 函数。

**超出范围的下标**

超出范围的下标值不会引起错误。

如果 `start` 下标比列表的最大下标 `end` ( `LLEN list` 减去 `1` )还要大，那么 [LRANGE](http://doc.redisfans.com/list/lrange.html#lrange) 返回一个空列表。

如果 `stop` 下标比 `end` 下标还要大，Redis将 `stop` 的值设置为 `end` 。

- **可用版本：**

  >= 1.0.0

- **时间复杂度:**

  O(S+N)， `S` 为偏移量 `start` ， `N` 为指定区间内元素的数量。

- **返回值:**

  一个列表，包含指定区间内的元素。

```
redis> RPUSH fp-language lisp
(integer) 1

redis> LRANGE fp-language 0 0
1) "lisp"

redis> RPUSH fp-language scheme
(integer) 2

redis> LRANGE fp-language 0 1
1) "lisp"
2) "scheme"
```

## LREM

**LREM key count value**

根据参数 `count` 的值，移除列表中与参数 `value` 相等的元素。

`count` 的值可以是以下几种：

- `count > 0` : 从表头开始向表尾搜索，移除与 `value` 相等的元素，数量为 `count` 。
- `count < 0` : 从表尾开始向表头搜索，移除与 `value` 相等的元素，数量为 `count` 的绝对值。
- `count = 0` : 移除表中所有与 `value` 相等的值。

- **可用版本：**

  >= 1.0.0

- **时间复杂度：**

  O(N)， `N` 为列表的长度。

- **返回值：**

  被移除元素的数量。因为不存在的 `key` 被视作空表(empty list)，所以当 `key` 不存在时， [LREM](http://doc.redisfans.com/list/lrem.html#lrem) 命令总是返回 `0` 。

```
# 先创建一个表，内容排列是
# morning hello morning helllo morning

redis> LPUSH greet "morning"
(integer) 1
redis> LPUSH greet "hello"
(integer) 2
redis> LPUSH greet "morning"
(integer) 3
redis> LPUSH greet "hello"
(integer) 4
redis> LPUSH greet "morning"
(integer) 5

redis> LRANGE greet 0 4         # 查看所有元素
1) "morning"
2) "hello"
3) "morning"
4) "hello"
5) "morning"

redis> LREM greet 2 morning     # 移除从表头到表尾，最先发现的两个 morning
(integer) 2                     # 两个元素被移除

redis> LLEN greet               # 还剩 3 个元素
(integer) 3

redis> LRANGE greet 0 2
1) "hello"
2) "hello"
3) "morning"

redis> LREM greet -1 morning    # 移除从表尾到表头，第一个 morning
(integer) 1

redis> LLEN greet               # 剩下两个元素
(integer) 2

redis> LRANGE greet 0 1
1) "hello"
2) "hello"

redis> LREM greet 0 hello      # 移除表中所有 hello
(integer) 2                    # 两个 hello 被移除

redis> LLEN greet
(integer) 0
```

## LSET

**LSET key index value**

将列表 `key` 下标为 `index` 的元素的值设置为 `value` 。

当 `index` 参数超出范围，或对一个空列表( `key` 不存在)进行 [LSET](http://doc.redisfans.com/list/lset.html#lset) 时，返回一个错误。

关于列表下标的更多信息，请参考 [*LINDEX*](http://doc.redisfans.com/list/lindex.html#lindex) 命令。

- **可用版本：**

  >= 1.0.0

- **时间复杂度：**

  对头元素或尾元素进行 [LSET](http://doc.redisfans.com/list/lset.html#lset) 操作，复杂度为 O(1)。其他情况下，为 O(N)， `N` 为列表的长度。

- **返回值：**

  操作成功返回 `ok` ，否则返回错误信息。

```
# 对空列表(key 不存在)进行 LSET

redis> EXISTS list
(integer) 0

redis> LSET list 0 item
(error) ERR no such key


# 对非空列表进行 LSET

redis> LPUSH job "cook food"
(integer) 1

redis> LRANGE job 0 0
1) "cook food"

redis> LSET job 0 "play game"
OK

redis> LRANGE job  0 0
1) "play game"


# index 超出范围

redis> LLEN list                    # 列表长度为 1
(integer) 1

redis> LSET list 3 'out of range'
(error) ERR index out of range
```

## LTRIM

**LTRIM key start stop**

对一个列表进行修剪(trim)，就是说，让列表只保留指定区间内的元素，不在指定区间之内的元素都将被删除。

举个例子，执行命令 `LTRIM list 0 2` ，表示只保留列表 `list` 的前三个元素，其余元素全部删除。

下标(index)参数 `start` 和 `stop` 都以 `0` 为底，也就是说，以 `0` 表示列表的第一个元素，以 `1` 表示列表的第二个元素，以此类推。

你也可以使用负数下标，以 `-1` 表示列表的最后一个元素， `-2` 表示列表的倒数第二个元素，以此类推。

当 `key` 不是列表类型时，返回一个错误。

[LTRIM](http://doc.redisfans.com/list/ltrim.html#ltrim) 命令通常和 [*LPUSH*](http://doc.redisfans.com/list/lpush.html#lpush) 命令或 [*RPUSH*](http://doc.redisfans.com/list/rpush.html#rpush) 命令配合使用，举个例子：

```
LPUSH log newest_log
LTRIM log 0 99
```

这个例子模拟了一个日志程序，每次将最新日志 `newest_log` 放到 `log` 列表中，并且只保留最新的 `100` 项。注意当这样使用 `LTRIM` 命令时，时间复杂度是O(1)，因为平均情况下，每次只有一个元素被移除。

**注意LTRIM命令和编程语言区间函数的区别**

假如你有一个包含一百个元素的列表 `list` ，对该列表执行 `LTRIM list 0 10` ，结果是一个包含11个元素的列表，这表明 `stop` 下标也在 [LTRIM](http://doc.redisfans.com/list/ltrim.html#ltrim) 命令的取值范围之内(闭区间)，这和某些语言的区间函数可能不一致，比如Ruby的 `Range.new` 、 `Array#slice` 和Python的 `range()` 函数。

**超出范围的下标**

超出范围的下标值不会引起错误。

如果 `start` 下标比列表的最大下标 `end` ( `LLEN list` 减去 `1` )还要大，或者 `start > stop` ， [LTRIM](http://doc.redisfans.com/list/ltrim.html#ltrim) 返回一个空列表(因为 [LTRIM](http://doc.redisfans.com/list/ltrim.html#ltrim) 已经将整个列表清空)。

如果 `stop` 下标比 `end` 下标还要大，Redis将 `stop` 的值设置为 `end` 。

- **可用版本：**

  >= 1.0.0

- **时间复杂度:**

  O(N)， `N` 为被移除的元素的数量。

- **返回值:**

  命令执行成功时，返回 `ok` 。

```
# 情况 1： 常见情况， start 和 stop 都在列表的索引范围之内
redis> LRANGE alpha 0 -1       # alpha 是一个包含 5 个字符串的列表
1) "h"
2) "e"
3) "l"
4) "l"
5) "o"

redis> LTRIM alpha 1 -1        # 删除 alpha 列表索引为 0 的元素
OK

redis> LRANGE alpha 0 -1       # "h" 被删除了
1) "e"
2) "l"
3) "l"
4) "o"

# 情况 2： stop 比列表的最大下标还要大
redis> LTRIM alpha 1 10086     # 保留 alpha 列表索引 1 至索引 10086 上的元素
OK

redis> LRANGE alpha 0 -1       # 只有索引 0 上的元素 "e" 被删除了，其他元素还在
1) "l"
2) "l"
3) "o"

# 情况 3： start 和 stop 都比列表的最大下标要大，并且 start < stop
redis> LTRIM alpha 10086 123321
OK

redis> LRANGE alpha 0 -1        # 列表被清空
(empty list or set)

# 情况 4： start 和 stop 都比列表的最大下标要大，并且 start > stop
redis> RPUSH new-alpha "h" "e" "l" "l" "o"     # 重新建立一个新列表
(integer) 5

redis> LRANGE new-alpha 0 -1
1) "h"
2) "e"
3) "l"
4) "l"
5) "o"

redis> LTRIM new-alpha 123321 10086    # 执行 LTRIM
OK

redis> LRANGE new-alpha 0 -1           # 同样被清空
(empty list or set)
```

## RPOP

**RPOP key**

移除并返回列表 `key` 的尾元素。

- **可用版本：**

  >= 1.0.0

- **时间复杂度：**

  O(1)

- **返回值：**

  列表的尾元素。当 `key` 不存在时，返回 `nil` 。

```
redis> RPUSH mylist "one"
(integer) 1

redis> RPUSH mylist "two"
(integer) 2

redis> RPUSH mylist "three"
(integer) 3

redis> RPOP mylist           # 返回被弹出的元素
"three"

redis> LRANGE mylist 0 -1    # 列表剩下的元素
1) "one"
2) "two"
```

## RPOPLPUSH

**RPOPLPUSH source destination**

命令 [RPOPLPUSH](http://doc.redisfans.com/list/rpoplpush.html#rpoplpush) 在一个原子时间内，执行以下两个动作：

- 将列表 `source` 中的最后一个元素(尾元素)弹出，并返回给客户端。
- 将 `source` 弹出的元素插入到列表 `destination` ，作为 `destination` 列表的的头元素。

举个例子，你有两个列表 `source` 和 `destination` ， `source` 列表有元素 `a, b, c` ， `destination` 列表有元素 `x, y, z` ，执行 `RPOPLPUSH source destination` 之后， `source` 列表包含元素 `a, b` ， `destination` 列表包含元素 `c, x, y, z` ，并且元素 `c` 会被返回给客户端。

如果 `source` 不存在，值 `nil` 被返回，并且不执行其他动作。

如果 `source` 和 `destination` 相同，则列表中的表尾元素被移动到表头，并返回该元素，可以把这种特殊情况视作列表的旋转(rotation)操作。

- **可用版本：**

  >= 1.2.0

- **时间复杂度：**

  O(1)

- **返回值：**

  被弹出的元素。

```
# source 和 destination 不同

redis> LRANGE alpha 0 -1         # 查看所有元素
1) "a"
2) "b"
3) "c"
4) "d"

redis> RPOPLPUSH alpha reciver   # 执行一次 RPOPLPUSH 看看
"d"

redis> LRANGE alpha 0 -1
1) "a"
2) "b"
3) "c"

redis> LRANGE reciver 0 -1
1) "d"

redis> RPOPLPUSH alpha reciver   # 再执行一次，证实 RPOP 和 LPUSH 的位置正确
"c"

redis> LRANGE alpha 0 -1
1) "a"
2) "b"

redis> LRANGE reciver 0 -1
1) "c"
2) "d"


# source 和 destination 相同

redis> LRANGE number 0 -1
1) "1"
2) "2"
3) "3"
4) "4"

redis> RPOPLPUSH number number
"4"

redis> LRANGE number 0 -1           # 4 被旋转到了表头
1) "4"
2) "1"
3) "2"
4) "3"

redis> RPOPLPUSH number number
"3"

redis> LRANGE number 0 -1           # 这次是 3 被旋转到了表头
1) "3"
2) "4"
3) "1"
4) "2"
```

### 模式： 安全的队列

Redis的列表经常被用作队列(queue)，用于在不同程序之间有序地交换消息(message)。一个客户端通过 [*LPUSH*](http://doc.redisfans.com/list/lpush.html#lpush) 命令将消息放入队列中，而另一个客户端通过 [*RPOP*](http://doc.redisfans.com/list/rpop.html#rpop) 或者 [*BRPOP*](http://doc.redisfans.com/list/brpop.html#brpop) 命令取出队列中等待时间最长的消息。

不幸的是，上面的队列方法是『不安全』的，因为在这个过程中，一个客户端可能在取出一个消息之后崩溃，而未处理完的消息也就因此丢失。

使用 [RPOPLPUSH](http://doc.redisfans.com/list/rpoplpush.html#rpoplpush) 命令(或者它的阻塞版本 [*BRPOPLPUSH*](http://doc.redisfans.com/list/brpoplpush.html#brpoplpush) )可以解决这个问题：因为它不仅返回一个消息，同时还将这个消息添加到另一个备份列表当中，如果一切正常的话，当一个客户端完成某个消息的处理之后，可以用 [*LREM*](http://doc.redisfans.com/list/lrem.html#lrem) 命令将这个消息从备份表删除。

最后，还可以添加一个客户端专门用于监视备份表，它自动地将超过一定处理时限的消息重新放入队列中去(负责处理该消息的客户端可能已经崩溃)，这样就不会丢失任何消息了。

### 模式：循环列表

通过使用相同的 `key` 作为 [RPOPLPUSH](http://doc.redisfans.com/list/rpoplpush.html#rpoplpush) 命令的两个参数，客户端可以用一个接一个地获取列表元素的方式，取得列表的所有元素，而不必像 [*LRANGE*](http://doc.redisfans.com/list/lrange.html#lrange) 命令那样一下子将所有列表元素都从服务器传送到客户端中(两种方式的总复杂度都是 O(N))。

以上的模式甚至在以下的两个情况下也能正常工作：

- 有多个客户端同时对同一个列表进行旋转(rotating)，它们获取不同的元素，直到所有元素都被读取完，之后又从头开始。
- 有客户端在向列表尾部(右边)添加新元素。

这个模式使得我们可以很容易实现这样一类系统：有 N 个客户端，需要连续不断地对一些元素进行处理，而且处理的过程必须尽可能地快。一个典型的例子就是服务器的监控程序：它们需要在尽可能短的时间内，并行地检查一组网站，确保它们的可访问性。

注意，使用这个模式的客户端是易于扩展(scala)且安全(reliable)的，因为就算接收到元素的客户端失败，元素还是保存在列表里面，不会丢失，等到下个迭代来临的时候，别的客户端又可以继续处理这些元素了。

## RPUSH

**RPUSH key value [value ...]**

将一个或多个值 `value` 插入到列表 `key` 的表尾(最右边)。

如果有多个 `value` 值，那么各个 `value` 值按从左到右的顺序依次插入到表尾：比如对一个空列表 `mylist` 执行 `RPUSH mylist a b c` ，得出的结果列表为 `a b c` ，等同于执行命令 `RPUSH mylist a` 、 `RPUSH mylist b` 、 `RPUSH mylist c` 。

如果 `key` 不存在，一个空列表会被创建并执行 [RPUSH](http://doc.redisfans.com/list/rpush.html#rpush) 操作。

当 `key` 存在但不是列表类型时，返回一个错误。

在 Redis 2.4 版本以前的 [RPUSH](http://doc.redisfans.com/list/rpush.html#rpush) 命令，都只接受单个 `value` 值。

- **可用版本：**

  >= 1.0.0

- **时间复杂度：**

  O(1)

- **返回值：**

  执行 [RPUSH](http://doc.redisfans.com/list/rpush.html#rpush) 操作后，表的长度。

```
# 添加单个元素
redis> RPUSH languages c
(integer) 1

# 添加重复元素
redis> RPUSH languages c
(integer) 2

redis> LRANGE languages 0 -1 # 列表允许重复元素
1) "c"
2) "c"

# 添加多个元素
redis> RPUSH mylist a b c
(integer) 3

redis> LRANGE mylist 0 -1
1) "a"
2) "b"
3) "c"
```

## RPUSHX

**RPUSHX key value**

将值 `value` 插入到列表 `key` 的表尾，当且仅当 `key` 存在并且是一个列表。

和 [*RPUSH*](http://doc.redisfans.com/list/rpush.html#rpush) 命令相反，当 `key` 不存在时， [RPUSHX](http://doc.redisfans.com/list/rpushx.html#rpushx) 命令什么也不做。

- **可用版本：**

  >= 2.2.0

- **时间复杂度：**

  O(1)

- **返回值：**

  [RPUSHX](http://doc.redisfans.com/list/rpushx.html#rpushx) 命令执行之后，表的长度。

```
# key不存在
redis> LLEN greet
(integer) 0

redis> RPUSHX greet "hello"     # 对不存在的 key 进行 RPUSHX，PUSH 失败。
(integer) 0

# key 存在且是一个非空列表
redis> RPUSH greet "hi"         # 先用 RPUSH 插入一个元素
(integer) 1

redis> RPUSHX greet "hello"     # greet 现在是一个列表类型，RPUSHX 操作成功。
(integer) 2

redis> LRANGE greet 0 -1
1) "hi"
2) "hello"
```

# Set（集合）

## SADD

**SADD key member [member ...]**

将一个或多个 `member` 元素加入到集合 `key` 当中，已经存在于集合的 `member` 元素将被忽略。

假如 `key` 不存在，则创建一个只包含 `member` 元素作成员的集合。

当 `key` 不是集合类型时，返回一个错误。

在Redis2.4版本以前， [SADD](http://doc.redisfans.com/set/sadd.html#sadd) 只接受单个 `member` 值。

- **可用版本：**

  >= 1.0.0

- **时间复杂度:**

  O(N)， `N` 是被添加的元素的数量。

- **返回值:**

  被添加到集合中的新元素的数量，不包括被忽略的元素。

```
# 添加单个元素
redis> SADD bbs "discuz.net"
(integer) 1

# 添加重复元素
redis> SADD bbs "discuz.net"
(integer) 0

# 添加多个元素
redis> SADD bbs "tianya.cn" "groups.google.com"
(integer) 2

redis> SMEMBERS bbs
1) "discuz.net"
2) "groups.google.com"
3) "tianya.cn"
```

## SCARD

**SCARD key**

返回集合 `key` 的基数(集合中元素的数量)。

- **可用版本：**

  >= 1.0.0

- **时间复杂度:**

  O(1)

- **返回值：**

  集合的基数。当 `key` 不存在时，返回 `0` 。

```
redis> SADD tool pc printer phone
(integer) 3

redis> SCARD tool   # 非空集合
(integer) 3

redis> DEL tool
(integer) 1

redis> SCARD tool   # 空集合
(integer) 0
```

## SDIFF

**SDIFF key [key ...]**

返回一个集合的全部成员，该集合是所有给定集合之间的差集。

不存在的 `key` 被视为空集。

- **可用版本：**

  >= 1.0.0

- **时间复杂度:**

  O(N)， `N` 是所有给定集合的成员数量之和。

- **返回值:**

  交集成员的列表。

```
redis> SADD peter\'s_movies "bet man" "start war" "2012"

redis> SMEMBERS peter's_movies
1) "bet man"
2) "start war"
3) "2012"

redis> SMEMBERS joe's_movies
1) "hi, lady"
2) "Fast Five"
3) "2012"

redis> SDIFF peter's_movies joe's_movies
1) "bet man"
2) "start war"
```

## SDIFFSTORE

**SDIFFSTORE destination key [key ...]**

这个命令的作用和 [*SDIFF*](http://doc.redisfans.com/set/sdiff.html#sdiff) 类似，但它将结果保存到 `destination` 集合，而不是简单地返回结果集。

如果 `destination` 集合已经存在，则将其覆盖。

`destination` 可以是 `key` 本身。

- **可用版本：**

  >= 1.0.0

- **时间复杂度:**

  O(N)， `N` 是所有给定集合的成员数量之和。

- **返回值:**

  结果集中的元素数量。

```
redis> SMEMBERS joe's_movies
1) "hi, lady"
2) "Fast Five"
3) "2012"

redis> SMEMBERS peter's_movies
1) "bet man"
2) "start war"
3) "2012"

redis> SDIFFSTORE joe_diff_peter joe's_movies peter's_movies
(integer) 2

redis> SMEMBERS joe_diff_peter
1) "hi, lady"
2) "Fast Five"
```

## SINTER

**SINTER key [key ...]**

返回一个集合的全部成员，该集合是所有给定集合的交集。

不存在的 `key` 被视为空集。

当给定集合当中有一个空集时，结果也为空集(根据集合运算定律)。

- **可用版本：**

  >= 1.0.0

- **时间复杂度:**

  O(N * M)， `N` 为给定集合当中基数最小的集合， `M` 为给定集合的个数。

- **返回值:**

  交集成员的列表。

```
redis> SMEMBERS group_1
1) "LI LEI"
2) "TOM"
3) "JACK"

redis> SMEMBERS group_2
1) "HAN MEIMEI"
2) "JACK"

redis> SINTER group_1 group_2
1) "JACK"
```

## SINTERSTORE

**SINTERSTORE destination key [key ...]**

这个命令类似于 [*SINTER*](http://doc.redisfans.com/set/sinter.html#sinter) 命令，但它将结果保存到 `destination` 集合，而不是简单地返回结果集。

如果 `destination` 集合已经存在，则将其覆盖。

`destination` 可以是 `key` 本身。

- **可用版本：**

  >= 1.0.0

- **时间复杂度:**

  O(N * M)， `N` 为给定集合当中基数最小的集合， `M` 为给定集合的个数。

- **返回值:**

  结果集中的成员数量。

```
redis> SMEMBERS songs
1) "good bye joe"
2) "hello,peter"

redis> SMEMBERS my_songs
1) "good bye joe"
2) "falling"

redis> SINTERSTORE song_interset songs my_songs
(integer) 1

redis> SMEMBERS song_interset
1) "good bye joe"
```

## SISMEMBER

**SISMEMBER key member**

判断 `member` 元素是否集合 `key` 的成员。

- **可用版本：**

  >= 1.0.0

- **时间复杂度:**

  O(1)

- **返回值:**

  如果 `member` 元素是集合的成员，返回 `1` 。如果 `member` 元素不是集合的成员，或 `key` 不存在，返回 `0` 。

```
redis> SMEMBERS joe's_movies
1) "hi, lady"
2) "Fast Five"
3) "2012"

redis> SISMEMBER joe's_movies "bet man"
(integer) 0

redis> SISMEMBER joe's_movies "Fast Five"
(integer) 1
```

## SMEMBERS

**SMEMBERS key**

返回集合 `key` 中的所有成员。

不存在的 `key` 被视为空集合。

- **可用版本：**

  >= 1.0.0

- **时间复杂度:**

  O(N)， `N` 为集合的基数。

- **返回值:**

  集合中的所有成员。

```
# key 不存在或集合为空

redis> EXISTS not_exists_key
(integer) 0

redis> SMEMBERS not_exists_key
(empty list or set)


# 非空集合

redis> SADD language Ruby Python Clojure
(integer) 3

redis> SMEMBERS language
1) "Python"
2) "Ruby"
3) "Clojure"
```

## SMOVE

**SMOVE source destination member**

将 `member` 元素从 `source` 集合移动到 `destination` 集合。

[SMOVE](http://doc.redisfans.com/set/smove.html#smove) 是原子性操作。

如果 `source` 集合不存在或不包含指定的 `member` 元素，则 [SMOVE](http://doc.redisfans.com/set/smove.html#smove) 命令不执行任何操作，仅返回 `0` 。否则， `member` 元素从 `source` 集合中被移除，并添加到 `destination` 集合中去。

当 `destination` 集合已经包含 `member` 元素时， [SMOVE](http://doc.redisfans.com/set/smove.html#smove) 命令只是简单地将 `source` 集合中的 `member` 元素删除。

当 `source` 或 `destination` 不是集合类型时，返回一个错误。

- **可用版本：**

  >= 1.0.0

- **时间复杂度:**

  O(1)

- **返回值:**

  如果 `member` 元素被成功移除，返回 `1` 。如果 `member` 元素不是 `source` 集合的成员，并且没有任何操作对 `destination` 集合执行，那么返回 `0` 。

```
redis> SMEMBERS songs
1) "Billie Jean"
2) "Believe Me"

redis> SMEMBERS my_songs
(empty list or set)

redis> SMOVE songs my_songs "Believe Me"
(integer) 1

redis> SMEMBERS songs
1) "Billie Jean"

redis> SMEMBERS my_songs
1) "Believe Me"
```

## SPOP

**SPOP key**

移除并返回集合中的一个随机元素。

如果只想获取一个随机元素，但不想该元素从集合中被移除的话，可以使用 [*SRANDMEMBER*](http://doc.redisfans.com/set/srandmember.html#srandmember) 命令。

- **可用版本：**

  >= 1.0.0

- **时间复杂度:**

  O(1)

- **返回值:**

  被移除的随机元素。当 `key` 不存在或 `key` 是空集时，返回 `nil` 。

```
redis> SMEMBERS db
1) "MySQL"
2) "MongoDB"
3) "Redis"

redis> SPOP db
"Redis"

redis> SMEMBERS db
1) "MySQL"
2) "MongoDB"

redis> SPOP db
"MySQL"

redis> SMEMBERS db
1) "MongoDB"
```

## SRANDMEMBER

**SRANDMEMBER key [count]**

如果命令执行时，只提供了 `key` 参数，那么返回集合中的一个随机元素。

从 Redis 2.6 版本开始， [SRANDMEMBER](http://doc.redisfans.com/set/srandmember.html#srandmember) 命令接受可选的 `count` 参数：

- 如果 `count` 为正数，且小于集合基数，那么命令返回一个包含 `count` 个元素的数组，数组中的元素**各不相同**。如果 `count` 大于等于集合基数，那么返回整个集合。
- 如果 `count` 为负数，那么命令返回一个数组，数组中的元素**可能会重复出现多次**，而数组的长度为 `count` 的绝对值。

该操作和 [*SPOP*](http://doc.redisfans.com/set/spop.html#spop) 相似，但 [*SPOP*](http://doc.redisfans.com/set/spop.html#spop) 将随机元素从集合中移除并返回，而 [SRANDMEMBER](http://doc.redisfans.com/set/srandmember.html#srandmember) 则仅仅返回随机元素，而不对集合进行任何改动。

- **可用版本：**

  >= 1.0.0

- **时间复杂度:**

  只提供 `key` 参数时为 O(1) 。如果提供了 `count` 参数，那么为 O(N) ，N 为返回数组的元素个数。

- **返回值:**

  只提供 `key` 参数时，返回一个元素；如果集合为空，返回 `nil` 。如果提供了 `count` 参数，那么返回一个数组；如果集合为空，返回空数组。

```
# 添加元素
redis> SADD fruit apple banana cherry
(integer) 3

# 只给定 key 参数，返回一个随机元素
redis> SRANDMEMBER fruit
"cherry"

redis> SRANDMEMBER fruit
"apple"

# 给定 3 为 count 参数，返回 3 个随机元素
# 每个随机元素都不相同

redis> SRANDMEMBER fruit 3
1) "apple"
2) "banana"
3) "cherry"

# 给定 -3 为 count 参数，返回 3 个随机元素
# 元素可能会重复出现多次

redis> SRANDMEMBER fruit -3
1) "banana"
2) "cherry"
3) "apple"

redis> SRANDMEMBER fruit -3
1) "apple"
2) "apple"
3) "cherry"

# 如果 count 是整数，且大于等于集合基数，那么返回整个集合
redis> SRANDMEMBER fruit 10
1) "apple"
2) "banana"
3) "cherry"

# 如果 count 是负数，且 count 的绝对值大于集合的基数
# 那么返回的数组的长度为 count 的绝对值

redis> SRANDMEMBER fruit -10
1) "banana"
2) "apple"
3) "banana"
4) "cherry"
5) "apple"
6) "apple"
7) "cherry"
8) "apple"
9) "apple"
10) "banana"

# SRANDMEMBER 并不会修改集合内容

redis> SMEMBERS fruit
1) "apple"
2) "cherry"
3) "banana"

# 集合为空时返回 nil 或者空数组

redis> SRANDMEMBER not-exists
(nil)

redis> SRANDMEMBER not-eixsts 10
(empty list or set)
```

## SREM

**SREM key member [member ...]**

移除集合 `key` 中的一个或多个 `member` 元素，不存在的 `member` 元素会被忽略。

当 `key` 不是集合类型，返回一个错误。

在 Redis 2.4 版本以前， [SREM](http://doc.redisfans.com/set/srem.html#srem) 只接受单个 `member` 值。

- **可用版本：**

  >= 1.0.0

- **时间复杂度:**

  O(N)， `N` 为给定 `member` 元素的数量。

- **返回值:**

  被成功移除的元素的数量，不包括被忽略的元素。

```
# 测试数据
redis> SMEMBERS languages
1) "c"
2) "lisp"
3) "python"
4) "ruby"

# 移除单个元素
redis> SREM languages ruby
(integer) 1

# 移除不存在元素
redis> SREM languages non-exists-language
(integer) 0

# 移除多个元素
redis> SREM languages lisp python c
(integer) 3

redis> SMEMBERS languages
(empty list or set)
```

## SUNION

**SUNION key [key ...]**

返回一个集合的全部成员，该集合是所有给定集合的并集。

不存在的 `key` 被视为空集。

- **可用版本：**

  >= 1.0.0

- **时间复杂度:**

  O(N)， `N` 是所有给定集合的成员数量之和。

- **返回值:**

  并集成员的列表。

```
redis> SMEMBERS songs
1) "Billie Jean"

redis> SMEMBERS my_songs
1) "Believe Me"

redis> SUNION songs my_songs
1) "Billie Jean"
2) "Believe Me"
```

## SUNIONSTORE

**SUNIONSTORE destination key [key ...]**

这个命令类似于 [*SUNION*](http://doc.redisfans.com/set/sunion.html#sunion) 命令，但它将结果保存到 `destination` 集合，而不是简单地返回结果集。

如果 `destination` 已经存在，则将其覆盖。

`destination` 可以是 `key` 本身。

- **可用版本：**

  >= 1.0.0

- **时间复杂度:**

  O(N)， `N` 是所有给定集合的成员数量之和。

- **返回值:**

  结果集中的元素数量。

```
redis> SMEMBERS NoSQL
1) "MongoDB"
2) "Redis"

redis> SMEMBERS SQL
1) "sqlite"
2) "MySQL"

redis> SUNIONSTORE db NoSQL SQL
(integer) 4

redis> SMEMBERS db
1) "MySQL"
2) "sqlite"
3) "MongoDB"
4) "Redis"
```

## SSCAN

**SSCAN key cursor [MATCH pattern] [COUNT count]**

详细信息请参考 [*SCAN*](http://doc.redisfans.com/key/scan.html#scan) 命令。

# SortedSet（有序集合）





















































































