# Redis

## 配置

### 后台启动(开启守护进程)

1. 首先找到redis的配置文件 
2. 将 daemonize no 修改为 daemonize yes 

### 开启日志

**配置方法：**

1、首先找到redis的配置文件 

2、**打开配置文件，找到logfile（可能有多个logfile，认准旁边有loglevel的那个），或者直接搜logfile ""**

3、**将路径填入logfile后面的引号内，例如：logfile "d:/redislog/redis.log" (注意斜杆的方向，这个和windows cmd中的斜杆方向是反的)**

4、**根据自己写的路径，手动将日志文件夹建好，日志文件不用建，建到文件夹即可，比如我就手动建立了d:\redislog 文件夹**

5、保存配置文件，以这个配置文件启动redis，然后这时候redis的启动框会变成一个黑框框，什么输出都没有，这就对了（因为输入全写到日志文件去了）

然后就可以去d:\redislog\redis.log文件夹去查看日志了

**其他注意事项：**

1、redis必须带配置文件启动，如果直接启动的话，它会使用默认配置（而且并不存在这个默认配置文件，所以不要想改它）。

2、如果出现

![redis-62.png](C:\Users\pi'ka'chu\Documents\redis\笔记\img\1574996545600181.png)

提示，说明没有指定配置文件或者配置文件读取不到（路径错误）

3、loglevel是用来设置日志等级的，具体可以看配置文件中上面的注释

### 最大客户端连接数

Linux可通过ulimit查看和设置系统当前用户进程的资源数。ulimit –a包含打开文件的参数，是单个用户可以同时打开的最大文件个数。

Redis允许有多个客户端通过网络进行连接，可以通过配置maxclients来限制最大客户端连接数。

Redis启动的时候，经常可以看到以下日志：

![img](C:\Users\pi'ka'chu\Documents\redis\笔记\img\70)

建议把maxclients设置成10032，默认是10000用来处理客户端,而且内部还会使用最多32个文件描述符。

但是Redis进程因为没有权限无法将open files设成10032

当前系统open files是4096，你可以将maxclinet设置成4096-32 = 4064个，如果想设置更高的maxclinets,使用ulimit –n来设置

### 持久化

**持久化定义**

```python
将数据从掉电易失的内存，放到永久存储的设备上
```

**为什么需要持久化**

```python
因为所有的数据都在内存上，所以必须得持久化
```

- **数据持久化分类之 - RDB模式（默认开启）**

**默认模式**

```python
1、保存真实的数据
2、将服务器包含的所有数据库数据以二进制文件的形式保存到硬盘里面
3、默认文件名 ：/var/lib/redis/dump.rdb
```

**创建rdb文件的两种方式**

**方式一：**服务器执行客户端发送的SAVE或者BGSAVE命令

```python
127.0.0.1:6379> SAVE
OK
# 特点
1、执行SAVE命令过程中，redis服务器将被阻塞，无法处理客户端发送的命令请求，在SAVE命令执行完毕后，服务器才会重新开始处理客户端发送的命令请求
2、如果RDB文件已经存在，那么服务器将自动使用新的RDB文件代替旧的RDB文件
# 工作中定时持久化保存一个文件

127.0.0.1:6379> BGSAVE
Background saving started
# 执行过程如下
1、客户端 发送 BGSAVE 给服务器
2、服务器马上返回 Background saving started 给客户端
3、服务器 fork() 子进程做这件事情
4、服务器继续提供服务
5、子进程创建完RDB文件后再告知Redis服务器

# 配置文件相关操作
/etc/redis/redis.conf
263行: dir /var/lib/redis # 表示rdb文件存放路径
253行: dbfilename dump.rdb  # 文件名

# 两个命令比较
SAVE比BGSAVE快，因为需要创建子进程，消耗额外的内存

# 补充：可以通过查看日志文件来查看redis都做了哪些操作
# 日志文件：配置文件中搜索 logfile
logfile /var/log/redis/redis-server.log
```

**方式二：\**设置配置文件条件满足时自动保存\**（使用最多）**

```python
# 命令行示例
redis>save 300 10
  表示如果距离上一次创建RDB文件已经过去了300秒，并且服务器的所有数据库总共已经发生了不少于10次修改，那么自动执行BGSAVE命令
redis>save 60 10000
  表示如果距离上一次创建rdb文件已经过去60秒，并且服务器所有数据库总共已经发生了不少于10000次修改，那么执行bgsave命令

# redis配置文件默认
218行: save 900 1
219行: save 300 10
220行: save 60 10000
  1、只要三个条件中的任意一个被满足时，服务器就会自动执行BGSAVE
  2、每次创建RDB文件之后，服务器为实现自动持久化而设置的时间计数器和次数计数器就会被清零，并重新开始计数，所以多个保存条件的效果不会叠加
```

- **数据持久化分类之 - AOF（AppendOnlyFile，默认未开启）**

**特点**

```python
1、存储的是命令，而不是真实数据
2、默认不开启
# 开启方式（修改配置文件）
1、/etc/redis/redis.conf
  672行: appendonly yes # 把 no 改为 yes
  676行: appendfilename "appendonly.aof"
2、重启服务
  sudo /etc/init.d/redis-server restart
```

**RDB缺点**

```python
1、创建RDB文件需要将服务器所有的数据库的数据都保存起来，这是一个非常消耗资源和时间的操作，所以服务器需要隔一段时间才创建一个新的RDB文件，也就是说，创建RDB文件不能执行的过于频繁，否则会严重影响服务器的性能
2、可能丢失数据
```

**AOF持久化原理及优点**

```python
# 原理
   1、每当有修改数据库的命令被执行时， 
   2、因为AOF文件里面存储了服务器执行过的所有数据库修改的命令，所以给定一个AOF文件，服务器只要重新执行一遍AOF文件里面包含的所有命令，就可以达到还原数据库的目的

# 优点
  用户可以根据自己的需要对AOF持久化进行调整，让Redis在遭遇意外停机时不丢失任何数据，或者只丢失一秒钟的数据，这比RDB持久化丢失的数据要少的多
```

**安全性问题考虑**

```python
# 因为
  虽然服务器执行一个修改数据库的命令，就会把执行的命令写入到AOF文件，但这并不意味着AOF文件持久化不会丢失任何数据，在目前常见的操作系统中，执行系统调用write函数，将一些内容写入到某个文件里面时，为了提高效率，系统通常不会直接将内容写入硬盘里面，而是将内容放入一个内存缓存区（buffer）里面，等到缓冲区被填满时才将存储在缓冲区里面的内容真正写入到硬盘里

# 所以
  1、AOF持久化：当一条命令真正的被写入到硬盘里面时，这条命令才不会因为停机而意外丢失
  2、AOF持久化在遭遇停机时丢失命令的数量，取决于命令被写入到硬盘的时间
  3、越早将命令写入到硬盘，发生意外停机时丢失的数据就越少，反之亦然
```

**策略 - 配置文件**

```python
# 打开配置文件:/etc/redis/redis.conf，找到相关策略如下
1、701行: alwarys
   服务器每写入一条命令，就将缓冲区里面的命令写入到硬盘里面，服务器就算意外停机，也不会丢失任何已经成功执行的命令数据
2、702行: everysec（# 默认）
   服务器每一秒将缓冲区里面的命令写入到硬盘里面，这种模式下，服务器即使遭遇意外停机，最多只丢失1秒的数据
3、703行: no
   服务器不主动将命令写入硬盘,由操作系统决定何时将缓冲区里面的命令写入到硬盘里面，丢失命令数量不确定

# 运行速度比较
always：速度慢
everysec和no都很快，默认值为everysec
```

**AOF文件中是否会产生很多的冗余命令？**

```python
为了让AOF文件的大小控制在合理范围，避免胡乱增长，redis提供了AOF重写功能，通过这个功能，服务器可以产生一个新的AOF文件
  -- 新的AOF文件记录的数据库数据和原由的AOF文件记录的数据库数据完全一样
  -- 新的AOF文件会使用尽可能少的命令来记录数据库数据，因此新的AOF文件的提及通常会小很多
  -- AOF重写期间，服务器不会被阻塞，可以正常处理客户端发送的命令请求
```

**示例**

| 原有AOF文件                | 重写后的AOF文件                         |
| -------------------------- | --------------------------------------- |
| select 0                   | SELECT 0                                |
| sadd myset peiqi           | SADD myset peiqi qiaozhi danni lingyang |
| sadd myset qiaozhi         | SET msg ‘hello tarena’                  |
| sadd myset danni           | RPUSH mylist 2 3 5                      |
| sadd myset lingyang        |                                         |
| INCR number                |                                         |
| INCR number                |                                         |
| DEL number                 |                                         |
| SET message ‘hello world’  |                                         |
| SET message ‘hello tarena’ |                                         |
| RPUSH mylist 1 2 3         |                                         |
| RPUSH mylist 5             |                                         |
| LPOP mylist                |                                         |

**AOF文件重写方法触发**

```python
1、客户端向服务器发送BGREWRITEAOF命令
   127.0.0.1:6379> BGREWRITEAOF
   Background append only file rewriting started

2、修改配置文件让服务器自动执行BGREWRITEAOF命令
  auto-aof-rewrite-percentage 100
  auto-aof-rewrite-min-size 64mb
  # 解释
    1、只有当AOF文件的增量大于100%时才进行重写，也就是大一倍的时候才触发
        # 第一次重写新增：64M
        # 第二次重写新增：128M
        # 第三次重写新增：256M（新增128M）
```

**RDB和AOF持久化对比**

| RDB持久化                                                    | AOF持久化                                     |
| ------------------------------------------------------------ | --------------------------------------------- |
| 全量备份，一次保存整个数据库                                 | 增量备份，一次保存一个修改数据库的命令        |
| 保存的间隔较长                                               | 保存的间隔默认为一秒钟                        |
| 数据还原速度快                                               | 数据还原速度一般，冗余命令多，还原速度慢      |
| 执行SAVE命令时会阻塞服务器，但手动或者自动触发的BGSAVE不会阻塞服务器 | 无论是平时还是进行AOF重写时，都不会阻塞服务器 |
|                                                              |                                               |

```python
# 用redis用来存储真正数据，每一条都不能丢失，都要用always，有的做缓存，有的保存真数据，我可以开多个redis服务，不同业务使用不同的持久化，新浪每个服务器上有4个redis服务，整个业务中有上千个redis服务，分不同的业务，每个持久化的级别都是不一样的。
```

**数据恢复（无需手动操作）**

```python
既有dump.rdb，又有appendonly.aof，恢复时找谁？
先找appendonly.aof
```

**配置文件常用配置总结**

```python
# 设置密码
1、requirepass password
# 开启远程连接
2、bind 127.0.0.1 ::1 注释掉
3、protected-mode no  把默认的 yes 改为 no
# rdb持久化-默认配置
4、dbfilename 'dump.rdb'
5、dir /var/lib/redis
# rdb持久化-自动触发(条件)
6、save 900 1
7、save 300 10 
8、save 60  10000
# aof持久化开启
9、appendonly yes
10、appendfilename 'appendonly.aof'
# aof持久化策略
11、appendfsync always
12、appendfsync everysec # 默认
13、appendfsync no
# aof重写触发
14、auto-aof-rewrite-percentage 100
15、auto-aof-rewrite-min-size 64mb
# 设置为从服务器
16、salveof <master-ip> <master-port>
```

**Redis相关文件存放路径**

```shell
1、配置文件: /etc/redis/redis.conf
2、备份文件: /var/lib/redis/*.rdb|*.aof
3、日志文件: /var/log/redis/redis-server.log
4、启动文件: /etc/init.d/redis-server
# /etc/下存放配置文件
# /etc/init.d/下存放服务启动文件
```

###  Redis设置密码

设置密码有两种方式。

#### 1. 命令行设置密码。

运行cmd切换到redis根目录，先启动服务端

```shell
>redis-server.exe
```

另开一个cmd切换到redis根目录，启动客户端

```css
>redis-cli.exe -h 127.0.0.1 -p 6379
```

客户端使用config get requirepass命令查看密码

```csharp
>config get requirepass
1)"requirepass"
2)""    //默认空
```

客户端使用config set requirepass yourpassword命令设置密码

```shell
>config set requirepass 123456
>OK
```

一旦设置密码，必须先验证通过密码，否则所有操作不可用

```lua
>config get requirepass
(error)NOAUTH Authentication required
```

使用auth password验证密码

```csharp
>auth 123456
>OK
>config get requirepass
1)"requirepass"
2)"123456"
```

也可以退出重新登录

```css
redis-cli.exe -h 127.0.0.1 -p 6379 -a 123456
```

命令行设置的密码在服务重启后失效，所以一般不使用这种方式。

#### 2. 配置文件设置密码

在redis根目录下找到redis.windows.conf配置文件，搜索requirepass，找到注释密码行，添加密码如下：

```cpp
# requirepass foobared
requirepass tenny     //注意，行前不能有空格
```

重启服务后，客户端重新登录后发现

```csharp
>config get requirepass
1)"requirepass"
2)""
```

密码还是空？

网上查询后的办法：创建redis-server.exe 的快捷方式， 右键快捷方式属性，在目标后面增加redis.windows.conf， 这里就是关键，你虽然修改了.conf文件，但是exe却没有使用这个conf，所以我们需要**手动指定**一下exe按照**修改后的conf**运行，就OK了。

所以，这里我再一次重启redis服务(指定配置文件)

```shell
>redis-server.exe redis.windows.conf
```

客户端再重新登录，OK了。

```csharp
>redis-cli.exe -h 127.0.0.1 -p 6379 -a 123456
>config get requirepass
1)"requirepass"
2)"123456"
```



## 整合springboot

1. 依赖:

```xml
        <dependency>
            <groupId>org.springframework.boot</groupId>
            <artifactId>spring-boot-starter</artifactId>
        </dependency>
        <dependency>
            <groupId>org.springframework.boot</groupId>
            <artifactId>spring-boot-starter-web</artifactId>
        </dependency>
        <dependency>
            <groupId>org.springframework.boot</groupId>
            <artifactId>spring-boot-starter-test</artifactId>
            <scope>test</scope>
        </dependency>
        <!-- redis -->
        <dependency>
            <groupId>org.springframework.boot</groupId>
            <artifactId>spring-boot-starter-data-redis</artifactId>
        </dependency>

        <!-- spring2.X集成redis所需common-pool2-->
        <dependency>
            <groupId>org.apache.commons</groupId>
            <artifactId>commons-pool2</artifactId>
            <version>2.6.0</version>
        </dependency>
        <!--json-->
        <dependency>
            <groupId>com.fasterxml.jackson.datatype</groupId>
            <artifactId>jackson-datatype-jsr310</artifactId>
            <version>2.9.2</version>
        </dependency>
```



2. 配置文件

```properties
#连接超时,单位秒
spring.redis.timeout=10000
spring.redis.lettuce.pool.max-active=20
spring.redis.jedis.pool.max-wait=-1
spring.redis.jedis.pool.max-idle=5
spring.redis.jedis.pool.min-idle=0
```

3. 配置类:RedisConfig

```java

import com.fasterxml.jackson.annotation.JsonAutoDetect;
import com.fasterxml.jackson.annotation.PropertyAccessor;
import com.fasterxml.jackson.databind.ObjectMapper;
import org.springframework.cache.CacheManager;
import org.springframework.cache.annotation.EnableCaching;
import org.springframework.context.annotation.Bean;
import org.springframework.context.annotation.Configuration;
import org.springframework.data.redis.cache.RedisCacheConfiguration;
import org.springframework.data.redis.cache.RedisCacheManager;
import org.springframework.data.redis.connection.RedisConnectionFactory;
import org.springframework.data.redis.core.RedisTemplate;
import org.springframework.data.redis.serializer.Jackson2JsonRedisSerializer;
import org.springframework.data.redis.serializer.RedisSerializationContext;
import org.springframework.data.redis.serializer.RedisSerializer;
import org.springframework.data.redis.serializer.StringRedisSerializer;

import java.time.Duration;

/**
 * Desc:
 *
 * @author pikachu
 * @date: 2022/9/12 10:44
 */
@EnableCaching
@Configuration
public class RedisConfig {
    public RedisConfig() {
    }

    @Bean
    public RedisTemplate<String, Object> redisTemplate(RedisConnectionFactory factory) {
        RedisTemplate<String, Object> template = new RedisTemplate<>();
        RedisSerializer<String> redisSerializer = new StringRedisSerializer();
        Jackson2JsonRedisSerializer jackson2JsonRedisSerializer = new Jackson2JsonRedisSerializer(Object.class);
        ObjectMapper om = new ObjectMapper();
        om.setVisibility(PropertyAccessor.ALL, JsonAutoDetect.Visibility.ANY);
        om.enableDefaultTyping(ObjectMapper.DefaultTyping.NON_FINAL);
        jackson2JsonRedisSerializer.setObjectMapper(om);
        template.setConnectionFactory(factory);
        //key序列化方式
        template.setKeySerializer(redisSerializer);
        //value序列化
        template.setValueSerializer(jackson2JsonRedisSerializer);
        //value hashmap序列化
        template.setHashValueSerializer(jackson2JsonRedisSerializer);
        return template;
    }

    @Bean
    public CacheManager cacheManager(RedisConnectionFactory factory) {
        RedisSerializer<String> redisSerializer = new StringRedisSerializer();

        Jackson2JsonRedisSerializer jackson2JsonRedisSerializer = new Jackson2JsonRedisSerializer(Object.class);
        //解决查询缓存转换异常的问题
        ObjectMapper om = new ObjectMapper();
        om.setVisibility(PropertyAccessor.ALL, JsonAutoDetect.Visibility.ANY);
        om.enableDefaultTyping(ObjectMapper.DefaultTyping.NON_FINAL);
        jackson2JsonRedisSerializer.setObjectMapper(om);
        // 配置序列化（解决乱码的问题）,过期时间600秒
        RedisCacheConfiguration config = RedisCacheConfiguration.defaultCacheConfig()
                .entryTtl(Duration.ofSeconds(600))
                .serializeKeysWith(RedisSerializationContext.SerializationPair.fromSerializer(redisSerializer))
                .serializeValuesWith(RedisSerializationContext.SerializationPair.fromSerializer(jackson2JsonRedisSerializer))
                .disableCachingNullValues();
        RedisCacheManager cacheManager = RedisCacheManager.builder(factory)
                .cacheDefaults(config)
                .build();
        return cacheManager;
    }
}
```





## 遇到的问题

### Jedis

#### 连接

> Failed to connect to any host resolved for DNS name.

解决：

1.启动redis-server

2.开放端口6379

```shell
iptables -I INPUT -p tcp --dport 6379 -j ACCEPT
```

3.修改 redis.conf 配置文件

```shell
protected-mode no   #redis开始了保护模式，默认是yes
将 bind 127.0.0.1 注释掉    #只有本机可以连接redis服务连接，所以我们把它注释掉
```



#### 事务

> ERR EXEC without MULTI

执行代码报错:

```java
redisTemplate.opsForHash().put("joker", "age", "27");
redisTemplate.watch("joker");
redisTemplate.multi();
redisTemplate.opsForHash().put("joker", "pet", "beibei");
redisTemplate.exec();
```



运行这段代码，程序就会给出`Caused by: org.springframework.data.redis.RedisSystemException: Error in execution; nested exception is io.lettuce.core.RedisCommandExecutionException: ERR EXEC without MULTI`错误，但是我明明执行`multi()`了呀~

1. 原因:

我们一层一层的剥开，可以找到这么一个干实事的函数：

```java
	/**
	 * Executes the given action object within a connection that can be exposed or not. Additionally, the connection can
	 * be pipelined. Note the results of the pipeline are discarded (making it suitable for write-only scenarios).
	 *
	 * @param <T> return type
	 * @param action callback object to execute
	 * @param exposeConnection whether to enforce exposure of the native Redis Connection to callback code
	 * @param pipeline whether to pipeline or not the connection for the execution
	 * @return object returned by the action
	 */
	@Nullable
	public <T> T execute(RedisCallback<T> action, boolean exposeConnection, boolean pipeline) {

		Assert.isTrue(initialized, "template not initialized; call afterPropertiesSet() before using it");
		Assert.notNull(action, "Callback object must not be null");

		RedisConnectionFactory factory = getRequiredConnectionFactory();
		RedisConnection conn = null;
		try {
            // 1
			if (enableTransactionSupport) {
				// only bind resources in case of potential transaction synchronization
				conn = RedisConnectionUtils.bindConnection(factory, enableTransactionSupport);
			} else {
				conn = RedisConnectionUtils.getConnection(factory);
			}

			boolean existingConnection = TransactionSynchronizationManager.hasResource(factory);

			RedisConnection connToUse = preProcessConnection(conn, existingConnection);

			boolean pipelineStatus = connToUse.isPipelined();
			if (pipeline && !pipelineStatus) {
				connToUse.openPipeline();
			}

			RedisConnection connToExpose = (exposeConnection ? connToUse : createRedisConnectionProxy(connToUse));
			T result = action.doInRedis(connToExpose);

			// close pipeline
			if (pipeline && !pipelineStatus) {
				connToUse.closePipeline();
			}

			// TODO: any other connection processing?
			return postProcessResult(result, connToUse, existingConnection);
		} finally {
			RedisConnectionUtils.releaseConnection(conn, factory, enableTransactionSupport);
		}
	}
```

在代码`1`处，可以看到有`enableTransactionSupport`这么一个参数，看一下他的值是`false`的话，那么会重新拿一个连接(而且他的默认值还就是`false`)，这也就解释了为啥我们明明执行`multi`了，但是还没说我们在`exec`前没有`multi`~
但是，如果`enableTransactionSupport`的值是`true`呢，他又干了啥呢？我们一路点进去，找到了这么一个函数：

```java
	/**
	 * Gets a Redis connection. Is aware of and will return any existing corresponding connections bound to the current
	 * thread, for example when using a transaction manager. Will create a new Connection otherwise, if
	 * {@code allowCreate} is <tt>true</tt>.
	 *
	 * @param factory connection factory for creating the connection.
	 * @param allowCreate whether a new (unbound) connection should be created when no connection can be found for the
	 *          current thread.
	 * @param bind binds the connection to the thread, in case one was created-
	 * @param transactionSupport whether transaction support is enabled.
	 * @return an active Redis connection.
	 */
	public static RedisConnection doGetConnection(RedisConnectionFactory factory, boolean allowCreate, boolean bind,
			boolean transactionSupport) {

		Assert.notNull(factory, "No RedisConnectionFactory specified");
        // 1
		RedisConnectionHolder connHolder = (RedisConnectionHolder) TransactionSynchronizationManager.getResource(factory);

		if (connHolder != null) { // 2
			if (transactionSupport) {
				potentiallyRegisterTransactionSynchronisation(connHolder, factory); // 3
			}
			return connHolder.getConnection();
		}

		if (!allowCreate) {
			throw new IllegalArgumentException("No connection found and allowCreate = false");
		}

		if (log.isDebugEnabled()) {
			log.debug("Opening RedisConnection");
		}

		RedisConnection conn = factory.getConnection(); // 4

		if (bind) {

			RedisConnection connectionToBind = conn;
			if (transactionSupport && isActualNonReadonlyTransactionActive()) {
				connectionToBind = createConnectionProxy(conn, factory);
			}

			connHolder = new RedisConnectionHolder(connectionToBind); 

			TransactionSynchronizationManager.bindResource(factory, connHolder);// 5
			if (transactionSupport) { 
				potentiallyRegisterTransactionSynchronisation(connHolder, factory);
			}

			return connHolder.getConnection(); // 8
		}

		return conn;
	}
```

说明：

1. 这里有一个新的东西：`TransactionSynchronizationManager`，这是由`spring`提供的，他里面有一个叫`resources`的成员，他是一个`ThreadLocal`。所以这一行代码，就很清楚了，他是去拿到跟当前线程绑定的连接。
2. 这里就是判断啊，当前线程是否绑定了这么一个连接。
3. 如果拿到了跟当前线程绑定的连接，且`enableTransactionSupport`的值是`true`，那么需要做一些操作~ 不过这些操作是同`spring`的事务相关的，在我们的代码中，不会执行~
4. 但是，我们第一次执行啊，好像没有给当前线程绑定过连接，所以上一步是执行不到的~ 这里创建一个连接~
5. 然后，在这里，我们把当前线程和连接绑定起来~

所以，综上，为啥我们的代码不对呢，因为`RedisTemplate`默认是不开启事务支持的，而且在执行`exec`方法时，会重新创建一个连接对象(或者从当前线程的`ThreadLocal`中拿到上一次绑定的连接)。所以，我们在不开启事务的情况下，自己在外面执行的`multi`方法时完全不会生效的(因为连接对象都换了)~

2. 解决

看到这，原因既然已经知道了，那么自然就迎刃而解了~
最简单的方式，既然默认是不开启事务支持的，那么我们手动把他打开不就好了~
执行: `redisTemplate.setEnableTransactionSupport(true);`即可~

可能有些地方描述的不是很清楚，我们还是拿我们的例子来说，还是上面那段代码：

```java
redisTemplate.opsForHash().put("joker", "age", "27"); // 1
redisTemplate.setEnableTransactionSupport(true); // 2
redisTemplate.watch("joker"); // 3
redisTemplate.multi(); // 4
redisTemplate.opsForHash().put("joker", "pet", "beibei"); // 5
redisTemplate.exec(); // 6
```

说明：

1. 初始化一条数据~
2. 开始事务支持
3. `watch`一个`key`，同时在这一步执行时，会创建一个新的连接并与当前线程绑定~
4. 执行`multi`，这里会拿到上一步与当前线程绑定的连接，并通过该连接调用`multi`方法~
5. 再加一条数据~
6. 执行`exec`方法，同样是拿到与线程绑定的连接后，通过该连接执行`exec`方法~ 因为该连接已经执行了`watch`和`multi`，所以在此之前，对应的`key`如果发生变化，那么，不会执行成功，我们的目的也就达到了~

不过，这种方法还有一个问题，大家可以顺着源代码继续往下捋~ 会发现，与当前线程绑定的连接不会解绑，更不会被`close`~
所以，感觉`RedisTemplate`提供的`SessionCallback`才是正解~

```java
redisTemplate.execute(new SessionCallback<List<Object>>() {
    public List<Object> execute(RedisOperations operations) throws DataAccessException {
        operations.watch("joker");
        operations.multi();
        operations.opsForHash().put("joker", "pet", "beibei");
        return operations.exec();
    }
});
```

`RedisTemplate`的`public <T> T execute(SessionCallback<T> session)`方法，会在`finally`中调用`RedisConnectionUtils.unbindConnection(factory);`来解除执行过程中与当前线程绑定的连接，并在随后关闭连接。



### “内存不足”

问题描述：

>  MISCONF Redis is configured to save RDB snapshots, but it is currently not able to persist on disk. Commands that may modify the data set are disabled, because this instance is configured to report errors during writes if RDB snapshotting fails (stop-writes-on-bgsave-error option). Please check the Redis logs for details about the RDB error.

@see

异常里面显示无法连接上jedis ，第一时间我去查看服务器上redis的运行情况，发现redis仍然是在运行中的，然后当我想ping一下redis测试一下的时候，出现了以下错误

```delphi
MISCONF Redis is configured to save RDB snapshots, but is currently not able to persist on disk
```

 意思是：Redis被配置为保存数据库快照，但它目前不能持久化到硬盘。用来修改集合数据的命令不能用。请查看Redis日志的详细错误信息。

去网上搜了一下这个问题，网上说原因是强制关闭Redis快照导致不能持久化。给出的解决方案是

1. 运行config set stop-writes-on-bgsave-error no　命令，关闭配置项stop-writes-on-bgsave-error解决该问题。
2. 或者在 redis.conf 中将 stop-writes-on-bgsave-error 设置为no

```shell
root@ubuntu:/usr/local/redis/bin# ./redis-cli
127.0.0.1:6379> config set stop-writes-on-bgsave-error no
```

完成以上操作后，重新刷新网页~真的可以了，牛逼！！然后我就去睡觉了。

然而又过了两天，网站又崩了~问题和上面一样，于是又去网上找解决方案，在这篇文章中找到了解决方案（https://www.cnblogs.com/qq78292959/p/3994349.html）

文章中说道将config set stop-writes-on-bgsave-error 设置为no仅仅是让redis忽略了这个异常，使得程序能够继续往下运行，但实际上数据还是会存储到硬盘失败。

查看redis的日志，会发现一行警告：

“WARNING overcommit_memory is set to 0! Background save may fail under low memory condition. To fix this issue add 'vm.overcommit_memory = 1' to /etc/sysctl.conf and then reboot or run the command 'sysctl vm.overcommit_memory=1' for this to take effect.”（警告：过量使用内存设置为0！在低内存环境下，后台保存可能失败。为了修正这个问题，请在/etc/sysctl.conf 添加一项 'vm.overcommit_memory = 1' ，然后重启（或者运行命令'sysctl vm.overcommit_memory=1' ）使其生效。）

意思是说我系统内存不足，我看了下系统内存 free -m，明明还有挺多内存的啊。

![img](C:\Users\pi'ka'chu\Documents\redis\笔记\img\20190427223743162.png)

迷迷糊糊中我跟着修改vm.overcommit_memory=1后问题果然解决了。有个问题，那明明系统内存还够，为什么redis会认为redis内存不足呢。上面链接中的文章也给出了问题原因分析

redis认为内存不足原因：http://www.linuxidc.com/Linux/2012-07/66079.htm，简单地说：Redis在保存数据到硬盘时为了避免主进程假死，需要Fork一份主进程，然后在Fork进程内完成数据保存到硬盘的操作，如果主进程使用了4GB的内存，Fork子进程的时候需要额外的4GB，此时内存就不够了，Fork失败，进而数据保存硬盘也失败了。

而将vm.overcommit_memory改为1有什么作用呢，网上看到一个博客是如下解释，我个人也比较同意

0 — 默认设置。个人理解：当应用进程尝试申请内存时，内核会做一个检测。内核将检查是否有足够的可用内存供应用进程使用；如果有足够的可用内存，内存申请允许；否则，内存申请失败，并把错误返回给应用进程。举个例子，比如1G的机器，A进程已经使用了500M，当有另外进程尝试malloc 500M的内存时，内核就会进行check，发现超出剩余可用内存，就会提示失败。
1 — 对于内存的申请请求，内核不会做任何check，直到物理内存用完，触发OOM杀用户态进程。同样是上面的例子，1G的机器，A进程500M，B进程尝试malloc 500M，会成功，但是一旦kernel发现内存使用率接近1个G(内核有策略)，就触发OOM，杀掉一些用户态的进程(有策略的杀)。
2 — 当 请求申请的内存 >= SWAP内存大小 + 物理内存 * N，则拒绝此次内存申请。解释下这个N：N是一个百分比，根据overcommit_ratio/100来确定，比如overcommit_ratio=50，那么N就是50%。

### 持久化

**问题：**

使用时redis断开，报错信息如下：

> Failed opening the RDB file dump.rdb (in server root dir /etc/profile.d) for saving: Permission denied

**原因：**

由于启动redis使用默认配置，持久化会在当前目录进行。

启动redis时在root权限目录下启动的，写权限不足，导致持久化时进程失败。

**解决：**

1. 停掉redis，在其他有写权限目录下再次重启解决。
2. 配置redis持久化目录

