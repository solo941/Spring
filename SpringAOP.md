# Sping AOP

## 定义

### Aop是什么

与OOP对比，面向切面，传统的OOP开发中的代码逻辑是自上而下的，而这些过程会产生一些横切性问题，这些横切性的问题和我们的住业务逻辑关系不大，这些横切性问题不会影响到主逻辑实现的，但是会散落到代码的各个部分，难以维护。AOP是处理一些横切性问题，AOP的编程思想就是把这些问题和主业务逻辑分开，达到与主业务逻辑解耦的目的。使代码的重用性和开发效率更高。

### aop的应用场景

1.日志记录

2.权限验证

3.效率检查

4.事务管理

5.exception

### aop概念

**aspect**:一定要给**spring**去管理  抽象  **aspectj-**>类  

**pointcut**:切点表示连接点的集合  **-------------------**>           表

PointCut告诉通知连接点在哪里，切点表达式决定 **JoinPoint** 的数量

**Joinpoint**:连接点   目标对象中的方法 **----------------**>    记录

JoinPoint是要关注和增强的方法，也就是我们要作用的点

**Weaving** :把代理逻辑加入到目标对象上的过程叫做织入

**target** 目标对象 原始对象

**aop** **Proxy** 代理对象  包含了原始对象的代码和增加后的代码的那个对象

**advice**:通知    (位置 + **logic**)，advice共有5种类型：

1.前置通知（Before）：在目标方法被调用之前调用通知功能。

2.后置通知（After）：在目标方法完成之后调用通知，此时不会关心方法的输出是什么。

3.返回通知（After-returning）：在目标方法成功执行之后调用通知。

4.异常通知（After-throwing）：在目标方法抛出异常后调用通知。

5.环绕通知（Around）：通知包裹了被通知的方法，在被通知的方法调用之前和调用之后执行自定义的行为。

## **springAop支持AspectJ**

SpringAop借助aspectj语法，使用javaconfig实现对方法的增强

```java
@Configuration
@ComponentScan("com")
//一定要有
@EnableAspectJAutoProxy
public class Appconfig {
}
```

```java
//bean交给spring管理，两个注释一定要有
@Component
@Aspect
public class UserAspectJ {
    //切入点表达式由@Pointcut注释表示。切入点声明由两部分组成:
    // 一个签名包含名称和任何参数，以及一个切入点表达式，该表达式确定我们对哪个方法执行感兴趣。
    @Pointcut("execution(* com.Aspetj.dao.*.*(..))")
    public void pointCut(){
        System.out.println("point cut");
    }
    //申明before通知,在pointCut切入点前执行
    @Before("com.Aspetj.config.UserAspectJ.pointCut()")
    public void before(){
        System.out.println("before");
    }
}
```

## JointPoint意义

### execution

execution(modifiers-pattern? ret-type-pattern declaring-type-pattern?name-pattern(param-pattern) throws-pattern?)

这里问号表示当前项可以有也可以没有，其中各项的语义如下

modifiers-pattern：方法的可见性，如**public**，**protected**；

ret-type-pattern：方法的返回值类型，如int，void等；

declaring-type-pattern：方法所在类的全路径名，如com.spring.Aspect；

name-pattern：方法名类型，如buisinessService()；

param-pattern：方法的参数类型，如java.lang.String；

throws-pattern：方法抛出的异常类型，如java.lang.Exception；

### within

**问：execution和within的区别**

within与execution相比，粒度更大，仅能实现到包和接口、类级别。而execution可以精确到方法的返回值，参数个数、修饰符、参数类型等

### args

args表达式的作用是匹配指定参数类型和指定参数数量的方法,与包名和类名无关。

args匹配的是运行时传递给方法的参数类型， execution(* *(java.io.Serializable))匹配的是方法在声明时指定的方法参数类型。

应用：两个方法，有参的不想增强，无参的增强

```java
@Repository
public class IndexDao {
    public void query(){
        System.out.println("query");
    }
    public void query(String str){
        System.out.println(str);
    }
}
-----------------------------------------
@Component
@Aspect
public class UserAspectJ {
    //切入点表达式由@Pointcut注释表示。切入点声明由两部分组成:
    // 一个签名包含名称和任何参数，以及一个切入点表达式，该表达式确定我们对哪个方法执行感兴趣。
    @Pointcut("execution(* com.Aspetj.dao.*.*(..))")
    public void pointCut(){
        System.out.println("point cut");
    }

    @Pointcut("args(java.lang.String)")
    public void pointCutArgs(){

    }

    //申明before通知,在pointCut切入点前执行
    @Before("com.Aspetj.config.UserAspectJ.pointCut() && !pointCutArgs()")
    public void before(){
        System.out.println("before");
    }
}
```

### 自定义注解

首先自定义注解

```java
@Retention(RetentionPolicy.RUNTIME)
public @interface Print {
}
```

在Dao中使用注解，作用在增强的方法上，切点声明

```java
 @Pointcut("@annotation(com.Aspetj.anno.Print)")
    public void pointCutAnnotation(){

    }

    //申明before通知,在pointCut切入点前执行
    @Before("com.Aspetj.config.UserAspectJ.pointCutAnnotation()")
    public void before(){
        System.out.println("before");
    }
```

### this

匹配代理对象的类型

### target

匹配目标对象

## 实现

动态代理：Spring Aop 共有两种方法JDK动态代理和CGLIB 动态代理

CGLIB基于继承，代理对象继承目标对象；JDK基于动态代理实现，代理对象和目标对象实现同一接口。因此如果采用JDK动态代理，因此getBean需要获取接口的类文件。

问题：JDK动态代理为什么采用聚合接口而不是继承实现？

因为java单继承，代理对象已经继承了Proxy对象，只能实现目标对象的接口。因此，在使用JDK动态代理时，annotationConfigApplicationContext拿到bean是proxy不等于目标对象，但是代理对象和目标对象实现了相同的接口。

```
@EnableAspectJAutoProxy(proxyTargetClass = true)  //使用CGlib
否则使用JDK动态代理。
```

静态代理：Aspectj Aop

完成代理后，spring容器中存在的是代理对象。

## 通过AOP实现spring事务

```java
EnableTransactionManagement
boolean proxyTargetClass() default false;
->---------------------------------------
ProxyTransactionManagementConfiguration
public TransactionInterceptor transactionInterceptor()
//拦截器的拦截方法，实现around类型的advice
public Object invoke(MethodInvocation invocation) throws Throwable {
		Class<?> targetClass = (invocation.getThis() != null ? AopUtils.getTargetClass(invocation.getThis()) : null);
		return invokeWithinTransaction(invocation.getMethod(), targetClass, invocation::proceed);
	}
-》----------------------------------------------------------------
    //创建事务->后置调用->提交事务或者回滚
TransactionInfo txInfo = createTransactionIfNecessary(tm, txAttr, joinpointIdentification);

			Object retVal;
			try {
				// This is an around advice: Invoke the next interceptor in the chain.
				// This will normally result in a target object being invoked.
				retVal = invocation.proceedWithInvocation();
			}
			catch (Throwable ex) {
				// target invocation exception
				completeTransactionAfterThrowing(txInfo, ex);
				throw ex;
			}
			finally {
				cleanupTransactionInfo(txInfo);
			}
			commitTransactionAfterReturning(txInfo);
```



## 常用注解

- @EnableAspectJAutoProxy
- @Aspect
- @Pointcut
- @Before
- @After/@AfterReturning/@AfterThrowing
- Around
- Order