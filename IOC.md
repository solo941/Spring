## 定义

控制反转是一种设计原则，降低代码直接的耦合度，最常见的实现方式：依赖注入(DI)，依赖查找(lookup)以及依赖拖拽。

面向抽象编程过程中会产生类的依赖，spring提供了管理对象的容器，类的产生过程交给spring。

## Spring实现IOC的方法

1.应用程序提供类和依赖关系（属性，构造方法）

2.通过xml,annotation,javaconfig，将需要交给容器管理的对象通过配置信息交给容器；

3.将类之间的依赖关系通过配置交给容器

## 注入方式

### xml

1. 构造方法：

   service->dao.IndexService(IndexDao dao)

   \<bean id="service" class="com.springtest.IndexService">

   ​	\<constructor-arg ref="dao">\</constructor-arg >

   \</bean>

2. set方法：

service->dao,setdao (IndexDao dao)

xml:

\<bean id="service" class="com.springtest.IndexService">

​	\<property name="dao" ref="dao">\</property>

\</bean>

### 注解

1.\<context:component-scan base-package="com">\</context:component-scan>

这一步既开启扫描又开启了注解

2.xmlns:context="http://www.springframework.org/schema/context"
http://www.springframework.org/schema/context/spring-context.xs">

这两种方法都使用ClassPathXmlApplicationContext

### JAVA 配置类

1.Spring.java

@Configuration

@ComponentScan("com.springtest")

2.AnnotationConfigApplicationContext annotationConfigApplicationContext = new AnnotationConfigApplicationContext(Spring.class)

如果dao的@Component没有使用，以bean形式写在xml中，需要使用@ImportResource

```java
@Configuration
@ComponentScan("com.springtest")
@ImportResource("classpath:spring.xml")
public class Spring {
}
```

## 自动装配

### 定义

IOC注入需要在类的定义中和在spring的配置中需要描述。使用自动装配，我们只需要在类的定义中提供依赖，对象交给容器管理，完成注入。

优点：

1.减少配置

2.自动更新配置

### 实现

#### xml中配置

可以全局指定

```
xml的beans中：
default-autowire="byType"
```

spring去类中找依赖，在容器中找与该依赖相同的类型赋给它。

问题：

同一个类型依赖多个匹配，解决方法：byName

byName看的是类中的依赖setxxx,其中xxx的首字母小写

也可以单独制定

\<bean>中 配置autowire

#### 注解

@Autowired默认使用的是bytype,如果bytype没有找到，根据byname

@Resource默认使用的是byname,根据属性名字，与方法名字无关

使用注解，可以不需要set方法

## 作用范围

@Scope

singleton:单例

prototype：原型

问题:单例A依赖原型B，B也会单例，只实例化一次，不能再次初始化B。

解决方法：applicationgContext.getBean()

使用解决方法前：

```java
@Autowired
    IndexDao dao;

    public void service(){
        System.out.println("service" + this.hashCode());
        System.out.println("dao" + dao.hashCode());
    }
```

```
service33320447
dao33320388
service33320447
dao33320388
service33320447
dao33320388
```

使用后

```java
public class IndexService implements ApplicationContextAware {
    private ApplicationContext applicationContext;
    IndexDao dao;

    public void service(){
        dao = (IndexDao) applicationContext.getBean("dao");
        System.out.println("service" + this.hashCode());
        System.out.println("dao" + dao.hashCode());
    }
}
```

```
service1482643464
dao32679524
service1455945662
dao5981722
service1462661556
dao12697616
```

但是这种方法与spring耦合度太高，侵入性太强

第二种解决方法：lookup

```java
@Service
public abstract class IndexService{
    @Lookup
    abstract IndexDao getDao();

    public void service(){
        System.out.println("service" + this.hashCode());
        System.out.println("dao" + getDao().hashCode());
    }
}
```

spring会通过getxxxx去找对应的xxx

# IOC应用

## 生命周期回调

类初始化，类销毁时的回调，三种方法

1.Initialization callbacks + destruction callbacks

```java
public class IndexDaoImpl  implements IndexDao, InitializingBean, DisposableBean {
    public IndexDaoImpl(){
        System.out.println("Construct");
    }

    public void afterPropertiesSet() throws Exception {
        System.out.println("init");
    }

    public void destroy() throws Exception {
        System.out.println("after");
    }
}
```

2. default Initialization and destroy methods，自定义可指定方法

bean中配置 init-method="init"

类中public void init(){

}

3.@PostConstruct，@Predestroy

```
@PostConstruct
    public void init() {
        System.out.println("init");
    }
```

第三种非侵入，最好

## 多个数据源

1.@Qualifier("bean的名字")  按名称装配Bean,与@Autowired组合使用,解决按类型匹配找到多个Bean问题。

2.@Primary告诉spring 优先选择哪一个POJO。

