---
title: 'Spring Boot 配置、Bean 生命周期与 AOP 代理机制'
description: '整理 Spring Boot 3.x 配置方式、Nacos 接入、Spring Bean 生命周期、AOP 代理机制以及 @Transactional 自调用失效的底层原因。'
pubDate: 2026-05-06
tags: ['Spring Boot', 'Spring', 'Java', 'AOP', 'Nacos']
author: 'liuzhne'
draft: false
---

## 前言

Spring Boot 项目看起来是从 `SpringApplication.run()` 启动的，但真正支撑业务运行的是 Spring 容器对 Bean 的创建、依赖注入、初始化、代理增强和销毁管理。

本文把三块内容串起来看：

1. Spring Boot 3.x 中配置文件与 Nacos 的推荐使用方式。
2. Spring Bean 从实例化到销毁的完整生命周期。
3. AOP 代理如何生成，以及 `@Transactional` 自调用为什么会失效。

## Spring Boot 3.x 的配置方式

在 Spring Boot 2.4+ 以及 Spring Boot 3.x 中，已经不再推荐继续使用 `bootstrap.yml` 作为默认配置入口。更推荐的做法是统一使用 `application.yml`，再配合新的配置导入机制来加载外部配置。

如果项目使用 Nacos，通常会同时用到两个能力：

- 服务发现：让服务注册到 Nacos，并能够发现其他服务。
- 配置中心：把环境相关配置托管到 Nacos，由应用启动时加载。

对应依赖通常包括：

```xml
<!-- Nacos Discovery -->
<dependency>
    <groupId>com.alibaba.cloud</groupId>
    <artifactId>spring-cloud-starter-alibaba-nacos-discovery</artifactId>
</dependency>

<!-- Nacos Config -->
<dependency>
    <groupId>com.alibaba.cloud</groupId>
    <artifactId>spring-cloud-starter-alibaba-nacos-config</artifactId>
</dependency>
```

这样做的核心思路是：Spring Boot 自身仍然以 `application.yml` 作为主配置入口，而外部配置通过 `spring.config.import` 等机制被导入。配置加载完成之后，Spring 才会继续创建容器中的 Bean。

## Bean 创建的核心调用链

Spring 容器启动后，`ApplicationContext` 的刷新流程最终会走到 Bean 的创建逻辑。可以按照下面这条调用链去理解：

```java
SpringApplication.run()
    ↳ SpringApplication#run
        ↳ refreshContext
            ↳ AbstractApplicationContext#refresh
                ↳ finishBeanFactoryInitialization
                    ↳ DefaultListableBeanFactory#preInstantiateSingletons
                        ↳ AbstractBeanFactory#getBean
                            ↳ AbstractBeanFactory#doGetBean
                                ↳ AbstractAutowireCapableBeanFactory#createBean
                                    ↳ AbstractAutowireCapableBeanFactory#doCreateBean
```

真正创建 Bean 的核心方法是 `doCreateBean`。它并不是简单地 `new` 一个对象，而是包含实例化、属性填充、初始化、后置增强等多个阶段。

## Spring Bean 生命周期

一个典型 Bean 的生命周期可以拆成以下阶段：

1. 实例化
2. 属性填充
3. Aware 接口回调
4. 前置处理
5. 初始化
6. 后置处理
7. 就绪使用
8. 销毁

### 1. 实例化

实例化对应 `createBeanInstance` 到 `instantiateBean` 这一段逻辑。

Spring 会根据 `BeanDefinition` 中的定义，通过反射创建一个原始 Java 对象。此时对象只是一个“空壳”，成员变量还没有被赋值，依赖关系也还没有建立。

如果构造器注入产生循环依赖，通常会在这个阶段直接抛出 `BeanCurrentlyInCreationException`。这是因为构造器依赖要求对象在创建时就必须拿到完整依赖，Spring 没有机会提前暴露半成品对象。

### 2. 属性填充

属性填充对应 `populateBean`。

实例化完成后，Spring 会解析并注入 Bean 依赖的资源，例如处理 `@Autowired`、`@Value` 等注解，把其他 Bean 或配置值注入到字段、Setter 方法或其他注入点中。

这个阶段会调用 `InstantiationAwareBeanPostProcessor` 的相关扩展方法。开发者也可以通过实现 `postProcessProperties` 来处理自定义注解或自定义注入逻辑。

需要注意的是：

- 字段注入和 Setter 注入在单例场景下可以借助三级缓存处理部分循环依赖。
- 构造器注入无法通过三级缓存解决循环依赖。

### 3. Aware 接口回调

Aware 回调发生在 `initializeBean` 内部的 `invokeAwareMethods` 阶段。

如果 Bean 实现了特定 Aware 接口，Spring 会把容器底层资源回调给当前 Bean：

- `BeanNameAware`：获取当前 Bean 在容器中的名称。
- `BeanFactoryAware`：获取 `BeanFactory`。
- `ApplicationContextAware`：获取 `ApplicationContext`。

在业务代码中不建议滥用 Aware 接口，因为这会让 Bean 和 Spring 容器耦合更深。但在通用组件、框架能力、事件发布、动态获取 Bean 等场景中，`ApplicationContextAware` 仍然很常见。

### 4. 前置处理

前置处理对应：

```java
initializeBean -> applyBeanPostProcessorsBeforeInitialization
```

Spring 会遍历所有注册的 `BeanPostProcessor`，执行它们的 `postProcessBeforeInitialization` 方法。

JSR-250 标准中的 `@PostConstruct` 注解方法，也是在这个阶段由 `CommonAnnotationBeanPostProcessor` 处理执行的。

### 5. 初始化

初始化对应：

```java
initializeBean -> invokeInitMethods
```

这一阶段会执行用户定义的初始化逻辑，常见顺序包括：

1. 如果实现了 `InitializingBean`，调用 `afterPropertiesSet()`。
2. 如果配置了 `init-method` 或 `@Bean(initMethod = "...")`，再通过反射调用自定义初始化方法。

这个阶段适合做资源预加载、参数校验、缓存初始化等准备工作。例如加载本地缓存、检查必要属性是否为空、启动后台线程等。

### 6. 后置处理：AOP 代理生成的关键阶段

后置处理对应：

```java
initializeBean -> applyBeanPostProcessorsAfterInitialization
```

Spring 会再次遍历所有 `BeanPostProcessor`，执行 `postProcessAfterInitialization`。

AOP 代理对象通常就是在这个阶段生成的。如果当前 Bean 需要被增强，例如方法上存在 `@Transactional`、`@Async` 等注解，`AbstractAutoProxyCreator` 会拦截该 Bean，并根据规则创建代理对象。

最终放入 Spring 单例池的，可能不是原始目标对象，而是代理对象。

### 7. 就绪使用

经过前面的所有阶段后，Bean 或 Bean 的代理对象会进入一级缓存 `singletonObjects`。此后，应用中通过依赖注入拿到的就是这个可用对象。

### 8. 销毁

当 Spring 容器关闭时，会触发销毁流程释放资源。常见执行顺序包括：

1. `@PreDestroy` 标注的方法。
2. 实现 `DisposableBean` 接口的 `destroy()` 方法。
3. 自定义 `destroy-method`。

销毁阶段适合释放连接、关闭线程池、刷盘缓存等。

## AOP 代理机制

Spring AOP 主要通过动态代理实现横切逻辑织入。常见代理方式有两类：JDK 动态代理和 CGLIB 代理。

| 特性 | JDK 动态代理 | CGLIB 代理 |
| :--- | :--- | :--- |
| 前提条件 | 目标类需要实现接口 | 目标类无需实现接口 |
| 实现方式 | 生成接口实现类 | 生成目标类子类 |
| 底层机制 | Java 反射、`InvocationHandler` | 字节码增强、`MethodInterceptor` |
| 限制 | 只能代理接口方法 | 不能代理 `final` 类或 `final` 方法 |

JDK 动态代理会生成一个实现相同接口的代理类，方法调用会进入 `InvocationHandler#invoke`，再由它织入增强逻辑并调用目标对象。

CGLIB 则通过继承目标类生成子类，并重写非 `final` 方法来拦截调用。由于它基于继承实现，所以 `final` 类、`final` 方法无法被代理。

在较新的 Spring Boot 版本中，默认更倾向于使用 CGLIB 代理，这样即使业务类没有显式接口，也可以被代理增强。

## @Transactional 自调用为什么会失效

理解 Bean 生命周期和 AOP 代理之后，就能解释一个非常常见的问题：为什么同一个类内部调用 `@Transactional` 方法时，事务不生效？

示例代码：

```java
@Service
public class UserService {

    public void methodA() {
        this.methodB();
    }

    @Transactional
    public void methodB() {
        // 业务逻辑
    }
}
```

外部调用 `userService.methodA()` 时，调用入口通常是 Spring 容器中的代理对象。但当执行进入目标对象的 `methodA` 后，`this.methodB()` 里的 `this` 指向的是目标对象本身，而不是代理对象。

也就是说，这次调用绕过了代理层，事务切面自然没有机会执行。

可以把调用过程拆成四步：

1. 外部调用的是容器中的 `UserService` 代理对象。
2. `methodA` 没有事务增强，代理对象把调用转发给目标对象。
3. 目标对象内部通过 `this.methodB()` 发起调用。
4. 这次调用没有经过代理对象，所以 `@Transactional` 失效。

解决思路也很明确：让调用重新经过代理对象。

推荐方案是拆分 Service，把 `methodB` 放到另一个 Spring Bean 中，通过依赖注入调用。这样调用链会经过容器中的代理对象。

也可以开启 `exposeProxy = true`，再通过 `AopContext.currentProxy()` 获取当前代理对象：

```java
((UserService) AopContext.currentProxy()).methodB();
```

不过这种方式会让业务代码显式感知 AOP 代理，耦合更重，通常不作为首选。

## 小结

Spring Boot 的配置加载、Bean 生命周期、AOP 代理并不是割裂的知识点。

配置先完成加载，容器再根据配置和 BeanDefinition 创建 Bean；Bean 在生命周期后置处理阶段可能被替换成代理对象；事务、异步等能力依赖代理对象完成增强。因此，很多看似“注解不生效”的问题，本质上都可以回到生命周期和代理调用链中定位。

掌握这条主线之后，再看循环依赖、事务失效、初始化顺序、BeanPostProcessor 扩展点等问题，会清晰很多。
