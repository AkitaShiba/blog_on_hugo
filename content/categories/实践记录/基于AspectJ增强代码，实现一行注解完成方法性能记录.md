---
title:       "基于AspectJ增强代码"
subtitle:    "实现一行注解完成方法性能记录"
description: ""
date:        2019-04-27
author:      "Jarvis"
image:       ""
tags:        ["AspectJ", "性能"]
categories:  ["实践记录" ]
---

## 背景 

因需要对项目性能测试，并追踪每一条日志进来后的处理状态.以性能记录为例，如果在每个方法前后都加上如下代码 

```
long beginTime = System.currentTimeMillis(); 

long endTime = System.currentTimeMillis(); 

long timeConsume = endTime - beginTime; 

```

显然会让代码变得非常冗杂，而且需要统计性能的地方可能还会有很多，如果可以单纯在方法上加上一个注解，就能实现方法执行耗时的记录，这样便能减少许多简单且繁琐的代码，更加方便的扩展 

## 技术选型 

像这种场景就是典型的AOP场景，搜索SpringAOP就能找到很多的代码样例，但是项目并不依赖于Spring， 也没有必要引入Spring。通常要实现这样的功能，有两种方案，一种是像SpringAOP那样，通过CGLib动态代理或JDK动态代理，在创建对象的时候，动态地生成代理类的对象，另一种是AspectJ，在编译代码的时候就将需要执行的逻辑织入到字节码中。 

对于动态代理，需要创建代理类的对象，才可以增强，而项目中，存在很多静态方法，在使用的时候并不通过对象来调用，而且即便是通过对象来调用的方法，没有Spring方便的IOC机制，也得修改所有代码中new对象的处理，才可以使用增强后的代理对象，略麻烦。而且如果是频繁的创建对象，因为还有一步创建代理对象的操作，性能上会有一定的损失。 

对于AspectJ这种方式，则可以对满足切点表达式的地方，都织入增强后的逻辑，但是需要依赖于织入工具的协助，来对编译后的字节码进行增强。幸好maven上已经有对应的aspectj编译插件，可以很方便的处理织入 

综合考虑之下，决定采用自定义注解（指定目标）+ ApsectJ(Aop增强) + aspectj的Maven编译插件来实现 

## 技术实现 

### 1、自定义注解 

```
@Retention(RetentionPolicy.RUNTIME) 

@Target(ElementType.METHOD) 

public @interface TimeConsumeLogAnnotation { 

} 

```

### 2、引入Aspectj依赖 

```
<dependency> 

​    <groupId>org.aspectj</groupId> 

​    <artifactId>aspectjrt</artifactId> 

​    <version>1.8.9</version> 

</dependency> 

```

### 3、Aspectj切面 

```
@Aspect 

public class TimeConsumeLogAspectJ { 

​    //通过ThreadLocal隔离不同线程的变量 

​    ThreadLocal<Long> timeRecord = new ThreadLocal<>(); 

​    @Pointcut("execution(* *(..)) && @annotation(com.sf.isic.rrcs.engine.annotation.TimeConsumeLogAnnotation)") 

​    public void jointPoint(){} 

​    @Before("jointPoint()") 

​    public void doBefore(JoinPoint joinPoint){ 

​        MethodSignature signature = (MethodSignature) joinPoint.getSignature(); 

​        Method method = signature.getMethod(); 

​        System.out.println("方法" + method.getName() + "开始"); 

​        timeRecord.set(System.currentTimeMillis()); 

​    } 

​    @After("jointPoint()") 

​    public void doAfter(JoinPoint joinPoint){ 

​        long beginTime = timeRecord.get(); 

​        System.out.println("方法" +joinPoint.getSignature().getName()+ "结束,耗时"+(System.currentTimeMillis()-beginTime) +"ms"); 

​    } 

} 

```

### 4、引入maven编译插件 

在maven-compiler-plugin处理完之后再工作 

```
<plugin> 

​    <groupId>org.codehaus.mojo</groupId> 

​    <artifactId>aspectj-maven-plugin</artifactId> 

​    <version>1.10</version> 

​    <configuration> 

​        <source>1.8</source> 

​        <target>1.8</target> 

​        <complianceLevel>1.8</complianceLevel> 

​    </configuration> 

​    <executions> 

​        <execution> 

​            <phase>compile</phase> 

​            <goals> 

​                <goal>compile</goal> 

​            </goals> 

​        </execution> 

​    </executions> 

</plugin> 

```

### 5、在目标方法上加入@TimeConsumeLogAnnotation注解编译运行即可 

```java 
@TimeConsumeLogAnnotation() 

public static void sayHelloWorld(String name) { 

​    System.out.println("Hello " + name); 

} 

```

编译后的字节码 

```java 
@TimeConsumeLogAnnotation 

public static void sayHelloWorld(String name) { 

​    JoinPoint var1 = Factory.makeJP(ajc$tjp_0, (Object)null, (Object)null, name); 

​    try { 

​        TimeConsumeLogAspectJ.aspectOf().doBefore(var1); 

​        System.out.println("Hello " + name); 

​    } catch (Throwable var4) { 

​        TimeConsumeLogAspectJ.aspectOf().doAfter(var1); 

​        throw var4; 

​    } 

​    TimeConsumeLogAspectJ.aspectOf().doAfter(var1); 

} 

```

效果 

```
方法sayHelloWorld开始 

Hello world 

方法sayHelloWorld结束,耗时1ms 

```

### 6、踩过的坑 

#### （1）切面执行两次 

在一开始切面的表达式为 

```java 
@Pointcut("@annotation(com.sf.isic.rrcs.engine.annotation.TimeConsumeLogAnnotation)") 


```

而aspectj的编译器会识别出**方法调用**和**方法执行**两个阶段的切入点，因为会在这两个阶段都执行 

通过将切面表达式修改为 

```java 
@Pointcut("execution(* *(..)) && @annotation(com.sf.isic.rrcs.engine.annotation.TimeConsumeLogAnnotation)") 


```

可以限定成只识别**方法执行**这个阶段 

#### （2）多模块项目aspectj编译失败 

> 如果在多模块项目，在具体的某个子模块声明切面类，定义切点表达式，但是连接点切分散在各个其他模块时，ajc扫描具到切点表达式时，只会在本模块扫描对应的连接点，其他模块的连接点是没有办法编绎期切入切面，ajc是不会在编绎其他模块时再去扫描有没有某个切点表达式与当前连接点匹配的 

通过在每个模块都加上自定义注解和切面，可解决编译的问题 

## 更多的操作 

由于自定义注解支持赋值，Aspectj切面又可以拦截到方法，并且通过反射获取到方法参数，因此可以在这基础做更多定制化的优化 

## 参考链接 

[Spring AOP 实现原理与 CGLIB 应用](https://www.ibm.com/developerworks/cn/java/j-lo-springaopcglib/index.html) 

[关于AspectJ你可能不知道的事](https://juejin.im/entry/5a40abb16fb9a0451e400886) 

[AspectJ切面执行两次原因分析](https://blog.csdn.net/u011116672/article/details/63685340) 

[多模块maven项目使用Eclipse的 AspectJ编绎期织入](http://blog.51cto.com/5914679/2096848) 