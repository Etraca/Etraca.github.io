---
layout:     post
title:      Spring IOC 容器源码分析 - 填充属性到 bean 原始对象
subtitle:   Spring IOC 容器源码分析 - 填充属性到 bean 原始对象
date:       2019-01-23
author:     WPF
header-img: img/post-bg-cook.jpg
catalog: true
tags:
- spring IOC
- java
---

## 1.简介
本篇文章，我们来一起了解一下 Spring 是如何将配置文件中的属性值填充到 bean 对象中的。我在前面几篇文章中介绍过 Spring 创建 bean 的流程，即 Spring 先通过反射创建一个原始的 bean 对象，然后再向这个原始的 bean 对象中填充属性。对于填充属性这个过程，简单点来说，JavaBean 的每个属性通常都有 getter/setter 方法，我们可以直接调用 setter 方法将属性值设置进去。当然，这样做还是太简单了，填充属性的过程中还有许多事情要做。比如在 Spring 配置中，所有属性值都是以字符串的形式进行配置的，我们在将这些属性值赋值给对象的成员变量时，要根据变量类型进行相应的类型转换。对于一些集合类的配置，比如、和，还要将这些配置转换成相应的集合对象才能进行后续的操作。除此之外，如果用户配置了自动注入（autowire = byName/byType），Spring 还要去为自动注入的属性寻找合适的注入项。由此可以见，属性填充的整个过程还是很复杂的，并非是简单调用 setter 方法设置属性值即可。

## 2. 源码分析
#### 2.1 populateBean 源码一览

```java
protected void populateBean(String beanName, RootBeanDefinition mbd, BeanWrapper bw) {
    // 获取属性列表
    PropertyValues pvs = mbd.getPropertyValues();

    if (bw == null) {
        if (!pvs.isEmpty()) {
            throw new BeanCreationException(
                    mbd.getResourceDescription(), beanName, "Cannot apply property values to null instance");
        }
        else {
            return;
        }
    }

    boolean continueWithPropertyPopulation = true;
    /*
     * 在属性被填充前，给 InstantiationAwareBeanPostProcessor 类型的后置处理器一个修改 
     * bean 状态的机会。关于这段后置引用，官方的解释是：让用户可以自定义属性注入。比如用户实现一
     * 个 InstantiationAwareBeanPostProcessor 类型的后置处理器，并通过 
     * postProcessAfterInstantiation 方法向 bean 的成员变量注入自定义的信息。当然，如果无
     * 特殊需求，直接使用配置中的信息注入即可。另外，Spring 并不建议大家直接实现 
     * InstantiationAwareBeanPostProcessor 接口，如果想实现这种类型的后置处理器，更建议
     * 通过继承 InstantiationAwareBeanPostProcessorAdapter 抽象类实现自定义后置处理器。
     */
    if (!mbd.isSynthetic() && hasInstantiationAwareBeanPostProcessors()) {
        for (BeanPostProcessor bp : getBeanPostProcessors()) {
            if (bp instanceof InstantiationAwareBeanPostProcessor) {
                InstantiationAwareBeanPostProcessor ibp = (InstantiationAwareBeanPostProcessor) bp;
                if (!ibp.postProcessAfterInstantiation(bw.getWrappedInstance(), beanName)) {
                    continueWithPropertyPopulation = false;
                    break;
                }
            }
        }
    }

    /* 
     * 如果上面设置 continueWithPropertyPopulation = false，表明用户可能已经自己填充了
     * bean 的属性，不需要 Spring 帮忙填充了。此时直接返回即可
     */
    if (!continueWithPropertyPopulation) {
        return;
    }

    // 根据名称或类型注入依赖
    if (mbd.getResolvedAutowireMode() == RootBeanDefinition.AUTOWIRE_BY_NAME ||
            mbd.getResolvedAutowireMode() == RootBeanDefinition.AUTOWIRE_BY_TYPE) {
        MutablePropertyValues newPvs = new MutablePropertyValues(pvs);

        // 通过属性名称注入依赖
        if (mbd.getResolvedAutowireMode() == RootBeanDefinition.AUTOWIRE_BY_NAME) {
            autowireByName(beanName, mbd, bw, newPvs);
        }

        // 通过属性类型注入依赖
        if (mbd.getResolvedAutowireMode() == RootBeanDefinition.AUTOWIRE_BY_TYPE) {
            autowireByType(beanName, mbd, bw, newPvs);
        }

        pvs = newPvs;
    }

    boolean hasInstAwareBpps = hasInstantiationAwareBeanPostProcessors();
    boolean needsDepCheck = (mbd.getDependencyCheck() != RootBeanDefinition.DEPENDENCY_CHECK_NONE);

    /*
     * 这里又是一种后置处理，用于在 Spring 填充属性到 bean 对象前，对属性的值进行相应的处理，
     * 比如可以修改某些属性的值。这时注入到 bean 中的值就不是配置文件中的内容了，
     * 而是经过后置处理器修改后的内容
     */ 
    if (hasInstAwareBpps || needsDepCheck) {
        PropertyDescriptor[] filteredPds = filterPropertyDescriptorsForDependencyCheck(bw, mbd.allowCaching);
        if (hasInstAwareBpps) {
            for (BeanPostProcessor bp : getBeanPostProcessors()) {
                if (bp instanceof InstantiationAwareBeanPostProcessor) {
                    InstantiationAwareBeanPostProcessor ibp = (InstantiationAwareBeanPostProcessor) bp;
                    // 对属性进行后置处理
                    pvs = ibp.postProcessPropertyValues(pvs, filteredPds, bw.getWrappedInstance(), beanName);
                    if (pvs == null) {
                        return;
                    }
                }
            }
        }
        if (needsDepCheck) {
            checkDependencies(beanName, mbd, filteredPds, pvs);
        }
    }

    // 应用属性值到 bean 对象中
    applyPropertyValues(beanName, mbd, bw, pvs);
}
```

上面的源码注释的比较详细了，下面我们来总结一下这个方法的执行流程。如下：
1. 获取属性列表 pvs
2. 在属性被填充到 bean 前，应用后置处理自定义属性填充
3. 根据名称或类型解析相关依赖
4. 再次应用后置处理，用于动态修改属性列表 pvs 的内容
5. 将属性应用到 bean 对象中

注意第3步，也就是根据名称或类型解析相关依赖（autowire）。该逻辑只会解析依赖，并不会将解析出的依赖立即注入到 bean 对象中。所有的属性值是在 applyPropertyValues 方法中统一被注入到 bean 对象中的。

在下面的章节中，我将会对 populateBean 方法中比较重要的几个方法调用进行分析，也就是第3步和第5步中的三个方法。好了，本节先到这里。

#### 2.2 autowireByName 方法分析

```java
protected void autowireByName(
        String beanName, AbstractBeanDefinition mbd, BeanWrapper bw, MutablePropertyValues pvs) {

    /*
     * 获取非简单类型属性的名称，且该属性未被配置在配置文件中。这里从反面解释一下什么是"非简单类型"
     * 属性，我们先来看看 Spring 认为的"简单类型"属性有哪些，如下：
     *   1. CharSequence 接口的实现类，比如 String
     *   2. Enum
     *   3. Date
     *   4. URI/URL
     *   5. Number 的继承类，比如 Integer/Long
     *   6. byte/short/int... 等基本类型
     *   7. Locale
     *   8. 以上所有类型的数组形式，比如 String[]、Date[]、int[] 等等
     * 
     * 除了要求非简单类型的属性外，还要求属性未在配置文件中配置过，也就是 pvs.contains(pd.getName()) = false。
     */
    String[] propertyNames = unsatisfiedNonSimpleProperties(mbd, bw);
    for (String propertyName : propertyNames) {
        // 检测是否存在与 propertyName 相关的 bean 或 BeanDefinition。若存在，则调用 BeanFactory.getBean 方法获取 bean 实例
        if (containsBean(propertyName)) {
            // 从容器中获取相应的 bean 实例
            Object bean = getBean(propertyName);
            // 将解析出的 bean 存入到属性值列表（pvs）中
            pvs.add(propertyName, bean);
            registerDependentBean(propertyName, beanName);
            if (logger.isDebugEnabled()) {
                logger.debug("Added autowiring by name from bean name '" + beanName +
                        "' via property '" + propertyName + "' to bean named '" + propertyName + "'");
            }
        }
        else {
            if (logger.isTraceEnabled()) {
                logger.trace("Not autowiring property '" + propertyName + "' of bean '" + beanName +
                        "' by name: no matching bean found");
            }
        }
    }
}
```

autowireByName 方法的逻辑比较简单，该方法首先获取非简单类型属性的名称，然后再根据名称到容器中获取相应的 bean 实例，最后再将获取到的 bean 添加到属性列表中即可。既然这个方法比较简单，那我也就不多说了，继续下面的分析。

#### 2.3 autowireByType 方法分析
本节我们来分析一下 autowireByName 的孪生兄弟 autowireByType，相较于 autowireByName，autowireByType 则要复杂一些，复杂之处在于解析依赖的过程。不过也没关系，如果我们不过于纠结细节，我们完全可以把一些复杂的地方当做一个黑盒，我们只需要要知道这个黑盒有什么用即可。这样可以在很大程度上降低源码分析的难度。好了，其他的就不多说了，咱们来分析源码吧。

```java
protected void autowireByType(
        String beanName, AbstractBeanDefinition mbd, BeanWrapper bw, MutablePropertyValues pvs) {

    TypeConverter converter = getCustomTypeConverter();
    if (converter == null) {
        converter = bw;
    }

    Set<String> autowiredBeanNames = new LinkedHashSet<String>(4);
    // 获取非简单类型的属性
    String[] propertyNames = unsatisfiedNonSimpleProperties(mbd, bw);
    for (String propertyName : propertyNames) {
        try {
            PropertyDescriptor pd = bw.getPropertyDescriptor(propertyName);
            // 如果属性类型为 Object，则忽略，不做解析
            if (Object.class != pd.getPropertyType()) {
                /*
                 * 获取 setter 方法（write method）的参数信息，比如参数在参数列表中的
                 * 位置，参数类型，以及该参数所归属的方法等信息
                 */
                MethodParameter methodParam = BeanUtils.getWriteMethodParameter(pd);

                // Do not allow eager init for type matching in case of a prioritized post-processor.
                boolean eager = !PriorityOrdered.class.isAssignableFrom(bw.getWrappedClass());
                // 创建依赖描述对象
                DependencyDescriptor desc = new AutowireByTypeDependencyDescriptor(methodParam, eager);
                /*
                 * 下面的方法用于解析依赖。过程比较复杂，先把这里看成一个黑盒，我们只要知道这
                 * 个方法可以帮我们解析出合适的依赖即可。
                 */
                Object autowiredArgument = resolveDependency(desc, beanName, autowiredBeanNames, converter);
                if (autowiredArgument != null) {
                    // 将解析出的 bean 存入到属性值列表（pvs）中
                    pvs.add(propertyName, autowiredArgument);
                }
                for (String autowiredBeanName : autowiredBeanNames) {
                    registerDependentBean(autowiredBeanName, beanName);
                    if (logger.isDebugEnabled()) {
                        logger.debug("Autowiring by type from bean name '" + beanName + "' via property '" +
                                propertyName + "' to bean named '" + autowiredBeanName + "'");
                    }
                }
                autowiredBeanNames.clear();
            }
        }
        catch (BeansException ex) {
            throw new UnsatisfiedDependencyException(mbd.getResourceDescription(), beanName, propertyName, ex);
        }
    }
}
```

如上所示，autowireByType 的代码本身并不复杂。和 autowireByName 一样，autowireByType 首先也是获取非简单类型属性的名称。然后再根据属性名获取属性描述符，并由属性描述符获取方法参数对象 MethodParameter，随后再根据 MethodParameter 对象获取依赖描述符对象，整个过程为 beanName → PropertyDescriptor → MethodParameter → DependencyDescriptor。在获取到依赖描述符对象后，再根据依赖描述符解析出合适的依赖。最后将解析出的结果存入属性列表 pvs 中即可。

关于 autowireByType 方法中出现的几种描述符对象，大家自己去看一下他们的实现吧，我就不分析了。接下来，我们来分析一下解析依赖的方法 resolveDependency。如下：

```java
public Object resolveDependency(DependencyDescriptor descriptor, String requestingBeanName,
        Set<String> autowiredBeanNames, TypeConverter typeConverter) throws BeansException {

    descriptor.initParameterNameDiscovery(getParameterNameDiscoverer());
    if (javaUtilOptionalClass == descriptor.getDependencyType()) {
        return new OptionalDependencyFactory().createOptionalDependency(descriptor, requestingBeanName);
    }
    else if (ObjectFactory.class == descriptor.getDependencyType() ||
            ObjectProvider.class == descriptor.getDependencyType()) {
        return new DependencyObjectProvider(descriptor, requestingBeanName);
    }
    else if (javaxInjectProviderClass == descriptor.getDependencyType()) {
        return new Jsr330ProviderFactory().createDependencyProvider(descriptor, requestingBeanName);
    }
    else {
        Object result = getAutowireCandidateResolver().getLazyResolutionProxyIfNecessary(
                descriptor, requestingBeanName);
        if (result == null) {
            // 解析依赖
            result = doResolveDependency(descriptor, requestingBeanName, autowiredBeanNames, typeConverter);
        }
        return result;
    }
}

public Object doResolveDependency(DependencyDescriptor descriptor, String beanName,
        Set<String> autowiredBeanNames, TypeConverter typeConverter) throws BeansException {

    InjectionPoint previousInjectionPoint = ConstructorResolver.setCurrentInjectionPoint(descriptor);
    try {
        // 该方法最终调用了 beanFactory.getBean(String, Class)，从容器中获取依赖
        Object shortcut = descriptor.resolveShortcut(this);
        // 如果容器中存在所需依赖，这里进行断路操作，提前结束依赖解析逻辑
        if (shortcut != null) {
            return shortcut;
        }

        Class<?> type = descriptor.getDependencyType();
        // 处理 @value 注解
        Object value = getAutowireCandidateResolver().getSuggestedValue(descriptor);
        if (value != null) {
            if (value instanceof String) {
                String strVal = resolveEmbeddedValue((String) value);
                BeanDefinition bd = (beanName != null && containsBean(beanName) ? getMergedBeanDefinition(beanName) : null);
                value = evaluateBeanDefinitionString(strVal, bd);
            }
            TypeConverter converter = (typeConverter != null ? typeConverter : getTypeConverter());
            return (descriptor.getField() != null ?
                    converter.convertIfNecessary(value, type, descriptor.getField()) :
                    converter.convertIfNecessary(value, type, descriptor.getMethodParameter()));
        }

        // 解析数组、list、map 等类型的依赖
        Object multipleBeans = resolveMultipleBeans(descriptor, beanName, autowiredBeanNames, typeConverter);
        if (multipleBeans != null) {
            return multipleBeans;
        }

        /*
         * 按类型查找候选列表，如果某个类型已经被实例化，则返回相应的实例。
         * 比如下面的配置：
         *
         *   <bean name="mongoDao" class="xyz.coolblog.autowire.MongoDao" primary="true"/>
         *   <bean name="service" class="xyz.coolblog.autowire.Service" autowire="byType"/>
         *   <bean name="mysqlDao" class="xyz.coolblog.autowire.MySqlDao"/>
         *
         * MongoDao 和 MySqlDao 均实现自 Dao 接口，Service 对象（不是接口）中有一个 Dao 
         * 类型的属性。现在根据类型自动注入 Dao 的实现类。这里有两个候选 bean，一个是 
         * mongoDao，另一个是 mysqlDao，其中 mongoDao 在 service 之前实例化，
         * mysqlDao 在 service 之后实例化。此时 findAutowireCandidates 方法会返回如下的结果：
         *
         *   matchingBeans = [ <mongoDao, Object@MongoDao>, <mysqlDao, Class@MySqlDao> ]
         *
         * 注意 mysqlDao 还未实例化，所以返回的是 MySqlDao.class。
         * 
         * findAutowireCandidates 这个方法逻辑比较复杂，我简单说一下它的工作流程吧，如下：
         *   1. 从 BeanFactory 中获取某种类型 bean 的名称，比如上面的配置中 
         *      mongoDao 和 mysqlDao 均实现了 Dao 接口，所以他们是同一种类型的 bean。
         *   2. 遍历上一步得到的名称列表，并判断 bean 名称对应的 bean 是否是合适的候选项，
         *      若合适则添加到候选列表中，并在最后返回候选列表
         *      
         * findAutowireCandidates 比较复杂，我并未完全搞懂，就不深入分析了。见谅
         */
        Map<String, Object> matchingBeans = findAutowireCandidates(beanName, type, descriptor);
        if (matchingBeans.isEmpty()) {
            if (isRequired(descriptor)) {
                // 抛出 NoSuchBeanDefinitionException 异常
                raiseNoMatchingBeanFound(type, descriptor.getResolvableType(), descriptor);
            }
            return null;
        }

        String autowiredBeanName;
        Object instanceCandidate;

        if (matchingBeans.size() > 1) {
            /*
             * matchingBeans.size() > 1，则表明存在多个可注入的候选项，这里判断使用哪一个
             * 候选项。比如下面的配置：
             *
             *   <bean name="mongoDao" class="xyz.coolblog.autowire.MongoDao" primary="true"/>
             *   <bean name="mysqlDao" class="xyz.coolblog.autowire.MySqlDao"/>
             *
             * mongoDao 的配置中存在 primary 属性，所以 mongoDao 会被选为最终的候选项。如
             * 果两个 bean 配置都没有 primary 属性，则需要根据优先级选择候选项。优先级这一块
             * 的逻辑没细看，不多说了。
             */
            autowiredBeanName = determineAutowireCandidate(matchingBeans, descriptor);
            if (autowiredBeanName == null) {
                if (isRequired(descriptor) || !indicatesMultipleBeans(type)) {
                    // 抛出 NoUniqueBeanDefinitionException 异常
                    return descriptor.resolveNotUnique(type, matchingBeans);
                }
                else {
                    return null;
                }
            }
            // 根据解析出的 autowiredBeanName，获取相应的候选项
            instanceCandidate = matchingBeans.get(autowiredBeanName);
        }
        else { // 只有一个候选项，直接取出来即可
            Map.Entry<String, Object> entry = matchingBeans.entrySet().iterator().next();
            autowiredBeanName = entry.getKey();
            instanceCandidate = entry.getValue();
        }

        if (autowiredBeanNames != null) {
            autowiredBeanNames.add(autowiredBeanName);
        }

        // 返回候选项实例，如果实例是 Class 类型，则调用 beanFactory.getBean(String, Class) 获取相应的 bean。否则直接返回即可
        return (instanceCandidate instanceof Class ?
                descriptor.resolveCandidate(autowiredBeanName, type, this) : instanceCandidate);
    }
    finally {
        ConstructorResolver.setCurrentInjectionPoint(previousInjectionPoint);
    }
}
```

由上面的代码可以看出，doResolveDependency 这个方法还是挺复杂的。这里我就不继续分析 doResolveDependency 所调用的方法了，对于这些方法，我也是似懂非懂。好了，本节的最后我们来总结一下 doResolveDependency 的执行流程吧，如下：
1. 首先将 beanName 和 requiredType 作为参数，并尝试从 BeanFactory 中获取与此对于的 bean。若获取成功，就可以提前结束 doResolveDependency 的逻辑。
2. 处理 @value 注解
3. 解析数组、List、Map 等类型的依赖，如果解析结果不为空，则返回结果
4. 根据类型查找合适的候选项
5. 如果候选项的数量为0，则抛出异常。为1，直接从候选列表中取出即可。若候选项数量 > 1，则在多个候选项中确定最优候选项，若无法确定则抛出异常
6. 若候选项是 Class 类型，表明候选项还没实例化，此时通过 BeanFactory.getBean 方法对其进行实例化。若候选项是非 Class 类型，则表明已经完成了实例化，此时直接返回即可。

#### 2.4 applyPropertyValues 方法分析
经过了上面的流程，现在终于可以将属性值注入到 bean 对象中了。当然，这里还不能立即将属性值注入到对象中，因为在 Spring 配置文件中属性值都是以 String 类型进行配置的，所以 Spring 框架需要对 String 类型进行转换。除此之外，对于 ref 属性，这里还需要根据 ref 属性值解析依赖。还有一些其他操作，这里就不多说了，更多的信息我们一起在源码探寻。

```java
protected void applyPropertyValues(String beanName, BeanDefinition mbd, BeanWrapper bw, PropertyValues pvs) {
    if (pvs == null || pvs.isEmpty()) {
        return;
    }

    if (System.getSecurityManager() != null && bw instanceof BeanWrapperImpl) {
        ((BeanWrapperImpl) bw).setSecurityContext(getAccessControlContext());
    }

    MutablePropertyValues mpvs = null;
    List<PropertyValue> original;

    if (pvs instanceof MutablePropertyValues) {
        mpvs = (MutablePropertyValues) pvs;
        // 如果属性列表 pvs 被转换过，则直接返回即可
        if (mpvs.isConverted()) {
            try {
                bw.setPropertyValues(mpvs);
                return;
            }
            catch (BeansException ex) {
                throw new BeanCreationException(
                        mbd.getResourceDescription(), beanName, "Error setting property values", ex);
            }
        }
        original = mpvs.getPropertyValueList();
    }
    else {
        original = Arrays.asList(pvs.getPropertyValues());
    }

    TypeConverter converter = getCustomTypeConverter();
    if (converter == null) {
        converter = bw;
    }
    BeanDefinitionValueResolver valueResolver = new BeanDefinitionValueResolver(this, beanName, mbd, converter);

    List<PropertyValue> deepCopy = new ArrayList<PropertyValue>(original.size());
    boolean resolveNecessary = false;
    // 遍历属性列表
    for (PropertyValue pv : original) {
        // 如果属性值被转换过，则就不需要再次转换
        if (pv.isConverted()) {
            deepCopy.add(pv);
        }
        else {
            String propertyName = pv.getName();
            Object originalValue = pv.getValue();
            /*
             * 解析属性值。举例说明，先看下面的配置：
             * 
             *   <bean id="macbook" class="MacBookPro">
             *       <property name="manufacturer" value="Apple"/>
             *       <property name="width" value="280"/>
             *       <property name="cpu" ref="cpu"/>
             *       <property name="interface">
             *           <list>
             *               <value>USB</value>
             *               <value>HDMI</value>
             *               <value>Thunderbolt</value>
             *           </list>
             *       </property>
             *   </bean>
             *
             * 上面是一款电脑的配置信息，每个 property 配置经过下面的方法解析后，返回如下结果：
             *   propertyName = "manufacturer", resolvedValue = "Apple"
             *   propertyName = "width", resolvedValue = "280"
             *   propertyName = "cpu", resolvedValue = "CPU@1234"  注：resolvedValue 是一个对象
             *   propertyName = "interface", resolvedValue = ["USB", "HDMI", "Thunderbolt"]
             *
             * 如上所示，resolveValueIfNecessary 会将 ref 解析为具体的对象，将 <list> 
             * 标签转换为 List 对象等。对于 int 类型的配置，这里并未做转换，所以 
             * width = "280"，还是字符串。除了解析上面几种类型，该方法还会解析 <set/>、
             * <map/>、<array/> 等集合配置
             */
            Object resolvedValue = valueResolver.resolveValueIfNecessary(pv, originalValue);
            Object convertedValue = resolvedValue;

            /*
             * convertible 表示属性值是否可转换，由两个条件合成而来。第一个条件不难理解，解释
             * 一下第二个条件。第二个条件用于检测 propertyName 是否是 nested 或者 indexed，
             * 直接举例说明吧：
             * 
             *   public class Room {
             *       private Door door = new Door();
             *   }
             *
             * room 对象里面包含了 door 对象，如果我们想向 door 对象中注入属性值，则可以这样配置：
             *
             *   <bean id="room" class="xyz.coolblog.Room">
             *      <property name="door.width" value="123"/>
             *   </bean>
             * 
             * isNestedOrIndexedProperty 会根据 propertyName 中是否包含 . 或 [  返回 
             * true 和 false。包含则返回 true，否则返回 false。关于 nested 类型的属性，我
             * 没在实践中用过，所以不知道上面举的例子是不是合理。若不合理，欢迎指正，也请多多指教。
             * 关于 nested 类型的属性，大家还可以参考 Spring 的官方文档：
             *     https://docs.spring.io/spring/docs/4.3.17.RELEASE/spring-framework-reference/htmlsingle/#beans-beans-conventions
             */
            boolean convertible = bw.isWritableProperty(propertyName) &&
                    !PropertyAccessorUtils.isNestedOrIndexedProperty(propertyName);
            // 对于一般的属性，convertible 通常为 true
            if (convertible) {
                // 对属性值的类型进行转换，比如将 String 类型的属性值 "123" 转为 Integer 类型的 123
                convertedValue = convertForProperty(resolvedValue, propertyName, bw, converter);
            }

            /*
             * 如果 originalValue 是通过 autowireByType 或 autowireByName 解析而来，
             * 那么此处条件成立，即 (resolvedValue == originalValue) = true
             */
            if (resolvedValue == originalValue) {
                if (convertible) {
                    // 将 convertedValue 设置到 pv 中，后续再次创建同一个 bean 时，就无需再次进行转换了
                    pv.setConvertedValue(convertedValue);
                }
                deepCopy.add(pv);
            }
            /*
             * 如果原始值 originalValue 是 TypedStringValue，且转换后的值 
             * convertedValue 不是 Collection 或数组类型，则将转换后的值存入到 pv 中。
             */
            else if (convertible && originalValue instanceof TypedStringValue &&
                    !((TypedStringValue) originalValue).isDynamic() &&
                    !(convertedValue instanceof Collection || ObjectUtils.isArray(convertedValue))) {
                pv.setConvertedValue(convertedValue);
                deepCopy.add(pv);
            }
            else {
                resolveNecessary = true;
                deepCopy.add(new PropertyValue(pv, convertedValue));
            }
        }
    }
    if (mpvs != null && !resolveNecessary) {
        mpvs.setConverted();
    }

    try {
        // 将所有的属性值设置到 bean 实例中
        bw.setPropertyValues(new MutablePropertyValues(deepCopy));
    }
    catch (BeansException ex) {
        throw new BeanCreationException(
                mbd.getResourceDescription(), beanName, "Error setting property values", ex);
    }
}

```

以上就是 applyPropertyValues 方法的源码，配合着我写的注释，应该可以理解这个方法的流程。这个方法也调用了很多其他的方法，如果大家跟下去的话，会发现这些方法的调用栈也是很深的，比较复杂。这里说一下 bw.setPropertyValues 这个方法，如果大家跟到这个方法的调用栈的最底部，会发现这个方法是通过调用对象的 setter 方法进行属性设置的。这里贴一下简化后的代码：

```java
public class BeanWrapperImpl extends AbstractNestablePropertyAccessor implements BeanWrapper {

    // 省略部分代码

    private class BeanPropertyHandler extends PropertyHandler {
        @Override
        public void setValue(final Object object, Object valueToApply) throws Exception {
            // 获取 writeMethod，也就是 setter 方法
            final Method writeMethod = this.pd.getWriteMethod();
            if (!Modifier.isPublic(writeMethod.getDeclaringClass().getModifiers()) && !writeMethod.isAccessible()) {
                writeMethod.setAccessible(true);
            }
            final Object value = valueToApply;
            // 调用 setter 方法，getWrappedInstance() 返回的是 bean 对象
            writeMethod.invoke(getWrappedInstance(), value);
        }
    }
}
```

好了，本节的最后来总结一下 applyPropertyValues 方法的执行流程吧，如下：
1. 检测属性值列表是否已转换过的，若转换过，则直接填充属性，无需再次转换
2. 遍历属性值列表 pvs，解析原始值 originalValue，得到解析值 resolvedValue
3. 对解析后的属性值 resolvedValue 进行类型转换
4. 将类型转换后的属性值设置到 PropertyValue 对象中，并将 PropertyValue 对象存入 deepCopy 集合中
5. 将 deepCopy 中的属性信息注入到 bean 对象中





