# Spring Aop实现

### 一、AOP的基本概念

AOP 中的概念和术语并不是 Spring 特有的，而是 AOP 联盟共同定义的。

Aspect: “切面”，表示横跨多个类的模块化关注点。
事务管理是企业级 Java 应用程序中横切关注点的一个很好的例子。
在 Spring AOP 中，Aspect 是通过 schema-based 或 @Aspect 注解来实现的。

Join point: “连接点”，表示程序执行过程中的一个点，如方法的执行或异常的处理。
在 Spring AOP 中，Join point 始终表示方法执行，对应到目标对象中的具体的方法。

Advice: “通知”，表示 Aspect 在特定的 Join point 采取的操作。包括 “around”, “before” and “after 等

Pointcut: “切点”，它是匹配连接点的谓词。可以说"Pointcut"表示的是"Join point"的集合。
通过切点表达式来匹配 Join point，Spring 默认使用 AspectJ 切入点表达式语言。
切点表达式匹配连接点的概念是 AOP 的核心。
Advice 与 Pointcut 关联之后，将在与切点表达式(Pointcut expression)匹配的任意连接点上运行。

Introduction: 代替一个类型声明附加的方法或字段。
Spring AOP 允许您向任何 Advice 的对象引入新接口（以及相应的实现）。例如，您可以使用 Introduction 使 bean 实现 IsModified 接口。（Introduction 在 AspectJ 社区中称为类型间声明。inter-type declaration）

Target object: 被代理的原始对象

Weaving: “织入”。把代理逻辑加入到目标对象上的过程叫织入。
Weaving 可以在编译时（例如使用AspectJ编译器）、加载时或运行时完成。Spring AOP 和其他纯 Java AOP 框架一样，在运行时执行织入。

### 二、aop实现

实现 Spring AOP 大体会分如下几步：

1. 找到与 bean 匹配的所有的 Advisor
2. 使用所有匹配的 Advisor 来为 bean 生成生成动态代理
3. 通过动态代理类执行 Advice
4. 将 Spring AOP 与 Spring IoC 进行结合

#### step-1 找到与 bean 匹配的所有的 Advisor

bean 中的每一个方法都是一个 join point，那么 join point 是如何与 Advisor 进行关联的呢？

答：Spring 中通过 Pointcut 来将 bean 中的 join point 与 Advisor 进行匹配。

Pointcut 匹配类有两个条件：类过滤器 + 方法匹配器。通过这两个条件来筛选匹配的类

```java
public interface Pointcut {
    ClassFilter getClassFilter();
    MethodMatcher getMethodMatcher();
}
```

这里我们主要研究一下 aspectj 表达式的匹配问题，所以，重点就是 `AspectJExpressionPointcut` 这个类。AspectJExpressionPointcut: 对 aspectj 表达式语法的支持。它是 Spring 中最常用的切入点匹配表达式。
它是通过 `AspectJExpressionPointcut#matches(Method, Class<?>, boolean hasIntroductions)` 方法来完成匹配的：

```java
public boolean matches(Method method, Class<?> targetClass, boolean hasIntroductions) {
    obtainPointcutExpression();
    ShadowMatch shadowMatch = getTargetShadowMatch(method, targetClass);

    // Special handling for this, target, @this, @target, @annotation
    // in Spring - we can optimize since we know we have exactly this class,
    // and there will never be matching subclass at runtime.
    if (shadowMatch.alwaysMatches()) {
        return true;
    } else if (shadowMatch.neverMatches()) {
        return false;
    } else {
        // the maybe case
        if (hasIntroductions) {
            return true;
        }
        // A match test returned maybe - if there are any subtype sensitive variables
        // involved in the test (this, target, at_this, at_target, at_annotation) then
        // we say this is not a match as in Spring there will never be a different
        // runtime subtype.
        RuntimeTestWalker walker = getRuntimeTestWalker(shadowMatch);
        return (!walker.testsSubtypeSensitiveVars() || walker.testTargetInstanceOfResidue(targetClass));
    }
}
```

#### step-2 使用所有匹配的 Advisor 来为 bean 生成生成动态代理

为了研究 Spring 是如何通过匹配的 Advisor 来给 bean 生成代理类的，我们需要先找到 proxy bean 是什么时候创建的。

> bean 的创建三步曲:
> 
> 1. createBeanInstance : 创建 bean 的实例
> 2. populateBean : 填充 bean 的依赖
> 3. initializeBean : 初始化 bean

通过阅读源码可以发现，proxy bean 是在执行 `AbstractAutoProxyCreator#postProcessAfterInitialization()` 时产生的

```java
public Object postProcessAfterInitialization(@Nullable Object bean, String beanName) {
    if (bean != null) {
        Object cacheKey = getCacheKey(bean.getClass(), beanName);
        if (this.earlyProxyReferences.remove(cacheKey) != bean) {
            return wrapIfNecessary(bean, beanName, cacheKey);
        }
    }
    return bean;
}

// AbstractAutoProxyCreator#wrapIfNecessary()
protected Object wrapIfNecessary(Object bean, String beanName, Object cacheKey) {
    ......

    // Create proxy if we have advice.
    Object[] specificInterceptors = getAdvicesAndAdvisorsForBean(bean.getClass(), beanName, null);
    if (specificInterceptors != DO_NOT_PROXY) {
        this.advisedBeans.put(cacheKey, Boolean.TRUE);
        // 创建代理 bean  
        Object proxy = createProxy(
                bean.getClass(), beanName, specificInterceptors, new SingletonTargetSource(bean));
        this.proxyTypes.put(cacheKey, proxy.getClass());
        return proxy;
    }

    this.advisedBeans.put(cacheKey, Boolean.FALSE);
    return bean;
}
```

#### step-3 通过动态代理类执行 Advice

Spring 的动态代理类分为两种： jdk proxy 和 cglib proxy。

ProxyFactory 是 Spring AOP 用于生成 AOP 代理类的核心类。
AopProxyFactory 是创建 AOP 代理类的工厂接口，它只有一个实现类 DefaultAopProxyFactory，它会根据规则来具体选用 JDK proxy 或者 CGLIB proxy。

##### JDK Proxy

JDK proxy 是通过 `java.lang.reflect.InvocationHandler#invoke()` 来进行方法拦截的。

```java
    // Get as late as possible to minimize the time we "own" the target,
            // in case it comes from a pool.
            target = targetSource.getTarget();
            Class<?> targetClass = (target != null ? target.getClass() : null);

            // Get the interception chain for this method.
            List<Object> chain = this.advised.getInterceptorsAndDynamicInterceptionAdvice(method, targetClass);

            // Check whether we have any advice. If we don't, we can fall back on direct
            // reflective invocation of the target, and avoid creating a MethodInvocation.
            if (chain.isEmpty()) {
                // We can skip creating a MethodInvocation: just invoke the target directly
                // Note that the final invoker must be an InvokerInterceptor so we know it does
                // nothing but a reflective operation on the target, and no hot swapping or fancy proxying.
                Object[] argsToUse = AopProxyUtils.adaptArgumentsIfNecessary(method, args);
                retVal = AopUtils.invokeJoinpointUsingReflection(target, method, argsToUse);
            }
            else {
                // We need to create a method invocation...
                MethodInvocation invocation =
                        new ReflectiveMethodInvocation(proxy, target, method, args, targetClass, chain);
                // Proceed to the joinpoint through the interceptor chain.
                retVal = invocation.proceed();
            }

            // Massage return value if necessary.
            Class<?> returnType = method.getReturnType();
            if (retVal != null && retVal == target &&
                    returnType != Object.class && returnType.isInstance(proxy) &&
                    !RawTargetAccess.class.isAssignableFrom(method.getDeclaringClass())) {
                // Special case: it returned "this" and the return type of the method
                // is type-compatible. Note that we can't help if the target sets
                // a reference to itself in another returned object.
                retVal = proxy;
            }
```

> 1. 获取与 method 匹配的 Advice 链(AdvisedSupport#getInterceptorsAndDynamicInterceptionAdvice)
> 
> 2. 使用 ReflectiveMethodInvocation 通过反射来执行 Advice 链 和 joinpoint（目标方法）

##### Cglib Proxy

cglib 是通过 `Enhancer#create()` 来生成代理类的，真正起拦截作用的是 Enhancer 中设置的 Callback 类。

Spring AOP 是如何创建 Enhancer 并设置 Callback 的，对应的源码是 `CglibAopProxy#getProxy()` :

```java
        // Choose an "aop" interceptor (used for AOP calls).
        Callback aopInterceptor = new DynamicAdvisedInterceptor(this.advised);
```

可以看出，普通的 Advice 是通过 `DynamicAdvisedInterceptor` 来拦截执行的

```java
        public Object intercept(Object proxy, Method method, Object[] args, MethodProxy methodProxy) throws Throwable {
            Object oldProxy = null;
            boolean setProxyContext = false;
            Object target = null;
            TargetSource targetSource = this.advised.getTargetSource();
            try {
                if (this.advised.exposeProxy) {
                    // Make invocation available if necessary.
                    oldProxy = AopContext.setCurrentProxy(proxy);
                    setProxyContext = true;
                }
                // Get as late as possible to minimize the time we "own" the target, in case it comes from a pool...
                target = targetSource.getTarget();
                Class<?> targetClass = (target != null ? target.getClass() : null);
                List<Object> chain = this.advised.getInterceptorsAndDynamicInterceptionAdvice(method, targetClass);
                Object retVal;
                // Check whether we only have one InvokerInterceptor: that is,
                // no real advice, but just reflective invocation of the target.
                if (chain.isEmpty()) {
                    // We can skip creating a MethodInvocation: just invoke the target directly.
                    // Note that the final invoker must be an InvokerInterceptor, so we know
                    // it does nothing but a reflective operation on the target, and no hot
                    // swapping or fancy proxying.
                    Object[] argsToUse = AopProxyUtils.adaptArgumentsIfNecessary(method, args);
                    retVal = AopUtils.invokeJoinpointUsingReflection(target, method, argsToUse);
                }
                else {
                    // We need to create a method invocation...
                    retVal = new CglibMethodInvocation(proxy, target, method, args, targetClass, chain, methodProxy).proceed();
                }
                return processReturnType(proxy, target, method, retVal);
            }
            finally {
                if (target != null && !targetSource.isStatic()) {
                    targetSource.releaseTarget(target);
                }
                if (setProxyContext) {
                    // Restore old proxy.
                    AopContext.setCurrentProxy(oldProxy);
                }
            }
        }
```

通过源码，可以看出，cglib proxy 最终拦截 method 之后的处理同 jdk proxy 一样，都是分为两步：

> 1. 获取与 method 匹配的 Advice 链(AdvisedSupport#getInterceptorsAndDynamicInterceptionAdvice)
> 
> 2. 使用 ReflectiveMethodInvocation 通过反射来执行 Advice 链 和 joinpoint（目标方法）

##### 总结：

JDK proxy

> 通过 JdkDynamicAopProxy#getProxy() 来产生代理对象，最终目标方法的拦截是通过 JdkDynamicAopProxy#invoke() 来实现的。

CGLIB proxy

> 通过 CglibAopProxy#getProxy() 来产生代理对象，最终目标方法的拦截主要是通过 
> DynamicAdvisedInterceptor#intercept() 来实现的。

不管是 JDK proxy 还是 CGLIB proxy，最终处理目标方法的拦截过程是相同，都会经过两步：

- 获取与 method 匹配的 Advice 链(AdvisedSupport#getInterceptorsAndDynamicInterceptionAdvice)

- 使用 ReflectiveMethodInvocation 通过反射来执行 Advice 链 和 joinpoint（目标方法）

获取与 method 匹配的 Advice 链是一个相对耗时的操作，所以 Spring AOP 将获取到的 Advice 链进行了缓存。

#### step-4：将 Spring AOP 与 Spring IoC 进行结合

Spring AOP 是一个相对独立的框架，它是通过 `org.springframework.aop.framework.ProxyFactory` 来对外提供代理功能的。

bean 创建时，创建 AOP 代理类
bean 在正常创建过程中，会分为三步：

- AbstractAutowireCapableBeanFactory#createBeanInstance()
  创建 bean 的实例

- AbstractAutowireCapableBeanFactory#populateBean()
  填充 bean 的依赖

- AbstractAutowireCapableBeanFactory#initializeBean()
  初始化 bean

spring AOP 代理类的产生是在 bean 创建的第三阶段 `initializeBean` 时: `AbstractAutoProxyCreator#wrapIfNecessary()`。  
产生代理的总入口是 `ProxyFactory#getProxy()`。

```java
// AbstractAutoProxyCreator#wrapIfNecessary()  
protected Object wrapIfNecessary(Object bean, String beanName, Object cacheKey) {
    if (StringUtils.hasLength(beanName) && this.targetSourcedBeans.contains(beanName)) {
        return bean;
    }
    if (Boolean.FALSE.equals(this.advisedBeans.get(cacheKey))) {
        return bean;
    }
    if (isInfrastructureClass(bean.getClass()) || shouldSkip(bean.getClass(), beanName)) {
        this.advisedBeans.put(cacheKey, Boolean.FALSE);
        return bean;
    }

    // Create proxy if we have advice.
    // 创建 Spring AOP 代理类
    Object[] specificInterceptors = getAdvicesAndAdvisorsForBean(bean.getClass(), beanName, null);
    if (specificInterceptors != DO_NOT_PROXY) {
        this.advisedBeans.put(cacheKey, Boolean.TRUE);
        Object proxy = createProxy(
                bean.getClass(), beanName, specificInterceptors, new SingletonTargetSource(bean));
        this.proxyTypes.put(cacheKey, proxy.getClass());
        return proxy;
    }

    this.advisedBeans.put(cacheKey, Boolean.FALSE);
    return bean;
}
```

#### extra：

个人感觉AdviseSupport有点像BeanDefinition，beanDifinition是维护了Bean的额外信息，AdviseSupport是维护了advice的额外信息







# 源码阅读

## 1、ApplicationContext context = new ClassPathXmlApplicationContext("classpath:spring-test.xml");这句代码是如何执行的？

[[读书笔记]AbstractApplicationContext中refresh方法详解](https://janus.blog.csdn.net/article/details/53893148)

## 2、spring event是怎么实现的？

## 3、spring 中让人印象深刻的设计？
