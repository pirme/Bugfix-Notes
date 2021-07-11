## 前言

Spring 在 3.0 版本之前都是通过 .xml 配置文件的形式来描述配置信息

在配置文件中显示声明 bena 标签或者扫描特定包下的类来注册 IOC 容器 Bean 对象等操作

@Import 是 Spring 3.0 之后通过 JavaConfig 方式提供注册 IOC Bean 的注解

需要配合 @Configuration 注解共同使用才能起作用，因为只有这样才会被 Spring 加载时扫描到

小伙伴阅读文章将收获：

1. @Import 日常使用方法
2. ImportBeanDefinitionRegistrar 配合 @Import 注册 Bean
3. ImportSelector 配合 @Import 注册 Bean
4. JavaConfig 方式引入 spring.xml 配置文件注册 Bean



## @Import 使用

先来看一下源码里是如何介绍这个小可爱的

```java
@Target(ElementType.TYPE)
@Retention(RetentionPolicy.RUNTIME)
@Documented
public @interface Import {

	/**
	 * {@link Configuration @Configuration}, {@link ImportSelector},
	 * {@link ImportBeanDefinitionRegistrar}, or regular component classes to import.
	 */
	Class<?>[] value();

}
```



源码类注释太长就不贴出来水字数了, 通过贴心的翻译软件，结论如下：

1. 表示要导入一个或多个普通、配置类（@Configuration ），将导入对象转换为 Spring IOC Bean
2. 支持 ImportBeanDefinitionRegistrar、ImportSelector 针对需要注册的 Bean 做更细致的填充
3. 如果使用的是 .xml 配置文件的形式注册 Bean，那么可以使用 @ImportResource
4. 通过导入注册的 Bean 对象可以正常被 @Autowired 注入 



先来简单看一下如何使用 @Import 来注册 Bean

```java
@Configuration
@Import(ImportBeanTest.ServiceBean.class)
public class ImportBeanTest implements ApplicationContextAware, BeanFactoryPostProcessor {
    
  private ApplicationContext applicationContext;

    @Override
    public void postProcessBeanFactory(ConfigurableListableBeanFactory configurableListableBeanFactory) throws BeansException {
        System.out.println("注册 ServiceBean 对象 :: " + applicationContext.getBean(ServiceBean.class));
    }

    @Override
    public void setApplicationContext(ApplicationContext applicationContext) throws BeansException {
        this.applicationContext = applicationContext;
    }

    static class ServiceBean {}
}
```



上面程序为了偷懒，直接使用了两个 Spring 接口，简单介绍下各自的作用

- **ApplicationContextAware**：为了获取 ApplicationContext 上下文对象，用来应该被注册的 Bean 实例
- **BeanFactoryPostProcessor**：后置处理器，程序用在查询 Bean 是否被注册 IOC 容器



这个使用了 @Import 注解的程序流程如下：

1. 类标记 @Configuration 表示是配置类，并通过 @Import 导入 ServiceBean
2. 通过 ApplicationContextAware 接口获取 ApplicationContext 上下文
3. 在后置处理器中，查看 ServiceBean 是否被注册成功



如果项目启动后能够正常打印对象信息，即注册成功



## @Configuration

小伙伴这次可以看到，@Import 中导入的类是被 @Configuration 所修饰的，也就意味着导入了一个配置类，同时其下包含多个 @Bean 方法

配置类中相关的 Bean 同时也会被加载

```java
@Configuration
@Import(ImportConfigurationTest.ImportConfiguration.class)
public class ImportConfigurationTest implements ApplicationContextAware, BeanFactoryPostProcessor {

    private ApplicationContext applicationContext;

    @Override
    public void postProcessBeanFactory(ConfigurableListableBeanFactory configurableListableBeanFactory) throws BeansException {
        System.out.println("注册 ServiceBeanA 对象 :: " + applicationContext.getBean(ServiceBeanA.class));
        System.out.println("注册 ServiceBeanB 对象 :: " + applicationContext.getBean(ServiceBeanB.class));
    }

    @Override
    public void setApplicationContext(ApplicationContext applicationContext) throws BeansException {
        this.applicationContext = applicationContext;
    }

    @Configuration
    static class ImportConfiguration {

        @Bean
        public ServiceBeanA getServiceBeanA() {return new ServiceBeanA();}

        @Bean
        public ServiceBeanB getServiceBeanB() {return new ServiceBeanB();}
    }

    static class ServiceBeanA {} static class ServiceBeanB {}
}
```



导入 @Configuration 和导入普通 Java Class 对象作用是一致的，相比于有两点好处：

1. 创建 Bean 时定义对应的方法以及属性值等操作
2. 将同一类型的 Bean 归结到一起，方便代码编写



## ImportSelector

ImportSelector 是一个接口，通过 @Import 进行导入，和 @Configuration 修饰类相似

两者作用域都是将多个类进行注入 IOC 容器，不同的是 ImportSelector 是将类的全限定名当作 IOC 容器的 ID



```java
@Configuration
@Import(ImportSelectorTest.RegistryConfig.class)
public class ImportSelectorTest implements ApplicationContextAware, BeanFactoryPostProcessor {

    private ApplicationContext applicationContext;

    @Override
    public void postProcessBeanFactory(ConfigurableListableBeanFactory configurableListableBeanFactory) throws BeansException {
        System.out.println("注册 ServiceBeanA 对象 :: " + applicationContext.getBean(ServiceBeanA.class));
        System.out.println("注册 ServiceBeanB 对象 :: " + applicationContext.getBean(ServiceBeanB.class));
    }

    @Override
    public void setApplicationContext(ApplicationContext applicationContext) throws BeansException {
        this.applicationContext = applicationContext;
    }

    static class RegistryConfig implements ImportSelector {
        @Override
        public String[] selectImports(AnnotationMetadata importingClassMetadata) {
            return new String[]{ServiceBeanA.class.getName(), ServiceBeanB.class.getName()};
        }
    }

    static class ServiceBeanA {} static class ServiceBeanB {}
}
```



selectImports 方法中的入参，存放的都是 **类元信息**，通过已序列化的 JSON 串看一下



![](https://images-machen.oss-cn-beijing.aliyuncs.com/image-20201127135816996.png)



另外通过后置处理器中，我们可以看到相关类已被注册，ID即是类全限定名称

![](https://images-machen.oss-cn-beijing.aliyuncs.com/image-20201127233423049.png)



## ImportBeanDefinitionRegistrar

ImportBeanDefinitionRegistrar 是一个接口，@Import 中可以加入此接口的实现

```java
@Configuration
@Import(ImportBeanDefinitionRegistrarTest.RegistrarConfig.class)
public class ImportBeanDefinitionRegistrarTest implements ApplicationContextAware, BeanFactoryPostProcessor {

    private ApplicationContext applicationContext;

    @Override
    public void postProcessBeanFactory(ConfigurableListableBeanFactory configurableListableBeanFactory) throws BeansException {
        System.out.println("注册 ServiceBean 对象 :: " + applicationContext.getBean(ServiceBean.class));
    }

    @Override
    public void setApplicationContext(ApplicationContext applicationContext) throws BeansException {
        this.applicationContext = applicationContext;
    }

    static class RegistrarConfig implements ImportBeanDefinitionRegistrar {
        @Override
        public void registerBeanDefinitions(AnnotationMetadata importingClassMetadata, BeanDefinitionRegistry registry) {
            BeanDefinition beanDefinition = BeanDefinitionBuilder
                    .genericBeanDefinition(ServiceBean.class)
                    .getBeanDefinition();
            registry.registerBeanDefinition("serviceBean", beanDefinition);
        }
    }

    static class ServiceBean {}
}
```



可以看到 registerBeanDefinitions 方法入参中不仅有类元信息，同时还包含 Bean 注册接口

同时可以为属性值、构造方法参数值以及更多实现信息进行赋值，不仅仅是示例代码中那么简单

![](https://images-machen.oss-cn-beijing.aliyuncs.com/image-20201127224507096.png)



## @ImportResource

如果说，你在用 Spring 或 SpringBoot 项目，需要使用 JavaConfig 这种方式来导入 .xml 文件运行

@ImportResource 绝对是救命稻草，但是作者认为 SpringBoot 项目更适合 JavaConfig 的方式进行配置



**registry.xml**

```xml
<?xml version="1.0" encoding="utf-8"?>
<beans xmlns="http://www.springframework.org/schema/beans"
       xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
       xsi:schemaLocation="http://www.springframework.org/schema/beans
       http://www.springframework.org/schema/beans/spring-beans.xsd">

    <bean id="serviceBean"
          class="cn.machen.study.springstudy.bean.importx.ImportResourceTest.ServiceBean" />

</beans>
```



**ImportResourceTest.java**

```java
@Configuration
@ImportResource(locations = "classpath:registry.xml")
public class ImportResourceTest implements ApplicationContextAware, BeanFactoryPostProcessor {

    private ApplicationContext applicationContext;

    @Override
    public void postProcessBeanFactory(ConfigurableListableBeanFactory configurableListableBeanFactory) throws BeansException {
        System.out.println("注册 ServiceBean 对象 :: " + applicationContext.getBean(ImportResourceTest.ServiceBean.class));
    }

    @Override
    public void setApplicationContext(ApplicationContext applicationContext) throws BeansException {
        this.applicationContext = applicationContext;
    }

    static class ServiceBean {}
}
```





## 结言

文章从最开始的 @Import 开始，讲述了注册普通 Java 类、@Configuration 修饰类、ImportSelector、ImportBeanDefinitionRegistrar 等方式以 JavaConfig 的形式注册 IOC Bean，同时也介绍了通过 @ImportResource 导入 .xml 配置文件的方式配置，希望在看的小伙伴都有所收获


由于作者水平有限, 欢迎大家能够反馈指正文章中错误不正确的地方, 感谢 🙏
