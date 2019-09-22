# Spring缓存抽象

## 定义

为java方法增加缓存，缓存执行结果，问题：哪些内容需要缓存，缓存在jvm中，还是redis中？

长久不变的，接受变化的延时：jvm内部，设置过期时间；

需要保证一致性：redis；

数据写比读多：不需要缓存。

## Spring中基于缓存的注解

@EnableCaching（基于AOP）

- Cacheable
- CacheEvict(缓存清理)
- CachePut
- Caching
- CacheConfig（缓存配置）

## Redis在spring中的其他用法

配置连接工厂

LettuceConnectionFactory

- RedisStandaloneConfiguration
- RedisSentinelConfiguration
- RedisClusterConfiguration

### Lettuce支持读写分离

通过配置`customizer`，实现只读主或者只读从；优先度主，优先读从

```java
@Bean
	public LettuceClientConfigurationBuilderCustomizer customizer(){
		return builder -> builder.readFrom(ReadFrom.MASTER_PREFERRED);
	}
```

## Spring与Redis的连接管理

### **RedisTemplate**

```java
@Bean
	public RedisTemplate<String,Coffee> redisTemplate(RedisConnectionFactory redisConnectionFactory){
		RedisTemplate<String, Coffee> template = new RedisTemplate<>();
		template.setConnectionFactory(redisConnectionFactory);
		return template;
	}
```

### **StringRedisTemplate与RedisTemplate区别点**

 

- 两者的关系是StringRedisTemplate继承RedisTemplate。
- 两者的数据是**不共通**的；也就是说StringRedisTemplate只能管理StringRedisTemplate里面的数据，RedisTemplate只能管理RedisTemplate中的数据。
- 其实他们两者之间的区别主要在于他们使用的序列化类:

　　　　RedisTemplate使用的是JdkSerializationRedisSerializer    存入数据会将数据先序列化成字节数组然后在存入Redis数据库。 

　　 　  StringRedisTemplate使用的是StringRedisSerializer

- 使用时注意事项：

　　　当你的redis数据库里面本来存的是字符串数据或者你要存取的数据就是字符串类型数据的时候，那么你就使用StringRedisTemplate即可。

　　　但是如果你的数据是复杂的对象类型，而取出的时候又不想做任何的数据转换，直接从Redis里面取出一个对象，那么使用RedisTemplate是更好的选择。

###  使用

springboot中使用注解@Autowired 即可

```java
 @Autowired
 private RedisTemplate<String,Coffee> redisTemplate;
```

此外还可以按照需要构造

```java
@Bean
	public RedisTemplate<String,Coffee> redisTemplate(RedisConnectionFactory redisConnectionFactory){
		RedisTemplate<String, Coffee> template = new RedisTemplate<>();
		template.setConnectionFactory(redisConnectionFactory);
		return template;
	}
```

## Redis Repository

问题：如何区分Repository

- 实体注解
- 接口类型
- 扫描不同的包

### 使用

1.Application加入注解@EnableRedisRepositories

2.创建缓存Entity,使用实体注解

```java
@RedisHash(value = "springbucks-coffee", timeToLive = 60)
@Data
@NoArgsConstructor
@AllArgsConstructor
@Builder
public class CoffeeCache {
    @Id
    private Long id;
    //使用二级索引
    @Indexed
    private String name;
    private Money price;
}
```

3.创建Redis Repository

```java
public interface CoffeeCacheRepository extends CrudRepository<CoffeeCache, Long> {
    Optional<CoffeeCache> findOneByName(String name);
}
```

4.处理类型转换

```java
@ReadingConverter
public class BytesToMoneyConverter implements Converter<byte[], Money> {
    @Override
    public Money convert(byte[] source) {
        String value = new String(source, StandardCharsets.UTF_8);
        return Money.ofMinor(CurrencyUnit.of("CNY"), Long.parseLong(value));
    }
}
@WritingConverter
public class MoneyToBytesConverter implements Converter<Money, byte[]> {
    @Override
    public byte[] convert(Money source) {
        String value = Long.toString(source.getAmountMinorLong());
        return value.getBytes(StandardCharsets.UTF_8);
    }
}
```

converter如何加载到Repository当中呢？

RedisRepositoryConfigurationExtension将Repository转换为bean的过程中，定义custormconvertions

```java
 RootBeanDefinition customConversions = new RootBeanDefinition(RedisCustomConversions.class);
```

因此，我们可以在application中自己定义custormconvertions

```java
@Bean
	public RedisCustomConversions redisCustomConversions() {
		return new RedisCustomConversions(
				Arrays.asList(new MoneyToBytesConverter(), new BytesToMoneyConverter()));
	}
```

### 查找缓存

```
127.0.0.1:6379> keys *
1) "springbucks-coffee:4"
2) "springbucks-coffee:4:idx"
3) "springbucks-coffee:name:mocha"
4) "springbucks-coffee:4:phantom"
5) "springbucks-coffee"
```

保存时会根据name创建二级索引

