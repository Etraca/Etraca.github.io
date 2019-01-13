---
layout:     post
title:      Spring IOC 容器源码分析 - 创建单例 bean 的过程
subtitle:   Spring IOC 容器源码分析 - 创建单例 bean 的过程
date:       2019-01-14
author:     WPF
header-img: img/post-bg-cook.jpg
catalog: true
tags:
- Spring
- IOC
- java
---

## 简介
在上一篇文章中，我比较详细的分析了获取 bean 的方法，也就是getBean(String)的实现逻辑。对于已实例化好的单例 bean，getBean(String) 方法并不会再一次去创建，而是从缓存中获取。如果某个 bean 还未实例化，这个时候就无法命中缓存。此时，就要根据 bean 的配置信息去创建这个 bean 了。相较于getBean(String)方法的实现逻辑，创建 bean 的方法createBean(String, RootBeanDefinition, Object[])及其所调用的方法逻辑上更为复杂一些。关于创建 bean 实例的过程，我将会分几篇文章进行分析。本篇文章会先从大体上分析 createBean(String, RootBeanDefinition, Object[])方法的代码逻辑，至于其所调用的方法将会在随后的文章中进行分析。

## 源码分析
#### 创建 bean 实例的入口
在正式分析createBean(String, RootBeanDefinition, Object[])方法前，我们先来看看 createBean 方法是在哪里被调用的。如下：

```java
public T doGetBean(...) {
    // 省略不相关代码
    if (mbd.isSingleton()) {
        sharedInstance = getSingleton(beanName, new ObjectFactory<Object>() {
            @Override
            public Object getObject() throws BeansException {
                try {
                    return createBean(beanName, mbd, args);
                }
                catch (BeansException ex) {
                    destroySingleton(beanName);
                    throw ex;
                }
            }
        });
        bean = getObjectForBeanInstance(sharedInstance, name, beanName, mbd);
    }
    // 省略不相关代码
}
```

上面是 doGetBean 方法的代码片段，从中可以发现 createBean 方法。createBean 方法被匿名工厂类的 getObject 方法包裹，但这个匿名工厂类对象并未直接调用 getObject 方法。而是将自身作为参数传给了getSingleton(String, ObjectFactory)方法，那么我们接下来再去看看一下getSingleton(String, ObjectFactory) 方法的实现。如下：

```java
public Object getSingleton(String beanName, ObjectFactory<?> singletonFactory) {
    Assert.notNull(beanName, "'beanName' must not be null");
    synchronized (this.singletonObjects) {
        // 从缓存中获取单例 bean，若不为空，则直接返回，不用再初始化
        Object singletonObject = this.singletonObjects.get(beanName);
        if (singletonObject == null) {
            if (this.singletonsCurrentlyInDestruction) {
                throw new BeanCreationNotAllowedException(beanName,
                        "Singleton bean creation not allowed while singletons of this factory are in destruction " +
                        "(Do not request a bean from a BeanFactory in a destroy method implementation!)");
            }
            if (logger.isDebugEnabled()) {
                logger.debug("Creating shared instance of singleton bean '" + beanName + "'");
            }
            /* 
             * 将 beanName 添加到 singletonsCurrentlyInCreation 集合中，
             * 用于表明 beanName 对应的 bean 正在创建中
             */
            beforeSingletonCreation(beanName);
            boolean newSingleton = false;
            boolean recordSuppressedExceptions = (this.suppressedExceptions == null);
            if (recordSuppressedExceptions) {
                this.suppressedExceptions = new LinkedHashSet<Exception>();
            }
            try {
                // 通过 getObject 方法调用 createBean 方法创建 bean 实例
                singletonObject = singletonFactory.getObject();
                newSingleton = true;
            }
            catch (IllegalStateException ex) {
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
                // 将 beanName 从 singletonsCurrentlyInCreation 移除
                afterSingletonCreation(beanName);
            }
            if (newSingleton) {
                /* 
                 * 将 <beanName, singletonObject> 键值对添加到 singletonObjects 集合中，
                 * 并从其他集合（比如 earlySingletonObjects）中移除 singletonObject 记录
                 */
                addSingleton(beanName, singletonObject);
            }
        }
        return (singletonObject != NULL_OBJECT ? singletonObject : null);
    }
}
```

上面的方法逻辑不是很复杂，这里简单总结一下。如下：
1. 先从 singletonObjects 集合获取 bean 实例，若不为空，则直接返回
2. 若为空，进入创建 bean 实例阶段。先将 beanName 添加到 singletonsCurrentlyInCreation
3. 通过 getObject 方法调用 createBean 方法创建 bean 实例
4. 将 beanName 从 singletonsCurrentlyInCreation 集合中移除
5. 将 <beanName, singletonObject> 映射缓存到 singletonObjects 集合中

从上面的分析中，我们知道了 createBean 方法在何处被调用的。那么接下来我们一起深入 createBean 方法的源码中，来看看这个方法具体都做了什么事情。

## createBean 方法全貌
createBean 和 getBean 方法类似，基本上都是空壳方法，只不过 createBean 的逻辑稍微多点，多做了一些事情。下面我们一起看看这个方法的实现逻辑，如下：

```java
protected Object createBean(String beanName, RootBeanDefinition mbd, Object[] args) throws BeanCreationException {
    if (logger.isDebugEnabled()) {
        logger.debug("Creating instance of bean '" + beanName + "'");
    }
    RootBeanDefinition mbdToUse = mbd;

    // 解析 bean 的类型
    Class<?> resolvedClass = resolveBeanClass(mbd, beanName);
    if (resolvedClass != null && !mbd.hasBeanClass() && mbd.getBeanClassName() != null) {
        mbdToUse = new RootBeanDefinition(mbd);
        mbdToUse.setBeanClass(resolvedClass);
    }

    try {
        // 处理 lookup-method 和 replace-method 配置，Spring 将这两个配置统称为 override method
        mbdToUse.prepareMethodOverrides();
    }
    catch (BeanDefinitionValidationException ex) {
        throw new BeanDefinitionStoreException(mbdToUse.getResourceDescription(),
                beanName, "Validation of method overrides failed", ex);
    }

    try {
        // 在 bean 初始化前应用后置处理，如果后置处理返回的 bean 不为空，则直接返回
        Object bean = resolveBeforeInstantiation(beanName, mbdToUse);
        if (bean != null) {
            return bean;
        }
    }
    catch (Throwable ex) {
        throw new BeanCreationException(mbdToUse.getResourceDescription(), beanName,
                "BeanPostProcessor before instantiation of bean failed", ex);
    }

    // 调用 doCreateBean 创建 bean
    Object beanInstance = doCreateBean(beanName, mbdToUse, args);
    if (logger.isDebugEnabled()) {
        logger.debug("Finished creating instance of bean '" + beanName + "'");
    }
    return beanInstance;
}
```

上面的代码不长，代码的执行流程比较容易看出，这里罗列一下：
1. 解析 bean 类型
2. 处理 lookup-method 和 replace-method 配置
3. 在 bean 初始化前应用后置处理，若后置处理返回的 bean 不为空，则直接返回
4. 若上一步后置处理返回的 bean 为空，则调用 doCreateBean 创建 bean 实例

###### 验证和准备 override 方法
当用户配置了 lookup-method 和 replace-method 时，Spring 需要对目标 bean 进行增强。在增强之前，需要做一些准备工作，也就是 prepareMethodOverrides 中的逻辑。下面来看看这个方法的源码：

```java
public void prepareMethodOverrides() throws BeanDefinitionValidationException {
    MethodOverrides methodOverrides = getMethodOverrides();
    if (!methodOverrides.isEmpty()) {
        Set<MethodOverride> overrides = methodOverrides.getOverrides();
        synchronized (overrides) {
            // 循环处理每个 MethodOverride 对象
            for (MethodOverride mo : overrides) {
                prepareMethodOverride(mo);
            }
        }
    }
}

protected void prepareMethodOverride(MethodOverride mo) throws BeanDefinitionValidationException {
    // 获取方法名为 mo.getMethodName() 的方法数量，当方法重载时，count 的值就会大于1
    int count = ClassUtils.getMethodCountForName(getBeanClass(), mo.getMethodName());
    // count = 0，表明根据方法名未找到相应的方法，此时抛出异常
    if (count == 0) {
        throw new BeanDefinitionValidationException(
                "Invalid method override: no method with name '" + mo.getMethodName() +
                "' on class [" + getBeanClassName() + "]");
    }
    // 若 count = 1，表明仅存在已方法名为 mo.getMethodName()，这意味着方法不存在重载
    else if (count == 1) {
        // 方法不存在重载，则将 overloaded 成员变量设为 false
        mo.setOverloaded(false);
    }
}
```

上面的源码中，prepareMethodOverrides方法循环调用了prepareMethodOverride方法，并没其他的太多逻辑。主要准备工作都是在 prepareMethodOverride 方法中进行的，所以我们重点关注一下这个方法。prepareMethodOverride 这个方法主要用于获取指定方法的方法数量 count，并根据 count 的值进行相应的处理。count = 0 时，表明方法不存在，此时抛出异常。count = 1 时，设置 MethodOverride 对象的overloaded成员变量为 false。这样做的目的在于，提前标注名称mo.getMethodName()的方法不存在重载，在使用 CGLIB 增强阶段就不需要进行校验，直接找到某个方法进行增强即可。

###### bean 实例化前的后置处理
后置处理是 Spring 的一个拓展点，用户通过实现 BeanPostProcessor 接口，并将实现类配置到 Spring 的配置文件中（或者使用注解），即可在 bean 初始化前后进行自定义操作。关于后置处理较为详细的说明，可以参考我的了一篇文章Spring IOC 容器源码分析系列文章导读，这里就不赘述了。下面我们来看看 createBean 方法中的后置处理逻辑，如下：

```java
protected Object resolveBeforeInstantiation(String beanName, RootBeanDefinition mbd) {
    Object bean = null;
    // 检测是否解析过，mbd.beforeInstantiationResolved 的值在下面的代码中会被设置
    if (!Boolean.FALSE.equals(mbd.beforeInstantiationResolved)) {
        if (!mbd.isSynthetic() && hasInstantiationAwareBeanPostProcessors()) {
            Class<?> targetType = determineTargetType(beanName, mbd);
            if (targetType != null) {
                // 应用前置处理
                bean = applyBeanPostProcessorsBeforeInstantiation(targetType, beanName);
                if (bean != null) {
                    // 应用后置处理
                    bean = applyBeanPostProcessorsAfterInitialization(bean, beanName);
                }
            }
        }
        // 设置 mbd.beforeInstantiationResolved
        mbd.beforeInstantiationResolved = (bean != null);
    }
    return bean;
}

protected Object applyBeanPostProcessorsBeforeInstantiation(Class<?> beanClass, String beanName) {
    for (BeanPostProcessor bp : getBeanPostProcessors()) {
        // InstantiationAwareBeanPostProcessor 一般在 Spring 框架内部使用，不建议用户直接使用
        if (bp instanceof InstantiationAwareBeanPostProcessor) {
            InstantiationAwareBeanPostProcessor ibp = (InstantiationAwareBeanPostProcessor) bp;
            // bean 初始化前置处理
            Object result = ibp.postProcessBeforeInstantiation(beanClass, beanName);
            if (result != null) {
                return result;
            }
        }
    }
    return null;
}

public Object applyBeanPostProcessorsAfterInitialization(Object existingBean, String beanName)
        throws BeansException {

    Object result = existingBean;
    for (BeanPostProcessor beanProcessor : getBeanPostProcessors()) {
       // bean 初始化后置处理
        result = beanProcessor.postProcessAfterInitialization(result, beanName);
        if (result == null) {
            return result;
        }
    }
    return result;
}
```
在 resolveBeforeInstantiation 方法中，当前置处理方法返回的 bean 不为空时，后置处理才会被执行。前置处理器是 InstantiationAwareBeanPostProcessor 类型的，该种类型的处理器一般用在 Spring 框架内部，比如 AOP 模块中的AbstractAutoProxyCreator抽象类间接实现了这个接口中的 postProcessBeforeInstantiation方法，所以 AOP 可以在这个方法中生成为目标类的代理对象。不过我在调试的过程中，发现 AOP 在此处生成代理对象是有条件的。一般情况下条件都不成立，也就不会在此处生成代理对象。至于这个条件为什么不成立，因 AOP 这一块的源码我还没来得及看，所以暂时还无法解答。等我看过 AOP 模块的源码后，我再来尝试分析这个条件。

###### 调用 doCreateBean 方法创建 bean
这一节，我们来分析一下doCreateBean方法的源码。在 Spring 中，做事情的方法基本上都是以do开头的，doCreateBean 也不例外。那下面我们就来看看这个方法都做了哪些事情。

```java
protected Object doCreateBean(final String beanName, final RootBeanDefinition mbd, final Object[] args)
        throws BeanCreationException {

    /* 
     * BeanWrapper 是一个基础接口，由接口名可看出这个接口的实现类用于包裹 bean 实例。
     * 通过 BeanWrapper 的实现类可以方便的设置/获取 bean 实例的属性
     */
    BeanWrapper instanceWrapper = null;
    if (mbd.isSingleton()) {
        // 从缓存中获取 BeanWrapper，并清理相关记录
        instanceWrapper = this.factoryBeanInstanceCache.remove(beanName);
    }
    if (instanceWrapper == null) {
        /* 
         * 创建 bean 实例，并将实例包裹在 BeanWrapper 实现类对象中返回。createBeanInstance 
         * 中包含三种创建 bean 实例的方式：
         *   1. 通过工厂方法创建 bean 实例
         *   2. 通过构造方法自动注入（autowire by constructor）的方式创建 bean 实例
         *   3. 通过无参构造方法方法创建 bean 实例
         *
         * 若 bean 的配置信息中配置了 lookup-method 和 replace-method，则会使用 CGLIB 
         * 增强 bean 实例。关于这个方法，后面会专门写一篇文章介绍，这里先说这么多。
         */
        instanceWrapper = createBeanInstance(beanName, mbd, args);
    }
    // 此处的 bean 可以认为是一个原始的 bean 实例，暂未填充属性
    final Object bean = (instanceWrapper != null ? instanceWrapper.getWrappedInstance() : null);
    Class<?> beanType = (instanceWrapper != null ? instanceWrapper.getWrappedClass() : null);
    mbd.resolvedTargetType = beanType;

    // 这里又遇到后置处理了，此处的后置处理是用于处理已“合并的 BeanDefinition”。关于这种后置处理器具体的实现细节就不深入理解了，大家有兴趣可以自己去看
    synchronized (mbd.postProcessingLock) {
        if (!mbd.postProcessed) {
            try {
                applyMergedBeanDefinitionPostProcessors(mbd, beanType, beanName);
            }
            catch (Throwable ex) {
                throw new BeanCreationException(mbd.getResourceDescription(), beanName,
                        "Post-processing of merged bean definition failed", ex);
            }
            mbd.postProcessed = true;
        }
    }

    /*
     * earlySingletonExposure 是一个重要的变量，这里要说明一下。该变量用于表示是否提前暴露
     * 单例 bean，用于解决循环依赖。earlySingletonExposure 由三个条件综合而成，如下：
     *   条件1：mbd.isSingleton() - 表示 bean 是否是单例类型
     *   条件2：allowCircularReferences - 是否允许循环依赖
     *   条件3：isSingletonCurrentlyInCreation(beanName) - 当前 bean 是否处于创建的状态中
     * 
     * earlySingletonExposure = 条件1 && 条件2 && 条件3 
     *                        = 单例 && 是否允许循环依赖 && 是否存于创建状态中。
     */
    boolean earlySingletonExposure = (mbd.isSingleton() && this.allowCircularReferences &&
            isSingletonCurrentlyInCreation(beanName));
    if (earlySingletonExposure) {
        if (logger.isDebugEnabled()) {
            logger.debug("Eagerly caching bean '" + beanName +
                    "' to allow for resolving potential circular references");
        }
        // 添加工厂对象到 singletonFactories 缓存中
        addSingletonFactory(beanName, new ObjectFactory<Object>() {
            @Override
            public Object getObject() throws BeansException {
                // 获取早期 bean 的引用，如果 bean 中的方法被 AOP 切点所匹配到，此时 AOP 相关逻辑会介入
                return getEarlyBeanReference(beanName, mbd, bean);
            }
        });
    }

    Object exposedObject = bean;
    try {
        // 向 bean 实例中填充属性，populateBean 方法也是一个很重要的方法，后面会专门写文章分析
        populateBean(beanName, mbd, instanceWrapper);
        if (exposedObject != null) {
            /*
             * 进行余下的初始化工作，详细如下：
             * 1. 判断 bean 是否实现了 BeanNameAware、BeanFactoryAware、
             *    BeanClassLoaderAware 等接口，并执行接口方法
             * 2. 应用 bean 初始化前置操作
             * 3. 如果 bean 实现了 InitializingBean 接口，则执行 afterPropertiesSet 
             *    方法。如果用户配置了 init-method，则调用相关方法执行自定义初始化逻辑
             * 4. 应用 bean 初始化后置操作
             * 
             * 另外，AOP 相关逻辑也会在该方法中织入切面逻辑，此时的 exposedObject 就变成了
             * 一个代理对象了
             */
            exposedObject = initializeBean(beanName, exposedObject, mbd);
        }
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
            // 若 initializeBean 方法未改变 exposedObject 的引用，则此处的条件为 true。
            if (exposedObject == bean) {
                exposedObject = earlySingletonReference;
            }
            // 下面的逻辑我也没完全搞懂，就不分析了。见谅。
            else if (!this.allowRawInjectionDespiteWrapping && hasDependentBean(beanName)) {
                String[] dependentBeans = getDependentBeans(beanName);
                Set<String> actualDependentBeans = new LinkedHashSet<String>(dependentBeans.length);
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
                            "'getBeanNamesOfType' with the 'allowEagerInit' flag turned off, for example.");
                }
            }
        }
    }

    try {
        // 注册销毁逻辑
        registerDisposableBeanIfNecessary(beanName, bean, mbd);
    }
    catch (BeanDefinitionValidationException ex) {
        throw new BeanCreationException(
                mbd.getResourceDescription(), beanName, "Invalid destruction signature", ex);
    }

    return exposedObject;
}
```

上面的注释比较多，分析的应该比较详细的。不过有一部分代码我暂时没看懂，就不分析了，见谅。下面我们来总结一下 doCreateBean 方法的执行流程吧，如下：
1. 从缓存中获取 BeanWrapper 实现类对象，并清理相关记录
2. 若未命中缓存，则创建 bean 实例，并将实例包裹在 BeanWrapper 实现类对象中返回
3. 应用 MergedBeanDefinitionPostProcessor 后置处理器相关逻辑
4. 根据条件决定是否提前暴露 bean 的早期引用（early reference），用于处理循环依赖问题
5. 调用 populateBean 方法向 bean 实例中填充属性
6. 调用 initializeBean 方法完成余下的初始化工作
7. 注册销毁逻辑

doCreateBean 方法的流程比较复杂，步骤略多。由此也可了解到创建一个 bean 还是很复杂的，这中间要做的事情繁多。比如填充属性、对 BeanPostProcessor 拓展点提供支持等。以上的步骤对应的方法具体是怎样实现的，本篇文章并不打算展开分析。在后续的文章中，我会单独写文章分析几个逻辑比较复杂的步骤。有兴趣的阅读的朋友可以稍微等待一下，相关文章本周会陆续进行更新。











