---
typora-root-url: ..\..\images
---

# 前言

1. 本文基于代码基于Spring5.0x源码，源码可自行到github上下载。

# 从一个小例子说起

```java
//Person.java
public class Person {
	public String getName (){
		return "person";
	}
}
//bean.xml
<?xml version="1.0" encoding="UTF-8"?>
<beans xmlns="http://www.springframework.org/schema/beans"
	   xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
	   xsi:schemaLocation="http://www.springframework.org/schema/beans
    http://www.springframework.org/schema/beans/spring-beans-3.0.xsd">

	<bean id="person" class="cn.fsc.bean.Person">
	</bean>
</beans>
//BeanFactoryTest
public class BeanFactoryTest {
	@Test
	public void testSimpleLoad(){
		BeanFactory beanFactory = new XmlBeanFactory(new ClassPathResource("bean.xml"));
		Person person = (Person) beanFactory.getBean("person");
		Assert.assertEquals("person",person.getName());
	}
}
```

​	上面的例子很简单，首先使用XmlBeanFactory读取bean.xml并解析，然后加载里面配置的bean到容器中，最后使用。bean的生命周期也就和上面代码一样，**解析注册->加载到Spring容器->使用**。本文分析的就是**解析注册**部分，也就是Spring如果解析xml文件中的bean，并将bean注册到Spring容器中（注意还没有被加载）。

# DefaultListableBeanFactory类

​	上面例子中的XmlBeanFactory继承于DefaultListableBeanFactory，DefaultListableBeanFactory是整个bean加载的核心部分，是Spring注册及加载bean的默认实现。XmlBeanFactory使用了自定义的XML读取器实现了个性化读取bean的定义。

​	再来看下DefaultListableBeanFactory的继承图，现在大概看一下下面类的作用即可，后面分析时再了解作用：

<img src="E:\拎着自己飞呀\笔记本\images\DefaultListableBeanFactory继承图.png" style="zoom:200%;" />

​	简单说下各个类的作用：
**AliasRegistry**：对alis的简单增删改查。
**SimpleAliasRegistry**：AliasRegistry的实现类，使用了map作为alias的缓存
**SingletonBeanRegistry**：定义对单例的注册及获取
**BeanFactory**：定义获取bean及bean的各种属性
**DefaultSingletonBeanRegistry**：SingletonBeanRegistry的实现类
**HierarchicalBeanFactory**：继承BeanFactory，增加了对parentFactory的支持
**BeanDefinitionRegistry**：定义了对BeanDefinition的CRUD
**FactoryBeanRegistrySupport**：在DefaultSingletonBeanRegistry的基础上增加了FactoryBean的特殊处理
**ConfigurableBeanFactory**：提供配置Factory的各种方法
**ListableBeanFactory**：根据各种条件获取bean的配置清单
**AbstractBeanFactory**：综合FactoryBeanRegistrySupport和ConfigurableBeanFactory的功能
**AutowireCapableBeanFactory**：提供创建bean，自动注入，初始化以及应用bean 的后处理器
**AbstractAutowireCapableBeanFactory**：综合AbstractBeanFactory并对接口AutowireCapableBeanFactory进行实现
**ConfigurableListableBeanFactory**：BeanFactory配置清单，指定忽略类型及接口等
**DefaultListableBeanFactory**：综合上面所有功能，主要是对bean注册后的处理。

# 源码分析

​	前面的铺垫结束了，现在就应该具体分析代码了。首先点进**new XmlBeanFactory(new ClassPathResource("bean.xml"));**这行代码看下里面：

```java
//XmlBeanFactory.java
	public XmlBeanFactory(Resource resource, BeanFactory parentBeanFactory) throws BeansException {
		super(parentBeanFactory);
        //核心 reader是XmlBeanDefinitionReader，这个类用来解析bean
		this.reader.loadBeanDefinitions(resource);
	}
```

​	XmlBeanDefinitionReader类才是解析xml文件的核心所在，XmlBeanFactory利用XmlBeanDefinitionReader实现xml的解析工作。所以再进行点进去**this.reader.loadBeanDefinitions(resource);**这行代码：

```java
//XmlBeanDefinitionReader.java
@Override
	public int loadBeanDefinitions(Resource resource) throws BeanDefinitionStoreException {
        //对资源进行编码 再继续看下一步
		return loadBeanDefinitions(new EncodedResource(resource));
}
	public int loadBeanDefinitions(EncodedResource encodedResource) throws BeanDefinitionStoreException {
		//传进来的EncodedResource是编码后的Resource
        //....省略
		//获取到当前线程加载的资源 currentResources是ThreadLocal
		Set<EncodedResource> currentResources = this.resourcesCurrentlyBeingLoaded.get();
        //判断当前线程是否加载过资源，如果没有就创建Set来加载
		if (currentResources == null) {
			currentResources = new HashSet<>(4);
			this.resourcesCurrentlyBeingLoaded.set(currentResources);
		}
		if (!currentResources.add(encodedResource)) {
			throw new BeanDefinitionStoreException(
					"Detected cyclic loading of " + encodedResource + " - check your import definitions!");
		}
		try {
            //获取到Resource的InputStream 用于后面的xml解析
			InputStream inputStream = encodedResource.getResource().getInputStream();
			try {
                //创建SAX解析xml需要使用到的InputSource
				InputSource inputSource = new InputSource(inputStream);
				if (encodedResource.getEncoding() != null) {
					inputSource.setEncoding(encodedResource.getEncoding());
				}
                //准备工作做好之后的核心逻辑
				return doLoadBeanDefinitions(inputSource, encodedResource.getResource());
			}
			finally {
				inputStream.close();
			}
		}
	}
	protected int doLoadBeanDefinitions(InputSource inputSource, Resource resource)
			throws BeanDefinitionStoreException {
		try {
            //将xml文件解析为一个SAX里面的Document对象 里面还用到了XML的验证模式
			Document doc = doLoadDocument(inputSource, resource);
            //下面才是关键 解析及注册BeanDefinition
			return registerBeanDefinitions(doc, resource);
		}
        //.....
	}
	public int registerBeanDefinitions(Document doc, Resource resource) throws BeanDefinitionStoreException {
        //BeanDefinitionDocumentReader对象 createBeanDefinitionDocumentReader方法就是创建这个接口的子类DefaultBeanDefinitionDocumentReader
		BeanDefinitionDocumentReader documentReader = createBeanDefinitionDocumentReader();
        //获取到当前BeanFactory注册的数量 这里可以点进去看下
        //这里使用的是BeanDefinitionRegistry对象，而DefaultListableBeanFactory便实现了这个接口，所以getRegistry()就是获取最开始的XmlBeanFactory
        //接下来再看getBeanDefinitionCount()方法 
		int countBefore = getRegistry().getBeanDefinitionCount();
        //注册Document中的Bean 再点进去看 下面源码接着这里看
		documentReader.registerBeanDefinitions(doc, createReaderContext(resource));
		return getRegistry().getBeanDefinitionCount() - countBefore;
	}
```

​	上面的代码可分为下面几个步骤：

  1. 将Rousure资源进行编码

  2. 判断当前线程是否加载过这个资源，如果没有则创建Set保存加载过的资源。这里用到了TreadLocal实现了线程隔离。

  3. 利用Reource生成使用SAX解析xml需要的InputStream和InputSource，再利用SAX解析xml生成Document

  4. 获取到已经注册为BeanDefinition对象的bean的数量（后面会讲到什么是BeanDefinition）。这行代码可以点进去看一下：

     ```java
     int countBefore = getRegistry().getBeanDefinitionCount();
     //1. getRegistry方法就是获取一个BeanDefinitionRegistry对象，而我们可以看DefaultListableBeanFactory类的继承图可以看到它实现了BeanDefinitionRegistry接口。
     //2. 而之前我们创建XmlBeanDefinitionReader对象是将DefaultListableBeanFactory（XmlBeanFactory）对象传入了XmlBeanDefinitionReader的构造方法。所以getBeanDefinitionCount方法实际上是调用DefaultListableBeanFactory对象的。
     //3. 所以看一下DefaultListableBeanFactory的getBeanDefinitionCount()
     	@Override
     	public int getBeanDefinitionCount() {
     		return this.beanDefinitionMap.size();
     	}
     //4. 再看beanDefinitionMap属性
     	private final Map<String, BeanDefinition> beanDefinitionMap = new ConcurrentHashMap<>(256);
     //5. 可以看到DefaultListableBeanFactory是使用ConcurrentHashMap来保存BeanDefinition的，ConcurrentHashMap是线程安全的HashMap。ConcurrentHashMap是juc中一个非常重要的类，有兴趣可自行查找资料。
     ```

     

  5. **使用BeanDefinitionDocumentReader对象将Document中的bean注册为BeanDefinition**



接着上面的源码继续看：

```java
//DefaultBeanDefinitionDocumentReader.java  BeanDefinitionDocumentReader的子类
	@Override
	public void registerBeanDefinitions(Document doc, XmlReaderContext readerContext) {
		this.readerContext = readerContext;
		logger.debug("Loading bean definitions");
        //从根元素开始解析
		Element root = doc.getDocumentElement();
        //继续接着看
		doRegisterBeanDefinitions(root);
	}
	protected void doRegisterBeanDefinitions(Element root) {
        //BeanDefinitionParserDelegate类用来解析XML中的bean元素
		BeanDefinitionParserDelegate parent = this.delegate;
		this.delegate = createDelegate(getReaderContext(), root, parent);

		if (this.delegate.isDefaultNamespace(root)) {
            //处理Profile属性
			String profileSpec = root.getAttribute(PROFILE_ATTRIBUTE);
			if (StringUtils.hasText(profileSpec)) {
				String[] specifiedProfiles = StringUtils.tokenizeToStringArray(
						profileSpec, BeanDefinitionParserDelegate.MULTI_VALUE_ATTRIBUTE_DELIMITERS);
				if (!getReaderContext().getEnvironment().acceptsProfiles(specifiedProfiles)) {
					if (logger.isInfoEnabled()) {
						logger.info("Skipped XML bean definition file due to specified profiles [" + profileSpec +
								"] not matching: " + getReaderContext().getResource());
					}
					return;
				}
			}
		}
        //解析前需要做的事，这里用到了模板方法模式，子类重写这个方法即可
		preProcessXml(root);
        //解析 接着看解析代码
		parseBeanDefinitions(root, this.delegate);
        //解析后需要做的事，这里用到了模板方法模式，子类重写这个方法即可
		postProcessXml(root);

		this.delegate = parent;
	}

	protected void parseBeanDefinitions(Element root, BeanDefinitionParserDelegate delegate) {
		if (delegate.isDefaultNamespace(root)) {
			NodeList nl = root.getChildNodes();
			for (int i = 0; i < nl.getLength(); i++) {
				Node node = nl.item(i);
				if (node instanceof Element) {
					Element ele = (Element) node;
					if (delegate.isDefaultNamespace(ele)) {
                        //使用默认方式解析并注册 下面源码接着这里
						parseDefaultElement(ele, delegate);
					}
					else {
                        //使用用户自定义方式解析
						delegate.parseCustomElement(ele);
					}
				}
			}
		}
		else {
			delegate.parseCustomElement(root);
		}
	}
```

​	上面源码步骤如下：

1. 从根元素开始解析
2. 创建用于解析bean元素的BeanDefinitionParserDelegate对象
3. 处理profile属性，这个属性就是我们平常使用的dev，prd，test
4. **使用默认的方式解析元素并注册**

注：上面使用到了模板方法模式，有兴趣可以自行学习。

接着上面的源码：

```java
//DefaultBeanDefinitionDocumentReader.java
private void parseDefaultElement(Element ele, BeanDefinitionParserDelegate delegate) {
    	//解析import元素
		if (delegate.nodeNameEquals(ele, IMPORT_ELEMENT)) {
			importBeanDefinitionResource(ele);
		}
    	//解析alias元素
		else if (delegate.nodeNameEquals(ele, ALIAS_ELEMENT)) {
			processAliasRegistration(ele);
		}
    	//解析bean元素 看这里就行了
		else if (delegate.nodeNameEquals(ele, BEAN_ELEMENT)) {
			processBeanDefinition(ele, delegate);
		}
    	//解析beans元素
		else if (delegate.nodeNameEquals(ele, NESTED_BEANS_ELEMENT)) {
			// recurse
			doRegisterBeanDefinitions(ele);
		}
	}
	protected void processBeanDefinition(Element ele, BeanDefinitionParserDelegate delegate) {
        //1. 解析Bean元素 bdHolder对象已经包括了我们配置文件中的各种属性了 这里面就是层传剥皮 下面源码分析接着这里
		BeanDefinitionHolder bdHolder = delegate.parseBeanDefinitionElement(ele);
        //不为空则继续解析自定义属性
		if (bdHolder != null) {
            //2. 根据需要对beanDefinition进行装饰 bean使用的是默认标签配置，但是子元素使用了自定义的配置时这句代码便有用了
			bdHolder = delegate.decorateBeanDefinitionIfRequired(ele, bdHolder);
			try {
				//3. 注册最终的实例 所以这里是关键 点进去看 go
				BeanDefinitionReaderUtils.registerBeanDefinition(bdHolder, getReaderContext().getRegistry());
			}
			catch (BeanDefinitionStoreException ex) {
				getReaderContext().error("Failed to register bean definition with name '" +
						bdHolder.getBeanName() + "'", ele, ex);
			}
			// 4. 发送注册事件
			getReaderContext().fireComponentRegistered(new BeanComponentDefinition(bdHolder));
		}
	}
```

## BeanDefintion接口

在进行上面的源码分析之前，先看一下BeanDefintion接口。BeanDefintion是配置文件<bean>元素标签在容器内部表示形式。。该接口存在三种实现：RootBeanDefinition，ChildBeanDefinition，GenericBeanDefinition。

![](E:\拎着自己飞呀\笔记本\images\BeanDefinition接口继承图.png)

父<bean>用RootBeanDefinition表示，子<bean>用ChildBeanDefinition表示，GenericBeanDefinition是2.5以后加入的bean文件配置属性定义类。

**Spring通过BeanDefinition将配置文件中的bean元素配置信息转换为容器内部表示，并将这些BeanDefinition注册到BeanDefintionRegistry中。**BeanDefintionRegistry就像内存数据库。



接着看上面源码的parseBeanDefinitionElement方法：

```java
//BeanDefinitionParserDelegate.java
	public BeanDefinitionHolder parseBeanDefinitionElement(Element ele) {
		return parseBeanDefinitionElement(ele, null);
	}
   	public BeanDefinitionHolder parseBeanDefinitionElement(Element ele, @Nullable BeanDefinition containingBean) {
           //解析到id属性
   		String id = ele.getAttribute(ID_ATTRIBUTE);
           //解析到name属性
   		String nameAttr = ele.getAttribute(NAME_ATTRIBUTE);
           //分割name属性因为一个bean可以有多个别名
   		List<String> aliases = new ArrayList<>();
   		if (StringUtils.hasLength(nameAttr)) {
   			String[] nameArr = StringUtils.tokenizeToStringArray(nameAttr, MULTI_VALUE_ATTRIBUTE_DELIMITERS);
   			aliases.addAll(Arrays.asList(nameArr));
   		}
        
   		String beanName = id;
		if (!StringUtils.hasText(beanName) && !aliases.isEmpty()) {
   			beanName = aliases.remove(0);
   			if (logger.isDebugEnabled()) {
   				logger.debug("No XML 'id' specified - using '" + beanName +
   						"' as bean name and " + aliases + " as aliases");
   			}
   		}
   
   		if (containingBean == null) {
   			checkNameUniqueness(beanName, aliases, ele);
   		}
           //上面讲到过的AbstractBeanDefinition go 继续看这行代码
   		AbstractBeanDefinition beanDefinition = parseBeanDefinitionElement(ele, beanName, containingBean);
   		if (beanDefinition != null) {
   			if (!StringUtils.hasText(beanName)) {
   				try {
                    //如果没指定beanname，那么根据默认命名规则生成beanname
   					if (containingBean != null) {
   						beanName = BeanDefinitionReaderUtils.generateBeanName(
   								beanDefinition, this.readerContext.getRegistry(), true);
   					}
   					else {
   						beanName = this.readerContext.generateBeanName(beanDefinition);
   						// Register an alias for the plain bean class name, if still possible,
   						// if the generator returned the class name plus a suffix.
   						// This is expected for Spring 1.2/2.0 backwards compatibility.
   						String beanClassName = beanDefinition.getBeanClassName();
   						if (beanClassName != null &&
   								beanName.startsWith(beanClassName) && beanName.length() > beanClassName.length() &&
   								!this.readerContext.getRegistry().isBeanNameInUse(beanClassName)) {
   							aliases.add(beanClassName);
   						}
   					}
   					if (logger.isDebugEnabled()) {
   						logger.debug("Neither XML 'id' nor 'name' specified - " +
   								"using generated bean name [" + beanName + "]");
   					}
   				}
   				catch (Exception ex) {
   					error(ex.getMessage(), ele);
   					return null;
   				}
   			}
   			String[] aliasesArray = StringUtils.toStringArray(aliases);
   			return new BeanDefinitionHolder(beanDefinition, beanName, aliasesArray);
   		}
   
   		return null;
   	}
//进一步解析其它所有属性并封装到GenericBeanDefinition对象中
   	public AbstractBeanDefinition parseBeanDefinitionElement(
   			Element ele, String beanName, @Nullable BeanDefinition containingBean) {
   
   		this.parseState.push(new BeanEntry(beanName));
   
   		String className = null;
           //解析class属性
   		if (ele.hasAttribute(CLASS_ATTRIBUTE)) {
   			className = ele.getAttribute(CLASS_ATTRIBUTE).trim();
   		}
   		String parent = null;
           //解析parent属性
   		if (ele.hasAttribute(PARENT_ATTRIBUTE)) {
   			parent = ele.getAttribute(PARENT_ATTRIBUTE);
   		}
   
   		try {
               //创建用于承载属性的AbstractBeanDefinition类型的GenericBeanDefinition
   			AbstractBeanDefinition bd = createBeanDefinition(className, parent);
   			//硬编码解析bean的各种特性 
   			parseBeanDefinitionAttributes(ele, beanName, containingBean, bd);
   			bd.setDescription(DomUtils.getChildElementValueByTagName(ele, DESCRIPTION_ELEMENT));
   			//下面都是各种解析......省略
   			parseMetaElements(ele, bd);
   			parseLookupOverrideSubElements(ele, bd.getMethodOverrides());
   			parseReplacedMethodSubElements(ele, bd.getMethodOverrides());
   
   			parseConstructorArgElements(ele, bd);
   			parsePropertyElements(ele, bd);
   			parseQualifierElements(ele, bd);
   
   			bd.setResource(this.readerContext.getResource());
   			bd.setSource(extractSource(ele));
   
   			return bd;
   		}
           //.....
   
   		return null;
   	}
```

简单概括一下DefaultBeanDefinitionDocumentReader对象processBeanDefinition方法中第一行代码的作用：

1. 解析元素中的id以及name属性
2. 进一步解析其它所有属性并封装到GenericBeanDefinition对象中
3. 如果检查到bean没有指定beanName，那么根据默认的规则为此bean生成beanName
4. 将获取到的信息封装到BeanDefinitionHolder实例中



源码再回到DefaultBeanDefinitionDocumentReader类中processBeanDefinition方法中的后面的代码，也就是 **\\\3. 注册最终的实例 所以这里是关键 点进去看 go**这一行：

```java
   //BeanDefinitionReaderUtils.java 这个类看名字都能看出来是注册BeanDefinition的工具类
   	public static void registerBeanDefinition(
   			BeanDefinitionHolder definitionHolder, BeanDefinitionRegistry registry)
   			throws BeanDefinitionStoreException {
   
   		// 使用beanname做唯一标识
   		String beanName = definitionHolder.getBeanName();
           //这里回到DefaultListableBeanFactory的registerBeanDefinition方法 go
   		registry.registerBeanDefinition(beanName, definitionHolder.getBeanDefinition());
   
   		// 注册所有别名
   		String[] aliases = definitionHolder.getAliases();
   		if (aliases != null) {
   			for (String alias : aliases) {
   				registry.registerAlias(beanName, alias);
   			}
   		}
   	}
   //DefaultListableBeanFactory.java
   	public void registerBeanDefinition(String beanName, BeanDefinition beanDefinition)
   			throws BeanDefinitionStoreException {
   
   		Assert.hasText(beanName, "Bean name must not be empty");
   		Assert.notNull(beanDefinition, "BeanDefinition must not be null");
   
   		if (beanDefinition instanceof AbstractBeanDefinition) {
   			try {
                   //参数最后的校验 主要对AbstractBeanDefinition属性中的methodOverrides校验
   				((AbstractBeanDefinition) beanDefinition).validate();
   			}
   			catch (BeanDefinitionValidationException ex) {
   				throw new BeanDefinitionStoreException(beanDefinition.getResourceDescription(), beanName,
   						"Validation of bean definition failed", ex);
   			}
   		}
   		//从ConcurrentHashMap中查看是否已经注册过该bean
   		BeanDefinition existingDefinition = this.beanDefinitionMap.get(beanName);
           //1. 如果不等于null那说明之前存在了相同的BeanDefinition 进行相应处理 不允许覆盖则直接报错
   		if (existingDefinition != null) {
   			if (!isAllowBeanDefinitionOverriding()) {
   				throw new BeanDefinitionStoreException(beanDefinition.getResourceDescription(), beanName,
   						"Cannot register bean definition [" + beanDefinition + "] for bean '" + beanName +
   						"': There is already [" + existingDefinition + "] bound.");
   			}
   			else if (existingDefinition.getRole() < beanDefinition.getRole()) {
   				// e.g. was ROLE_APPLICATION, now overriding with ROLE_SUPPORT or ROLE_INFRASTRUCTURE
   				if (logger.isWarnEnabled()) {
   					logger.warn("Overriding user-defined bean definition for bean '" + beanName +
   							"' with a framework-generated bean definition: replacing [" +
   							existingDefinition + "] with [" + beanDefinition + "]");
   				}
   			}
   			else if (!beanDefinition.equals(existingDefinition)) {
   				if (logger.isInfoEnabled()) {
   					logger.info("Overriding bean definition for bean '" + beanName +
   							"' with a different definition: replacing [" + existingDefinition +
   							"] with [" + beanDefinition + "]");
   				}
   			}
   			else {
   				if (logger.isDebugEnabled()) {
   					logger.debug("Overriding bean definition for bean '" + beanName +
   							"' with an equivalent definition: replacing [" + existingDefinition +
   							"] with [" + beanDefinition + "]");
   				}
   			}
   			this.beanDefinitionMap.put(beanName, beanDefinition);
   		}
   		else {
               //进到这里说明是第一次加载这个BeanDefinition
               
               //检查该工厂的bean创建阶段是否已经开始 也就是检查该工厂的Bean创建阶段是否已经开始，即，在此期间是否已将任何Bean标记为已创建
               //第一次getbean的时候会将beanname加入到alreadyCreated集合中去
   			if (hasBeanCreationStarted()) {
   				// Cannot modify startup-time collection elements anymore (for stable iteration)
                   //已经加载过bean则需要通过锁来避免全局访问情况 
   				synchronized (this.beanDefinitionMap) {
   					this.beanDefinitionMap.put(beanName, beanDefinition);
   					List<String> updatedDefinitions = new ArrayList<>(this.beanDefinitionNames.size() + 1);
   					updatedDefinitions.addAll(this.beanDefinitionNames);
   					updatedDefinitions.add(beanName);
   					this.beanDefinitionNames = updatedDefinitions;
   					if (this.manualSingletonNames.contains(beanName)) {
   						Set<String> updatedSingletons = new LinkedHashSet<>(this.manualSingletonNames);
   						updatedSingletons.remove(beanName);
   						this.manualSingletonNames = updatedSingletons;
   					}
   				}
   			}
   			else {
   				//工厂仍处于注册阶段则直接往HashMap里面插数据即可 记录beanName
   				this.beanDefinitionMap.put(beanName, beanDefinition);
   				this.beanDefinitionNames.add(beanName);
   				this.manualSingletonNames.remove(beanName);
   			}
   			this.frozenBeanDefinitionNames = null;
   		}
   
   		if (existingDefinition != null || containsSingleton(beanName)) {
               //清缓存
   			resetBeanDefinition(beanName);
   		}
   		else if (isConfigurationFrozen()) {
   			clearByTypeCache();
   		}
   	}
```

上面代码比较多，概括如下：

1. 对AbstractBeanDefinition的校验。
2. 对beanName已经注册的情况处理
3. 加入map缓存
4. 清楚解析之前留下的对于beanName的缓存

**注**：上面源码的if分支if(hasBeanCreationStarted())这个代码块我也不是很理解，我源码点进去看是有一个Set集合存放已经创建的bean，但是要在从BeanFactory里面getbean()的时候才会把bean加入这个集合，不是很能理解这个分支，不太理解为什么进入这个分支需要加锁,有理解的可以交流一下。















