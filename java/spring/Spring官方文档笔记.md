# Spring官方文档笔记

## IOC容器

### 依赖

1. 默认情况下 ApplicationContext 实现会预先实例化单例 bean。在实际需要这些 bean 之前，需要花费一些前期时间和内存来创建它们，因此在创建 ApplicationContext 时(而不是以后)会发现配置问题。您仍然可以覆盖这个默认行为，以便单例 bean 以惰性方式初始化，而不是预先实例化。

> 默认Spring会预先实例化单例bean的原因是为了尽早发现一些配置问题。例如循环依赖，无效属性这些问题，如果等到使用bean的时候才发现就已经晚了。

2. 官网文档更建议使用构造方法注入而不是Setter注入。（强制性属性使用构造，可选使用Setter）

> 这是因为可以将bean实现为不可变对象，并且可以确保所需的依赖项不为空。

3. depend-on的作用：保证depend-on的bean总是在当前bean之前被创建，在当前bean之后被销毁。
4. 配置bean的lazy-init属性可以让bean在被第一次请求时才创建。

> 前提是这个bean不是一个依赖项

### Bean的Scope

1. 将一个Scope的bean注入到Singleton Bean中要注意使用AOP，这样在每次使用的使用都会根据Scope去创建这个注入的scope。

### 定制Bean

#### 生命周期的回调

1. 要与 bean 生命周期的容器管理交互，可以实现 Spring InitializingBean 和 DisposableBean 接口。容器为前者调用 afterPropertiesSet () ，为后者调用 destroy () ，让 bean 在初始化和销毁 bean 时执行某些操作。

> 这里更加推荐使用@PostConstruct 和@Predestroy。耦合性较低。

2. 在内部，Spring 框架使用 BeanPostProcessor 实现来处理它能够找到并调用适当方法的任何回调接口。如果需要自定义特性或其他 Spring 默认不提供的生命周期行为，可以自己实现 BeanPostProcessor。多个BeanPostProcessor之间用order属性来控制顺序。

> BeanPostProcessor中的生命周期行为是针对所有Bean的。
>
> 调式代码看BeanPostProcessor的时候发现一个问题，使用XmlBeanFactory创建的容器不会执行BeanPostProcess，但是使用ClassPathXmlApplicationContext创建的容器会执行BeanPostProcess，后续再跟进下这里的代码。
>
> **解答：**1. 首先XmlBeanFactory的Bean在getBean的时候才会去创建一个预初始化的Bean。
>
> 			2. ClassPathXmlApplicationContext的父类AbstractApplicationContext会往AbstractBeanFactory.beanPostProcessors添加BeanFactoryProcessors，而XmlBeanFactory的父类DefaultListableBeanFactory并没有做这步操作。难怪官方不推荐使用XmlBeanFactory了。这里埋个坑，后续有时间好好把这里逻辑给理顺一下。
>    			3. 官网文档其实也是解释：因为 ApplicationContext 包含 BeanFactory 的所有功能，所以通常推荐使用普通的 BeanFactory，但需要对 bean 处理进行完全控制的场景除外。在 ApplicationContext (例如 GenericApplicationContext 实现)中，通过约定检测几种 bean (即通过 bean 名称或通过 bean 类型检测，特别是后处理器) ，而普通 DefaultListableBeanFactory 对任何特殊 bean 都是不可知的。
>
> 下图是执行BeanPostProcessor的代码：

![image-20210206014029441](https://fanshanchao.oss-cn-shenzhen.aliyuncs.com/img/image-20210206014029441.png)

3. Spring中控制Bean的生命周期有三个方法：

   * 使用InitializingBean和DisposableBean接口（不推荐）
   * 自定义init和destory方法，然后统一给bean指定默认的init方法，例如下面的例子：

   ```xml
   <beans default-init-method="init">
   
       <bean id="blogService" class="com.something.DefaultBlogService">
           <property name="blogDao" ref="blogDao" />
       </bean>
   
   </beans>
   ```

   * 使用@PostConstruct 和@predestroy（推荐）

4. 要想对整个ApplicationContext进行回调可以使用Lifecycle 接口，详细使用看官方文档。

5. 使用接口ApplicationContextAware回调，可以将ApplicationContext注入到Bean中。BeanNameAware接口回调可以将beanName注入到bean中去使用。BeanNameAware在普通 bean 属性填充之后，在诸如 InitializingBean、 afterPropertiesSet 或自定义 init-method 等初始化回调之前调用该回调。

### 容器扩展

1. 使用BeanFactory接口去创建复杂的bean。当您需要向容器请求一个实际的 FactoryBean 实例本身而不是它生成的 bean 时，在调用 ApplicationContext 的 getBean ()方法时，在 bean 的 id 前面加上 & 符号。

### 基于注解的Spring容器配置

1. 为什么使用一个特定的唯一ID推荐使用@ReSource而不是@Autowired+@Qualifiers？

> @Resource 注释，这个注释在语义上定义为根据特定的目标组件的唯一名称标识该组件，声明的类型与匹配过程无关。但是要注意@Reource仅支持字段和setter方法，并且只有一个参数
>
> 使用@Qualifiers会把所有符合条件的bean都注入，而不是唯一，所以可能导致报错。

​		这部分直接看[官网文档](https://docs.spring.io/spring-framework/docs/current/reference/html/core.html#beans-factory-lifecycle-combined-effects)

### 关于@Component

1. @Repository的用法之一是自动翻译异常。其实就是将标注的类中抛出的异常封装为Spring的数据访问异常类型。
2. Spring的Component扫描可以使用索引机制来提高效率。

### Environment 

Environment 接口是集成在容器中的抽象，它为应用程序环境的两个关键方面建模: 概要文件和属性。

### ApplicationContext

​		ApplicationContext 中的事件处理是通过 ApplicationEvent 类和 ApplicationListener 接口提供的。如果一个实现 ApplicationListener 接口的 bean 被部署到上下文中，那么每次一个 ApplicationEvent 被发布到 ApplicationContext 时，就会通知该 bean。本质上，这是标准的观察者设计模式。

### BeanFactory

1. BeanFactory API 为 Spring 的 IoC 功能提供了基础。它的特定契约主要用于集成 Spring 的其他部分和相关的第三方框架，它的 DefaultListableBeanFactory 实现是高级 GenericApplicationContext 容器中的一个关键委托。
2. 选择BeanFactory还是ApplicationContext？

> ApplicationContext 包含 BeanFactory 的所有功能。除非有充分的理由，否则应该使用 ApplicationContext，而 GenericApplicationContext 及其子类AnnotationConfigApplicationContext 是自定义引导的通用实现。这些是 Spring 核心容器的主要入口点，用于所有常见目的: 加载配置文件、触发类路径扫描、以编程方式注册 bean 定义和带注释的类，以及(从5.0开始)注册函数 bean 定义。

![image-20210206232756108](https://fanshanchao.oss-cn-shenzhen.aliyuncs.com/img/image-20210206232756108.png)

## Resource

​		Spring 的 Resource 接口旨在成为一个更有能力的接口，用于抽象对底层资源的访问。

```java
ublic interface Resource extends InputStreamSource {

    boolean exists();

    boolean isOpen();

    URL getURL() throws IOException;

    File getFile() throws IOException;

    Resource createRelative(String relativePath) throws IOException;

    String getFilename();

    String getDescription();
}
```

### 内置Resource实现

1. **UrlResource：**UrlResource 包装了一个 java. net. URL，可以用来访问通常可以通过 URL 访问的任何对象，比如文件、 HTTP 目标、 FTP 目标等。
2. **ClassPathResource：**此类表示应该从类路径获取的资源。它使用线程上下文类装入器、给定的类装入器或给定的类装入资源。
3. **FileSystemResource：**这是一个 java.io 的资源实现。文件和 java.nio.File。路径句柄。它支持分辨率作为一个文件和一个 URL。
4. **ServletContestResource：**这是 ServletContext 资源的一个资源实现，它解释了相关 web 应用程序根目录中的相对路径。
5. **ByteArrayResource：**这是一个给定字节数组的资源实现，它为给定的字节数组创建一个 bytearayinputstream。

### ResourceLoader

​		ResourceLoader 接口意味着由可以返回(即加载) Resource 实例的对象实现。

###  Application Contexts and Resource Paths

```java
ApplicationContext ctx =
    new FileSystemXmlApplicationContext("conf/appContext.xml");
```

## 验证、数据绑定和类型转换

​		数据绑定对于让用户输入动态地绑定到应用程序的域模型(或用于处理用户输入的任何对象)非常有用。Spring 提供了名副其实的 **DataBinder** 来完成这个任务。**Validator** 和 **DataBinder** 构成了验证包，它们主要用于但不限于 web 层。

### Validator接口

```java
//自定义Person实例的Validator
public class PersonValidator implements Validator {

    /**
     * This Validator validates only Person instances
     */
    public boolean supports(Class clazz) {
        return Person.class.equals(clazz);
    }

    public void validate(Object obj, Errors e) {
        ValidationUtils.rejectIfEmpty(e, "name", "name.empty");
        Person p = (Person) obj;
        if (p.getAge() < 0) {
            e.rejectValue("age", "negativevalue");
        } else if (p.getAge() > 110) {
            e.rejectValue("age", "too.darn.old");
        }
    }
}
```

### Bean Manipulation and the BeanWrapper

​		BeanWrapper接口使用直接看官网。

## Spring测试

### Spring 单元测试

Spring包括了四个可以用来进行单元测试的包：

1. Environment
2. JNDI
3. Servelt API
4. Spring Web Reactive：用于测试WebFlux应用程序

Spring有许多可以帮助进行单元测试的类，可分为：

1. General Testing Utilities：通用测试工具

2. Spring MVC Testing Utilities：Spring MVC 测试工具

## Spring事务

### Spring声明式事务

1. Spring声明式事务使用Aop实现。
2. Spring事务调用方法的流程图：

![image-20210223222013575](https://fanshanchao.oss-cn-shenzhen.aliyuncs.com/img/image-20210223222013575.png)

### 事务传播的级别

> 吐槽一下机翻，看的也太费劲了，也怪自己的英语太渣了。

1. **PROPAGATION_REQUIRED：**强制执行一个物理事务，可以是当前范围的本地事务(如果尚未存在事务) ，也可以是参与为更大范围定义的现有“外部”事务
2. **PROPAGATION _ require _ new：**它总是为每个受影响的事务范围使用独立的物理事务，从不参与外部范围的现有事务。
3. **PROPAGATION _ nested ：**使用具有多个保存点的单个物理事务，可以回滚到这些保存点。

## Spring Dao支持

1. 配置Dao层的bean最好使用@repository注解

## Spring MVC

### DispatcherServlet

对于许多应用程序来说，拥有一个单一的 WebApplicationContext 是简单且足够的。还可以有一个上下文层次结构，其中一个根 WebApplicationContext 在多个 DispatcherServlet (或其他 Servlet)实例之间共享，每个实例都有自己的子 WebApplicationContext 配置。RootWebApplicationContext 通常包含基础架构 bean，例如需要跨多个 Servlet 实例共享的数据存储库和业务服务。这些 bean 可以被有效地继承，并且可以在特定于 Servlet  WebApplicationContext 中重写(即重新声明) ，后者通常包含给定 Servlet 的本地 bean。。

![image-20210224221536819](https://fanshanchao.oss-cn-shenzhen.aliyuncs.com/img/image-20210224221536819.png)

#### 一些特殊的bean

1. **HandlerMapping：**将请求映射到处理程序以及用于预处理和后处理的拦截器列表。
2. **HandlerAdapter：**帮助 DispatcherServlet 调用映射到请求的处理程序，而不管该处理程序实际上是如何调用的。

#### Interception 拦截器

Interceptor和Filter的区别：

1. Interceptor基于Java的反射，也就是Spring中的AOP。而Filter依赖于回调。
2. Filter需要在web.xml中配置，依赖于Servlet
3. Interceptor需要在SpringMVC中配置，依赖于框架，所以可以使用依赖注入，拦截器中可以获取到Controller。
4. Interceptor只能对Controller请求进行拦截，对静态资源无法拦截
5. Filter的执行顺序在Interceptor之前，具体的流程见下图：

![image-20210224223720189](https://fanshanchao.oss-cn-shenzhen.aliyuncs.com/img/image-20210224223720189.png)

Interceptor和Filter以及Aspect的区别和执行顺序：
![image-20210224224906608](https://fanshanchao.oss-cn-shenzhen.aliyuncs.com/img/image-20210224224906608.png)

![image-20210224225111946](https://fanshanchao.oss-cn-shenzhen.aliyuncs.com/img/image-20210224225111946.png)

## Spring Boot

1. 需要通过向@configuration 类中添加@enableautoconfiguration 或@springbootapplication 注释来选择自动配置。这里所谓的自动配置就是Spring Boot 自动配置尝试根据添加的 jar 依赖项自动配置 Spring 应用程序。

2. @springbootapplication 包括@Configuration @EnableAutoConfiguration @ComponentScan三个注解。