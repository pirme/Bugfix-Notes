## 前言

最近在构思中台如何通过一个接口，发布出多种协议。比如说：**发布 dubbo、feign 或者阿里 hsf 接口的同时发布出同语义的 http 接口**

最后想到的是扫描自定义注解，然后注册到 mvc 中。所以就去看了下 mvc 注册 mapping 流程，就引发了 **本文的主角 InitializingBean**



> 本文大纲如下：
>
> - InitializingBean 是什么
> - 耍一耍 InitializingBean
> - InitializingBean 如何被加载
> - mvc 源码中如何使用
> - 结言



## InitializingBean 是什么

InitializingBean 接口为 bean 提供了 **初始化方法的方式**，接口只包括一个无返回值的 afterPropertiesSet 方法，凡是继承该接口的类，**在初始化 bean 的时候都会执行该方法**



```java
/**
 * Interface to be implemented by beans that need to react once all their properties
 * have been set by a {@link BeanFactory}: e.g. to perform custom initialization,
 * or merely to check that all mandatory properties have been set.
 *
 * <p>An alternative to implementing {@code InitializingBean} is specifying a custom
 * init method, for example in an XML bean definition. For a list of all bean
 * lifecycle methods, see the {@link BeanFactory BeanFactory javadocs}.
 *
 * @author Rod Johnson
 * @author Juergen Hoeller
 * @see DisposableBean
 * @see org.springframework.beans.factory.config.BeanDefinition#getPropertyValues()
 * @see org.springframework.beans.factory.support.AbstractBeanDefinition#getInitMethodName()
 */
public interface InitializingBean {

	/**
	 * Invoked by the containing {@code BeanFactory} after it has set all bean properties
	 * and satisfied {@link BeanFactoryAware}, {@code ApplicationContextAware} etc.
	 * <p>This method allows the bean instance to perform validation of its overall
	 * configuration and final initialization when all bean properties have been set.
	 * @throws Exception in the event of misconfiguration (such as failure to set an
	 * essential property) or if initialization fails for any other reason
	 */
	void afterPropertiesSet() throws Exception;

}
```



还是老规矩，凭借作者大学一级水平的英语给大家翻译一下类注释

> 由 BeanFactory **设置完所有属性后** 需要进行响应实现此接口的 bean。例如：**执行自定义初始化**，或检查所有 **必需属性是否已设置**。也可以使用自定义 initMethod 方法替代



从类注释得到了啥信息呢

1. 必须是 BeanFactory 设置完所有属性以后才执行
2. 其次，实现此接口的必须要是一个 ioc 容器的 bean
3. 这玩意不是不可替代，可以使用 initMethod 方法代替



afterPropertiesSet 方法注释别看写的不少，咱们挑重点说

> 该方法会在设置了所有 bean 属性的条件下调用。**其实和类注释表达意思一样**，毕竟小子还是要听老子的



在不看源码的情况下，我们来简单总结一下。bean 实现了 InitializingBean 接口，那么就会在 BeanFactory 设置完所有属性后，调用 afterPropertiesSet 方法



## 耍一耍 InitializingBean

老话说得好：**光说不练假把式**；那我们就来耍一耍这个接口

> 大家要在 spring 项目或 springboot 项目里进行哈，**示例代码使用的 lombok 打印日志**



这个小程序比较简短，做了两件事情

1. 项目启动时，看是否会打印代码里的日志，打印即代表成功
2. 输出容器里 bean 数量

```java
@Slf4j
@Component
public class InitializingBeanTest implements InitializingBean {
    @Autowired
    private ApplicationContext applicationContext;
    @Override
    public void afterPropertiesSet() throws Exception {
        log.info("======");
        log.info("  >>> InitializingBeanTest 执行 :: afterPropertiesSet ");
        log.info("  >>> BeanNames :: {}", applicationContext.getBeanDefinitionCount());
        log.info("======");
    }
  	/**
     * 运行结果：
     * ======
     *   >>> InitializingBeanTest 执行 :: afterPropertiesSet 
     *   >>> BeanNames :: 128
     * ======
     */
}
```



想到 bean 实例化后的初始化方法，大家一定会想到声明 bean 时的 initMethod 属性，下面会向大家详细介绍下两者的区别

> 其实，InitializingBean 大有所为，具体功能这一块在下面会讲到



**InitializingBean 和 initMethod 的区别**



**实现方式：**

1. InitializingBean 是一个接口，需要实现此接口的类是一个 ioc 容器 bean
2. init 需要在声明 bean 时在 initMethod 指定，通过 xml 或者 @Bean 的形式声明



**执行时机：**

1. BeanFactory 设置完所有属性之后，会执行实现了 InitializingBean 接口的 bean
2. initMethod 的执行时机在 InitializingBean 之后

改动一下上面的 demo 再运行试试


```java
@Slf4j
public class InitializingBeanTest implements InitializingBean {

    @Configuration
    static class BeanConfiguration {
        @Bean(initMethod = "init")
        public InitializingBeanTest getInitializingBeanTest() {
            return new InitializingBeanTest();
        }
    }

    @Autowired
    private ApplicationContext applicationContext;

    public void init() {
        log.info("  >>> bean 指定 init 执行... ");
    }

    @Override
    public void afterPropertiesSet() {
        log.info("======");
        log.info("  >>> InitializingBeanTest 执行 :: afterPropertiesSet ");
        log.info("  >>> BeanNames :: {}", applicationContext.getBeanDefinitionCount());
        log.info("======");
    }
    /**
     * 运行结果：
     * ======
     *   >>> InitializingBeanTest 执行 :: afterPropertiesSet
     *   >>> BeanNames :: 128
     * ======
     * 
     *   >>> bean 指定 init 执行... 
     */
}
```

根据运行结果得知，initMethod 运行在了 InitializingBean 之后，接下来我们通过源码的角度来讲解这是为什么？


## InitializingBean 如何加载

可以通过 spring 加载 bean 的源码类 AbstractAutowiredCapableBeanFactory

查看具体的调用逻辑，invokeInitMethods 方法负责调用实现 InitializingBean 接口以及指定 initMethod 方法的 bean 

```java
protected void invokeInitMethods(String beanName, Object bean, @Nullable RootBeanDefinition mbd)
        throws Throwable {
		// 判断 bean 是否实现了 InitializingBean 接口
    boolean isInitializingBean = (bean instanceof InitializingBean);
    if (isInitializingBean && (mbd == null || !mbd.isExternallyManagedInitMethod("afterPropertiesSet"))) {
        if (logger.isTraceEnabled()) {
            logger.trace("Invoking afterPropertiesSet() on bean with name '" + beanName + "'");
        }
      	// 系统安全处理器为空则直接执行 else 流程, 调用 afterPropertiesSet 方法
      	// 默认为空，所以直接 else 流程
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
          	// 直接调用 afterPropertiesSet
            ((InitializingBean) bean).afterPropertiesSet();
        }
    }

    if (mbd != null && bean.getClass() != NullBean.class) {
      	// 判断是否指定了 initMethod 方法, 如果指定会进行调用
        String initMethodName = mbd.getInitMethodName();
      	// 如果 initMethod 方法名称为 “afterPropertiesSet”, 则不尽兴调用
        if (StringUtils.hasLength(initMethodName) &&
                !(isInitializingBean && "afterPropertiesSet".equals(initMethodName)) &&
                !mbd.isExternallyManagedInitMethod(initMethodName)) {
          	// 通过反射调用 initMethod 指定方法
            invokeCustomInitMethod(beanName, bean, mbd);
        }
    }
}
```



通过源码我们都已经很明白了，其实就是在依赖注入完成的时候，spring 会去检查 bean 是否实现了 InitializingBean 接口，已实现就会去调用这个类的 afterPropertiesSet 方法

另外总结三个小知识点供大家参考：

1. 晓得了为什么 InitializingBean 接口在 initMethod 方法之前
2. 如果 bean 对象实现了 InitializingBean 接口，而且声明 initMethod 方法名称为 "afterPropertiesSet" 时，是不会重复调用的
3. initMethod 方法是通过反射执行的，而 InitializingBean 为直接调用，大家可以自行选择初始化方式



## mvc 源码中如何使用

关于 springmvc 的源代码不会过多讲解，会在下一篇 mvc 源码解析中说明

这篇文章算是投石问路，先来看下 mvc 是如何利用 InitializingBean 初始化方法特性完成映射关系的



```java
@RestController
@RequestMapping("/test")
public class MvcController {

    @GetMapping("/say/hello/{name}")
    public String sayHello(@PathVariable("name") String name) {
        return "Hello World " + name;
    }
}
```



为了避免大家对这块没有任何印象，大致说一下：就是会将 Mapping 相关 url 路由信息注册



![](https://images-machen.oss-cn-beijing.aliyuncs.com/image-20201209203708756.png)



springmvc 源码还是很复杂的，主要说明 AbstractHandlerMethodMapping 是如何将 url 转换为 HandlerMethod 使用

> springmvc HandlerMethod 属于 springmvc 专属，不必关注



我们先来看一波最关键的，那就是实现了 InitializingBean 接口，并在 afterPropertiesSet 方法中调用了初始化逻辑

```java
public abstract class AbstractHandlerMethodMapping<T> extends AbstractHandlerMapping implements InitializingBean {
   @Override
	  public void afterPropertiesSet() {
		    initHandlerMethods();
	  }
}
```



这个方法会遍历 spring ioc 容器中所有的 bean，然后将符合条件拥有 @Controller、@RequestMapping 等注解的 bean，维护 url 的映射关系

```java
protected void initHandlerMethods() {
    if (logger.isDebugEnabled()) {
        logger.debug("Looking for request mappings in application context: " + getApplicationContext());
    }
  	// 获取 ioc 容器中所有 bean 名称
    String[] beanNames = (this.detectHandlerMethodsInAncestorContexts ?
            BeanFactoryUtils.beanNamesForTypeIncludingAncestors(obtainApplicationContext(), Object.class) :
            obtainApplicationContext().getBeanNamesForType(Object.class));
		// 循环 bean 名称数组, 找到符合条件的进行关系维护
    for (String beanName : beanNames) {
        if (!beanName.startsWith(SCOPED_TARGET_NAME_PREFIX)) {
            Class<?> beanType = null;
            try {
                beanType = obtainApplicationContext().getType(beanName);
            }
            catch (Throwable ex) {
                // An unresolvable bean type, probably from a lazy bean - let's ignore it.
                if (logger.isDebugEnabled()) {
                    logger.debug("Could not resolve target class for bean with name '" + beanName + "'", ex);
                }
            }
          	// isHandler 即判断是否拥有 @Controller、@RequestMapping
            if (beanType != null && isHandler(beanType)) {
              	// 符合条件, 进行关系维护
                detectHandlerMethods(beanName);
            }
        }
    }
    handlerMethodsInitialized(getHandlerMethods());
}
```



## 结言

文章详细讲解什么是 InitializingBean，并且通过实际例子以及源码讲解的形式，讲述了和声明 bean 时指定的 initMethod 有什么区别

最后通过 mvc 框架实际例子讲述了是如何通过 InitializingBean接口，将 url 与自定义 Handler 实体相互绑定的，因为这样可以更好的帮助我们理解与项目实际使用



