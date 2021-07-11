## 前言

Spring 的出现是对广大 Java Web 开发者的福音，帮助我们解决了众多问题并且提供了很多便利

写此文章主要来说明对于 Spring IOC 容器中注册 Bean 的方式主要有哪几种


如果你以为我会告诉你

- @Bean
- @Import、@ImportSelector...
- @Service、@Component...


很遗憾的说：**格局小了兄弟...**

虽然这篇文章是作者看 Mybatis Spring 集成源码之后，决定先水一篇探路的文章

但是！**水也得水出高级感么不是**，怎么不得让小伙伴有点收获，看点工作中不常用到的



## 注册 IOC 容器 Bean

除了上文中说到的注册 Bean 方式，文章主要使用 BeanDefinitionRegistryPostProcessor（以下简称：BD） 来进行 Bean 注册

名词可能没听说过，但是如果说它是BeanFactoryPostProcessor 后置处理器（以下简称BF）子类，应该就清楚很多了

![](https://images-machen.oss-cn-beijing.aliyuncs.com/image-20201125221631186.png)



整体的关系比较简洁，只是继承了 Spring 后置处理器，先看下接口定义是怎么介绍的

```java
/**
 * Extension to the standard {@link BeanFactoryPostProcessor} SPI, allowing for
 * the registration of further bean definitions <i>before</i> regular
 * BeanFactoryPostProcessor detection kicks in. In particular,
 * BeanDefinitionRegistryPostProcessor may register further bean definitions
 * which in turn define BeanFactoryPostProcessor instances.
 *
 * @author Juergen Hoeller
 * @since 3.0.1
 * @see org.springframework.context.annotation.ConfigurationClassPostProcessor
 */
public interface BeanDefinitionRegistryPostProcessor extends BeanFactoryPostProcessor {

	/**
	 * Modify the application context's internal bean definition registry after its
	 * standard initialization. All regular bean definitions will have been loaded,
	 * but no beans will have been instantiated yet. This allows for adding further
	 * bean definitions before the next post-processing phase kicks in.
	 * @param registry the bean definition registry used by the application context
	 * @throws org.springframework.beans.BeansException in case of errors
	 */
	void postProcessBeanDefinitionRegistry(BeanDefinitionRegistry registry) throws BeansException;
}
```



接口上面这一段让人看不懂的英文说的啥意思呢？

> 大致说的是当前接口是后置处理器的 SPI 扩展，会在后置处理器工作前定义更多的 Bean



因为 BD（可注册Bean接口） 继承了 BF（后置处理器） 接口，所以如果实现 BP 就需要实现两个方法

```java
-- BeanFactoryPostProcessor
void postProcessBeanFactory(ConfigurableListableBeanFactory beanFactory) throws BeansException;
-- BeanDefinitionRegistryPostProcessor
void postProcessBeanDefinitionRegistry(BeanDefinitionRegistry registry) throws BeansException;
```



**mybatis 与 spring 源码集成上也是使用此方式，将 DAO 对象注册 Spring IOC 容器**



**postProcessBeanFactory**

该方法主要对已注册 Spring Bean 的定义做出改变



**postProcessBeanDefinitionRegistry**

该方法主要是将 Bean 定义注册到 Spring IOC 容器中，具体的 Bean 细节体现入参 BeanDefinitionRegistry



**BeanDefinitionRegistry**

BeanDefinitionRegistry 是一个接口并实现了 AliasRegistry 接口，其中方法定义了对 Bean 的常用操作

```java
// 向IOC容器中注册BeanDefinition实例
void registerBeanDefinition(String beanName, BeanDefinition beanDefinition)
			throws BeanDefinitionStoreException;
// 移除已存在的BeanDefinition实例
void removeBeanDefinition(String beanName) throws NoSuchBeanDefinitionException;
// 获取BeanDefinition实例
BeanDefinition getBeanDefinition(String beanName) throws NoSuchBeanDefinitionException;
// 判断beanName是否在IOC容器中
boolean containsBeanDefinition(String beanName);
// 获取IOC容器中所有实例名称
String[] getBeanDefinitionNames();
// 获取IOC容器中实例数量
int getBeanDefinitionCount();
// 判断beanName是否被占用
boolean isBeanNameInUse(String beanName);
```



postProcessBeanDefinitionRegistry 方法直接通过入参就可以完成 Bean 的注册、移除、判断等等操作



AliasRegistry 接口中定义的是 IOC 容器 Bean 别名管理，就不再一一介绍了



## 代码实战

创建一个 SpringBoot 的项目或者在已有项目测试都可以

```java
@Component
public class BeanRegistryTest implements BeanDefinitionRegistryPostProcessor, ApplicationContextAware {

    private ApplicationContext applicationContext;

    @Override
    public void postProcessBeanDefinitionRegistry(BeanDefinitionRegistry beanDefinitionRegistry) throws BeansException {
        GenericBeanDefinition parentBeanDef = new GenericBeanDefinition();
        parentBeanDef.setBeanClass(BeanRegistry.class);
        beanDefinitionRegistry.registerBeanDefinition("beanRegistry", rootBeanDefinition);
    }

    @Override
    public void postProcessBeanFactory(ConfigurableListableBeanFactory configurableListableBeanFactory) throws BeansException {
        System.out.println("注册Bean对象 :: " + applicationContext.getBean("beanRegistry"));
    }

    @Override
    public void setApplicationContext(ApplicationContext applicationContext) throws BeansException {
        this.applicationContext = applicationContext;
    }

    static class BeanRegistry { }
}
```



GenericBeanDefinition 是 BeanDefinition 接口通用实现，负责记录 Bean 对象信息





代码比较简单明了，为了方便在一个类中实现，通过以下步骤完成 Bean 注册测试



1. 通过实现 ApplicationContextAware 接口赋值 ApplicationContext 对象
2. BeanDefinitionRegistryPostProcessor 接口完成对象注册以及打印对应注册信息
3. postProcessBeanDefinitionRegistry 方法中注册 BeanRegistry 到 Spring IOC 容器中
4. postProcessBeanFactory 打印 Spring IOC 容器中的 BeanRegistry 对象信息



## 结言

由于作者水平有限, 欢迎大家能够反馈指正文章中错误不正确的地方, 感谢 🙏
