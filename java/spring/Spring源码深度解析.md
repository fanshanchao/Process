# 核心实现

架构图：

![](E:\拎着自己飞呀\笔记本\images\Spring架构图.png)



## 容器的基本实现

### 核心类DefaultListableBeanFactory

利用idea查看出这个类的继承关系图如下：

![](E:\拎着自己飞呀\笔记本\images\DefaultListableBeanFactory继承图.png)

### 读取XML中的bean

#### 1. 核心类XmlBeanFactory

```
XmlBeanFactory继承自DefaultListableBeanFactory，是读取XML配置的关键，而XmlBeanFactory类里面有一个关键类XmlBeanDefinitionReader，这个类用来读取资源文件，解析及注册xml中的bean。
```

#### 2. XmlBeanDefinitionReader

XmlBeanDefinitionReader的大概流程如下：

1. 通过继承自XmlBeanDefinitionReader中的方法，使用ResourLoader将资源文件路径转换为对于的Resource文件。
2. 通过DocumentLoader对Resource文件进行转换，将Reousrce转换为Document
3. 对Document进行解析，注册bean信息。

#### 3. XmlBeanFactory的初始化过程

1. xml文件封装到Resource中。
2. 加载Reousrce中的bean并返回加载bean 的数量

#### 4. 从代码分析bean的解析和注册

1. 从第一行代码开始：

   ```java
   //XmlBeanFactory.java
   	public XmlBeanFactory(Resource resource, BeanFactory parentBeanFactory) throws BeansException {
   		super(parentBeanFactory);
           //核心 reader是XmlBeanDefinitionReader，这个类用来解析bean
   		this.reader.loadBeanDefinitions(resource);
   	}
   //XmlBeanDefinitionReader.java
   	public int loadBeanDefinitions(EncodedResource encodedResource) throws BeanDefinitionStoreException {
   		//传进来的EncodedResource是编码后的Resource
           //....省略
   		//获取到当前线程加载的资源 currentResources是ThreadLocal
   		Set<EncodedResource> currentResources = this.resourcesCurrentlyBeingLoaded.get();
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
                   //下一步核心
   				return doLoadBeanDefinitions(inputSource, encodedResource.getResource());
   			}
   			finally {
   				inputStream.close();
   			}
   		}
   		catch (IOException ex) {
   			throw new BeanDefinitionStoreException(
   					"IOException parsing XML document from " + encodedResource.getResource(), ex);
   		}
   		finally {
   			currentResources.remove(encodedResource);
   			if (currentResources.isEmpty()) {
   				this.resourcesCurrentlyBeingLoaded.remove();
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
           //BeanDefinitionDocumentReader对象 就是这个接口的子类DefaultBeanDefinitionDocumentReader
   		BeanDefinitionDocumentReader documentReader = createBeanDefinitionDocumentReader();
           //获取到当前BeanFactory注册的数量 这里可以点进去看下
           //这里使用的是BeanDefinitionRegistry对象，而DefaultListableBeanFactory便实现了这个接口，所以getRegistry()就是获取最开始的XmlBeanFactory
           //接下来再看getBeanDefinitionCount()方法 
   		int countBefore = getRegistry().getBeanDefinitionCount();
           //注册Document中的Bean 再点进去看
   		documentReader.registerBeanDefinitions(doc, createReaderContext(resource));
   		return getRegistry().getBeanDefinitionCount() - countBefore;
   	}
   //DefaultListableBeanFactory.java
   	@Override
   	public int getBeanDefinitionCount() {
   		return this.beanDefinitionMap.size();
   	}
   //继续看这个属性 所以我们可以得到一个结论DefaultListableBeanFactory使用ConcurrentHashMap保存BeanDefinition
   private final Map<String, BeanDefinition> beanDefinitionMap = new ConcurrentHashMap<>(256);
   
   //DefaultBeanDefinitionDocumentReader.java
   	@Override
   	public void registerBeanDefinitions(Document doc, XmlReaderContext readerContext) {
   		this.readerContext = readerContext;
   		logger.debug("Loading bean definitions");
           //获取到根元素
   		Element root = doc.getDocumentElement();
           //从根元素开始解析
   		doRegisterBeanDefinitions(root);
   	}
   
   
   	protected void doRegisterBeanDefinitions(Element root) {
           //BeanDefinitionParserDelegate类用来解析bean元素
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
           //解析
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
                           //使用默认方式解析并注册bean
   						parseDefaultElement(ele, delegate);
   					}
   					else {
   						delegate.parseCustomElement(ele);
   					}
   				}
   			}
   		}
   		else {
   			delegate.parseCustomElement(root);
   		}
   	}
   
   	private void parseDefaultElement(Element ele, BeanDefinitionParserDelegate delegate) {
   		if (delegate.nodeNameEquals(ele, IMPORT_ELEMENT)) {
   			importBeanDefinitionResource(ele);
   		}
   		else if (delegate.nodeNameEquals(ele, ALIAS_ELEMENT)) {
   			processAliasRegistration(ele);
   		}
           //直接看bean元素即可
   		else if (delegate.nodeNameEquals(ele, BEAN_ELEMENT)) {
   			processBeanDefinition(ele, delegate);
   		}
   		else if (delegate.nodeNameEquals(ele, NESTED_BEANS_ELEMENT)) {
   			// recurse
   			doRegisterBeanDefinitions(ele);
   		}
   	}
   
   	protected void processBeanDefinition(Element ele, BeanDefinitionParserDelegate delegate) {
           //1. 解析Bean元素 bdHolder对象已经包括了我们配置文件中的各种属性了 这里面就是层传剥皮 go
   		BeanDefinitionHolder bdHolder = delegate.parseBeanDefinitionElement(ele);
           //不为空则继续解析自定义属性
   		if (bdHolder != null) {
               //2. 根据需要对beanDefinition进行装饰 bean使用的是默认标签配置，但是子元素使用了自定义的配置时这句代码便有用了
   			bdHolder = delegate.decorateBeanDefinitionIfRequired(ele, bdHolder);
   			try {
   				//2. 注册最终的实例 所以这里是关键 点进去看 go
   				BeanDefinitionReaderUtils.registerBeanDefinition(bdHolder, getReaderContext().getRegistry());
   			}
   			catch (BeanDefinitionStoreException ex) {
   				getReaderContext().error("Failed to register bean definition with name '" +
   						bdHolder.getBeanName() + "'", ele, ex);
   			}
   			// 发送注册事件
   			getReaderContext().fireComponentRegistered(new BeanComponentDefinition(bdHolder));
   		}
   	}
   ```

   
#### 5. BeanDefintion接口

该接口存在三种实现：RootBeanDefinition，ChildBeanDefinition，GenericBeanDefinition。

![](E:\拎着自己飞呀\笔记本\images\BeanDefinition接口继承图.png)

BeanDefintion是配置文件<bean>元素标签在容器内部表示形式。父<bean>用RootBeanDefinition表示，子<bean>用ChildBeanDefinition表示，GenericBeanDefinition是2.5以后加入的bean文件配置属性定义类。

**Spring通过BeanDefinition将配置文件中的bean元素配置信息转换为容器内部表示，并将这些BeanDefinition注册到BeanDefintionRegistry中。**BeanDefintionRegistry就像内存数据库。

#### 6. 从代码分析bean的解析和注册（2）

   ```java
   //BeanDefinitionParserDelegate.java
   //继续看下spring是如何使用BeanDefinitionParserDelegate层层剥皮解析xml的
   	@Nullable
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
           //上面讲到过的AbstractBeanDefinition go
   		AbstractBeanDefinition beanDefinition = parseBeanDefinitionElement(ele, beanName, containingBean);
   		if (beanDefinition != null) {
   			if (!StringUtils.hasText(beanName)) {
   				try {
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
   //接下来又回到processBeanDefinition方法中的BeanDefinitionReaderUtils.registerBeanDefinition(bdHolder, getReaderContext().getRegistry()); 对beanDefinition进行注册
   //BeanDefinitionReaderUtils.java
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
   				//工厂仍处于注册阶段则直接往HashMap里面插数据即可
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

   

   ### Bean的加载

new xmlBeanFactory()这行代码会完成对xml的解析和bean的注册。xmlBeanFactory调用getBean()的时候才会开始加载bean。

```java
//AbstractBeanFactory.java getBean()会调用doGetBean
	protected <T> T doGetBean(
			String name, @Nullable Class<T> requiredType, @Nullable Object[] args, boolean typeCheckOnly)
			throws BeansException {
		//获取对应的beanName
		String beanName = transformedBeanName(name);
		Object bean;

		/* 检查缓存中或者实例工厂中是否有对应的实例
		*  需要检查的原因在于，创建单例bean的原则时候会存在依赖注入的情况，而在创建依赖的时候为了避免循环依赖，Spring在创建bean的原则是不等bean创建完成就会将创建的bean的ObjectFactory提早曝光，也就是将ObjectFactory加入到缓存中，一旦下个bean创建时候需要依赖上个bean则直接使用
		*/
        //直接尝试从缓存或者singletonFactories的ObjectFactory中获取、
        //ObjectFactory为了解决循环依赖问题，提早暴露的一个半成品bean
		Object sharedInstance = getSingleton(beanName);
		if (sharedInstance != null && args == null) {
			if (logger.isDebugEnabled()) {
				if (isSingletonCurrentlyInCreation(beanName)) {
					logger.debug("Returning eagerly cached instance of singleton bean '" + beanName +
							"' that is not fully initialized yet - a consequence of a circular reference");
				}
				else {
					logger.debug("Returning cached instance of singleton bean '" + beanName + "'");
				}
			}
            //对从缓存中获取到的原始状态bean进行初始化
			bean = getObjectForBeanInstance(sharedInstance, name, beanName, null);
		}

		else {
            //只有在单例情况下才尝试解决循环依赖，原型模式下，如果存在A中有B的属性，B中有A的属性，依赖注入时会产生循环依赖问题，也就是下面的情况
			if (isPrototypeCurrentlyInCreation(beanName)) {
				throw new BeanCurrentlyInCreationException(beanName);
			}

			// Check if bean definition exists in this factory.
			BeanFactory parentBeanFactory = getParentBeanFactory();
            //如果beanDefinitionMap中所有已经加载的类不包括beanName则尝试从parentBeanFactory中检测
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
				else {
					// No args -> delegate to standard getBean method.
					return parentBeanFactory.getBean(nameToLookup, requiredType);
				}
			}
			//如果不是仅仅做类型检查时创建bean，这里要进行记录
			if (!typeCheckOnly) {
				markBeanAsCreated(beanName);
			}
	
			try {
                //将GernericBeanDefinition转换为RootBeanDefinition
				RootBeanDefinition mbd = getMergedLocalBeanDefinition(beanName);
				checkMergedBeanDefinition(mbd, beanName, args);

				// Guarantee initialization of beans that the current bean depends on.
				String[] dependsOn = mbd.getDependsOn();
                //如果存在依赖则需要递归实例化依赖的bean
				if (dependsOn != null) {
					for (String dep : dependsOn) {
						if (isDependent(beanName, dep)) {
							throw new BeanCreationException(mbd.getResourceDescription(), beanName,
									"Circular depends-on relationship between '" + beanName + "' and '" + dep + "'");
						}
                        //缓存依赖调用
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

				// 实例化依赖的bean后便可以实例化mdb本身了
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
                    //
					bean = getObjectForBeanInstance(sharedInstance, name, beanName, mbd);
				}
				//prototype模式的创建 ？？？？（原型模型）
				else if (mbd.isPrototype()) {
					// It's a prototype -> create a new instance.
					Object prototypeInstance = null;
					try {
						beforePrototypeCreation(beanName);
						prototypeInstance = createBean(beanName, mbd, args);
					}
					finally {
						afterPrototypeCreation(beanName);
					}
					bean = getObjectForBeanInstance(prototypeInstance, name, beanName, mbd);
				}

				else {
                    //指定的scope上实例化bean
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
        //检查需要的类型是否符合bean的实际类型
		if (requiredType != null && !requiredType.isInstance(bean)) {
			try {
				T convertedBean = getTypeConverter().convertIfNecessary(bean, requiredType);
				if (convertedBean == null) {
					throw new BeanNotOfRequiredTypeException(name, requiredType, bean.getClass());
				}
				return convertedBean;
			}
			catch (TypeMismatchException ex) {
				if (logger.isDebugEnabled()) {
					logger.debug("Failed to convert bean '" + name + "' to required type '" +
							ClassUtils.getQualifiedName(requiredType) + "'", ex);
				}
				throw new BeanNotOfRequiredTypeException(name, requiredType, bean.getClass());
			}
		}
		return (T) bean;
	}
```

上面代码大概步骤:

1. 转换对应的beanName
2. 尝试从缓存中加载单例
3. bean的实例化
4. 原型模型的依赖检查
5. 检测parentFactory
6. 将存储XML配置文件的GernericBeanDefinition转换为RootBeanDefinition
7. 寻找依赖
8. 针对不同的scope进行bean的创建
9. 类型转换

下面接着分析每一个步骤



#### FactoryBean

有时候我们需要创建一个复制的bean，为了避免编写大量的配置，可以实现FactoryBean接口。FactoryBean接口里面有个getObject方法，将创建bean的逻辑写在这个方法里面。



#### 从缓存中获取bean

接着上面的代码：getSingleton(beanName)

```java
//DefaultSingletonBeanRegistry.java	
public Object getSingleton(String beanName) {
    	//设置为true允许早期依赖
		return getSingleton(beanName, true);
	}
	protected Object getSingleton(String beanName, boolean allowEarlyReference) {
        //检查缓存中是否存在实例 是一个ConcurrentHashMap singletonObjects用于保存BeanName和创建bean实例之间的关系bean name --> bean instance
		Object singletonObject = this.singletonObjects.get(beanName);
        //如果不存在，且当前单例正在创建中 那么进行同步器锁内创建bean
		if (singletonObject == null && isSingletonCurrentlyInCreation(beanName)) {
			synchronized (this.singletonObjects) {
                //earlySingletonObjects（HashMap）也是保存BeanName和创建bean实例之间的关系，不同于singletonObjects，当一个单例bean被放到这里后，那么当bean还在创建过程中，就可以通过getBean获取到了，其目的是用来检测循环依赖
				singletonObject = this.earlySingletonObjects.get(beanName);
                //仍然获取不到则进入下一步 为了处理循环依赖问题
				if (singletonObject == null && allowEarlyReference) {
                    //singletonFactories（HashMap）保存了BeanName和创建bean的工厂直接的关系，bean name --> ObjectFactory
					ObjectFactory<?> singletonFactory = this.singletonFactories.get(beanName);
					if (singletonFactory != null) {
                        //调用预设的getObject方法
						singletonObject = singletonFactory.getObject();
						this.earlySingletonObjects.put(beanName, singletonObject);
                        //移除这个beanName的ObjectFactory
						this.singletonFactories.remove(beanName);
					}
				}
			}
		}
		return singletonObject;
	}
```

#### 从bean的实例中获取对象

也就是bean加载代码中的getObjectForBeanInstance()方法。无论是从缓存中获得的bean还是根据不同的scope策略加载bean。我们得到bean的实例后要做的第一件事就是调用这个方法来检测一下正确性，其实就是检测当前bean是否是FactoryBean类型的bean。

```java
//AbstractBeanFactory.java
	protected Object getObjectForBeanInstance(
			Object beanInstance, String name, String beanName, @Nullable RootBeanDefinition mbd) {


        //翻译过来就是 如果执行的name是工厂相关(以&为前缀)且beanInstance又不是Factory类型则校验不通过
		if (BeanFactoryUtils.isFactoryDereference(name)) {
			if (beanInstance instanceof NullBean) {
				return beanInstance;
			}
			if (!(beanInstance instanceof FactoryBean)) {
				throw new BeanIsNotAFactoryException(beanName, beanInstance.getClass());
			}
		}

        //这里的意思就是，如果当前bean不是FactoryBean，那么直接返回即可
		if (!(beanInstance instanceof FactoryBean) || BeanFactoryUtils.isFactoryDereference(name)) {
			return beanInstance;
		}

        //下面就是对FactoryBean进行加载操作了
		Object object = null;
		if (mbd == null) {
            //从缓存中加载？？为什么还要从缓存中加载
            //答：这里是FactoryBean的缓存，和上面的缓存不是一个东西
            
			object = getCachedObjectForFactoryBean(beanName);
		}
		if (object == null) {
			// 到这里可以肯定类型是FactoryBean了
			FactoryBean<?> factory = (FactoryBean<?>) beanInstance;
			// 这里就是看DefaultListableBeanFactory的beanDefinitionMap中是否有BeanDefinition
			if (mbd == null && containsBeanDefinition(beanName)) {
                //将存储XML配置文件的GernericBeanDefinition转换为RootBeanDefinition
				mbd = getMergedLocalBeanDefinition(beanName);
			}
            //是否是用户定义而不是程序自定义
			boolean synthetic = (mbd != null && mbd.isSynthetic());
            //关键就在这里
			object = getObjectFromFactoryBean(factory, beanName, !synthetic);
		}
		return object;
	}
```

上面代码的步骤：

1. 对FactoryBean的正确校验
2. 对非FactoryBean不做任何处理
3. 对bean进行转行
4. **将从Factory中解析bean 的操作委托给getObjectFromFactoryBean方法**

继续看上面的步骤4：

```java
//FactoryBeanRegistrySupport.java
	protected Object getObjectFromFactoryBean(FactoryBean<?> factory, String beanName, boolean shouldPostProcess) {
        //针对单例bean进行处理
		if (factory.isSingleton() && containsSingleton(beanName)) {
			synchronized (getSingletonMutex()) {
                //这里从缓存中获取FactoryBean和上面的那个源码中的缓存是同一个缓存
                //难道因为是缓存，所以要多获取几次?
				Object object = this.factoryBeanObjectCache.get(beanName);
				if (object == null) {
                    //这里非常关键，看方法名都能看出来doGetObjectFromFactoryBean，根据FactoryBean的getObject方法来创建bean
					object = doGetObjectFromFactoryBean(factory, beanName);
					// Only post-process and store if not put there already during getObject() call above
					// (e.g. because of circular reference processing triggered by custom getBean calls)
					Object alreadyThere = this.factoryBeanObjectCache.get(beanName);
					if (alreadyThere != null) {
						object = alreadyThere;
					}
					else {
						if (shouldPostProcess) {
							if (isSingletonCurrentlyInCreation(beanName)) {
								// Temporarily return non-post-processed object, not storing it yet..
								return object;
							}
                            //调用bean的前置处理器
							beforeSingletonCreation(beanName);
							try {
								object = postProcessObjectFromFactoryBean(object, beanName);
							}
							catch (Throwable ex) {
								throw new BeanCreationException(beanName,
										"Post-processing of FactoryBean's singleton object failed", ex);
							}
							finally {
                                //调用bean的额后置处理器
								afterSingletonCreation(beanName);
							}
						}
						if (containsSingleton(beanName)) {
                            //最终放入到缓存中去
							this.factoryBeanObjectCache.put(beanName, object);
						}
					}
				}
				return object;
			}
		}
		else {
			Object object = doGetObjectFromFactoryBean(factory, beanName);
			if (shouldPostProcess) {
				try {
					object = postProcessObjectFromFactoryBean(object, beanName);
				}
				catch (Throwable ex) {
					throw new BeanCreationException(beanName, "Post-processing of FactoryBean's object failed", ex);
				}
			}
			return object;
		}
	}
```

#### 创建Bean

上面这一段都是AbstractBeanFactory.doGetBean中从缓存中获取单例的过程，如果缓存中不存在已经加载的单例，就需要重新加载bean，而加载bean的代码就在这个方法下面，也就是getSingleton的重载方法，我们接着看这个方法是怎么实现bean加载过程的。

```java
//DefaultSingletonBeanRegistry.java
	public Object getSingleton(String beanName, ObjectFactory<?> singletonFactory) {
		Assert.notNull(beanName, "Bean name must not be null");
        //全局变量需要同步
		synchronized (this.singletonObjects) {
            //查看是否已经加载过
			Object singletonObject = this.singletonObjects.get(beanName);
            //没有再进行加载操作
			if (singletonObject == null) {
				if (this.singletonsCurrentlyInDestruction) {
					throw new BeanCreationNotAllowedException(beanName,
							"Singleton bean creation not allowed while singletons of this factory are in destruction " +
							"(Do not request a bean from a BeanFactory in a destroy method implementation!)");
				}
				if (logger.isDebugEnabled()) {
					logger.debug("Creating shared instance of singleton bean '" + beanName + "'");
				}
                //这里做了一个很重要的操作：记录加载状态，也就是将当前正在创建的bean记录在缓存中，然后就可以对循环依赖进行检测
				beforeSingletonCreation(beanName);
				boolean newSingleton = false;
				boolean recordSuppressedExceptions = (this.suppressedExceptions == null);
				if (recordSuppressedExceptions) {
					this.suppressedExceptions = new LinkedHashSet<>();
				}
				try {
                    //加载bean 这里的核心代码还要跳回到前面的传参进行来的createBean方法中去
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
                    //创建后要做的事 其实就是移除前面beforeSingletonCreation保存的加载状态
					afterSingletonCreation(beanName);
				}
				if (newSingleton) {
                    //加入缓存 这里其实就是往前面getSingleton(beanName)方法中的那几个Map插数据
					addSingleton(beanName, singletonObject);
				}
			}
			return singletonObject;
		}
	}
```

上面代码的核心逻辑：

1. 检查缓存是否已经加载过
2. 若没有加载，则记录加载状态
3. 加载单例前记录加载状态 ？？？
4. 通过调用参数传入的ObjectFactory的个体的getObejct方法实例化bean
5. 加载单例后处理方法调用
6. 将结果记录到缓存并删除加载bean过程中所记录的各种辅助状态
7. 返回结果





前面源码中也讲到，核心创建逻辑在createBean方法中，所以我们应该回到createBean方法去看bean的创建过程

```java
//AbstractAutowireCapableBeanFactory.java
	protected Object createBean(String beanName, RootBeanDefinition mbd, @Nullable Object[] args)
			throws BeanCreationException {

		if (logger.isDebugEnabled()) {
			logger.debug("Creating instance of bean '" + beanName + "'");
		}
		RootBeanDefinition mbdToUse = mbd;

		//锁定class，根据设置的class属性或者根据className来解析Class
		Class<?> resolvedClass = resolveBeanClass(mbd, beanName);
		if (resolvedClass != null && !mbd.hasBeanClass() && mbd.getBeanClassName() != null) {
			mbdToUse = new RootBeanDefinition(mbd);
			mbdToUse.setBeanClass(resolvedClass);
		}

		// Prepare method overrides.
		try {
            //验证及准备覆盖的方法 
            //这里其实就是针对lookup-method和replace-method的
			mbdToUse.prepareMethodOverrides();
		}
		catch (BeanDefinitionValidationException ex) {
			throw new BeanDefinitionStoreException(mbdToUse.getResourceDescription(),
					beanName, "Validation of method overrides failed", ex);
		}

		try {
			//给BeanPostProcessors一个机会来返回代理来替代真正的实例
			Object bean = resolveBeforeInstantiation(beanName, mbdToUse);
            //如果前置处理解析BeanDefinition可以创建bean，那么直接返回即可
			if (bean != null) {
				return bean;
			}
		}
		catch (Throwable ex) {
			throw new BeanCreationException(mbdToUse.getResourceDescription(), beanName,
					"BeanPostProcessor before instantiation of bean failed", ex);
		}

		try {
            //这里才是真正创建bean的关键
			Object beanInstance = doCreateBean(beanName, mbdToUse, args);
			if (logger.isDebugEnabled()) {
				logger.debug("Finished creating instance of bean '" + beanName + "'");
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

上面代码的核心逻辑：

1. 根据设置的class属性或者根据className类解析Class

2. 对override属性进行标记及验证

3. 应用初始化前的后处理器，解析指定bean是否存在初始化前的断路操作

4. **创建bean**

   

   #### 循环依赖

   ##### 构造器循环依赖

   ​	此类依赖是无法解决的，只能抛出BeanCurrentlyInCreationException。

   ​	Spring容器将每一个正在创建的bean标识符放在一个**当前创建bean池**中，bean标识符会在创建过程中一直保持在这个池中，因此如果在创建bean过程中发现自己已经在这个标识符池里，就会抛出异常表示循环依赖，而对于创建完毕的bean将从**当前创建bean池**中清除掉。

   

   ##### setter循环依赖

   ​	对于setter注入造成的依赖是通过Spring容器提交暴露刚完成构造器注入但未完成其他步骤（如setter注入）的bean来完成的，而且只能解决单例作用域的循环依赖。

   

   ##### prototype范围的依赖处理

   ​	对于prototype作用域bean，Spring容器无法完成依赖注入，因为Spring容器不进行缓存prototype作用域的bean，因此无法提前暴露一个创建中的bean。

   

   #### 创建Bean doCreateBean方法

   ​	接下来进行上面的源码分析，doCreateBean方法，接着常规bean的创建。

   ```java
   //AbstractAutowireCapableBeanFactory.java
   	protected Object doCreateBean(String beanName, RootBeanDefinition mbd, @Nullable Object[] args)
   			throws BeanCreationException {
   
   		// Instantiate the bean.
   		BeanWrapper instanceWrapper = null;
   		if (mbd.isSingleton()) {
               //不太理解这里为什么要去FactoryBean的缓存中清除一下，防止已经生成了FactoryBean？？
   			instanceWrapper = this.factoryBeanInstanceCache.remove(beanName);
   		}
   		if (instanceWrapper == null) {
               //根据指定bean使用对于的策略创建新的实例。例如工厂方法，构造函数自动注入
   			instanceWrapper = createBeanInstance(beanName, mbd, args);
   		}
   		Object bean = instanceWrapper.getWrappedInstance();
   		Class<?> beanType = instanceWrapper.getWrappedClass();
   		if (beanType != NullBean.class) {
   			mbd.resolvedTargetType = beanType;
   		}
   
   		// Allow post-processors to modify the merged bean definition.
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
   		
           //判断是否需要提前曝光：单例&允许循环依赖&当前bean正在创建中，检测循环依赖
   		boolean earlySingletonExposure = (mbd.isSingleton() && this.allowCircularReferences &&
   				isSingletonCurrentlyInCreation(beanName));
   		if (earlySingletonExposure) {
   			if (logger.isDebugEnabled()) {
   				logger.debug("Eagerly caching bean '" + beanName +
   						"' to allow for resolving potential circular references");
   			}
               //为了避免后期循环依赖，可以在bean初始化完成前将创建实例的ObjectFactory加入工厂
               //这里就已经暴露出了ObjectFactory
               //AOP就是在getEarlyBeanReference这里将advice动态织入bean中
   			addSingletonFactory(beanName, () -> getEarlyBeanReference(beanName, mbd, bean));
   		}
   
   		// Initialize the bean instance.
   		Object exposedObject = bean;
   		try {
               //对bean进行填充，将各个属性注入，其中可以依赖于其他bean的属性，则会递归初始化依赖bean
   			populateBean(beanName, mbd, instanceWrapper);
               //调用初始化方法
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
               //只有检测到有循环依赖才不会为空
   			if (earlySingletonReference != null) 
                   //如果exposedObject没有在初始化方法中被改变，也就是没有被增强
   				if (exposedObject == bean) {
   					exposedObject = earlySingletonReference;
   				}
   				else if (!this.allowRawInjectionDespiteWrapping && hasDependentBean(beanName)) {
   					String[] dependentBeans = getDependentBeans(beanName);
   					Set<String> actualDependentBeans = new LinkedHashSet<>(dependentBeans.length);
   					for (String dependentBean : dependentBeans) {
                           //检测依赖
   						if (!removeSingletonIfCreatedForTypeCheckOnly(dependentBean)) {
   							actualDependentBeans.add(dependentBean);
   						}
   					}
                       /**因为bean创建后其依赖的bean一定是已经创建的
                       * actualDependentBeans不为空则表示当前bean创建后其依赖的bean却没有被全部创建完，也就是存在循环依赖
                       */
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
               //根据scope注册bean
   			registerDisposableBeanIfNecessary(beanName, bean, mbd);
   		}
   		catch (BeanDefinitionValidationException ex) {
   			throw new BeanCreationException(
   					mbd.getResourceDescription(), beanName, "Invalid destruction signature", ex);
   		}
   
   		return exposedObject;
   	}
   ```

   分析下上面的逻辑：

   1. 如果是单例bean就需要先清除缓存
   2. 实例化bean，将BeanDefinnition转行为BeanWrapper
      	* 如果存在工厂方法则使用工厂方法进行初始化（bean的factory-method指定的方法）
         	* 一个类有多个构造方法，选择合适的构造方法进行初始化
            	* 上面两种情况都不存在则使用默认的构造函数进行bean的实例化
   3. MergedBeanDefinitionPostProcessors的应用，也就是bean合并后的处理
   4. 依赖处理
   5. 属性填充
   6. 循环依赖检查：因为解决循环依赖只对单例bean有效，所以这里发现非单例bean出现循环依赖就抛出异常
   7. 注册DisposableBean，如果配置了destroy-method，这里需要注册以便后面销毁时使用
   8. 完成创建并返回

   ##### 创建bean的实例

   ​	首先分析下bean的实例是如何创建的，也就是上面的步骤2，也就是createBeanInstance方法：

   ```java
   //AbstractAutowireCapableBeanFactory.java	
   protected BeanWrapper createBeanInstance(String beanName, RootBeanDefinition mbd, @Nullable Object[] args) {
   		// Make sure bean class is actually resolved at this point.
   		Class<?> beanClass = resolveBeanClass(mbd, beanName);
   
   		if (beanClass != null && !Modifier.isPublic(beanClass.getModifiers()) && !mbd.isNonPublicAccessAllowed()) {
   			throw new BeanCreationException(mbd.getResourceDescription(), beanName,
   					"Bean class isn't public, and non-public access not allowed: " + beanClass.getName());
   		}
   		//下面这段代码暂时还没弄清楚是做什么的 ???
       	//看方法注释，返回创建bean的回调
   		Supplier<?> instanceSupplier = mbd.getInstanceSupplier();
   		if (instanceSupplier != null) {
   			return obtainFromSupplier(instanceSupplier, beanName);
   		}
   		//如果bean配置了factory-method属性，那么就使用工厂方法初始化策略
   		if (mbd.getFactoryMethodName() != null) {
   			return instantiateUsingFactoryMethod(beanName, mbd, args);
   		}
   
   		// Shortcut when re-creating the same bean...
   		boolean resolved = false;
   		boolean autowireNecessary = false;
   		if (args == null) {
   			synchronized (mbd.constructorArgumentLock) {
                   //一个类有多个构造函数，每个构造器都有不同的参数，所以调用前要先锁定构造方法
   				if (mbd.resolvedConstructorOrFactoryMethod != null) {
   					resolved = true;
   					autowireNecessary = mbd.constructorArgumentsResolved;
   				}
   			}
   		}
   		if (resolved) {
   			if (autowireNecessary) {
                   //构造函数自动注入
   				return autowireConstructor(beanName, mbd, null, null);
   			}
   			else {
                   //使用默认构造函数构造
   				return instantiateBean(beanName, mbd);
   			}
   		}
   
   		// 需要根据参数解析构造方法
   		Constructor<?>[] ctors = determineConstructorsFromBeanPostProcessors(beanClass, beanName);
   		if (ctors != null || mbd.getResolvedAutowireMode() == AUTOWIRE_CONSTRUCTOR ||
   				mbd.hasConstructorArgumentValues() || !ObjectUtils.isEmpty(args)) {
               //构造方法自动注入
   			return autowireConstructor(beanName, mbd, ctors, args);
   		}
   
   		// No special handling: simply use no-arg constructor.
       	//使用默认构造方法构造
   		return instantiateBean(beanName, mbd);
   	}
   ```
   
   ​	上面代码逻辑：
   
   1. 判断是否有factory-method属性配置，如果有则使用工厂方法加载类
   2. 锁定构造方法创建bean，并且这里如果构造方法已经被解析过会使用缓存机制，不会重复进行解析。
   
   
   
   