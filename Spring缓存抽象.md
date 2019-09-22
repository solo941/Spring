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

