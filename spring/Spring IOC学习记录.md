# Spring IoC学习总结

## Spring简介

    世界上最受欢迎的java应用层级框架(官网)，可以更快、更轻松、更安全的进行java编程,核心是控制反转（IoC）和依赖注入（DI）

>IoC简介

本质就是一个***容器***，负责创建并管理对象的整个生命周期（创建至销毁）,也就是由容器来完成new一个实例的过程，控制权重程序员转向IoC容器，实现控制反转

```java
String arg = "IoC很强大"
TestIoc testIoc = new TestIoc(arg);
```

## IoC容器

>Spring中IoC容器实现主要分两种：**BeanFactory**和**ApplicationContext**

主要学习下ApplicationContext容器，它间接继承至BeanFactory,因此也包含了BeanFactory容器的所有功能（但ApplicationContext并不是BeanFactory的实现类，后面会提到），具体继承类图如下:
![ApplicationContext的类继承图](https://xymbucket.oss-cn-beijing.aliyuncs.com/blogs/Images/20210902111907.png)

>BeanFactory源码也贴出来看下：这个接口其实也没有具体功能实现，只是提取了容器使用中的一部分公共方法

- 获取Bean实例
- 判断是否单例
- 获取别名
- ...

![BeanFactory类方法结构图](https://xymbucket.oss-cn-beijing.aliyuncs.com/blogs/Images/20210902141316.png)

---
---

Spring中容器启动：

```java
ApplicationContext context = new ClassPathXmlApplicationContext("classpath:application.xml");
```

注意:ClassPathXmlApplicationContext只是Spring容器启动的一个分支实现而已，根据classpath下的xml文件来创建SpringContext容器和初始化bean实例

## 源码分析

>关注ClassPathXmlApplicationContext类

首先看下它的构造方法：

```java
public ClassPathXmlApplicationContext(
        String[] configLocations, boolean refresh, @Nullable ApplicationContext parent)
        throws BeansException {

    super(parent);
    //配置文件路径
    setConfigLocations(configLocations);
    if (refresh) {
        //IoC容器构建初始化的关键方法
        refresh();
    }
}
```

>直接进到refresh()方法：整个容器构建的过程全部放在了这个方法里头，先熟悉下各个方法的大概功能，然后再分析关键方法的具体实现

```java
public void refresh() throws BeansException, IllegalStateException {
//加锁，防止其他进程初始化销毁容器
synchronized (this.startupShutdownMonitor) {
    StartupStep contextRefresh = this.applicationStartup.start("spring.context.refresh");

    //准备工作，标记启动状态、验证配置文件是否为可解析状态
    prepareRefresh();

    // 更新BeanFactory，并解析bean信息到BeanFactory中
    ConfigurableListableBeanFactory beanFactory = obtainFreshBeanFactory();

    // 设置类加载器，添加BeanPostProcessor，手动注册几个特殊的 bean
    prepareBeanFactory(beanFactory);

    try {
        // 提供给子类的扩展点，此时bean还没有初始化，子类可以在该方法添加一些BeanFactoryPostProcess实现类
        postProcessBeanFactory(beanFactory);
        StartupStep beanPostProcess = this.applicationStartup.start("spring.context.beans.post-process");
        invokeBeanFactoryPostProcessors(beanFactory);
        registerBeanPostProcessors(beanFactory);
        beanPostProcess.end();

        // MessageSource和时间广播器
        initMessageSource();
        initApplicationEventMulticaster();

        // 钩子方法，子类初是化一些特殊bean(webServerGracefulShutdown,webServerStartStop等)
        onRefresh();

        // 注册事件监听器
        registerListeners();

        // 初始化所有单例bean,懒加载除外
        finishBeanFactoryInitialization(beanFactory);

        // 清缓存、广播事件...
        finishRefresh();
    }
    catch (BeansException ex) {
        ...
    }
    finally {
        resetCommonCaches();
        contextRefresh.end();
    }
}
```

### refresh方法中具体方法解析

>obtainFreshBeanFactory：容器构建的关键方法，初始化 BeanFactory、加载 Bean、注册 Bean等，但这是还没有对bean进行初始化，既还没生成bean实例   
具体实现为：AbstractRefreshableApplicationContext.class 121

```java
protected final void refreshBeanFactory() throws BeansException {
    //当前是否存在BeanFactory，有则销毁所有Bean并将其关闭
    if (hasBeanFactory()) {
        destroyBeans();
        closeBeanFactory();
    }
    try {
        //这个对象里面存了很多'货',包括<beanName,BeanDefinition>的map结构
        DefaultListableBeanFactory beanFactory = createBeanFactory();
        //序列化    
        beanFactory.setSerializationId(getId());
        //设置bean是否可覆盖以及循环引用
        customizeBeanFactory(beanFactory);
        //读取配置文件中的信息放到BeanDefinition当中，具体实现内容有点多，暂且省略
        loadBeanDefinitions(beanFactory);
        /**
        ApplicationContext中内部持有一个BeanFactory,所有BeanFactory操作实委托内部实例进行的==>装饰模式？
        */
        this.beanFactory = beanFactory;
    }
    catch (IOException ex) {
        throw new ApplicationContextException("I/O error parsing bean definition source for " + getDisplayName(), ex);
    }
}
```

BeanDefinition对象

![BeanDefinition对象](https://xymbucket.oss-cn-beijing.aliyuncs.com/blogs/Images/20210902163358.png)

>prepareBeanFactory:  处理一些特殊的bean  
具体实现=>AbstractApplicationContext.class 680

```java
protected void prepareBeanFactory(ConfigurableListableBeanFactory beanFactory) {
    //设置类加载器
    beanFactory.setBeanClassLoader(getClassLoader());
    if (!shouldIgnoreSpel) {
        beanFactory.setBeanExpressionResolver(new StandardBeanExpressionResolver(beanFactory.getBeanClassLoader()));
    }
    //添加属性编辑器
    beanFactory.addPropertyEditorRegistrar(new ResourceEditorRegistrar(this, getEnvironment()));

    //主要负责回调（postProcessBeforeInitialization方法）
    beanFactory.addBeanPostProcessor(new ApplicationContextAwareProcessor(this));
    //bean依赖下面接口时，自动装配忽略
    beanFactory.ignoreDependencyInterface(EnvironmentAware.class);
    beanFactory.ignoreDependencyInterface(EmbeddedValueResolverAware.class);
    beanFactory.ignoreDependencyInterface(ResourceLoaderAware.class);
    beanFactory.ignoreDependencyInterface(ApplicationEventPublisherAware.class);
    beanFactory.ignoreDependencyInterface(MessageSourceAware.class);
    beanFactory.ignoreDependencyInterface(ApplicationContextAware.class);
    beanFactory.ignoreDependencyInterface(ApplicationStartupAware.class);

    //为特殊bean赋值，自动装配（其中beanFactory也进行了复制，内部持有）
    beanFactory.registerResolvableDependency(BeanFactory.class, beanFactory);
    beanFactory.registerResolvableDependency(ResourceLoader.class, this);
    beanFactory.registerResolvableDependency(ApplicationEventPublisher.class, this);
    beanFactory.registerResolvableDependency(ApplicationContext.class, this);

    // 时间监听器
    beanFactory.addBeanPostProcessor(new ApplicationListenerDetector(this));

    // LoadTimeWeaver
    if (!NativeDetector.inNativeImage() && beanFactory.containsBean(LOAD_TIME_WEAVER_BEAN_NAME)) {
        beanFactory.addBeanPostProcessor(new LoadTimeWeaverAwareProcessor(beanFactory));
        // Set a temporary ClassLoader for type matching.
        beanFactory.setTempClassLoader(new ContextTypeMatchClassLoader(beanFactory.getBeanClassLoader()));
    }

    // Spring"自动"注册bean
    if (!beanFactory.containsLocalBean(ENVIRONMENT_BEAN_NAME)) {
        beanFactory.registerSingleton(ENVIRONMENT_BEAN_NAME, getEnvironment());
    }
    if (!beanFactory.containsLocalBean(SYSTEM_PROPERTIES_BEAN_NAME)) {
        beanFactory.registerSingleton(SYSTEM_PROPERTIES_BEAN_NAME, getEnvironment().getSystemProperties());
    }
    if (!beanFactory.containsLocalBean(SYSTEM_ENVIRONMENT_BEAN_NAME)) {
        beanFactory.registerSingleton(SYSTEM_ENVIRONMENT_BEAN_NAME, getEnvironment().getSystemEnvironment());
    }
    if (!beanFactory.containsLocalBean(APPLICATION_STARTUP_BEAN_NAME)) {
        beanFactory.registerSingleton(APPLICATION_STARTUP_BEAN_NAME, getApplicationStartup());
    }
}
```

>finishBeanFactoryInitialization：最最最重要的方法，前面方法已经将BeanFactory创建好了，配置文件中的Bean和实现了BeanFactoryPostProcessor 接口的 Bean 都已经初始化，并且其中的 ``postProcessBeanFactory(Beanfactory)`` 方法已经得到回调执行了，所有Bean信息存在BeanDefinition当中，该方法则是将Bean实例进行初始化，上源码

```java
protected void finishBeanFactoryInitialization(ConfigurableListableBeanFactory beanFactory) {
    // 初始化类型转换service==>初始化的具体实现封装在了getBean
    if (beanFactory.containsBean(CONVERSION_SERVICE_BEAN_NAME) &&
            beanFactory.isTypeMatch(CONVERSION_SERVICE_BEAN_NAME, ConversionService.class)) {
        beanFactory.setConversionService(
                beanFactory.getBean(CONVERSION_SERVICE_BEAN_NAME, ConversionService.class));
    }

    //注册值解析器
    if (!beanFactory.hasEmbeddedValueResolver()) {
        beanFactory.addEmbeddedValueResolver(strVal -> getEnvironment().resolvePlaceholders(strVal));
    }

    //初始化LoadTimeWeaverAware=>AspectJ=>相关内容
    String[] weaverAwareNames = beanFactory.getBeanNamesForType(LoadTimeWeaverAware.class, false, false);
    for (String weaverAwareName : weaverAwareNames) {
        getBean(weaverAwareName);
    }
    //标记状态
    beanFactory.setTempClassLoader(null);
    beanFactory.freezeConfiguration();

    // 开始初始化
    beanFactory.preInstantiateSingletons();
}
```

直接进到preInstantiateSingletons方法中的关键部分，无关代码直接忽略了,发现最终初始化都是调用的getBean方法实现
```java
for (String beanName : beanNames) {
    RootBeanDefinition bd = getMergedLocalBeanDefinition(beanName);
    //判断非抽象、非懒加载、单例对象进行初始化
    if (!bd.isAbstract() && bd.isSingleton() && !bd.isLazyInit()) {
        //FactoryBean处理特殊的bean实例，如数据库连接池实例等
        if (isFactoryBean(beanName)) {
            //重新构建beanName = '&' + beanName
            Object bean = getBean(FACTORY_BEAN_PREFIX + beanName);
            if (bean instanceof FactoryBean) {
                FactoryBean<?> factory = (FactoryBean<?>) bean;
                boolean isEagerInit;
                if (System.getSecurityManager() != null && factory instanceof SmartFactoryBean) {
                    isEagerInit = AccessController.doPrivileged(
                            (PrivilegedAction<Boolean>) ((SmartFactoryBean<?>) factory)::isEagerInit,
                            getAccessControlContext());
                }
                else {
                    isEagerInit = (factory instanceof SmartFactoryBean &&
                            ((SmartFactoryBean<?>) factory).isEagerInit());
                }
                if (isEagerInit) {
                    getBean(beanName);
                }
            }
        }
        else {
            //非特殊bean，直接调用getBean进行初始化
            getBean(beanName);
        }
    }
```

接下来进到getBean的方法实现：doGetBean方法

```java
protected <T> T doGetBean(
        String name, @Nullable Class<T> requiredType, @Nullable Object[] args, boolean typeCheckOnly)
        throws BeansException {

    //转换FactoryBean 或 别名转换
    String beanName = transformedBeanName(name);
    Object beanInstance;

    // 检查缓存是否已经创建
    Object sharedInstance = getSingleton(beanName);
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
        beanInstance = getObjectForBeanInstance(sharedInstance, name, beanName, null);
    }

    else {
        //校验prototype类型bean
        if (isPrototypeCurrentlyInCreation(beanName)) {
            throw new BeanCurrentlyInCreationException(beanName);
        }

        // 
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

        StartupStep beanCreation = this.applicationStartup.start("spring.beans.instantiate")
                .tag("beanName", name);
        try {
            if (requiredType != null) {
                beanCreation.tag("beanType", requiredType::toString);
            }
            RootBeanDefinition mbd = getMergedLocalBeanDefinition(beanName);
            checkMergedBeanDefinition(mbd, beanName, args);

            // 检查循环依赖
            String[] dependsOn = mbd.getDependsOn();
            if (dependsOn != null) {
                for (String dep : dependsOn) {
                    if (isDependent(beanName, dep)) {
                        throw new BeanCreationException(mbd.getResourceDescription(), beanName,
                                "Circular depends-on relationship between '" + beanName + "' and '" + dep + "'");
                    }
                    //注册依赖关系
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
            // Singleton 实例
            if (mbd.isSingleton()) {
                sharedInstance = getSingleton(beanName, () -> {
                    try {、
                        //执行Bean创建
                        return createBean(beanName, mbd, args);
                    }
                    catch (BeansException ex) {
                        destroySingleton(beanName);
                        throw ex;
                    }
                });
                beanInstance = getObjectForBeanInstance(sharedInstance, name, beanName, mbd);
            }
            //prototype 实例
            else if (mbd.isPrototype()) {
                // 创建新的instance
                Object prototypeInstance = null;
                try {
                    beforePrototypeCreation(beanName);
                    //执行Bean创建
                    prototypeInstance = createBean(beanName, mbd, args);
                }
                finally {
                    afterPrototypeCreation(beanName);
                }
                beanInstance = getObjectForBeanInstance(prototypeInstance, name, beanName, mbd);
            }

            else {
                //非Singleton、prototype,委托实现类处理
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
                            //执行Bean创建
                            return createBean(beanName, mbd, args);
                        }
                        finally {
                            afterPrototypeCreation(beanName);
                        }
                    });
                    beanInstance = getObjectForBeanInstance(scopedInstance, name, beanName, mbd);
                }
                catch (IllegalStateException ex) {
                    throw new ScopeNotActiveException(beanName, scopeName, ex);
                }
            }
        }
        catch (BeansException ex) {
            beanCreation.tag("exception", ex.getClass().toString());
            beanCreation.tag("message", String.valueOf(ex.getMessage()));
            cleanupAfterBeanCreationFailure(beanName);
            throw ex;
        }
        finally {
            beanCreation.end();
        }
    }

    return adaptBeanInstance(name, beanInstance, requiredType);
}
```

>createBean方法对Bean具体执行创建这边就暂不做分析了，方法返回bean实例
```java
try {
    //回调前面说到的BeanPostProcessor实现类中的方法
    Object bean = resolveBeforeInstantiation(beanName, mbdToUse);
    if (bean != null) {
        return bean;
    }
}
```

over













