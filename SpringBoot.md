# SpringBoot笔记

## 配置SLL证书

1. 申请证书并下载

   ![image-20221012095110277](imgs/image-20221012095110277.png)

2. 将.pfx文件复制到resources下

   ![image-20221012095126881](imgs/image-20221012095126881.png)

3. 配置application.yml

```yaml
spring:
  application:
    name: pika
server:
  port: 443
  ssl:
    key-store: classpath:8604835_host.pikachuvirtual.top.pfx
    key-store-password: Qni83X9e
    key-store-type: PKCS12
    ciphers: TLS_RSA_WITH_AES_128_CBC_SHA,TLS_RSA_WITH_AES_256_CBC_SHA,TLS_ECDHE_RSA_WITH_AES_128_CBC_SHA,TLS_ECDHE_RSA_WITH_AES_128_CBC_SHA256,TLS_RSA_WITH_AES_128_CBC_SHA256,TLS_RSA_WITH_AES_256_CBC_SHA256

```

https 访问有锁:

![image-20221012095220026](imgs/image-20221012095220026.png)

http 访问显示不安全:

![image-20221012095250171](imgs/image-20221012095250171.png)



## 问题

###  'java.lang.String' that could not be found.

> Description:
>
> Parameter 0 of method secKill in com.pika.config.SecKillController required a bean of type 'java.lang.String' that could not be found.

**因为注释的时候没有把@Autowired一同注释掉，有一个空的@Autowired引起报错，导致项目启动报错。**

### Jackson冲突

> Error creating bean with name ‘requestMappingHandlerAdapter’ defined in class path resource [org/[springframework](https://so.csdn.net/so/search?q=springframework&spm=1001.2101.3001.7020)/boot/autoconfigure/web/servlet/WebMvcAutoConfiguration$EnableWebMvcConfiguration.class]: Bean instantiation via factory method failed; nested exception is org.springframework.beans

```java
org.springframework.beans.factory.BeanCreationException: Error creating bean with name 'requestMappingHandlerAdapter' defined in class path resource [org/springframework/boot/autoconfigure/web/servlet/WebMvcAutoConfiguration$EnableWebMvcConfiguration.class]: Bean instantiation via factory method failed; nested exception is org.springframework.beans.BeanInstantiationException: Failed to instantiate [org.springframework.web.servlet.mvc.method.annotation.RequestMappingHandlerAdapter]: Factory method 'requestMappingHandlerAdapter' threw exception; nested exception is org.springframework.beans.BeanInstantiationException: Failed to instantiate [com.fasterxml.jackson.datatype.jsr310.JavaTimeModule]: Unresolvable class definition; nested exception is java.lang.NoClassDefFoundError: com/fasterxml/jackson/databind/ser/std/ToStringSerializerBase


Caused by: org.springframework.beans.BeanInstantiationException: Failed to instantiate [org.springframework.web.servlet.mvc.method.annotation.RequestMappingHandlerAdapter]: Factory method 'requestMappingHandlerAdapter' threw exception; nested exception is org.springframework.beans.BeanInstantiationException: Failed to instantiate [com.fasterxml.jackson.datatype.jsr310.JavaTimeModule]: Unresolvable class definition; nested exception is java.lang.NoClassDefFoundError: com/fasterxml/jackson/databind/ser/std/ToStringSerializerBase


Caused by: org.springframework.beans.BeanInstantiationException: Failed to instantiate [com.fasterxml.jackson.datatype.jsr310.JavaTimeModule]: Unresolvable class definition; nested exception is java.lang.NoClassDefFoundError: com/fasterxml/jackson/databind/ser/std/ToStringSerializerBase
	at org.springframework.beans.BeanUtils.instantiateClass(BeanUtils.java:157)
Caused by: java.lang.NoClassDefFoundError: com/fasterxml/jackson/databind/ser/std/ToStringSerializerBase


Caused by: org.springframework.beans.factory.BeanCreationException: Error creating bean with name 'requestMappingHandlerAdapter' defined in class path resource [org/springframework/boot/autoconfigure/web/servlet/WebMvcAutoConfiguration$EnableWebMvcConfiguration.class]: Bean instantiation via factory method failed; nested exception is org.springframework.beans.BeanInstantiationException: Failed to instantiate [org.springframework.web.servlet.mvc.method.annotation.RequestMappingHandlerAdapter]: Factory method 'requestMappingHandlerAdapter' threw exception; nested exception is org.springframework.beans.BeanInstantiationException: Failed to instantiate [com.fasterxml.jackson.datatype.jsr310.JavaTimeModule]: Unresolvable class definition; nested exception is java.lang.NoClassDefFoundError: com/fasterxml/jackson/databind/ser/std/ToStringSerializerBase

Caused by: org.springframework.beans.BeanInstantiationException: Failed to instantiate [org.springframework.web.servlet.mvc.method.annotation.RequestMappingHandlerAdapter]: Factory method 'requestMappingHandlerAdapter' threw exception; nested exception is org.springframework.beans.BeanInstantiationException: Failed to instantiate [com.fasterxml.jackson.datatype.jsr310.JavaTimeModule]: Unresolvable class definition; nested exception is java.lang.NoClassDefFoundError: com/fasterxml/jackson/databind/ser/std/ToStringSerializerBase

java.lang.IllegalStateException: Failed to load ApplicationContext

	
Caused by: org.springframework.beans.factory.BeanCreationException: Error creating bean with name 'requestMappingHandlerAdapter' defined in class path resource [org/springframework/boot/autoconfigure/web/servlet/WebMvcAutoConfiguration$EnableWebMvcConfiguration.class]: Bean instantiation via factory method failed; nested exception is org.springframework.beans.BeanInstantiationException: Failed to instantiate [org.springframework.web.servlet.mvc.method.annotation.RequestMappingHandlerAdapter]: Factory method 'requestMappingHandlerAdapter' threw exception; nested exception is org.springframework.beans.BeanInstantiationException: Failed to instantiate [com.fasterxml.jackson.datatype.jsr310.JavaTimeModule]: Unresolvable class definition; nested exception is java.lang.NoClassDefFoundError: com/fasterxml/jackson/databind/ser/std/ToStringSerializerBase
	org.springframework.test.context.cache.DefaultCacheAwareContextLoaderDelegate.loadContext(DefaultCacheAwareContextLoaderDelegate.java:124)
	... 25 more
Caused by: org.springframework.beans.BeanInstantiationException: Failed to instantiate [org.springframework.web.servlet.mvc.method.annotation.RequestMappingHandlerAdapter]: Factory method 'requestMappingHandlerAdapter' threw exception; nested exception is org.springframework.beans.BeanInstantiationException: Failed to instantiate [com.fasterxml.jackson.datatype.jsr310.JavaTimeModule]: Unresolvable class definition; nested exception is java.lang.NoClassDefFoundError: com/fasterxml/jackson/databind/ser/std/ToStringSerializerBase
	
Caused by: org.springframework.beans.BeanInstantiationException: Failed to instantiate [com.fasterxml.jackson.datatype.jsr310.JavaTimeModule]: Unresolvable class definition; nested exception is java.lang.NoClassDefFoundError: com/fasterxml/jackson/databind/ser/std/ToStringSerializerBase
	at org.springframework.beans.BeanUtils.instantiateClass(BeanUtils.java:157)
	
Caused by: java.lang.ClassNotFoundException: com.fasterxml.jackson.databind.ser.std.ToStringSerializerBase

	... 73 more
```

大概有以下几点：

- 有的说是 `jackson jar包`版本低了 把版本修改了就行了
- 还有可能是`jar 包冲突了` 解决冲突也能行
- 还有可能是 兼容问题哈

```xml
<!--json-->
<dependency>
    <groupId>com.alibaba</groupId>
    <artifactId>fastjson</artifactId>
    <version>1.2.33</version>
</dependency>
<dependency>
    <groupId>com.fasterxml.jackson.core</groupId>
    <artifactId>jackson-databind</artifactId>
    <version>2.9.0</version>
</dependency>
1234567891011
```

可以先都试一试哈，没有给大家找完全哈。

#### 解决

最后我找到的解决方式 就是因为这句话 `使Jackson支持JSR310标准`

然后最后导入了下面这个依赖：

```xml
<dependency>
    <groupId>com.fasterxml.jackson.datatype</groupId>
    <artifactId>jackson-datatype-jsr310</artifactId>
    <version>2.9.2</version>
</dependency>
```