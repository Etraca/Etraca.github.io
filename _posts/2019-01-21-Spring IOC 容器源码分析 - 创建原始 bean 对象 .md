---
layout:     post
title:      Spring IOC 容器源码分析 - 创建原始 bean 对象
subtitle:   Spring IOC 容器源码分析 - 创建原始 bean 对象
date:       2019-01-21
author:     WPF
header-img: img/post-bg-cook.jpg
catalog: true
tags:
- spring IOC
- java
---

## 1.简介
本篇文章是上一篇文章（创建单例 bean 的过程）的延续。在上一篇文章中，我们从战略层面上领略了doCreateBean方法的全过程。本篇文章，我们就从战术的层面上，详细分析doCreateBean方法中的一个重要的调用，即createBeanInstance方法。在本篇文章中，你将看到三种不同的构造 bean 对象的方式。你也会了解到构造 bean 对象的两种策略。如果你对这些内容感兴趣，那么不妨继续往下读。我会在代码进行大量的注解，相信能帮助你理解代码逻辑。好了，其他的就不多说了，进入正题吧。

## 2.源码分析
#### 2.1 创建 bean 对象的过程
本节，我们一起来来分析一下本篇文章的主角createBeanInstance方法。按照惯例，我们还是先分析一下方法的大致脉络，然后我们再按照这个脉络去分析一些重要的调用。So. Let`s go → ↓

```java
protected BeanWrapper createBeanInstance(String beanName, RootBeanDefinition mbd, Object[] args) {
    Class<?> beanClass = resolveBeanClass(mbd, beanName);

    /*
     * 检测类的访问权限。默认情况下，对于非 public 的类，是允许访问的。
     * 若禁止访问，这里会抛出异常
     */
    if (beanClass != null && !Modifier.isPublic(beanClass.getModifiers()) && !mbd.isNonPublicAccessAllowed()) {
        throw new BeanCreationException(mbd.getResourceDescription(), beanName,
                "Bean class isn't public, and non-public access not allowed: " + beanClass.getName());
    }

    /*
     * 如果工厂方法不为空，则通过工厂方法构建 bean 对象。这种构建 bean 的方式
     * 就不深入分析了，有兴趣的朋友可以自己去看一下。
     */
    if (mbd.getFactoryMethodName() != null)  {
        // 通过“工厂方法”的方式构建 bean 对象
        return instantiateUsingFactoryMethod(beanName, mbd, args);
    }

    /*
     * 当多次构建同一个 bean 时，可以使用此处的快捷路径，即无需再次推断应该使用哪种方式构造实例，
     * 以提高效率。比如在多次构建同一个 prototype 类型的 bean 时，就可以走此处的捷径。
     * 这里的 resolved 和 mbd.constructorArgumentsResolved 将会在 bean 第一次实例
     * 化的过程中被设置，在后面的源码中会分析到，先继续往下看。
     */
    boolean resolved = false;
    boolean autowireNecessary = false;
    if (args == null) {
        synchronized (mbd.constructorArgumentLock) {
            if (mbd.resolvedConstructorOrFactoryMethod != null) {
                resolved = true;
                autowireNecessary = mbd.constructorArgumentsResolved;
            }
        }
    }
    if (resolved) {
        if (autowireNecessary) {
            // 通过“构造方法自动注入”的方式构造 bean 对象
            return autowireConstructor(beanName, mbd, null, null);
        }
        else {
            // 通过“默认构造方法”的方式构造 bean 对象
            return instantiateBean(beanName, mbd);
        }
    }

    // 由后置处理器决定返回哪些构造方法，这里不深入分析了
    Constructor<?>[] ctors = determineConstructorsFromBeanPostProcessors(beanClass, beanName);
    /*
     * 下面的条件分支条件用于判断使用什么方式构造 bean 实例，有两种方式可选 - 构造方法自动
     * 注入和默认构造方法。判断的条件由4部分综合而成，如下：
     * 
     *    条件1：ctors != null -> 后置处理器返回构造方法数组是否为空
     *    
     *    条件2：mbd.getResolvedAutowireMode() == RootBeanDefinition.AUTOWIRE_CONSTRUCTOR 
     *              -> bean 配置中的 autowire 属性是否为 constructor    
     *    条件3：mbd.hasConstructorArgumentValues() 
     *              -> constructorArgumentValues 是否存在元素，即 bean 配置文件中
     *                 是否配置了 <construct-arg/>
     *    条件4：!ObjectUtils.isEmpty(args) 
     *              -> args 数组是否存在元素，args 是由用户调用 
     *                 getBean(String name, Object... args) 传入的
     * 
     * 上面4个条件，只要有一个为 true，就会通过构造方法自动注入的方式构造 bean 实例
     */
    if (ctors != null ||
            mbd.getResolvedAutowireMode() == RootBeanDefinition.AUTOWIRE_CONSTRUCTOR ||
            mbd.hasConstructorArgumentValues() || !ObjectUtils.isEmpty(args))  {
        // 通过“构造方法自动注入”的方式构造 bean 对象
        return autowireConstructor(beanName, mbd, ctors, args);
    }

    // 通过“默认构造方法”的方式构造 bean 对象
    return instantiateBean(beanName, mbd);
}
```

以上就是 createBeanInstance 方法的源码，不是很长。配合着注释，应该不是很难懂。下面我们来总结一下这个方法的执行流程，如下：
1. 检测类的访问权限，若禁止访问，则抛出异常
2. 若工厂方法不为空，则通过工厂方法构建 bean 对象，并返回结果
3. 若构造方式已解析过，则走快捷路径构建 bean 对象，并返回结果
4. 如第三步不满足，则通过组合条件决定使用哪种方式构建 bean 对象

这里有三种构造 bean 对象的方式，如下：
1. 通过“工厂方法”的方式构造 bean 对象
2. 通过“构造方法自动注入”的方式构造 bean 对象
3. 通过“默认构造方法”的方式构造 bean 对象

下面我将会分析第2和第3种构造 bean 对象方式的实现源码。至于第1种方式，实现逻辑和第2种方式较为相似。所以就不分析了，大家有兴趣可以自己看一下。

#### 2.2 通过构造方法自动注入的方式创建 bean 实例
本节，我将会分析构造方法自动注入的实现逻辑。代码逻辑较为复杂，需要大家耐心阅读。代码如下：

```java
protected BeanWrapper autowireConstructor(
        String beanName, RootBeanDefinition mbd, Constructor<?>[] ctors, Object[] explicitArgs) {

    // 创建 ConstructorResolver 对象，并调用其 autowireConstructor 方法
    return new ConstructorResolver(this).autowireConstructor(beanName, mbd, ctors, explicitArgs);
}

public BeanWrapper autowireConstructor(final String beanName, final RootBeanDefinition mbd,
        Constructor<?>[] chosenCtors, final Object[] explicitArgs) {

    // 创建 BeanWrapperImpl 对象
    BeanWrapperImpl bw = new BeanWrapperImpl();
    this.beanFactory.initBeanWrapper(bw);

    Constructor<?> constructorToUse = null;
    ArgumentsHolder argsHolderToUse = null;
    Object[] argsToUse = null;

    // 确定参数值列表（argsToUse）
    if (explicitArgs != null) {
        argsToUse = explicitArgs;
    }
    else {
        Object[] argsToResolve = null;
        synchronized (mbd.constructorArgumentLock) {
            // 获取已解析的构造方法
            constructorToUse = (Constructor<?>) mbd.resolvedConstructorOrFactoryMethod;
            if (constructorToUse != null && mbd.constructorArgumentsResolved) {
                // 获取已解析的构造方法参数列表
                argsToUse = mbd.resolvedConstructorArguments;
                if (argsToUse == null) {
                    // 若 argsToUse 为空，则获取未解析的构造方法参数列表
                    argsToResolve = mbd.preparedConstructorArguments;
                }
            }
        }
        if (argsToResolve != null) {
            // 解析参数列表
            argsToUse = resolvePreparedArguments(beanName, mbd, bw, constructorToUse, argsToResolve);
        }
    }

    if (constructorToUse == null) {
        boolean autowiring = (chosenCtors != null ||
                mbd.getResolvedAutowireMode() == RootBeanDefinition.AUTOWIRE_CONSTRUCTOR);
        ConstructorArgumentValues resolvedValues = null;

        int minNrOfArgs;
        if (explicitArgs != null) {
            minNrOfArgs = explicitArgs.length;
        }
        else {
            ConstructorArgumentValues cargs = mbd.getConstructorArgumentValues();
            resolvedValues = new ConstructorArgumentValues();
            /*
             * 确定构造方法参数数量，比如下面的配置：
             *     <bean id="persion" class="xyz.coolblog.autowire.Person">
             *         <constructor-arg index="0" value="xiaoming"/>
             *         <constructor-arg index="1" value="1"/>
             *         <constructor-arg index="2" value="man"/>
             *     </bean>
             *
             * 此时 minNrOfArgs = maxIndex + 1 = 2 + 1 = 3，除了计算 minNrOfArgs，
             * 下面的方法还会将 cargs 中的参数数据转存到 resolvedValues 中
             */
            minNrOfArgs = resolveConstructorArguments(beanName, mbd, bw, cargs, resolvedValues);
        }

        // 获取构造方法列表
        Constructor<?>[] candidates = chosenCtors;
        if (candidates == null) {
            Class<?> beanClass = mbd.getBeanClass();
            try {
                candidates = (mbd.isNonPublicAccessAllowed() ?
                        beanClass.getDeclaredConstructors() : beanClass.getConstructors());
            }
            catch (Throwable ex) {
                throw new BeanCreationException(mbd.getResourceDescription(), beanName,
                        "Resolution of declared constructors on bean Class [" + beanClass.getName() +
                        "] from ClassLoader [" + beanClass.getClassLoader() + "] failed", ex);
            }
        }

        // 按照构造方法的访问权限级别和参数数量进行排序
        AutowireUtils.sortConstructors(candidates);

        int minTypeDiffWeight = Integer.MAX_VALUE;
        Set<Constructor<?>> ambiguousConstructors = null;
        LinkedList<UnsatisfiedDependencyException> causes = null;

        for (Constructor<?> candidate : candidates) {
            Class<?>[] paramTypes = candidate.getParameterTypes();

            /*
             * 下面的 if 分支的用途是：若匹配到到合适的构造方法了，提前结束 for 循环
             * constructorToUse != null 这个条件比较好理解，下面分析一下条件 argsToUse.length > paramTypes.length：
             * 前面说到 AutowireUtils.sortConstructors(candidates) 用于对构造方法进行
             * 排序，排序规则如下：
             *   1. 具有 public 访问权限的构造方法排在非 public 构造方法前
             *   2. 参数数量多的构造方法排在前面
             *
             * 假设现在有一组构造方法按照上面的排序规则进行排序，排序结果如下（省略参数名称）：
             *
             *   1. public Hello(Object, Object, Object)
             *   2. public Hello(Object, Object)
             *   3. public Hello(Object)
             *   4. protected Hello(Integer, Object, Object, Object)
             *   5. protected Hello(Integer, Object, Object)
             *   6. protected Hello(Integer, Object)
             *
             * argsToUse = [num1, obj2]，可以匹配上的构造方法2和构造方法6。由于构造方法2有
             * 更高的访问权限，所以没理由不选他（尽管后者在参数类型上更加匹配）。由于构造方法3
             * 参数数量 < argsToUse.length，参数数量上不匹配，也不应该选。所以 
             * argsToUse.length > paramTypes.length 这个条件用途是：在条件 
             * constructorToUse != null 成立的情况下，通过判断参数数量与参数值数量
             * （argsToUse.length）是否一致，来决定是否提前终止构造方法匹配逻辑。
             */
            if (constructorToUse != null && argsToUse.length > paramTypes.length) {
                break;
            }

            /*
             * 构造方法参数数量低于配置的参数数量，则忽略当前构造方法，并重试。比如 
             * argsToUse = [obj1, obj2, obj3, obj4]，上面的构造方法列表中，
             * 构造方法1、2和3显然不是合适选择，忽略之。
             */
            if (paramTypes.length < minNrOfArgs) {
                continue;
            }

            ArgumentsHolder argsHolder;
            if (resolvedValues != null) {
                try {
                    /*
                     * 判断否则方法是否有 ConstructorProperties 注解，若有，则取注解中的
                     * 值。比如下面的代码：
                     * 
                     *  public class Persion {
                     *      private String name;
                     *      private Integer age;
                     *
                     *      @ConstructorProperties(value = {"coolblog", "20"})
                     *      public Persion(String name, Integer age) {
                     *          this.name = name;
                     *          this.age = age;
                     *      }
                     * }
                     */
                    String[] paramNames = ConstructorPropertiesChecker.evaluate(candidate, paramTypes.length);
                    if (paramNames == null) {
                        ParameterNameDiscoverer pnd = this.beanFactory.getParameterNameDiscoverer();
                        if (pnd != null) {
                            /*
                             * 获取构造方法参数名称列表，比如有这样一个构造方法:
                             *   public Person(String name, int age, String sex)
                             *   
                             * 调用 getParameterNames 方法返回 paramNames = [name, age, sex]
                             */
                            paramNames = pnd.getParameterNames(candidate);
                        }
                    }

                    /* 
                     * 创建参数值列表，返回 argsHolder 会包含进行类型转换后的参数值，比如下
                     * 面的配置:
                     *
                     *     <bean id="persion" class="xyz.coolblog.autowire.Person">
                     *         <constructor-arg name="name" value="xiaoming"/>
                     *         <constructor-arg name="age" value="1"/>
                     *         <constructor-arg name="sex" value="man"/>
                     *     </bean>
                     *
                     * Person 的成员变量 age 是 Integer 类型的，但由于在 Spring 配置中
                     * 只能配成 String 类型，所以这里要进行类型转换。
                     */
                    argsHolder = createArgumentArray(beanName, mbd, resolvedValues, bw, paramTypes, paramNames,
                            getUserDeclaredConstructor(candidate), autowiring);
                }
                catch (UnsatisfiedDependencyException ex) {
                    if (this.beanFactory.logger.isTraceEnabled()) {
                        this.beanFactory.logger.trace(
                                "Ignoring constructor [" + candidate + "] of bean '" + beanName + "': " + ex);
                    }
                    if (causes == null) {
                        causes = new LinkedList<UnsatisfiedDependencyException>();
                    }
                    causes.add(ex);
                    continue;
                }
            }
            else {
                if (paramTypes.length != explicitArgs.length) {
                    continue;
                }
                argsHolder = new ArgumentsHolder(explicitArgs);
            }

            /*
             * 计算参数值（argsHolder.arguments）每个参数类型与构造方法参数列表
             * （paramTypes）中参数的类型差异量，差异量越大表明参数类型差异越大。参数类型差异
             * 越大，表明当前构造方法并不是一个最合适的候选项。引入差异量（typeDiffWeight）
             * 变量目的：是将候选构造方法的参数列表类型与参数值列表类型的差异进行量化，通过量化
             * 后的数值筛选出最合适的构造方法。
             * 
             * 讲完差异量，再来说说 mbd.isLenientConstructorResolution() 条件。
             * 官方的解释是：返回构造方法的解析模式，有宽松模式（lenient mode）和严格模式
             * （strict mode）两种类型可选。具体的细节没去研究，就不多说了。
             */
            int typeDiffWeight = (mbd.isLenientConstructorResolution() ?
                    argsHolder.getTypeDifferenceWeight(paramTypes) : argsHolder.getAssignabilityWeight(paramTypes));
            if (typeDiffWeight < minTypeDiffWeight) {
                constructorToUse = candidate;
                argsHolderToUse = argsHolder;
                argsToUse = argsHolder.arguments;
                minTypeDiffWeight = typeDiffWeight;
                ambiguousConstructors = null;
            }
            /* 
             * 如果两个构造方法与参数值类型列表之间的差异量一致，那么这两个方法都可以作为
             * 候选项，这个时候就出现歧义了，这里先把有歧义的构造方法放入 
             * ambiguousConstructors 集合中
             */
            else if (constructorToUse != null && typeDiffWeight == minTypeDiffWeight) {
                if (ambiguousConstructors == null) {
                    ambiguousConstructors = new LinkedHashSet<Constructor<?>>();
                    ambiguousConstructors.add(constructorToUse);
                }
                ambiguousConstructors.add(candidate);
            }
        }

        // 若上面未能筛选出合适的构造方法，这里将抛出 BeanCreationException 异常
        if (constructorToUse == null) {
            if (causes != null) {
                UnsatisfiedDependencyException ex = causes.removeLast();
                for (Exception cause : causes) {
                    this.beanFactory.onSuppressedException(cause);
                }
                throw ex;
            }
            throw new BeanCreationException(mbd.getResourceDescription(), beanName,
                    "Could not resolve matching constructor " +
                    "(hint: specify index/type/name arguments for simple parameters to avoid type ambiguities)");
        }
        /*
         * 如果 constructorToUse != null，且 ambiguousConstructors 也不为空，表明解析
         * 出了多个的合适的构造方法，此时就出现歧义了。Spring 不会擅自决定使用哪个构造方法，
         * 所以抛出异常。
         */
        else if (ambiguousConstructors != null && !mbd.isLenientConstructorResolution()) {
            throw new BeanCreationException(mbd.getResourceDescription(), beanName,
                    "Ambiguous constructor matches found in bean '" + beanName + "' " +
                    "(hint: specify index/type/name arguments for simple parameters to avoid type ambiguities): " +
                    ambiguousConstructors);
        }

        if (explicitArgs == null) {
            /*
             * 缓存相关信息，比如：
             *   1. 已解析出的构造方法对象 resolvedConstructorOrFactoryMethod
             *   2. 构造方法参数列表是否已解析标志 constructorArgumentsResolved
             *   3. 参数值列表 resolvedConstructorArguments 或 preparedConstructorArguments
             *
             * 这些信息可用在其他地方，用于进行快捷判断
             */
            argsHolderToUse.storeCache(mbd, constructorToUse);
        }
    }

    try {
        Object beanInstance;

        if (System.getSecurityManager() != null) {
            final Constructor<?> ctorToUse = constructorToUse;
            final Object[] argumentsToUse = argsToUse;
            beanInstance = AccessController.doPrivileged(new PrivilegedAction<Object>() {
                @Override
                public Object run() {
                    return beanFactory.getInstantiationStrategy().instantiate(
                            mbd, beanName, beanFactory, ctorToUse, argumentsToUse);
                }
            }, beanFactory.getAccessControlContext());
        }
        else {
            /*
             * 调用实例化策略创建实例，默认情况下使用反射创建实例。如果 bean 的配置信息中
             * 包含 lookup-method 和 replace-method，则通过 CGLIB 增强 bean 实例
             */
            beanInstance = this.beanFactory.getInstantiationStrategy().instantiate(
                    mbd, beanName, this.beanFactory, constructorToUse, argsToUse);
        }

        // 设置 beanInstance 到 BeanWrapperImpl 对象中
        bw.setBeanInstance(beanInstance);
        return bw;
    }
    catch (Throwable ex) {
        throw new BeanCreationException(mbd.getResourceDescription(), beanName,
                "Bean instantiation via constructor failed", ex);
    }
}
```

上面的方法逻辑比较复杂，做了不少事情，该方法的核心逻辑是根据参数值类型筛选合适的构造方法。解析出合适的构造方法后，剩下的工作就是构建 bean 对象了，这个工作交给了实例化策略去做。下面罗列一下这个方法的工作流程吧：
1. 创建 BeanWrapperImpl 对象
2. 解析构造方法参数，并算出 minNrOfArgs
3. 获取构造方法列表，并排序
4. 遍历排序好的构造方法列表，筛选合适的构造方法
    - 获取构造方法参数列表中每个参数的名称
    - 再次解析参数，此次解析会将value 属性值进行类型转换，由 String 转为合适的类型。
    - 计算构造方法参数列表与参数值列表之间的类型差异量，以筛选出更为合适的构造方法
5. 缓存已筛选出的构造方法以及参数值列表，若再次创建 bean 实例时，可直接使用，无需再次进行筛选
6. 使用初始化策略创建 bean 对象
7. 将 bean 对象放入 BeanWrapperImpl 对象中，并返回该对象

由上面的流程可以看得出，通过构造方法自动注入的方式构造 bean 对象的过程还是很复杂的。为了看懂这个流程，我进行了多次调试，算是勉强弄懂大致逻辑。由于时间有限，我并未能详细分析 autowireConstructor 方法及其所调用的一些方法，比如 resolveConstructorArguments、 autowireConstructor 等。关于这些方法，这里只写了个大概，有兴趣的朋友自己去探索吧。

#### 2.3 通过默认构造方法创建 bean 对象
看完了上面冗长的逻辑，本节来看点轻松的吧 - 通过默认构造方法创建 bean 对象。如下：

```java
protected BeanWrapper instantiateBean(final String beanName, final RootBeanDefinition mbd) {
    try {
        Object beanInstance;
        final BeanFactory parent = this;
        // if 条件分支里的一大坨是 Java 安全相关的代码，可以忽略，直接看 else 分支
        if (System.getSecurityManager() != null) {
            beanInstance = AccessController.doPrivileged(new PrivilegedAction<Object>() {
                @Override
                public Object run() {
                    return getInstantiationStrategy().instantiate(mbd, beanName, parent);
                }
            }, getAccessControlContext());
        }
        else {
            /*
             * 调用实例化策略创建实例，默认情况下使用反射创建对象。如果 bean 的配置信息中
             * 包含 lookup-method 和 replace-method，则通过 CGLIB 创建 bean 对象
             */
            beanInstance = getInstantiationStrategy().instantiate(mbd, beanName, parent);
        }
        // 创建 BeanWrapperImpl 对象
        BeanWrapper bw = new BeanWrapperImpl(beanInstance);
        initBeanWrapper(bw);
        return bw;
    }
    catch (Throwable ex) {
        throw new BeanCreationException(
                mbd.getResourceDescription(), beanName, "Instantiation of bean failed", ex);
    }
}

public Object instantiate(RootBeanDefinition bd, String beanName, BeanFactory owner) {
    // 检测 bean 配置中是否配置了 lookup-method 或 replace-method，若配置了，则需使用 CGLIB 构建 bean 对象
    if (bd.getMethodOverrides().isEmpty()) {
        Constructor<?> constructorToUse;
        synchronized (bd.constructorArgumentLock) {
            constructorToUse = (Constructor<?>) bd.resolvedConstructorOrFactoryMethod;
            if (constructorToUse == null) {
                final Class<?> clazz = bd.getBeanClass();
                if (clazz.isInterface()) {
                    throw new BeanInstantiationException(clazz, "Specified class is an interface");
                }
                try {
                    if (System.getSecurityManager() != null) {
                        constructorToUse = AccessController.doPrivileged(new PrivilegedExceptionAction<Constructor<?>>() {
                            @Override
                            public Constructor<?> run() throws Exception {
                                return clazz.getDeclaredConstructor((Class[]) null);
                            }
                        });
                    }
                    else {
                        // 获取默认构造方法
                        constructorToUse = clazz.getDeclaredConstructor((Class[]) null);
                    }
                    // 设置 resolvedConstructorOrFactoryMethod
                    bd.resolvedConstructorOrFactoryMethod = constructorToUse;
                }
                catch (Throwable ex) {
                    throw new BeanInstantiationException(clazz, "No default constructor found", ex);
                }
            }
        }
        // 通过无参构造方法创建 bean 对象
        return BeanUtils.instantiateClass(constructorToUse);
    }
    else {
        // 使用 GCLIG 创建 bean 对象
        return instantiateWithMethodInjection(bd, beanName, owner);
    }
}
```

上面就是通过默认构造方法创建 bean 对象的过程，比较简单，就不多说了。最后我们再来看看简单看看通过无参构造方法刚创建 bean 对象的代码（通过 CGLIB 创建 bean 对象的方式就不看了）是怎样的，如下：

```java
public static <T> T instantiateClass(Constructor<T> ctor, Object... args) throws BeanInstantiationException {
    Assert.notNull(ctor, "Constructor must not be null");
    try {
        // 设置构造方法为可访问
        ReflectionUtils.makeAccessible(ctor);
        // 通过反射创建 bean 实例，这里的 args 是一个没有元素的空数组
        return ctor.newInstance(args);
    }
    catch (InstantiationException ex) {
        throw new BeanInstantiationException(ctor, "Is it an abstract class?", ex);
    }
    catch (IllegalAccessException ex) {
        throw new BeanInstantiationException(ctor, "Is the constructor accessible?", ex);
    }
    catch (IllegalArgumentException ex) {
        throw new BeanInstantiationException(ctor, "Illegal arguments for constructor", ex);
    }
    catch (InvocationTargetException ex) {
        throw new BeanInstantiationException(ctor, "Constructor threw exception", ex.getTargetException());
    }
}
```









