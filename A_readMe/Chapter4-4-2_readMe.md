# chapter4-4-2：使用Redis做集中式缓存


上一篇介绍了在Spring Boot中如何引入缓存、缓存注解的使用、以及EhCache的整合。

虽然EhCache已经能够适用很多应用场景，但是由于EhCache是进程内的缓存框架，在集群模式下时，
各应用服务器之间的缓存都是独立的，因此在不同服务器的进程间会存在缓存不一致的情况。
即使EhCache提供了集群环境下的缓存同步策略，但是同步依然需要一定的时间，短暂的缓存不一致依然存在。

在一些要求高一致性（任何数据变化都能及时的被查询到）的系统和应用中，就不能再使用EhCache来解决了，
这个时候使用集中式缓存是个不错的选择，因此本文将介绍如何在Spring Boot的缓存支持中使用Redis进行数据缓存。

下面以上一篇的例子作为基础进行改造，将缓存内容迁移到redis中。

### 准备工作

可以下载案例Chapter4-4-1，进行下面改造步骤。

先来回顾一下在此案例中，我们做了什么内容：

- 引入了spring-data-jpa和EhCache
- 定义了User实体，包含id、name、age字段
- 使用spring-data-jpa实现了对User对象的数据访问接口UserRepository
- 使用Cache相关注解配置了缓存
- 单元测试，通过连续的查询和更新数据后的查询来验证缓存是否生效

### 开始改造

- 删除EhCache的配置文件src/main/resources/ehcache.xml

- pom.xml中删除EhCache的依赖，增加redis的依赖：
```
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-redis</artifactId>
</dependency>
```

- application.properties中增加redis配置，以本地运行为例，比如：

```
spring.redis.host=localhost
spring.redis.port=6379
spring.redis.pool.max-idle=8
spring.redis.pool.min-idle=0
spring.redis.pool.max-active=8
spring.redis.pool.max-wait=-1
```

我们需要做的配置到这里就已经完成了，Spring Boot会在侦测到存在Redis的依赖并且Redis的配置是可用的情况下，
使用RedisCacheManager初始化CacheManager。

为此，我们可以单步运行我们的单元测试，可以观察到此时CacheManager的实例是org.springframework.data.redis.cache.RedisCacheManager，并获得下面的执行结果：
```
Hibernate: insert into user (age, name) values (?, ?)
Hibernate: select user0_.id as id1_0_, user0_.age as age2_0_, user0_.name as name3_0_ from user user0_ where user0_.name=?
第一次查询：10
第二次查询：10
Hibernate: select user0_.id as id1_0_0_, user0_.age as age2_0_0_, user0_.name as name3_0_0_ from user user0_ where user0_.id=?
Hibernate: update user set age=?, name=? where id=?
第三次查询：10
```
可以观察到，在第一次查询的时候，执行了select语句；第二次查询没有执行select语句，说明是从缓存中获得了结果
而第三次查询，我们获得了一个错误的结果，根据我们的测试逻辑，在查询之前我们已经将age更新为20，但是我们从缓存中获取到的age还是为10。

### 问题思考

为什么同样的逻辑在EhCache中没有问题，但是到Redis中会出现这个问题呢？

在EhCache缓存时没有问题，主要是由于EhCache是进程内的缓存框架，第一次通过select查询出的结果被加入到EhCache缓存中，
第二次查询从EhCache取出的对象与第一次查询对象实际上是同一个对象（可以在使用Chapter4-4-1工程中，观察u1==u2来看看是否是同一个对象），
因此我们在更新age的时候，实际已经更新了EhCache中的缓存对象。

而Redis的缓存独立存在于我们的Spring应用之外，我们对数据库中数据做了更新操作之后，没有通知Redis去更新相应的内容，
因此我们取到了缓存中未修改的数据，导致了数据库与缓存中数据的不一致。

因此我们在使用缓存的时候，要注意缓存的生命周期，利用好上一篇上提到的几个注解来做好缓存的更新、删除

## 进一步修改

针对上面的问题，我们只需要在更新age的时候，通过@CachePut来让数据更新操作同步到缓存中，就像下面这样：

```
@CacheConfig(cacheNames = "users")
public interface UserRepository extends JpaRepository<User, Long> {

    @Cacheable(key = "#p0")
    User findByName(String name);

    @CachePut(key = "#p0.name")
    User save(User user);

}
```

在redis-cli中flushdb，清空一下之前的缓存内容，再执行单元测试，可以获得下面的结果：

```
Hibernate: insert into user (age, name) values (?, ?)
第一次查询：10
第二次查询：10
Hibernate: select user0_.id as id1_0_0_, user0_.age as age2_0_0_, user0_.name as name3_0_0_ from user user0_ where user0_.id=?
Hibernate: update user set age=?, name=? where id=?
第三次查询：20
```
可以看到，我们的第三次查询获得了正确的结果！同时，我们的第一次查询也不是通过select查询获得的，
因为在初始化数据的时候，调用save方法时，就已经将这条数据加入了redis缓存中，因此后续的查询就直接从redis中获取了。

本文内容到此为止，主要介绍了为什么要使用Redis做缓存，以及如何在Spring Boot中使用Redis做缓存，并且通过一个小问题来帮助大家理解缓存机制，
在使用过程中，一定要注意缓存生命周期的控制，防止数据不一致的情况出现。

完整示例：Chapter-4-4-2