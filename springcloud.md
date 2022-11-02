# What’s your problems

## Stream

### Rabbitmq连接

1. 错误的配置:

```yaml
spring:
  application:
    name: cloud-stream-provider
  cloud:
    stream:
      binders: # 在此处配置要绑定的rabbitmq的服务信息；
        defaultRabbit: # 表示定义的名称，用于于binding整合
          type: rabbit # 消息组件类型
          environment: # 设置rabbitmq的相关的环境配置
            #注意仅在这里配置rabbitmq服务并没有生效,会连接Attempting to connect to: [localhost:5672]
            #还需要在spring.rabbitmq下配置连接属性,基表如此也不能将其删除
            spring:
              rabbitmq:
                host: 192.168.10.111
                port: 5672
                username: admin
                password: 123
      bindings: # 服务的整合处理
        output: # 这个名字是一个通道的名称
          binder: defaultRabbit # 设置要绑定的消息服务的具体设置
          destination: studyExchange # 表示要使用的Exchange名称定义
          content-type: application/json # 设置消息类型，本次为json，文本则设置“text/plain”
```

按以上配置启动程序后,日志输出:

```java
2022-10-28 09:31:36.477  INFO 10736 --- [192.168.112.230] o.s.a.r.c.CachingConnectionFactory       : Attempting to connect to: [localhost:5672]
2022-10-28 09:31:40.585  WARN 10736 --- [192.168.112.230] o.s.b.a.amqp.RabbitHealthIndicator       : Rabbit health check failed

org.springframework.amqp.AmqpConnectException: java.net.ConnectException: Connection refused: connect
```

可以看到连接的rabbitmq并不是machine111,这是由于引入了actuactor的监控检测默认配置,若不设置默认rabbitmq,则会使用默认配置进行尝试,localhost:5672



2. 正确写法

   需要在spring.rabbitmq下配置连接属性,即使如此也不能将其删除

   ```yaml
   server:
     port: 8801
   spring:
     rabbitmq:
       host: machine111
       port: 5672
       username: admin
       password: 123
     application:
       name: cloud-stream-provider
     cloud:
       stream:
         binders: # 在此处配置要绑定的rabbitmq的服务信息；
           defaultRabbit: # 表示定义的名称，用于于binding整合
             type: rabbit # 消息组件类型
             environment: # 设置rabbitmq的相关的环境配置
               #注意仅在这里配置rabbitmq服务并没有生效,会连接Attempting to connect to: [localhost:5672]
               #还需要在spring.rabbitmq下配置连接属性,即使如此也不能将其删除
               spring:
                 rabbitmq:
                   host: 192.168.10.111
                   port: 5672
                   username: admin
                   password: 123
         bindings: # 服务的整合处理
           output: # 这个名字是一个通道的名称
             binder: defaultRabbit # 设置要绑定的消息服务的具体设置
             destination: studyExchange # 表示要使用的Exchange名称定义
             content-type: application/json # 设置消息类型，本次为json，文本则设置“text/plain”
   eureka:
     client:
       service-url:
         defaultZone: http://eureka7001.com:7001/eureka
       register-with-eureka: true
       fetch-registry: true
     instance:
       instance-id: send-8001.com
       prefer-ip-address: true
       lease-renewal-interval-in-seconds: 2
       lease-expiration-duration-in-seconds: 5
   ```

   此时日志打印信息:

   ```java
2022-10-28 09:51:57.238  INFO 5488 --- [192.168.112.230] o.s.a.r.c.CachingConnectionFactory       : Attempting to connect to: [machine111:5672]
   2022-10-28 09:51:57.259  INFO 5488 --- [192.168.112.230] o.s.a.r.c.CachingConnectionFactory       : Created new connection: rabbitConnectionFactory#26b9c5b9:0/SimpleConnection@62fc9a44 [delegate=amqp://admin@192.168.10.111:5672/, localPort= 9505]
   ```
```
   
   
   
   ### 参考
   
   #### 创建生产者

   ##### 1. 引入依赖

   ```javascript
<dependency>
               <groupId>org.springframework.cloud</groupId>
               <artifactId>spring-cloud-starter-stream-rabbit</artifactId>
           </dependency>
```

   

   ##### 2. 定义配置文件

   ```javascript
   spring:
     cloud:
       stream:
         binders:
           test:
             type: rabbit
             environment:
               spring:
                 rabbitmq:
                   addresses: 10.0.20.132
                   port: 5672
                username: root
                   password: root
                virtual-host: /unicode-pay
         bindings:
           testOutPut:
             destination: testRabbit
          content-type: application/json
             default-binder: test
   ```

   现在来解释一下这些配置的含义

   1. binders： 这是一组binder的集合，这里配置了一个名为test的binder，这个binder中是包含了一个rabbit的连接信息
   2. bindings：这是一组binding的集合，这里配置了一个名为testOutPut的binding，这个binding中配置了指向名test的binder下的一个交换机testRabbit。
   3. 扩展： 如果我们项目中不仅集成了rabbit还集成了kafka那么就可以新增一个类型为kafka的binder、如果项目中会使用多个交换机那么就使用多个binding，

##### 3.创建通道

```javascript
   public interface  MqMessageSource {
    String TEST_OUT_PUT = "testOutPut";
       @Output(TEST_OUT_PUT)
       MessageChannel testOutPut();
   }
```

   这个通道的名字就是上方binding的名字

   ##### 4. 发送消息

   ```javascript
   @EnableBinding(MqMessageSource.class)
   public class MqMessageProducer {
    @Autowired
       @Output(MqMessageSource.TEST_OUT_PUT)
    private MessageChannel channel;
       public void sendMsg(String msg) {
        channel.send(MessageBuilder.withPayload(msg).build());
           System.err.println("消息发送成功："+msg);
    }
   }
   ```

   

   这里就是使用上方的通道来发送到指定的交换机了。需要注意的是withPayload方法你可以传入任何类型的对象，但是需要实现序列化接口

   ##### 5. 创建测试接口

   EnableBinding注解绑定的类默认是被Spring管理的，我们可以在controller中注入它

```javascript
   @Autowired
private MqMessageProducer mqMessageProducer;
   @GetMapping(value = "/testMq")
public String testMq(@RequestParam("msg")String msg){
       mqMessageProducer.sendMsg(msg);
    return "发送成功";
   }
```

   

   生产者的代码到此已经完成了。

   ### 创建消费者

   ##### 1. 引入依赖

   ```javascript
<dependency>
               <groupId>org.springframework.cloud</groupId>
               <artifactId>spring-cloud-starter-stream-rabbit</artifactId>
           </dependency>
   ```

   

   ##### 2. 定义配置文件

   ```javascript
   spring:
     cloud:
       stream:
         binders:
           test:
             type: rabbit
             environment:
               spring:
                 rabbitmq:
                   addresses: 10.0.20.132
                   port: 5672
                username: root
                   password: root
                virtual-host: /unicode-pay
         bindings:
        testInPut:
             destination: testRabbit
          content-type: application/json
             default-binder: test
   ```

   

   这里与生产者唯一不同的地方就是testIntPut了，相信你已经明白了，它是binding的名字，也是通道与交换机绑定的关键

##### 3.创建通道

```javascript
   public interface  MqMessageSource {
    String TEST_IN_PUT = "testInPut";
       @Input(TEST_IN_PUT)
       SubscribableChannel testInPut();
   }
```

   

   ##### 4. 接受消息

```javascript
   @EnableBinding(MqMessageSource.class)
public class MqMessageConsumer {
       @StreamListener(MqMessageSource.TEST_IN_PUT)
       public void messageInPut(Message<String> message) {
           System.err.println(" 消息接收成功：" + message.getPayload());
       }
   }
```

   

   这个时候启动Eureka、消息生产者和消费者，然后调用生产者的接口应该就可以接受到来自mq的消息了。



## Nacos

版本2.1.2

### 启动出错

#### 1.1 Caused by: java.net.UnknownHostException: jmenv.tbsite.net

解决: 以单机模式启动

```shell
startup.cmd -m standalone
```



#### 1.2  No DataSource set

```java
Caused by: java.lang.IllegalStateException: No DataSource set
        at org.springframework.util.Assert.state(Assert.java:76)
        at org.springframework.jdbc.support.JdbcAccessor.obtainDataSource(JdbcAccessor.java:86)
        at org.springframework.jdbc.core.JdbcTemplate.execute(JdbcTemplate.java:376)
        at org.springframework.jdbc.core.JdbcTemplate.query(JdbcTemplate.java:465)
        at org.springframework.jdbc.core.JdbcTemplate.query(JdbcTemplate.java:475)
        at org.springframework.jdbc.core.JdbcTemplate.queryForObject(JdbcTemplate.java:508)
        at org.springframework.jdbc.core.JdbcTemplate.queryForObject(JdbcTemplate.java:515)
        at com.alibaba.nacos.config.server.service.repository.extrnal.ExternalStoragePersistServiceImpl.findConfigMaxId(ExternalStoragePersistServiceImpl.java:674)
        at com.alibaba.nacos.config.server.service.dump.processor.DumpAllProcessor.process(DumpAllProcessor.java:51)
        at com.alibaba.nacos.config.server.service.dump.DumpService.dumpConfigInfo(DumpService.java:282)
        at com.alibaba.nacos.config.server.service.dump.DumpService.dumpOperate(DumpService.java:195)
        ... 65 common frames omitted
```

配置文件(数据库及连接池部分):

```properties

#*************** Config Module Related Configurations ***************#
### If use MySQL as datasource:
spring.datasource.platform=mysql

### Count of DB:
db.num=1

### Connect URL of DB:
db.url.0=jdbc:mysql://localhost:13306/nacos_config?characterEncoding=utf8&useUnicode=true&useSSL=false
db.user=root
db.password=root

### Connection pool configuration: hikariCP
db.pool.config.connectionTimeout=30000
db.pool.config.validationTimeout=10000
db.pool.config.maximumPoolSize=20
db.pool.config.minimumIdle=2
```



可能原因: 

未初始化创建配置nacos数据库及相关配置信息表,

数据库用户密码填写错误(这里使用mysql5.x),nacos自带的jdbc驱动的mysql8.x,同时兼容mysql5.x

创建表:

```sql
CREATE DATABASE `nacos_config` CHARACTER SET 'utf8' COLLATE 'utf8_general_ci';
```

执行sql脚本:mysq-schema.sql

![image-20221029151450008](imgs/image-20221029151450008.png)



```sql
/*
 * Copyright 1999-2018 Alibaba Group Holding Ltd.
 *
 * Licensed under the Apache License, Version 2.0 (the "License");
 * you may not use this file except in compliance with the License.
 * You may obtain a copy of the License at
 *
 *      http://www.apache.org/licenses/LICENSE-2.0
 *
 * Unless required by applicable law or agreed to in writing, software
 * distributed under the License is distributed on an "AS IS" BASIS,
 * WITHOUT WARRANTIES OR CONDITIONS OF ANY KIND, either express or implied.
 * See the License for the specific language governing permissions and
 * limitations under the License.
 */

/******************************************/
/*   数据库全名 = nacos_config   */
/*   表名称 = config_info   */
/******************************************/
CREATE TABLE `config_info` (
  `id` bigint(20) NOT NULL AUTO_INCREMENT COMMENT 'id',
  `data_id` varchar(255) NOT NULL COMMENT 'data_id',
  `group_id` varchar(255) DEFAULT NULL,
  `content` longtext NOT NULL COMMENT 'content',
  `md5` varchar(32) DEFAULT NULL COMMENT 'md5',
  `gmt_create` datetime NOT NULL DEFAULT CURRENT_TIMESTAMP COMMENT '创建时间',
  `gmt_modified` datetime NOT NULL DEFAULT CURRENT_TIMESTAMP COMMENT '修改时间',
  `src_user` text COMMENT 'source user',
  `src_ip` varchar(50) DEFAULT NULL COMMENT 'source ip',
  `app_name` varchar(128) DEFAULT NULL,
  `tenant_id` varchar(128) DEFAULT '' COMMENT '租户字段',
  `c_desc` varchar(256) DEFAULT NULL,
  `c_use` varchar(64) DEFAULT NULL,
  `effect` varchar(64) DEFAULT NULL,
  `type` varchar(64) DEFAULT NULL,
  `c_schema` text,
  `encrypted_data_key` text NOT NULL COMMENT '秘钥',
  PRIMARY KEY (`id`),
  UNIQUE KEY `uk_configinfo_datagrouptenant` (`data_id`,`group_id`,`tenant_id`)
) ENGINE=InnoDB DEFAULT CHARSET=utf8 COLLATE=utf8_bin COMMENT='config_info';

/******************************************/
/*   数据库全名 = nacos_config   */
/*   表名称 = config_info_aggr   */
/******************************************/
CREATE TABLE `config_info_aggr` (
  `id` bigint(20) NOT NULL AUTO_INCREMENT COMMENT 'id',
  `data_id` varchar(255) NOT NULL COMMENT 'data_id',
  `group_id` varchar(255) NOT NULL COMMENT 'group_id',
  `datum_id` varchar(255) NOT NULL COMMENT 'datum_id',
  `content` longtext NOT NULL COMMENT '内容',
  `gmt_modified` datetime NOT NULL COMMENT '修改时间',
  `app_name` varchar(128) DEFAULT NULL,
  `tenant_id` varchar(128) DEFAULT '' COMMENT '租户字段',
  PRIMARY KEY (`id`),
  UNIQUE KEY `uk_configinfoaggr_datagrouptenantdatum` (`data_id`,`group_id`,`tenant_id`,`datum_id`)
) ENGINE=InnoDB DEFAULT CHARSET=utf8 COLLATE=utf8_bin COMMENT='增加租户字段';


/******************************************/
/*   数据库全名 = nacos_config   */
/*   表名称 = config_info_beta   */
/******************************************/
CREATE TABLE `config_info_beta` (
  `id` bigint(20) NOT NULL AUTO_INCREMENT COMMENT 'id',
  `data_id` varchar(255) NOT NULL COMMENT 'data_id',
  `group_id` varchar(128) NOT NULL COMMENT 'group_id',
  `app_name` varchar(128) DEFAULT NULL COMMENT 'app_name',
  `content` longtext NOT NULL COMMENT 'content',
  `beta_ips` varchar(1024) DEFAULT NULL COMMENT 'betaIps',
  `md5` varchar(32) DEFAULT NULL COMMENT 'md5',
  `gmt_create` datetime NOT NULL DEFAULT CURRENT_TIMESTAMP COMMENT '创建时间',
  `gmt_modified` datetime NOT NULL DEFAULT CURRENT_TIMESTAMP COMMENT '修改时间',
  `src_user` text COMMENT 'source user',
  `src_ip` varchar(50) DEFAULT NULL COMMENT 'source ip',
  `tenant_id` varchar(128) DEFAULT '' COMMENT '租户字段',
  `encrypted_data_key` text NOT NULL COMMENT '秘钥',
  PRIMARY KEY (`id`),
  UNIQUE KEY `uk_configinfobeta_datagrouptenant` (`data_id`,`group_id`,`tenant_id`)
) ENGINE=InnoDB DEFAULT CHARSET=utf8 COLLATE=utf8_bin COMMENT='config_info_beta';

/******************************************/
/*   数据库全名 = nacos_config   */
/*   表名称 = config_info_tag   */
/******************************************/
CREATE TABLE `config_info_tag` (
  `id` bigint(20) NOT NULL AUTO_INCREMENT COMMENT 'id',
  `data_id` varchar(255) NOT NULL COMMENT 'data_id',
  `group_id` varchar(128) NOT NULL COMMENT 'group_id',
  `tenant_id` varchar(128) DEFAULT '' COMMENT 'tenant_id',
  `tag_id` varchar(128) NOT NULL COMMENT 'tag_id',
  `app_name` varchar(128) DEFAULT NULL COMMENT 'app_name',
  `content` longtext NOT NULL COMMENT 'content',
  `md5` varchar(32) DEFAULT NULL COMMENT 'md5',
  `gmt_create` datetime NOT NULL DEFAULT CURRENT_TIMESTAMP COMMENT '创建时间',
  `gmt_modified` datetime NOT NULL DEFAULT CURRENT_TIMESTAMP COMMENT '修改时间',
  `src_user` text COMMENT 'source user',
  `src_ip` varchar(50) DEFAULT NULL COMMENT 'source ip',
  PRIMARY KEY (`id`),
  UNIQUE KEY `uk_configinfotag_datagrouptenanttag` (`data_id`,`group_id`,`tenant_id`,`tag_id`)
) ENGINE=InnoDB DEFAULT CHARSET=utf8 COLLATE=utf8_bin COMMENT='config_info_tag';

/******************************************/
/*   数据库全名 = nacos_config   */
/*   表名称 = config_tags_relation   */
/******************************************/
CREATE TABLE `config_tags_relation` (
  `id` bigint(20) NOT NULL COMMENT 'id',
  `tag_name` varchar(128) NOT NULL COMMENT 'tag_name',
  `tag_type` varchar(64) DEFAULT NULL COMMENT 'tag_type',
  `data_id` varchar(255) NOT NULL COMMENT 'data_id',
  `group_id` varchar(128) NOT NULL COMMENT 'group_id',
  `tenant_id` varchar(128) DEFAULT '' COMMENT 'tenant_id',
  `nid` bigint(20) NOT NULL AUTO_INCREMENT,
  PRIMARY KEY (`nid`),
  UNIQUE KEY `uk_configtagrelation_configidtag` (`id`,`tag_name`,`tag_type`),
  KEY `idx_tenant_id` (`tenant_id`)
) ENGINE=InnoDB DEFAULT CHARSET=utf8 COLLATE=utf8_bin COMMENT='config_tag_relation';

/******************************************/
/*   数据库全名 = nacos_config   */
/*   表名称 = group_capacity   */
/******************************************/
CREATE TABLE `group_capacity` (
  `id` bigint(20) unsigned NOT NULL AUTO_INCREMENT COMMENT '主键ID',
  `group_id` varchar(128) NOT NULL DEFAULT '' COMMENT 'Group ID，空字符表示整个集群',
  `quota` int(10) unsigned NOT NULL DEFAULT '0' COMMENT '配额，0表示使用默认值',
  `usage` int(10) unsigned NOT NULL DEFAULT '0' COMMENT '使用量',
  `max_size` int(10) unsigned NOT NULL DEFAULT '0' COMMENT '单个配置大小上限，单位为字节，0表示使用默认值',
  `max_aggr_count` int(10) unsigned NOT NULL DEFAULT '0' COMMENT '聚合子配置最大个数，，0表示使用默认值',
  `max_aggr_size` int(10) unsigned NOT NULL DEFAULT '0' COMMENT '单个聚合数据的子配置大小上限，单位为字节，0表示使用默认值',
  `max_history_count` int(10) unsigned NOT NULL DEFAULT '0' COMMENT '最大变更历史数量',
  `gmt_create` datetime NOT NULL DEFAULT CURRENT_TIMESTAMP COMMENT '创建时间',
  `gmt_modified` datetime NOT NULL DEFAULT CURRENT_TIMESTAMP COMMENT '修改时间',
  PRIMARY KEY (`id`),
  UNIQUE KEY `uk_group_id` (`group_id`)
) ENGINE=InnoDB DEFAULT CHARSET=utf8 COLLATE=utf8_bin COMMENT='集群、各Group容量信息表';

/******************************************/
/*   数据库全名 = nacos_config   */
/*   表名称 = his_config_info   */
/******************************************/
CREATE TABLE `his_config_info` (
  `id` bigint(20) unsigned NOT NULL,
  `nid` bigint(20) unsigned NOT NULL AUTO_INCREMENT,
  `data_id` varchar(255) NOT NULL,
  `group_id` varchar(128) NOT NULL,
  `app_name` varchar(128) DEFAULT NULL COMMENT 'app_name',
  `content` longtext NOT NULL,
  `md5` varchar(32) DEFAULT NULL,
  `gmt_create` datetime NOT NULL DEFAULT CURRENT_TIMESTAMP,
  `gmt_modified` datetime NOT NULL DEFAULT CURRENT_TIMESTAMP,
  `src_user` text,
  `src_ip` varchar(50) DEFAULT NULL,
  `op_type` char(10) DEFAULT NULL,
  `tenant_id` varchar(128) DEFAULT '' COMMENT '租户字段',
  `encrypted_data_key` text NOT NULL COMMENT '秘钥',
  PRIMARY KEY (`nid`),
  KEY `idx_gmt_create` (`gmt_create`),
  KEY `idx_gmt_modified` (`gmt_modified`),
  KEY `idx_did` (`data_id`)
) ENGINE=InnoDB DEFAULT CHARSET=utf8 COLLATE=utf8_bin COMMENT='多租户改造';


/******************************************/
/*   数据库全名 = nacos_config   */
/*   表名称 = tenant_capacity   */
/******************************************/
CREATE TABLE `tenant_capacity` (
  `id` bigint(20) unsigned NOT NULL AUTO_INCREMENT COMMENT '主键ID',
  `tenant_id` varchar(128) NOT NULL DEFAULT '' COMMENT 'Tenant ID',
  `quota` int(10) unsigned NOT NULL DEFAULT '0' COMMENT '配额，0表示使用默认值',
  `usage` int(10) unsigned NOT NULL DEFAULT '0' COMMENT '使用量',
  `max_size` int(10) unsigned NOT NULL DEFAULT '0' COMMENT '单个配置大小上限，单位为字节，0表示使用默认值',
  `max_aggr_count` int(10) unsigned NOT NULL DEFAULT '0' COMMENT '聚合子配置最大个数',
  `max_aggr_size` int(10) unsigned NOT NULL DEFAULT '0' COMMENT '单个聚合数据的子配置大小上限，单位为字节，0表示使用默认值',
  `max_history_count` int(10) unsigned NOT NULL DEFAULT '0' COMMENT '最大变更历史数量',
  `gmt_create` datetime NOT NULL DEFAULT CURRENT_TIMESTAMP COMMENT '创建时间',
  `gmt_modified` datetime NOT NULL DEFAULT CURRENT_TIMESTAMP COMMENT '修改时间',
  PRIMARY KEY (`id`),
  UNIQUE KEY `uk_tenant_id` (`tenant_id`)
) ENGINE=InnoDB DEFAULT CHARSET=utf8 COLLATE=utf8_bin COMMENT='租户容量信息表';


CREATE TABLE `tenant_info` (
  `id` bigint(20) NOT NULL AUTO_INCREMENT COMMENT 'id',
  `kp` varchar(128) NOT NULL COMMENT 'kp',
  `tenant_id` varchar(128) default '' COMMENT 'tenant_id',
  `tenant_name` varchar(128) default '' COMMENT 'tenant_name',
  `tenant_desc` varchar(256) DEFAULT NULL COMMENT 'tenant_desc',
  `create_source` varchar(32) DEFAULT NULL COMMENT 'create_source',
  `gmt_create` bigint(20) NOT NULL COMMENT '创建时间',
  `gmt_modified` bigint(20) NOT NULL COMMENT '修改时间',
  PRIMARY KEY (`id`),
  UNIQUE KEY `uk_tenant_info_kptenantid` (`kp`,`tenant_id`),
  KEY `idx_tenant_id` (`tenant_id`)
) ENGINE=InnoDB DEFAULT CHARSET=utf8 COLLATE=utf8_bin COMMENT='tenant_info';

CREATE TABLE `users` (
	`username` varchar(50) NOT NULL PRIMARY KEY,
	`password` varchar(500) NOT NULL,
	`enabled` boolean NOT NULL
);

CREATE TABLE `roles` (
	`username` varchar(50) NOT NULL,
	`role` varchar(50) NOT NULL,
	UNIQUE INDEX `idx_user_role` (`username` ASC, `role` ASC) USING BTREE
);

CREATE TABLE `permissions` (
    `role` varchar(50) NOT NULL,
    `resource` varchar(255) NOT NULL,
    `action` varchar(8) NOT NULL,
    UNIQUE INDEX `uk_role_permission` (`role`,`resource`,`action`) USING BTREE
);

INSERT INTO users (username, password, enabled) VALUES ('nacos', '$2a$10$EuWPZHzz32dJN7jexM34MOeYirDdFAZm2kuWj7VEOJhhZkDrxfvUu', TRUE);

INSERT INTO roles (username, role) VALUES ('nacos', 'ROLE_ADMIN');

```



完成以上步骤再次启动,成功.

![image-20221029152101993](imgs/image-20221029152101993.png)

默认的用户名和密码为nacos

![image-20221029152529891](imgs/image-20221029152529891.png)





###  参考数据库配置参数

#### 单机模式下运行Nacos

Linux/Unix/Mac

- Standalone means it is non-cluster Mode. * sh [startup.sh](http://startup.sh/) -m standalone

Windows

- Standalone means it is non-cluster Mode. * cmd startup.cmd -m standalone

单机模式支持mysql

在0.7版本之前，在单机模式时nacos使用嵌入式数据库实现数据的存储，不方便观察数据存储的基本情况。0.7版本增加了支持mysql数据源能力，具体的操作步骤：

- 1.安装数据库，版本要求：5.6.5+
- 2.初始化mysql数据库，数据库初始化文件：mysql-schema.sql
- 3.修改conf/application.properties文件，增加支持mysql数据源配置（目前只支持mysql），添加mysql数据源的url、用户名和密码。

```
spring.datasource.platform=mysql

db.num=1
db.url.0=jdbc:mysql://11.162.196.16:3306/nacos_devtest?characterEncoding=utf8&connectTimeout=1000&socketTimeout=3000&autoReconnect=true
db.user=nacos_devtest
db.password=youdontknow
```

再以单机模式启动nacos，nacos所有写嵌入式数据库的数据都写到了mysql



#### Config模块

| 参数名                                           | 含义                                                         | 可选值 | 默认值                 | 支持版本 |
| ------------------------------------------------ | ------------------------------------------------------------ | ------ | ---------------------- | -------- |
| db.num                                           | 数据库数目                                                   | 正整数 | 0                      | >= 0.1.0 |
| db.url.0                                         | 第一个数据库的URL                                            | 字符串 | 空                     | >= 0.1.0 |
| db.url.1                                         | 第二个数据库的URL                                            | 字符串 | 空                     | >= 0.1.0 |
| db.user                                          | 数据库连接的用户名                                           | 字符串 | 空                     | >= 0.1.0 |
| db.password                                      | 数据库连接的密码                                             | 字符串 | 空                     | >= 0.1.0 |
| spring.datasource.platform                       | 数据库类型                                                   | 字符串 | mysql                  | >=1.3.0  |
| [db.pool.config.xxx](http://db.pool.config.xxx/) | 数据库连接池参数，使用的是hikari连接池，参数与hikari连接池相同，如`db.pool.config.connectionTimeout`或`db.pool.config.maximumPoolSize` | 字符串 | 同hikariCp对应默认配置 | >=1.4.1  |

当前数据库配置支持多数据源。通过`db.num`来指定数据源个数，`db.url.index`为对应的数据库的链接。`db.user`以及`db.password`没有设置`index`时,所有的链接都以`db.user`和`db.password`用作认证。如果不同数据源的用户名称或者用户密码不一样时，可以通过符号`,`来进行切割，或者指定`db.user.index`,`db.user.password`来设置对应数据库链接的用户或者密码。需要注意的是，当`db.user`和`db.password`没有指定下标时，因为当前机制会根据`,`进行切割。所以当用户名或者密码存在`,`时，会把`,`切割后前面的值当成最后的值进行认证，会导致认证失败。

Nacos从1.3版本开始使用HikariCP连接池，但在1.4.1版本前，连接池配置由系统默认值定义，无法自定义配置。在1.4.1后，提供了一个方法能够配置HikariCP连接池。 `db.pool.config`为配置前缀，`xxx`为实际的hikariCP配置，如`db.pool.config.connectionTimeout`或`db.pool.config.maximumPoolSize`等。更多hikariCP的配置请查看[HikariCP](https://github.com/brettwooldridge/HikariCP) 需要注意的是，url,user,password会由`db.url.n`,`db.user`,`db.password`覆盖，driverClassName则是默认的MySQL8 driver（该版本mysql driver支持mysql5.x)



## Ribbon

### RestTemplate

使用RestTemplate时首先要往容器中注入bean,并添加 @loadbalance 注解

```java
@Configuration
public class FeignLogConfiguration {
    @Bean
    public Logger.Level level() {
        return Logger.Level.FULL;
    }
}
```



### OpenFeign

若未集成,则需要手动添加依赖

```xml
        <dependency>
            <groupId>org.springframework.cloud</groupId>
            <artifactId>spring-cloud-starter-openfeign</artifactId>
        </dependency>
```

在配置类中添加 @EnableFeignClients

```java
@SpringBootApplication
@EnableDiscoveryClient
@EnableFeignClients
public class ConsumerMain83 {
    public static void main(String[] args) {
        SpringApplication.run(ConsumerMain83.class, args);
    }
}
```

服务接口中添加: @Component

```java
@FeignClient(name = "${my.remote-server}")
@Component
public interface FeignTestService {
    @GetMapping("/nacos/payment/test")
    String test();
}
```



## Sentinel

### 限流规则

#### 流控模式

![img](imgs/061b8c23ef694d6e9d18abe3647c9dbe.png)

一、流控模式-直接

添加规则：

![img](imgs/6d5781b7da1345f59e3321b3c8806571.png)

 ![img](imgs/8df1b81beca34a90a71af4aa2f1b8026.png)



测试例子分析：

![img](imgs/39b96b130c9046078bc6d632580aff3e.png)



 启动测试

![img](imgs/5ec171ee24494c7488e3b09e3d7d8b47.png)



点击 **察看结果树**

![img](imgs/cc093a02f2be44fd938dfcc603a06ca3.png)

 上面测试例子，到[Sentinel](https://so.csdn.net/so/search?q=Sentinel&spm=1001.2101.3001.7020)控制台的实时监控可以看到![img](imgs/3dd6594966864a3a8f8d230c66387980.png)

 ![img](imgs/57592224a235458b826fc55819f5ea3d.png)



二、流控模式-关联

• ***\*关联模式\**** **：**统计与当前资源相关的另一个资源，触发阈值时，对当前资源限流

• **使用场景** ：比如用户支付时需要修改订单状态，同时用户要查询订单。查询和修改操作会争            抢数据库锁，产生竞争。业务需求是有限支付和更新订单的业务，因此当修改订            单业务触发阈值时，需要对查询订单业务限流。

![img](imgs/b8cf0fa094f0443db55cd6567f70b192.png)

当**/write**资源访问量触发阈值时，就会对***\*/read\****资源限流，避免影响/write资源。

### 案例：

​    需求：  

​     •在OrderController新建两个端点：/order/query和/order/update，无需实现业务  

​     •配置流控规则，当/order/ update资源被访问的QPS超过5时，对/order/query请求限流

1. 编写测试controller方法:

![img](imgs/dfda223212c6437a96852a8aa4d744e0.png)

2. 添加规则（想给谁限流，就给谁添加规则）

![img](imgs/9fc5937b14504fce8dde85b0c1a5cfb6.png)

![img](imgs/d93155d750694872a0818677f4cb81a8.png)

![img](imgs/ddfa4ee5ed00436c9c0f3e147d8c7b6d.png)

3. 借助JMeter进行测试：

![img](imgs/5ff99d435d124355914f816b525b9a7a.png)



![img](imgs/38c110824b1f478883e86b43a4df0017.png) 4. 去网页访问验证:![img](imgs/6d7e4884e3314d76a707bf0e0e6548d0.png)

query被限流 ![img](imgs/21342817472a4d2587669ada29df161c.png)

5. 总结： 满足下面条件可以使用关联模式

1. 两个有竞争关系的资源  

2. 一个优先级较高，一个优先级较低（优先级高的触发阈值时（本案例的order），对优先级低的限流（本案例的query））



三、流控模式-链路

![img](imgs/56e2dc9f451441a2bbb8483750137bf4.png)

 案例：

![img](imgs/00c8c659e1c44e098c46b7ce5fd7b219.png)



1. 编写测试代码：

![img](imgs/540402db64874fab953436a286717e02.png)

 ![img](imgs/32565b6a61534e3d87a9de1d1d9b7446.png)



![img](imgs/b0c343dd8ae54320bed317a725bda5d6.png)

​     

2. 注意:

 Sentinel默认只标记Controller中的方法为资源，如果要标记其它方法，需要利用**@SentinelResource**注解

去配置文件里配置，关闭context，就可以让controller里的方法单独成为一个链路；不关闭context的话，controller里的方法都会默认进去sentinel默认的根链路里，这样就只有一条链路，无法流控链路模式

![img](imgs/dba8e68e6bc049b18fceab90848e5a24.png)



3. 启动之后，并到网页里分别访问了/order/query和/order/save接口后

![img](imgs/bb6e754ffb7e4e4f8b715fe3910cbecd.png)

4. 添加规则：(对query做限制，save没有做限制)

![img](imgs/cc38bc601afc440f9405972e04257f5b.png)

 ![img](imgs/feb37cb8556e49c4a151ff9c8397497f.png)



5. 借助JMeter来测试：

![img](imgs/ed662f7afdb54e60a0455ce48d55c18e.png)

![img](imgs/7f1e916b436d43379585d1e9a52828a4.png)

启动测试 ![img](imgs/4917bbf6a6b34670aed4ad9d3c751b02.png)

![img](imgs/45dd3918b3124af6983bfb648adfbb3a.png)

![img](imgs/df349c02421040b9bc8e00a398be0c9d.png)



## Seata

### 启动失败

版本

seata: 1.5.1	(事务管理)

nacos: 2.1.2	(注册和配置中心)

pom文件信息

```xml
        <!--nacos-->
        <dependency>
            <groupId>com.alibaba.cloud</groupId>
            <artifactId>spring-cloud-starter-alibaba-nacos-discovery</artifactId>
            <version>2.1.2.RELEASE</version>
            <exclusions>
                <exclusion>
                    <groupId>com.alibaba.nacos</groupId>
                    <artifactId>nacos-client</artifactId>
                </exclusion>
            </exclusions>
        </dependency>
        <!--nacos-config-->
        <dependency>
            <groupId>com.alibaba.cloud</groupId>
            <artifactId>spring-cloud-starter-alibaba-nacos-config</artifactId>
            <version>2.1.2.RELEASE</version>
            <exclusions>
                <exclusion>
                    <groupId>com.alibaba.nacos</groupId>
                    <artifactId>nacos-client</artifactId>
                </exclusion>
            </exclusions>
        </dependency>
        <!-- https://mvnrepository.com/artifact/com.alibaba.nacos/nacos-client -->
        <dependency>
            <groupId>com.alibaba.nacos</groupId>
            <artifactId>nacos-client</artifactId>
            <version>2.1.0</version>
        </dependency>

        <!--seata-->
        <dependency>
            <groupId>io.seata</groupId>
            <artifactId>seata-spring-boot-starter</artifactId>
            <version>1.5.1</version>
        </dependency>

        <dependency>
            <groupId>io.seata</groupId>
            <artifactId>seata-all</artifactId>
            <version>1.5.1</version>
        </dependency>

        <dependency>
            <groupId>com.alibaba.cloud</groupId>
            <artifactId>spring-cloud-starter-alibaba-seata</artifactId>
            <version>2.2.0.RELEASE</version>
            <exclusions>
                <exclusion>
                    <groupId>io.seata</groupId>
                    <artifactId>seata-spring-boot-starter</artifactId>
                </exclusion>
            </exclusions>
        </dependency>
```

yaml文件配置:

> nacos服务注册到 nacos的namespace,group,application,cluster要与client端一致



server:

```yaml
server:
  port: 7091

spring:
  application:
    name: seata-server

logging:
  config: classpath:logback-spring.xml
  file:
    path: ${user.home}/logs/seata
  extend:
    logstash-appender:
      destination: 127.0.0.1:4560
    kafka-appender:
      bootstrap-servers: 127.0.0.1:9092
      topic: logback_to_logstash

console:
  user:
    username: seata
    password: seata

seata:
  tx-service-group: my_test_tx_group
  config:
    # support: nacos, consul, apollo, zk, etcd3
    type: nacos
    nacos:
      server-addr: 127.0.0.1:8848
      namespace: 1008611
      group: SEATA_GROUP
      username: nacos
      password: nacos
      ##if use MSE Nacos with auth, mutex with username/password attribute
      #access-key: ""
      #secret-key: ""
      data-id: seata.properties
  registry:
    # support: nacos, eureka, redis, zk, consul, etcd3, sofa
    type: nacos
    preferred-networks: 30.240.*
    nacos:
      application: seata-server
      server-addr: 127.0.0.1:8848
      group: SEATA_GROUP
      namespace: 1008611
      username: nacos
      password: nacos
      cluster: seata-server
      
  store:
    # support: file 、 db 、 redis
    mode: db
    db:
      datasource: druid
      db-type: mysql
      driver-class-name: com.mysql.jdbc.Driver
      url: jdbc:mysql://127.0.0.1:13306/seata?rewriteBatchedStatements=true
      user: root
      password: root
      min-conn: 5
      max-conn: 100
      global-table: global_table
      branch-table: branch_table
      lock-table: lock_table
      distributed-lock-table: distributed_lock
      query-limit: 100
      max-wait: 5000
#  server:
#    service-port: 8091 #If not configured, the default is '${server.port} + 1000'
  security:
    secretKey: SeataSecretKey0c382ef121d778043159209298fd40bf3850a017
    tokenValidityInMilliseconds: 1800000
    ignore:
      urls: /,/**/*.css,/**/*.js,/**/*.html,/**/*.map,/**/*.svg,/**/*.png,/**/*.ico,/console-fe/public/**,/api/v1/auth/login
```



client:

```yaml
server:
  port: 3001
spring:
  application:
    name: seata-order-service
  cloud:
    nacos:
      discovery:
        server-addr: localhost:8848
        namespace: 1008611
        group: SEATA_GROUP
        register-enabled: true
  datasource:
    driver-class-name: com.mysql.jdbc.Driver
    url: jdbc:mysql://localhost:13306/seata_order
    username: root
    password: root
feign:
  hystrix:
    enabled: false

logging:
  level:
    io:
      seata: info
mybatis:
  mapperLocations: classpath:mapper/*.xml
  type-aliases-package: classpath:com.pika.domain.*

#自定义事务组名称需要与seata-server中的对应
seata:
  application-id: seata-order-service
  registry:
    type: nacos
    preferred-networks: 30.240.*
    nacos:
      application: seata-server
      server-addr: 127.0.0.1:8848
      group: SEATA_GROUP
      namespace: 1008611
      username: nacos
      password: nacos
      cluster: seata-server
  enable-auto-data-source-proxy: false
  config:
    type: nacos # !!!!不要忘了指定配置类型
    nacos:
      data-id: seata.properties
      username: nacos
      password: nacos
      group: SEATA_GROUP
      namespace: 1008611
      server-addr: localhost:8848

  enabled: true
  tx-service-group: my_test_tx_group
```



#### 事务分组



```java
i.s.c.r.netty.NettyClientChannelManager  : can not get cluster name in registry config 'service.vgroupMapping.seata-order-service-fescar-service-group', please make sure registry config correct
```

解决:在nacos的seata.properties中添加:

```properties
service.vgroupMapping.seata-order-service-fescar-service-group=seata-server
```



> 从v1.4.2版本开始，已支持从一个Nacos dataId中获取所有配置信息,你只需要额外添加一个dataId配置项。
>
> 1.4.2版本以下的可以使用官方脚本逐个上传配置(本次的1.5.1使用的是dataId,使用脚本上传会逐个设置每个配置项作为单独dataId,不便管理)
>
> https://github.com/seata/seata/tree/develop/script/config-center



nacos中完整的seata.properties,源文件地址:https://github.com/seata/seata/blob/develop/script/config-center/config.txt

```properties
#补充配置
service.vgroupMapping.seata-order-service-fescar-service-group=seata-server
# suppress inspection "SpringBootApplicationProperties" for whole file
#For details about configuration items, see https://seata.io/zh-cn/docs/user/configurations.html
#Transport configuration, for client and server
transport.type=TCP
transport.server=NIO
transport.heartbeat=true
transport.enableTmClientBatchSendRequest=false
transport.enableRmClientBatchSendRequest=true
transport.enableTcServerBatchSendResponse=false
transport.rpcRmRequestTimeout=30000
transport.rpcTmRequestTimeout=30000
transport.rpcTcRequestTimeout=30000
transport.threadFactory.bossThreadPrefix=NettyBoss
transport.threadFactory.workerThreadPrefix=NettyServerNIOWorker
transport.threadFactory.serverExecutorThreadPrefix=NettyServerBizHandler
transport.threadFactory.shareBossWorker=false
transport.threadFactory.clientSelectorThreadPrefix=NettyClientSelector
transport.threadFactory.clientSelectorThreadSize=1
transport.threadFactory.clientWorkerThreadPrefix=NettyClientWorkerThread
transport.threadFactory.bossThreadSize=1
transport.threadFactory.workerThreadSize=default
transport.shutdown.wait=3
transport.serialization=seata
transport.compressor=none

#Transaction routing rules configuration, only for the client
service.vgroupMapping.default_tx_group=seata-server
#If you use a registry, you can ignore it
service.default.grouplist=127.0.0.1:8091
service.enableDegrade=false
service.disableGlobalTransaction=false

#Transaction rule configuration, only for the client
client.rm.asyncCommitBufferLimit=10000
client.rm.lock.retryInterval=10
client.rm.lock.retryTimes=30
client.rm.lock.retryPolicyBranchRollbackOnConflict=true
client.rm.reportRetryCount=5
client.rm.tableMetaCheckEnable=true
client.rm.tableMetaCheckerInterval=60000
client.rm.sqlParserType=druid
client.rm.reportSuccessEnable=false
client.rm.sagaBranchRegisterEnable=false
client.rm.sagaJsonParser=fastjson
client.rm.tccActionInterceptorOrder=-2147482648
client.tm.commitRetryCount=5
client.tm.rollbackRetryCount=5
client.tm.defaultGlobalTransactionTimeout=60000
client.tm.degradeCheck=false
client.tm.degradeCheckAllowTimes=10
client.tm.degradeCheckPeriod=2000
client.tm.interceptorOrder=-2147482648
client.undo.dataValidation=true
client.undo.logSerialization=jackson
client.undo.onlyCareUpdateColumns=true
server.undo.logSaveDays=7
server.undo.logDeletePeriod=86400000
client.undo.logTable=undo_log
client.undo.compress.enable=true
client.undo.compress.type=zip
client.undo.compress.threshold=64k
#For TCC transaction mode
tcc.fence.logTableName=tcc_fence_log
tcc.fence.cleanPeriod=1h

#Log rule configuration, for client and server
log.exceptionRate=100

#Transaction storage configuration, only for the server. The file, db, and redis configuration values are optional.
store.mode=db
store.lock.mode=db
store.session.mode=db
#Used for password encryption
store.publicKey=

#If `store.mode,store.lock.mode,store.session.mode` are not equal to `file`, you can remove the configuration block.
#store.file.dir=file_store/data
#store.file.maxBranchSessionSize=16384
#store.file.maxGlobalSessionSize=512
#store.file.fileWriteBufferCacheSize=16384
#store.file.flushDiskMode=async
#store.file.sessionReloadReadSize=100

#These configurations are required if the `store mode` is `db`. If `store.mode,store.lock.mode,store.session.mode` are not equal to `db`, you can remove the configuration block.
store.db.datasource=druid
store.db.dbType=mysql
store.db.driverClassName=com.mysql.jdbc.Driver
store.db.url=jdbc:mysql://127.0.0.1:13306/seata?useUnicode=true&rewriteBatchedStatements=true
store.db.user=root
store.db.password=root
store.db.minConn=5
store.db.maxConn=30
store.db.globalTable=global_table
store.db.branchTable=branch_table
store.db.distributedLockTable=distributed_lock
store.db.queryLimit=100
store.db.lockTable=lock_table
store.db.maxWait=5000

#These configurations are required if the `store mode` is `redis`. If `store.mode,store.lock.mode,store.session.mode` are not equal to `redis`, you can remove the configuration block.
#store.redis.mode=single
#store.redis.single.host=127.0.0.1
#store.redis.single.port=6379
#store.redis.sentinel.masterName=
#store.redis.sentinel.sentinelHosts=
#store.redis.maxConn=10
#store.redis.minConn=1
#store.redis.maxTotal=100
#store.redis.database=0
#store.redis.password=
#store.redis.queryLimit=100

#Transaction rule configuration, only for the server
server.recovery.committingRetryPeriod=1000
server.recovery.asynCommittingRetryPeriod=1000
server.recovery.rollbackingRetryPeriod=1000
server.recovery.timeoutRetryPeriod=1000
server.maxCommitRetryTimeout=-1
server.maxRollbackRetryTimeout=-1
server.rollbackRetryTimeoutUnlockEnable=false
server.distributedLockExpireTime=10000
server.xaerNotaRetryTimeout=60000
server.session.branchAsyncQueueSize=5000
server.session.enableBranchAsyncRemove=false
server.enableParallelRequestHandle=false

#Metrics configuration, only for the server
metrics.enabled=false
metrics.registryType=compact
metrics.exporterList=prometheus
metrics.exporterPrometheusPort=9898
```

![image-20221102202423929](imgs/image-20221102202423929.png)

重新启动后:控制台已无Error

![image-20221102203608884](imgs/image-20221102203608884.png)



![image-20221102203511334](imgs/image-20221102203511334.png)



```java
i.s.c.r.netty.NettyClientChannelManager  : will connect to 192.168.231.130:8091
i.s.core.rpc.netty.NettyPoolableFactory  : NettyPool create channel to transactionRole:TMROLE,address:192.168.231.130:8091,msg:< RegisterTMRequest{applicationId='seata-order-service', transactionServiceGroup='seata-order-service-fescar-service-group'} >
```



#### nacos 拉取配置

```java
no available service found in cluster ‘seata-server’, please make sure registry config correct and keep your seata server running
```

seata-server服务注册在nacos的集群名称是否为seata-server(互相对应)

将seata.config.type配置为nacos后依然出错

```java
Failed to get available servers: {}
```

检查client的seata.config.type有没有配置为nacos(默认为file),以及seata.config.nacos.*的各项配置,

或者进入io.seata.core.rpc.netty.NettyClientChannelManager类中调试availInetSocketAddressList是否为null

```java
    private List<String> getAvailServerList(String transactionServiceGroup) throws Exception {
        List<InetSocketAddress> availInetSocketAddressList = RegistryFactory.getInstance()
                .lookup(transactionServiceGroup);
        if (CollectionUtils.isEmpty(availInetSocketAddressList)) {
            return Collections.emptyList();
        }

        return availInetSocketAddressList.stream()
                .map(NetUtil::toStringAddress)
                .collect(Collectors.toList());
    }
```

调试io.seata.config.nacos.NacosConfiguration查看配置是否生效





