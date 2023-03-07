# doGetBean()和createBean()详解

## 一、doGetBean()整体逻辑

```java
// 整体逻辑
protected <T> T doGetBean(
            String name, @Nullable Class<T> requiredType, @Nullable Object[] args, boolean typeCheckOnly)
            throws BeansException {
        //参数传进来的name，可能是一个别名，也可能是&开头的name
        //1.别名，alias，需要重定向出一个真实的beanName
        //2、&name，说明要获取的对象是一个FactoryBean对象
        //FactoryBean：如果某个bean的配置非常复杂，使用Spring管理不容易，不灵活，想使用直接编码的方式去构建
        //那么可以提供构建该bean实例的工厂，这个工厂就是FactoryBean接口实现类。FactoryBean接口实现类需要使用Spring管理。
        //这里涉及到两种对象，一种是FactoryBean接口实现类（IOC管理的），另一种就是FactoryBean接口内部管理的对象、
        //如果要拿到FactoryBean接口实现类，使用getBean时传的beanName需要带 & 开头
        //如果要拿到FactoryBean内部管理的对象，直接传beanName不需要带 &
        String beanName = transformedBeanName(name);
        Object bean;

        // Eagerly check singleton cache for manually registered singletons.
        //到缓存中获取共享单实例。这里是第一次getSingleton(beanName)
        Object sharedInstance = getSingleton(beanName);

        //CASE1：缓存中有对应的数据，缓存中的数据可能是普通单实例，也可能是factoryBean，所以需要根据name来判断返回的数据
        if (sharedInstance != null && args == null) {
            if (logger.isTraceEnabled()) {
                if (isSingletonCurrentlyInCreation(beanName)) {
                    logger.trace("Returning eagerly cached instance of singleton bean '" + beanName +
                            "' that is not fully initialized yet - a consequence of a circular reference");
                }
                else {
                    logger.trace("Returning cached instance of singleton bean '" + beanName + "'");
                }
            }

            //这里为什么又要套？不直接拿去用
            //因为拿到的对象可能是普通的单实例，也可能是FactoryBean
            //如果是FactoryBean，还需要进行处理，主要看是否带“&”
            //带&，说明拿的就是FactoryBean，否则拿的是普通bean
            bean = getObjectForBeanInstance(sharedInstance, name, beanName, null);
        }
        //CASE2:缓存中没有数据，需要自己创建
        else {
            // Fail if we're already creating this bean instance:
            // We're assumably within a circular reference.

            //1、原型循环依赖问题判断
            //举个例子：
            //prototypeA -> B, B -> prototypeA
            //1、会向正在创建中的原型集合中添加一个字符串"prototypeA"
            //2、创建prototypeA对象，只是一个早期对象
            //3、处理prototypeA的依赖，发现A依赖了B对象
            //4、触发了Spring.getBean("B")的操作
            //5、根据B的构造方法反射创建出来了B的早期实例
            //6、Spinrg处理B对象的依赖，发现了依赖A
            //7、Spring转头去获取A，Spring.getBean("prototypeA")
            //8、条件会返回true，最终抛出异常，算是结束了循环依赖注入
            if (isPrototypeCurrentlyInCreation(beanName)) {
                throw new BeanCurrentlyInCreationException(beanName);
            }

            // Check if bean definition exists in this factory.
            BeanFactory parentBeanFactory = getParentBeanFactory();
            if (parentBeanFactory != null && !containsBeanDefinition(beanName)) {
                // Not found -> check parent.
                String nameToLookup = originalBeanName(name);
                if (parentBeanFactory instanceof AbstractBeanFactory) {
                    return ((AbstractBeanFactory) parentBeanFactory).doGetBean(
                            nameToLookup, requiredType, args, typeCheckOnly);
                }
                else if (args != null) {
                    // Delegation to parent with explicit args.
                    return (T) parentBeanFactory.getBean(nameToLookup, args);
                }
                else if (requiredType != null) {
                    // No args -> delegate to standard getBean method.
                    return parentBeanFactory.getBean(nameToLookup, requiredType);
                }
                else {
                    return (T) parentBeanFactory.getBean(nameToLookup);
                }
            }

            if (!typeCheckOnly) {
                markBeanAsCreated(beanName);
            }

            try {
                //2、获取合并md信息
                //为什么需要合并?
                //因为bd支持继承，子bd可以继承到父bd的所有信息
                //  例如：    
                   //    <bean id="template" abstract="true">
                   //         <property name="name" value="exampleA"></property>
                   //         <property name="age" value="20"></property>
                   //     </bean>
                   //     
                   //     <!-- exampleB会继承父bean template中name和age的属性值 -->
                   //     <bean id="exampleB" class="cn.example.spring.boke.ExampleB" parent="template"></bean>

                RootBeanDefinition mbd = getMergedLocalBeanDefinition(beanName);
                //判断当前bd是否为抽象bd，抽象bd不能创建实例，只能父bd让子bd继承
                checkMergedBeanDefinition(mbd, beanName, args);

                // Guarantee initialization of beans that the current bean depends on.
                //3.depends-on属性处理..
                //<bean name="A" depends-on="B" ... />
                //<bean name="B" .../>
                //循环依赖问题
                //<bean name="A" depends-on="B" ... />
                //<bean name="B" depends-on="A" .../>
                //Spring是处理不了这种情况的，需要报错..
                //Spring需要发现这种情况的产生。
                //怎么发现呢? 依靠两个Map，一个map是 dependentBeanMap 另一个是 dependenciesForBeanMap
                //1. dependentBeanMap 记录依赖当前beanName的其他beanName
                //2. dependenciesForBeanMap 记录当前beanName依赖的其它beanName集合
                String[] dependsOn = mbd.getDependsOn();
                if (dependsOn != null) {
                    for (String dep : dependsOn) {
                        //判断循环依赖
                        if (isDependent(beanName, dep)) {
                            throw new BeanCreationException(mbd.getResourceDescription(), beanName,
                                    "Circular depends-on relationship between '" + beanName + "' and '" + dep + "'");
                        }
                        //假设<bean name="A" depends-on="B" ... />
                        //dep:B，beanName:A
                        //以B为视角 dependentBeanMap {"B"：{"A"}}
                        //以A为视角 dependenciesForBeanMap {"A" :{"B"}}
                        registerDependentBean(dep, beanName);
                        try {
                            getBean(dep);
                        }
                        catch (NoSuchBeanDefinitionException ex) {
                            throw new BeanCreationException(mbd.getResourceDescription(), beanName,
                                    "'" + beanName + "' depends on missing bean '" + dep + "'", ex);
                        }
                    }
                }

                // Create bean instance.
                //CASE-SINGLETON的情况
                if (mbd.isSingleton()) {
                    sharedInstance = getSingleton(beanName, () -> {
                        try {
                            return createBean(beanName, mbd, args);
                        }
                        catch (BeansException ex) {
                            // Explicitly remove instance from singleton cache: It might have been put there
                            // eagerly by the creation process, to allow for circular reference resolution.
                            // Also remove any beans that received a temporary reference to the bean.
                            destroySingleton(beanName);
                            throw ex;
                        }
                    });
                    //这里为啥不直接返回，还调用getObjectForBeanInstance
                    //因为可能当前的bean是factoryBean，但是要的name带着&，此时需要的是factoryBean内部管理的对象
                    bean = getObjectForBeanInstance(sharedInstance, name, beanName, mbd);
                }
                //CASE-PROTOTYPE的情况
                else if (mbd.isPrototype()) {
                    // It's a prototype -> create a new instance.
                    Object prototypeInstance = null;
                    try {
                        //记录当前线程正在创建的原型对象的beanName
                        beforePrototypeCreation(beanName);
                        //创建对象
                        prototypeInstance = createBean(beanName, mbd, args);
                    }
                    finally {
                        //创建完成，移除当前线程正在创建的原型对象的beanName
                        afterPrototypeCreation(beanName);
                    }
                    bean = getObjectForBeanInstance(prototypeInstance, name, beanName, mbd);
                }
                //CASE-OTHER的情况（不管）
                else {
                    String scopeName = mbd.getScope();
                    if (!StringUtils.hasLength(scopeName)) {
                        throw new IllegalStateException("No scope name defined for bean ´" + beanName + "'");
                    }
                    Scope scope = this.scopes.get(scopeName);
                    if (scope == null) {
                        throw new IllegalStateException("No Scope registered for scope name '" + scopeName + "'");
                    }
                    try {
                        Object scopedInstance = scope.get(beanName, () -> {
                            beforePrototypeCreation(beanName);
                            try {
                                return createBean(beanName, mbd, args);
                            }
                            finally {
                                afterPrototypeCreation(beanName);
                            }
                        });
                        bean = getObjectForBeanInstance(scopedInstance, name, beanName, mbd);
                    }
                    catch (IllegalStateException ex) {
                        throw new BeanCreationException(beanName,
                                "Scope '" + scopeName + "' is not active for the current thread; consider " +
                                "defining a scoped proxy for this bean if you intend to refer to it from a singleton",
                                ex);
                    }
                }
            }
            catch (BeansException ex) {
                cleanupAfterBeanCreationFailure(beanName);
                throw ex;
            }
        }

        // Check if required type matches the type of the actual bean instance.
        if (requiredType != null && !requiredType.isInstance(bean)) {
            try {
                T convertedBean = getTypeConverter().convertIfNecessary(bean, requiredType);
                if (convertedBean == null) {
                    throw new BeanNotOfRequiredTypeException(name, requiredType, bean.getClass());
                }
                return convertedBean;
            }
            catch (TypeMismatchException ex) {
                if (logger.isTraceEnabled()) {
                    logger.trace("Failed to convert bean '" + name + "' to required type '" +
                            ClassUtils.getQualifiedName(requiredType) + "'", ex);
                }
                throw new BeanNotOfRequiredTypeException(name, requiredType, bean.getClass());
            }
        }
        return (T) bean;
    }
```

### 1、处理别名

```java
//参数传进来的name，可能是一个别名，也可能是一个&开头的name
//1.别名，重定向出来真实beanName
//2.&开头的name，说明，你要获取的bean实例对象，是一个FactoryBean对象。
//FactoryBean：如果某个bean的配置非常复杂，使用Spring管理不容易...不够灵活，想要使用编码的形式去构建它，那么你就可以提供
//一个构建该bean实例的工厂，这个工厂就是FactoryBean接口实现类。FactoryBean接口实现类还是需要使用Spring管理的。
//这里就涉及到两种对象，一种是FactoryBean接口实现类（IOC管理的），另一个就是FactoryBean接口内部管理的对象。
//如果要拿FactoryBean接口实现类，使用getBean时传的beanName需要带“&”开头。
//如果你要FactoryBean内部管理的对象，你直接传beanName不需要带“&”开头。
String beanName = transformedBeanName(name);
```

#### 1-1、关于FactoryBean的扩展

当配置文件中的bean的class是FactoryBean的实现类的时候,通过调用getBean()方法返回的不是FactoryBean本身,而是FactoryBean 调用getObject()方法返回的对象,相当于FactoryBean用getObject()方法代理了getBean()方法。

```java
// 这里就涉及到两种对象，一种是FactoryBean接口实现类（IOC管理的），另一个就是FactoryBean接口内部管理的对象。
// 上面这句话如何理解？我们举例说明：

// 1、定义一个MyFactoryBean实现FactoryBean,包装一个Cat对象
public class MyFactoryBean implements FactoryBean<Cat> {
    private String catName;
    @Override
    public Cat getObject() throws Exception {
        return new Cat(catName);
    }

    @Override
    public Class<?> getObjectType() {
        return Cat.class;
    }

    public String getCatName() {
        return catName;
    }

    public void setCatName(String catName) {
        this.catName = catName;
    }


// 定义一个cat对象
class Cat {
        private String name;

        public Cat() {
        }

        @Override
        public String toString() {
            return "Cat{" +
                    "name='" + name + '\'' +
                '}';
        }

        public Cat(String name) {
            this.name = name;
        }
    }
}




// 2、spring.xml文件中定义好bean
<bean id="cat" class="it.wdz.MyFactoryBean">
        <property name="catName" value="xiao7" />
</bean>



// 3、测试输出

        ClassPathXmlApplicationContext context = new ClassPathXmlApplicationContext("classpath:spring.xml");
        Cat cat = (Cat)context.getBean("cat");

        try {
            System.out.println(cat.toString());
        } catch (Exception e) {
            e.printStackTrace();
        }
// 输出结果为： Cat{name='xiao7'}
```

调用顺序：

1. `AbstractBeanFactory`#`transformedBeanName(String name)`

2. `BeanFactoryUtils`#`transformedBeanName(String name)`

3. `AbstractBeanFactory`#`canonicalName(String name)`

```java
// AbstractBeanFactory#transformedBeanName(String name) 封装方法
protected String transformedBeanName(String name) {
    return canonicalName(BeanFactoryUtils.transformedBeanName(name));
}


// BeanFactoryUtils#transformedBeanName(String name) 处理“&”开头的FactoryBean的名称
public static String transformedBeanName(String name) {
    Assert.notNull(name, "'name' must not be null");
    //条件成立：说明beanName是普通bean，直接返回
    if (!name.startsWith(BeanFactory.FACTORY_BEAN_PREFIX)) {
        return name;
    }
    // 如果是FactoryBean，进行去“&”处理，将处理后的名字保存在缓存中。
    // map.computeIfAbsent()方法：
    // 一个向map中put元素的操作方法，如果放入成功，返回新放入的value，如果放入失败，则返回map中原有的value
    return transformedBeanNameCache.computeIfAbsent(name, beanName -> {
        do {
            beanName = beanName.substring(BeanFactory.FACTORY_BEAN_PREFIX.length());
        }
        while (beanName.startsWith(BeanFactory.FACTORY_BEAN_PREFIX));
        return beanName;
    });
}


// AbstractBeanFactory#canonicalName(String name) 处理别名
public String canonicalName(String name) {
    String canonicalName = name;
    // Handle aliasing...
    String resolvedName;
    do {
        resolvedName = this.aliasMap.get(canonicalName);
        if (resolvedName != null) {
            canonicalName = resolvedName;
        }
    }
    while (resolvedName != null);
    return canonicalName;
}
```

### 2、到缓存中获取共享单实例

第一个getSingleton(String beanName) 一个参数的，后面还有一个getSingleton(String beanName, ObjectFactory<?> singletonFactory)，不要搞混了。  

```java
Object sharedInstance = getSingleton(beanName);
```

调用顺序：

1. `AbstractBeanFactory`#`getSingleton(String beanName)`

2. `DefaultSingletonBeanRegistry`#`getSingleton(String beanName, boolean allowEarlyReference)`

### 3、getSingleton(String beanName, boolean allowEarlyReference)方法

`DefaultSingletonBeanRegistry`#`getSingleton(String beanName, boolean allowEarlyReference)`方法，这里涉及到了循环依赖的解决，代码处理了setter循环依赖，原因就在于提前暴露对象到了第3级缓存中，在接下来发现循环依赖的时候，可以从第3级缓存中获取。

```java
    //allowEarlyReference:是否运行拿到早期的引用
    @Nullable
    protected Object getSingleton(String beanName, boolean allowEarlyReference) {
        //到一级缓存中获取beanName对应的单实例对象。
        Object singletonObject = this.singletonObjects.get(beanName);

        //条件一成立（singletonObject == null）： 有几种可能呢？？
        //1.单实例确实尚未创建呢..
        //2.单实例正在创建中，当前发生循环依赖了...

        //什么循环依赖？
        //A->B B->A , A->B  B->C  C->A
        //单实例有几种循环依赖呢？
        //1.构造方法循环依赖    （无解）
        //2.setter造成的循环依赖  （有解，靠三级缓存解决。）

        //三级缓存怎么解决的setter造成的循环依赖呢？
        //举个例子：
        //A->B,B->A   （setter依赖）
        //    1.假设Spring先实例化A，首先拿到A的构造方法，进行反射创建出来A的早期实例对象，这个时候，这个早期对象被包装成了ObjectFactory对象，放到了3级缓存。（AbstractAutowireCapableBeanFactory#CreateBeanInstance方法中的obtainFromSupplier方法将beanName存放到缓存中。）
        //    2.处理A的依赖数据，检查发现，A它依赖了B对象，所以接下来，Spring就会去根据B类型到容器中去getBean(B.class)，这里就递归了。
        //    3.拿到B的构造方法，进行反射创建出来B的早期实例对象，它也会把B包装成ObjectFactory对象，放到3级缓存。
        //    4.处理B的依赖数据，检查发现，B它依赖了A对象，所以接下来，Spring就会去根据A类型到容器中去getBean(A.class)，去拿A对象，这个又递归了。
        //    5.程序还会走到当前这个方法。getSingleton这个方法。
        //    6.条件一成立，条件二也会成立。
        if (singletonObject == null && isSingletonCurrentlyInCreation(beanName)) {
            synchronized (this.singletonObjects) {
                //检查二级缓存
                singletonObject = this.earlySingletonObjects.get(beanName);
                //条件成立：说明二级没有，到三级缓存查看。
                if (singletonObject == null && allowEarlyReference) {
                    //Spring为什么需要有3级缓存存在，而不是只有2级缓存呢？
                    //AOP，靠什么实现的呢？动态代理
                    //静态代理：需要手动写代码，实现一个新的java文件，这个java类 和 需要代理的对象 实现同一个接口，内部维护一个被代理对象（原生）
                    //代理类，在调用原生对象前后，可以加一些逻辑. 代理对象 和 被代理对象 是两个不同的对象，内存地址一定是不一样的。
                    //动态代理：不需要人为写代码了，而是依靠字节码框架动态生成class字节码文件，然后jvm再加载，然后也一样 也是去new代理对象，这个
                    //代理对象 没啥特殊的，也是内部保留了 原生对象，然后在调用原生对象前后 实现的 字节码增强。
                    //3级缓存在这里有什么目的呢？
                    //3级缓存里面保存的是对象工厂，这个对象工厂内部保留着最原生的对象引用，ObjectFactory的实现类，getObject()方法，它需要考虑一个问题。
                    //它到底要返回原生的，还是增强后的。
                    //getObject会判断当前这个早期实例 是否需要被增强，如果是，那么提前完成动态代理增强，返回代理对象。否则，返回原生对象。
                    ObjectFactory<?> singletonFactory = this.singletonFactories.get(beanName);
                    //条件成立：3级有数据。这里涉及到缓存升级。
                    if (singletonFactory != null) {
                        singletonObject = singletonFactory.getObject();
                        //向2级缓存存数据
                        this.earlySingletonObjects.put(beanName, singletonObject);
                        //将3级缓存的数据干掉。
                        this.singletonFactories.remove(beanName);
                    }
                }
            }
        }
        return singletonObject;
    }
```

### 3、mbd的一些处理

#### 3-1、合并beanDefinition

```java
 //为什么需要合并?
                //因为bd支持继承，子bd可以继承到父bd的所有信息
                //  例如：    
                   //    <bean id="template" abstract="true">
                   //         <property name="name" value="exampleA"></property>
                   //         <property name="age" value="20"></property>
                   //     </bean>
                   //     
                   //     <!-- exampleB会继承父bean template中name和age的属性值 -->
                   //     <bean id="exampleB" class="cn.example.spring.boke.ExampleB" parent="template"></bean>
RootBeanDefinition mbd = getMergedLocalBeanDefinition(beanName);
```

#### 3-2、beanDefinition depends-on属性的循环依赖处理

涉及到两个map：①dependentBeanMap ②dependenciesForBeanMap

这里比较绕，尤其是多个bd循环依赖的时候的判断，需要自己结合注释去看，要有耐心。

但是整个判断逻辑就一句话：<mark>`我依赖的`和`依赖我的`只要有重复就是循环依赖</mark>

```java
 //3.depends-on属性处理..
                //<bean name="A" depends-on="B" ... />
                //<bean name="B" .../>
                //循环依赖问题
                //<bean name="A" depends-on="B" ... />
                //<bean name="B" depends-on="A" .../>
                //Spring是处理不了这种情况的，需要报错..
                //Spring需要发现这种情况的产生。
                //怎么发现呢? 依靠两个Map，一个map是 dependentBeanMap 另一个是 dependenciesForBeanMap
                //1. dependentBeanMap 记录依赖当前beanName的其他beanName
                //2. dependenciesForBeanMap 记录当前beanName依赖的其它beanName集合
                String[] dependsOn = mbd.getDependsOn();
                if (dependsOn != null) {
                    for (String dep : dependsOn) {
                        //判断循环依赖
                        if (isDependent(beanName, dep)) {
                            throw new BeanCreationException(mbd.getResourceDescription(), beanName,
                                    "Circular depends-on relationship between '" + beanName + "' and '" + dep + "'");
                        }
                        //假设<bean name="A" depends-on="B" ... />
                        //dep:B，beanName:A
                        //以B为视角 dependentBeanMap {"B"：{"A"}}
                        //以A为视角 dependenciesForBeanMap {"A" :{"B"}}
                         // 填充 dependentBeanMap 和 dependenciesForBeanMap
                        registerDependentBean(dep, beanName);


 // DefaultSingletonBeanRegistry#isDependent(String beanName, String dependentBeanName, @Nullable Set<String> alreadySeen)
 private boolean isDependent(String beanName, String dependentBeanName, @Nullable Set<String> alreadySeen) {
        //beanName=A, dependentBeanName=B  dependentBeanMap={A:{"B"}}
        if (alreadySeen != null && alreadySeen.contains(beanName)) {
            return false;
        }
        //如果A没有别名的话,canonicalName=A, beanName=A, dependentBeanName=B
        String canonicalName = canonicalName(beanName);
        Set<String> dependentBeans = this.dependentBeanMap.get(canonicalName);
        //dependentBeans={"B"}
        if (dependentBeans == null) {
            return false;
        }
        // {"B"} 包含了 B, 说明发生了循环依赖
        if (dependentBeans.contains(dependentBeanName)) {
            return true;
        }

        // 多个循环依赖
        // <bean name=A depends-on = B ...>
        // <bean name=B depends-on = C ...>
        // <bean name=C depends-on = A ...>
        // 假设: A,B均已成功加载,未发现循环依赖,此时运行到加载C...
        // beanName=C, dependentBeanName=A, dependentBeans={"B"}
        for (String transitiveDependency : dependentBeans) {
            if (alreadySeen == null) {
                alreadySeen = new HashSet<>();
            }
            alreadySeen.add(beanName);
            // 遍历{"B"} transitiveDependency = B, dependentBeanName=A, 回到AB互相依赖的情况, 返回true, 发生循环依赖
            if (isDependent(transitiveDependency, dependentBeanName, alreadySeen)) {
                return true;
            }
        }
        return false;
    }
```

### 4、单例bean构造的情况

#### 4-1、CASE-SINGLETON：

这里是第二个getSingleton方法，`DefaultSingletonBeanRegistry`#`getSingleton(String beanName, ObjectFactory<?> singletonFactory)`这里第二个参数不是boolean，而是一个单例工厂。

这里查询的缓存是`singletonObjects`

```java
// 如果bean是单例
if (mbd.isSingleton()) {
                    // 1、拿到单例对象
                    sharedInstance = getSingleton(beanName, () -> {
                        try {
                            return createBean(beanName, mbd, args);
                        }
                        catch (BeansException ex) {
                            // Explicitly remove instance from singleton cache: It might have been put there
                            // eagerly by the creation process, to allow for circular reference resolution.
                            // Also remove any beans that received a temporary reference to the bean.
                            destroySingleton(beanName);
                            throw ex;
                        }
                    });
                    // 2、
                    bean = getObjectForBeanInstance(sharedInstance, name, beanName, mbd);
                }
```

```java
public Object getSingleton(String beanName, ObjectFactory<?> singletonFactory) {
        Assert.notNull(beanName, "Bean name must not be null");
        synchronized (this.singletonObjects) {
            Object singletonObject = this.singletonObjects.get(beanName);
            if (singletonObject == null) {
                //容器销毁时，会设置这个属性为true，这个时候就不能再创建bean实例了，直接抛错。
                if (this.singletonsCurrentlyInDestruction) {
                    throw new BeanCreationNotAllowedException(beanName,
                            "Singleton bean creation not allowed wile singletons of this factory are in destruction " +
                            "(Do not request a bean from a BeanFactory in a destroy method implementation!)");
                }

                if (logger.isDebugEnabled()) {
                    logger.debug("Creating shared instance of singleton bean '" + beanName + "'");
                }

                //将当前beanName放入到“正在创建中单实例集合”，放入成功，说明没有产生循环依赖，失败，则产生循环依赖，里面会抛异常。
                //举个例子：构造方法参数依赖
                //A->B    B->A
                //1.加载A，根据A的构造方法，想要去实例化A对象，但是发现A的构造方法有一个参数是B（在这之前，已经向这个集合中添加了 {A}）
                //2.因为A的构造方法依赖B，所以触发了加载B的逻辑..
                //3.加载B，根据B的构造方法，想要去实例化B对象，但是发现B的构造方法有一个参数是A（在这之前，已经向这个集合中添加了 {A，B}）
                //4.因为B的构造方法依赖A，所以再次触发了加载A的逻辑..
                //5.再次来到这个getSingleton方法里，调用beforeSingletonCreation(A),因为创建中集合 已经有A了，所以添加失败，抛出异常
                //完事。
                beforeSingletonCreation(beanName);

                boolean newSingleton = false;
                boolean recordSuppressedExceptions = (this.suppressedExceptions == null);
                if (recordSuppressedExceptions) {
                    this.suppressedExceptions = new LinkedHashSet<>();
                }
                try {
                    singletonObject = singletonFactory.getObject();
                    newSingleton = true;
                }
                catch (IllegalStateException ex) {
                    // Has the singleton object implicitly appeared in the meantime ->
                    // if yes, proceed with it since the exception indicates that state.
                    singletonObject = this.singletonObjects.get(beanName);
                    if (singletonObject == null) {
                        throw ex;
                    }
                }
                catch (BeanCreationException ex) {
                    if (recordSuppressedExceptions) {
                        for (Exception suppressedException : this.suppressedExceptions) {
                            ex.addRelatedCause(suppressedException);
                        }
                    }
                    throw ex;
                }
                finally {
                    if (recordSuppressedExceptions) {
                        this.suppressedExceptions = null;
                    }
                    //将beanName从当前正在创建的bean的集合移除
                    afterSingletonCreation(beanName);
                }
                if (newSingleton) {
                    //添加到缓存
                    addSingleton(beanName, singletonObject);
                }
            }
            return singletonObject;
        }
    }


    //判断当前创建的bean是否有循环依赖
    protected void beforeSingletonCreation(String beanName) {
        //条件成立：发生循环依赖
        if (!this.inCreationCheckExclusions.contains(beanName) && !this.singletonsCurrentlyInCreation.add(beanName)) {
            throw new BeanCurrentlyInCreationException(beanName);
        }
    }


    // 创建结束，最终删除记录
    protected void afterSingletonCreation(String beanName) {
        if (!this.inCreationCheckExclusions.contains(beanName) && !this.singletonsCurrentlyInCreation.remove(beanName)) {
            throw new IllegalStateException("Singleton '" + beanName + "' isn't currently in creation");
        }
    }
```

#### 4-2、CASE-PROTOTYPE

和singleton类似，区别在于查询的缓存不同。循环依赖的问题在处理mbd之前就已经判断过了

这里查询的是`prototypesCurrentlyInCreation：ThreadLocal<Object>`

```java
//CASE-PROTOTYPE的情况
                else if (mbd.isPrototype()) {
                    // It's a prototype -> create a new instance.
                    Object prototypeInstance = null;
                    try {
                        //记录当前线程正在创建的原型对象的beanName
                        beforePrototypeCreation(beanName);
                        //创建对象
                        prototypeInstance = createBean(beanName, mbd, args);
                    }
                    finally {
                        //创建完成，移除当前线程正在创建的原型对象的beanName
                        afterPrototypeCreation(beanName);
                    }
                    // 
                    bean = getObjectForBeanInstance(prototypeInstance, name, beanName, mbd);
                }


    protected void beforePrototypeCreation(String beanName) {
        //prototypesCurrentlyInCreation：ThreadLocal<Object>
        Object curVal = this.prototypesCurrentlyInCreation.get();
        //记录当前线程正在创建的bean，并且还没有创建完的bean
        if (curVal == null) {
            this.prototypesCurrentlyInCreation.set(beanName);
        }
        else if (curVal instanceof String) {
            Set<String> beanNameSet = new HashSet<>(2);
            beanNameSet.add((String) curVal);
            beanNameSet.add(beanName);
            this.prototypesCurrentlyInCreation.set(beanNameSet);
        }
        else {
            Set<String> beanNameSet = (Set<String>) curVal;
            beanNameSet.add(beanName);
        }
    }
```

### 5、doGetBean总结

doGetBean的主要逻辑：

1. 先从缓存中获取bean实例，获取不到那么就根据需要获取的bean实例是单实例还是prototype去走对应的逻辑

2. 在获取单实例或者prototype实例之前，会先处理depends-on依赖的实例，这里涉及到了循环依赖问题，不能解决，只能检测并且抛出异常。（做法就是提供两个map，一个是谁依赖了我，一个是我依赖了谁，等发现存在依赖bean的时候会去走getBean的逻辑，发现我要依赖的bean，它已经依赖了我，这个时候抛出异常。）

3. 单实例的获取，主要就是走getSingleton的逻辑，并且提供了一个createBean的ObjectFactory。getSingleton的逻辑首先会去一级缓存获取bean实例，没有拿到那么就需要创建单实例了，创建单实例可能存在构造方法循环依赖，那么就要检测到循环依赖，抛出异常了。具体的操作就是，
   
   1. 先记录现在要创建的beanName存放到set集合中，存放成功则说明没有发送循环依赖，失败则说明发送了循环依赖；
   
   2. 存放成功，那么就去走ObjectFactory.getObject()，这里会去走createBean的逻辑；
   
   3. 最终将创建的bean存放到一级缓存中，并且从set集合中删除记录的beanName

4. 在走对应的逻辑之前，都会先判断是否发生prototype循环依赖，其实判断机制实现的前置内容都是在创建bean之前，会记录当前正在创建的beanName到threadLocal中，然后再去创建bean，最后再从threadLocal中删除记录的beanName

## 二、createBean()整体逻辑

#### 6-1、 createBean(beanName, mbd, args);

```java
    @Override
    protected Object createBean(String beanName, RootBeanDefinition mbd, @Nullable Object[] args)
            throws BeanCreationException {

        if (logger.isTraceEnabled()) {
            logger.trace("Creating instance of bean '" + beanName + "'");
        }
        RootBeanDefinition mbdToUse = mbd;

        // Make sure bean class is actually resolved at this point, and
        // clone the bean definition in case of a dynamically resolved Class
        // which cannot be stored in the shared merged bean definition.

        //判断当前mbd中的class是否已经加载到jvm中，如果未加载，则使用类加载器加载到jvm中，，并且返回class对象
        Class<?> resolvedClass = resolveBeanClass(mbd, beanName);

        //条件1：说明拿到了mbd实例化对象时的真实Class对象
        //条件2：成立说明mbd在resolveBeanClass之前是没有Class对象的
        //条件3：条件成立，说明mbd中有className
        if (resolvedClass != null && !mbd.hasBeanClass() && mbd.getBeanClassName() != null) {
            mbdToUse = new RootBeanDefinition(mbd);
            mbdToUse.setBeanClass(resolvedClass);
        }

        // Prepare method overrides.
        //不重要，跳过...
        try {
            mbdToUse.prepareMethodOverrides();
        }
        catch (BeanDefinitionValidationException ex) {
            throw new BeanDefinitionStoreException(mbdToUse.getResourceDescription(),
                    beanName, "Validation of method overrides failed", ex);
        }


        try {
            // Give BeanPostProcessors a chance to return a proxy instead of the target bean instance.
            //跳过后处理器返回一个代理实例对象...注意这里的代理对象不是springAop逻辑实现的地方
            //instantiation：实例化之前的解析。不要和init混淆，init是初始化
            //后处理器调用点：创建实例之前的调用点
            Object bean = resolveBeforeInstantiation(beanName, mbdToUse);

            //条件成立：形成一个短路操作，直接返回。一般不会直接返回，不是正常的执行逻辑
            if (bean != null) {
                return bean;
            }
        }
        catch (Throwable ex) {
            throw new BeanCreationException(mbdToUse.getResourceDescription(), beanName,
                    "BeanPostProcessor before instantiation of bean failed", ex);
        }

        try {
            //核心方法：创建bean实例对象，并且声明周期的动作大部分在这里
            Object beanInstance = doCreateBean(beanName, mbdToUse, args);
            if (logger.isTraceEnabled()) {
                logger.trace("Finished creating instance of bean '" + beanName + "'");
            }
            return beanInstance;
        }
        catch (BeanCreationException | ImplicitlyAppearedSingletonException ex) {
            // A previously detected exception with proper bean creation context already,
            // or illegal singleton state to be communicated up to DefaultSingletonBeanRegistry.
            throw ex;
        }
        catch (Throwable ex) {
            throw new BeanCreationException(
                    mbdToUse.getResourceDescription(), beanName, "Unexpected exception during bean creation", ex);
        }
    }
```

#### 6-2、doCreateBean(beanName, mbdToUse, args)

```java
protected Object doCreateBean(String beanName, RootBeanDefinition mbd, @Nullable Object[] args)
            throws BeanCreationException {

        // Instantiate the bean.
        //包装对象，内部最核心的就是真实的bean实例。另外提供的额外的接口，如：属性访问器
        BeanWrapper instanceWrapper = null;


        if (mbd.isSingleton()) {
            instanceWrapper = this.factoryBeanInstanceCache.remove(beanName);
        }


        if (instanceWrapper == null) {
            //创建真实的bean实例，并且将其包装到beanWrapper实例中
            instanceWrapper = createBeanInstance(beanName, mbd, args);
        }

        //获取创建出来的真实的bean
        Object bean = instanceWrapper.getWrappedInstance();
        //获取创建出来的真实的bean的类型
        Class<?> beanType = instanceWrapper.getWrappedClass();
        if (beanType != NullBean.class) {
            //记录实例类型
            mbd.resolvedTargetType = beanType;
        }

        // Allow post-processors to modify the merged bean definition.
        synchronized (mbd.postProcessingLock) {
            //每个bd只能执行一次applyMergedBeanDefinitionPostProcessors
            if (!mbd.postProcessed) {
                try {
                    //后处理器调用点：合并bd信息，因为接下来就是populate处理依赖了。（保存@Autowired，@Value，@Inject注解信息到缓存）
                    applyMergedBeanDefinitionPostProcessors(mbd, beanType, beanName);
                }
                catch (Throwable ex) {
                    throw new BeanCreationException(mbd.getResourceDescription(), beanName,
                            "Post-processing of merged bean definition failed", ex);
                }
                mbd.postProcessed = true;
            }
        }

        // Eagerly cache singletons to be able to resolve circular references
        // even when triggered by lifecycle interfaces like BeanFactoryAware.
        //决定了早期实例是否要暴露到第3级缓存中
        boolean earlySingletonExposure = (mbd.isSingleton() && this.allowCircularReferences &&
                isSingletonCurrentlyInCreation(beanName));



        if (earlySingletonExposure) {
            if (logger.isTraceEnabled()) {
                logger.trace("Eagerly caching bean '" + beanName +
                        "' to allow for resolving potential circular references");
            }
            //参数2：将bean包装成一个ObjectFactory，getObject调用的是getEarlyBeanReference的逻辑
            addSingletonFactory(beanName, () -> getEarlyBeanReference(beanName, mbd, bean));
        }




        // Initialize the bean instance.
        Object exposedObject = bean;
        try {
            //处理当前实例的依赖数据的，依赖注入在这一步完成（出来普通的property还有@Autowired，@Value，@Inject信息的注入）
            populateBean(beanName, mbd, instanceWrapper);
            //生命周期中初始化方法的调用
            exposedObject = initializeBean(beanName, exposedObject, mbd);
        }
        catch (Throwable ex) {
            if (ex instanceof BeanCreationException && beanName.equals(((BeanCreationException) ex).getBeanName())) {
                throw (BeanCreationException) ex;
            }
            else {
                throw new BeanCreationException(
                        mbd.getResourceDescription(), beanName, "Initialization of bean failed", ex);
            }
        }




        if (earlySingletonExposure) {
            Object earlySingletonReference = getSingleton(beanName, false);
            if (earlySingletonReference != null) {
                if (exposedObject == bean) {
                    exposedObject = earlySingletonReference;
                }
                else if (!this.allowRawInjectionDespiteWrapping && hasDependentBean(beanName)) {
                    String[] dependentBeans = getDependentBeans(beanName);
                    Set<String> actualDependentBeans = new LinkedHashSet<>(dependentBeans.length);
                    for (String dependentBean : dependentBeans) {
                        if (!removeSingletonIfCreatedForTypeCheckOnly(dependentBean)) {
                            actualDependentBeans.add(dependentBean);
                        }
                    }
                    if (!actualDependentBeans.isEmpty()) {
                        throw new BeanCurrentlyInCreationException(beanName,
                                "Bean with name '" + beanName + "' has been injected into other beans [" +
                                StringUtils.collectionToCommaDelimitedString(actualDependentBeans) +
                                "] in its raw version as part of a circular reference, but has eventually been " +
                                "wrapped. This means that said other beans do not use the final version of the " +
                                "bean. This is often the result of over-eager type matching - consider using " +
                                "'getBeanNamesForType' with the 'allowEagerInit' flag turned off, for example.");
                    }
                }
            }
        }

        // Register bean as disposable.
        try {
            //判断当前bean是否需要注册析构回调，即容器销毁的时候，会执行bean的销毁方法
            registerDisposableBeanIfNecessary(beanName, bean, mbd);
        }
        catch (BeanDefinitionValidationException ex) {
            throw new BeanCreationException(
                    mbd.getResourceDescription(), beanName, "Invalid destruction signature", ex);
        }

        return exposedObject;
    }
```

#### 6-3、createBeanInstance(beanName, mbd, args)

这个方法就是选择哪个构造函数的，往里深了跟就是选择最优的构造函数，很繁琐。抓住主要逻辑就可以。走出这个方法就得到了早期bean实例

```java
protected BeanWrapper createBeanInstance(String beanName, RootBeanDefinition mbd, @Nullable Object[] args) {
        // Make sure bean class is actually resolved at this point.
        //返回mbd中真实bean的Class对象。
        Class<?> beanClass = resolveBeanClass(mbd, beanName);


        //三个条件想表达的意思：bd中如果nonPublicAccessAllowed字段值为true，表示class是非公开类型的 也可以创建实例，反之false，说明是无法创建的..
        if (beanClass != null && !Modifier.isPublic(beanClass.getModifiers()) && !mbd.isNonPublicAccessAllowed()) {
            throw new BeanCreationException(mbd.getResourceDescription(), beanName,
                    "Bean class isn't public, and non-public access not allowed: " + beanClass.getName());
        }



        Supplier<?> instanceSupplier = mbd.getInstanceSupplier();
        if (instanceSupplier != null) {
            return obtainFromSupplier(instanceSupplier, beanName);
        }


        //bean标签中配置了 factory-method的情况处理，这个分支也挺复杂的...跳过..
        if (mbd.getFactoryMethodName() != null) {
            return instantiateUsingFactoryMethod(beanName, mbd, args);
        }



        // Shortcut when re-creating the same bean...
        //表示bd对应的构造信息是否已经解析成可以反射调用的构造方法method信息了...
        boolean resolved = false;
        //是否自动匹配构造方法..
        boolean autowireNecessary = false;

        if (args == null) {
            synchronized (mbd.constructorArgumentLock) {
                //条件成立：说明bd的构造信息已经转化成可以反射调用的method了...
                if (mbd.resolvedConstructorOrFactoryMethod != null) {
                    resolved = true;
                    //当resolvedConstructorOrFactoryMethod 有值时，且构造方法有参数，那么可以认为这个字段值就是true。
                    //只有什么情况下这个字段是false呢？
                    //1.resolvedConstructorOrFactoryMethod == null
                    //2.当resolvedConstructorOrFactoryMethod 表示的是默认的构造方法，无参构造方法。
                    autowireNecessary = mbd.constructorArgumentsResolved;
                }

            }
        }

        if (resolved) {
            if (autowireNecessary) {
                //有参数，那么就需要根据参数 去匹配合适的构造方法了...
                //拿出当前Class的所有构造器，然后根据参数信息 去匹配出一个最优的选项，然后执行最优的 构造器 创建出实例。
                return autowireConstructor(beanName, mbd, null, null);
            }
            else {
                //无参构造方法处理
                return instantiateBean(beanName, mbd);
            }
        }


        // Candidate constructors for autowiring?
        //典型的应用：@Autowired 注解打在了 构造器方法上。
        Constructor<?>[] ctors = determineConstructorsFromBeanPostProcessors(beanClass, beanName);
        //条件一：成立，后处理指定了构造方法数组
        //条件二：mbd autowiredMode 一般情况是no
        //条件三：条件成立，说明bean信息中配置了 构造参数信息。
        //条件四：getBean时，args有参数..
        if (ctors != null || mbd.getResolvedAutowireMode() == AUTOWIRE_CONSTRUCTOR ||
                mbd.hasConstructorArgumentValues() || !ObjectUtils.isEmpty(args)) {
            return autowireConstructor(beanName, mbd, ctors, args);
        }


        // Preferred constructors for default construction?

        ctors = mbd.getPreferredConstructors();
        if (ctors != null) {
            return autowireConstructor(beanName, mbd, ctors, null);
        }

        //未指定构造参数，未设定偏好...使用默认的 无参数的构造方法进行创建实例。
        // No special handling: simply use no-arg constructor.
        return instantiateBean(beanName, mbd);
    }
```

#### 6-4、applyMergedBeanDefinitionPostProcessors

```java
        // Allow post-processors to modify the merged bean definition.
        synchronized (mbd.postProcessingLock) {
            //每个bd只能执行一次applyMergedBeanDefinitionPostProcessors
            if (!mbd.postProcessed) {
                try {
                    //后处理器调用点：合并bd信息，因为接下来就是populate处理依赖了。（保存@Autowired，@Value，@Inject注解信息到缓存）
                    applyMergedBeanDefinitionPostProcessors(mbd, beanType, beanName);
                }
                catch (Throwable ex) {
                    throw new BeanCreationException(mbd.getResourceDescription(), beanName,
                            "Post-processing of merged bean definition failed", ex);
                }
                mbd.postProcessed = true;
            }
        }
```

#### 6-5、早期实例提前暴露

```java
if (earlySingletonExposure) {
    if (logger.isTraceEnabled()) {
        logger.trace("Eagerly caching bean '" + beanName +
                "' to allow for resolving potential circular references");
    }
    //参数2：将bean包装成一个ObjectFactory，getObject调用的是getEarlyBeanReference的逻辑
    addSingletonFactory(beanName, () -> getEarlyBeanReference(beanName, mbd, bean));
}


// 提前暴露到三级缓存
protected void addSingletonFactory(String beanName, ObjectFactory<?> singletonFactory) {
    Assert.notNull(singletonFactory, "Singleton factory must not be null");
    // 串行化
    synchronized (this.singletonObjects) {
        if (!this.singletonObjects.containsKey(beanName)) {
            //存放到第3级缓存
            this.singletonFactories.put(beanName, singletonFactory);
            this.earlySingletonObjects.remove(beanName);
            this.registeredSingletons.add(beanName);
        }
    }
}
```

#### 6-6、populateBean 依赖注入

这里的逻辑完成了依赖注入操作，注入了普通的实例，还将之前后置处理器中的注解元数据转为propertyValue并且注入依赖。

```java
Object exposedObject = bean;
    try {
        //处理当前实例的依赖数据的，依赖注入在这一步完成（出来普通的property还有@Autowired，@Value，@Inject信息的注入）
        populateBean(beanName, mbd, instanceWrapper);
        //生命周期中初始化方法的调用
        exposedObject = initializeBean(beanName, exposedObject, mbd);
    }
    catch (Throwable ex) {
        if (ex instanceof BeanCreationException && beanName.equals(((BeanCreationException) ex).getBeanName())) {
            throw (BeanCreationException) ex;
        }
        else {
            throw new BeanCreationException(
                    mbd.getResourceDescription(), beanName, "Initialization of bean failed", ex);
        }
    }
```

```java
protected void populateBean(String beanName, RootBeanDefinition mbd, @Nullable BeanWrapper bw) {
    if (bw == null) {
        if (mbd.hasPropertyValues()) {
            throw new BeanCreationException(
                    mbd.getResourceDescription(), beanName, "Cannot apply property values to null instance");
        }
        else {
            // Skip property population phase for null instance.
            return;
        }
    }

    // Give any InstantiationAwareBeanPostProcessors the opportunity to modify the
    // state of the bean before properties are set. This can be used, for example,
    // to support styles of field injection.
    if (!mbd.isSynthetic() && hasInstantiationAwareBeanPostProcessors()) {
        for (BeanPostProcessor bp : getBeanPostProcessors()) {
            if (bp instanceof InstantiationAwareBeanPostProcessor) {
                InstantiationAwareBeanPostProcessor ibp = (InstantiationAwareBeanPostProcessor) bp;
                //AfterInstantiation实例化之后的后处理器调用，返回值决定了当前实例是否需要在进行依赖注入处理。
                //默认ibp.postProcessAfterInstantiation返回ture，如果不想要依赖注入，可以自己写一个InstantiationAwareBeanPostProcessor实现类
                if (!ibp.postProcessAfterInstantiation(bw.getWrappedInstance(), beanName)) {
                    return;
                }
            }
        }
    }


    //来到这里：处理依赖注入的逻辑

    //<property name="id" value="1"></property>
    //<property name="name" value="sfan"></property>
    //<property name="age" value="18"></property>
    PropertyValues pvs = (mbd.hasPropertyValues() ? mbd.getPropertyValues() : null);


    int resolvedAutowireMode = mbd.getResolvedAutowireMode();
    //条件成立：说明bd的autowireMode是byName或者byType
    if (resolvedAutowireMode == AUTOWIRE_BY_NAME || resolvedAutowireMode == AUTOWIRE_BY_TYPE) {
        //包装成MutablePropertyValues
        MutablePropertyValues newPvs = new MutablePropertyValues(pvs);
        // Add property values based on autowire by name if applicable.
        if (resolvedAutowireMode == AUTOWIRE_BY_NAME) {
            //byName应该怎么处理？
            //根据字段名称去查找依赖的bean，然后完成注入
            autowireByName(beanName, mbd, bw, newPvs);
        }


        // Add property values based on autowire by type if applicable.
        if (resolvedAutowireMode == AUTOWIRE_BY_TYPE) {

            autowireByType(beanName, mbd, bw, newPvs);
        }
        //newPvs相当于处理了依赖数据之后的pvs
        pvs = newPvs;
    }


    //表示当前是否有InstantiationAwareBeanPostProcessors的后置处理器
    boolean hasInstAwareBpps = hasInstantiationAwareBeanPostProcessors();
    //不重要
    boolean needsDepCheck = (mbd.getDependencyCheck() != AbstractBeanDefinition.DEPENDENCY_CHECK_NONE);

    PropertyDescriptor[] filteredPds = null;

    if (hasInstAwareBpps) {
        if (pvs == null) {
            pvs = mbd.getPropertyValues();
        }


        //后处理器调点：InstantiationAwareBeanPostProcessor.postProcessProperties
        for (BeanPostProcessor bp : getBeanPostProcessors()) {
            if (bp instanceof InstantiationAwareBeanPostProcessor) {
                //典型应用：@Autowired注解的注入（之前扫描了注解信息，并且存放到了InstantiationAwareBeanPostProcessor缓存中，
                // 来到这里将注解元数据转为PropertyValues，注入value到bean的指定field中）
                InstantiationAwareBeanPostProcessor ibp = (InstantiationAwareBeanPostProcessor) bp;
                PropertyValues pvsToUse = ibp.postProcessProperties(pvs, bw.getWrappedInstance(), beanName);
                if (pvsToUse == null) {
                    if (filteredPds == null) {
                        filteredPds = filterPropertyDescriptorsForDependencyCheck(bw, mbd.allowCaching);
                    }
                    pvsToUse = ibp.postProcessPropertyValues(pvs, filteredPds, bw.getWrappedInstance(), beanName);
                    if (pvsToUse == null) {
                        return;
                    }
                }
                pvs = pvsToUse;
            }
        }



    }
    if (needsDepCheck) {
        if (filteredPds == null) {
            filteredPds = filterPropertyDescriptorsForDependencyCheck(bw, mbd.allowCaching);
        }
        checkDependencies(beanName, mbd, filteredPds, pvs);
    }

    if (pvs != null) {
        //完成的依赖注入合并后的pvs，应用到真实的实例中
        applyPropertyValues(beanName, mbd, bw, pvs);
    }
}
```

#### 6-7、initializeBean 初始化

两件事情：

1. 对于实现aware接口方法的实例注入  

2. 两个后置处理器和一个初始化方法的执行。

```java
protected Object initializeBean(String beanName, Object bean, @Nullable RootBeanDefinition mbd) {
        if (System.getSecurityManager() != null) {
            AccessController.doPrivileged((PrivilegedAction<Object>) () -> {
                invokeAwareMethods(beanName, bean);
                return null;
            }, getAccessControlContext());
        }
        else {
            //aware接口对应方法的参数注入
            invokeAwareMethods(beanName, bean);
        }



        Object wrappedBean = bean;
        if (mbd == null || !mbd.isSynthetic()) {
            //后处理器调用点：BeforeInitialization初始化之前的后处理器调用点，主要还是aware接口方法对于实例的注入
            wrappedBean = applyBeanPostProcessorsBeforeInitialization(wrappedBean, beanName);
        }

        try {
            //执行初始化方法
            invokeInitMethods(beanName, wrappedBean, mbd);
        }
        catch (Throwable ex) {
            throw new BeanCreationException(
                    (mbd != null ? mbd.getResourceDescription() : null),
                    beanName, "Invocation of init method failed", ex);
        }

        //后置处理器调用点：AfterInitialization初始化之后的后处理器调用点
        //典型应用：Spring AOP的实现，暂时不涉及，后面再来
        if (mbd == null || !mbd.isSynthetic()) {
            wrappedBean = applyBeanPostProcessorsAfterInitialization(wrappedBean, beanName);
        }

        return wrappedBean;
    }初始化方法：两种方式，要么是实现InitializingBean接口，要么是指定init-method，两个都有那么执行接口的方法两种方式，要么是实现InitializingBean接口，要么是指定init-method，两个都有那么执行接口的方法protected void invokeInitMethods(String beanName, Object bean, @Nullable RootBeanDefinition mbd)
            throws Throwable {
        //执行初始化方法有两种方式，一种是实现了InitializingBean，一种是xml中的bean标签指定了init-method属性    boolean isInitializingBean = (bean instanceof InitializingBean);
    //条件成立：说明实现了InitializingBean接口
    if (isInitializingBean && (mbd == null || !mbd.isExternallyManagedInitMethod("afterPropertiesSet"))) {
        if (logger.isTraceEnabled()) {
            logger.trace("Invoking afterPropertiesSet() on bean with name '" + beanName + "'");
        }
        if (System.getSecurityManager() != null) {
            try {
                AccessController.doPrivileged((PrivilegedExceptionAction<Object>) () -> {
                    ((InitializingBean) bean).afterPropertiesSet();
                    return null;
                }, getAccessControlContext());
            }
            catch (PrivilegedActionException pae) {
                throw pae.getException();
            }
        }
        else {
            //完成afterPropertiesSet接口方法调用
            ((InitializingBean) bean).afterPropertiesSet();
        }
    }

    //条件成立：
    if (mbd != null && bean.getClass() != NullBean.class) {
        //bd中获取init-method指定方法名称
        String initMethodName = mbd.getInitMethodName();

        if (StringUtils.hasLength(initMethodName) &&
                //通过接口实现了初始化方法，isInitializingBean就会是true，取反得到false，直接退出
                !(isInitializingBean && "afterPropertiesSet".equals(initMethodName)) &&
                !mbd.isExternallyManagedInitMethod(initMethodName)) {
            //反射执行初始化方法
            invokeCustomInitMethod(beanName, bean, mbd);
        }
    }
}
```



执行初始化方法：  
两种方式，要么是实现InitializingBean接口，要么是指定init-method，两个都有那么执行接口的方法

```java
protected void invokeInitMethods(String beanName, Object bean, @Nullable RootBeanDefinition mbd)
			throws Throwable {
		//执行初始化方法有两种方式，一种是实现了InitializingBean，一种是xml中的bean标签指定了init-method属性

		boolean isInitializingBean = (bean instanceof InitializingBean);
		//条件成立：说明实现了InitializingBean接口
		if (isInitializingBean && (mbd == null || !mbd.isExternallyManagedInitMethod("afterPropertiesSet"))) {
			if (logger.isTraceEnabled()) {
				logger.trace("Invoking afterPropertiesSet() on bean with name '" + beanName + "'");
			}
			if (System.getSecurityManager() != null) {
				try {
					AccessController.doPrivileged((PrivilegedExceptionAction<Object>) () -> {
						((InitializingBean) bean).afterPropertiesSet();
						return null;
					}, getAccessControlContext());
				}
				catch (PrivilegedActionException pae) {
					throw pae.getException();
				}
			}
			else {
				//完成afterPropertiesSet接口方法调用
				((InitializingBean) bean).afterPropertiesSet();
			}
		}

		//条件成立：
		if (mbd != null && bean.getClass() != NullBean.class) {
			//bd中获取init-method指定方法名称
			String initMethodName = mbd.getInitMethodName();

			if (StringUtils.hasLength(initMethodName) &&
					//通过接口实现了初始化方法，isInitializingBean就会是true，取反得到false，直接退出
					!(isInitializingBean && "afterPropertiesSet".equals(initMethodName)) &&
					!mbd.isExternallyManagedInitMethod(initMethodName)) {
				//反射执行初始化方法
				invokeCustomInitMethod(beanName, bean, mbd);
			}
		}
	}

```



#### 6-8、注册销毁回调

```java
/判断当前bean是否需要注册销毁回调，即容器销毁的时候，会执行bean的销毁方法
	registerDisposableBeanIfNecessary(beanName, bean, mbd);
```

```java
	protected void registerDisposableBeanIfNecessary(String beanName, Object bean, RootBeanDefinition mbd) {
		AccessControlContext acc = (System.getSecurityManager() != null ? getAccessControlContext() : null);
		//条件1：原型不会注册销毁回调
		//条件2：判断是否需要注册销毁回调
		if (!mbd.isPrototype() && requiresDestruction(bean, mbd)) {
			if (mbd.isSingleton()) {
				// Register a DisposableBean implementation that performs all destruction
				// work for the given bean: DestructionAwareBeanPostProcessors,
				// DisposableBean interface, custom destroy method.
				//给当前单实例注册回调adapter适配器。适配器内根据当前的bean是继承接口还是通过bd来决定是通过调用哪个方法来完成销毁
				registerDisposableBean(beanName,
						new DisposableBeanAdapter(bean, beanName, mbd, getBeanPostProcessors(), acc));
			}
			else {
				// A bean with a custom scope...
				Scope scope = this.scopes.get(mbd.getScope());
				if (scope == null) {
					throw new IllegalStateException("No Scope registered for scope name '" + mbd.getScope() + "'");
				}
				scope.registerDestructionCallback(beanName,
						new DisposableBeanAdapter(bean, beanName, mbd, getBeanPostProcessors(), acc));
			}
		}
	}

```

 

### 7、doCreateBean总结

#### 7-1、整体的createBean流程：

1. 执行beforeInstantiation后置处理器，进入doCreateBean逻辑 

2. createBeanInstance创建出来早期bean（这里最复杂的是匹配最优构造器，知道就行，不要深入） 

3. 将注解信息（@Autowired，@Value，@Inject注解信息到缓存）缓存到AutowiredAnnotationBeanPostProcessor中的Map<String, InjectionMetadata> injectionMetadataCache中 

4. 暴露早期bean到第3级缓存 

5. 注入依赖信息 
   
   1. 注入依赖信息中，有一个核心逻辑就是将AutowiredAnnotationBeanPostProcessor缓存的注解元信息转为PropertyValue合并到bd中的PropertyValues中。最后才是执行真正的注入逻辑applyPropertyValues 

6. 初始化bean实例：
   
   1. aware接口对应方法的参数注入；
   
   2. BeforeInitialization后置处理器调用，主要还是和aware接口相关；
   
   3. 执行init方法；
   
   4. AfterInitialization后置处理器调用，与AOP有关 
   
   5. 注册销毁回调 

7. 返回bean

#### 7-2、各个后置处理器大致出现的位置及其作用：

1. 一个是合并bd阶段调用的AutowiredAnnotationBeanPostProcessor后置处理器，扫描类继承体系上的注解元数据，并且缓存下来

2. 第二个是依赖注入阶段，需要将AutowiredAnnotationBeanPostProcessor后置处理器中的注解元数据转换为propertyValue存放到propertyValues中

3. 第三个bean初始化阶段，BeforeInitialization初始化之前的后处理器调用点，主要是aware接口参数实例的注入

4. 第四个bean初始化阶段，AfterInitialization初始化之后的后处理器调用点，主要是AOP逻辑，后面再来分析
