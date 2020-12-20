# Bean的加载

实际上bean是在我们需要去使用的时候才会被加载，也就是下面这行代码执行时才会真正的加载bean。

```jaja
Person person = (Person) beanFactory.getBean("person");
```

追进去里面的代码：

```java
//AbstractBeanFactory.java
protected <T> T doGetBean(
			String name, @Nullable Class<T> requiredType, @Nullable Object[] args, boolean typeCheckOnly)
			throws BeansException {
		//1. 获取相应的beanName
		String beanName = transformedBeanName(name);
		Object bean;

		// Eagerly check singleton cache for manually registered singletons.
    	//这里创建依赖时为了避免循环依赖，Spring创建bean的原则是不等bean的创建完成就会将创建bean的ObjectFactory提早暴露，也就是将ObjectFactory加入到缓存中，一旦下一个bean创建时候需要依赖上个bean则直接使用
    	//直接尝试中缓存中获取或者singletonFactories中的ObjectFactory中获取
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
            //返回对于的实例
			bean = getObjectForBeanInstance(sharedInstance, name, beanName, null);
		}

		else {
            //进入这个分支说明我们正在加载这个bean，而且可能存在循环引用的问题
            
            
            //下面是为了处理原型模式下
			if (isPrototypeCurrentlyInCreation(beanName)) {
				throw new BeanCurrentlyInCreationException(beanName);
			}
			
			// Check if bean definition exists in this factory.
			BeanFactory parentBeanFactory = getParentBeanFactory();
            //如果当前beanDefinitionMap中不包括beanName，则尝试从父工厂中加载
            //这里源码点进去能发现这个map就是之前看过的beanDefinitionMap
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
			//如果不仅仅做类型检查则是创建bean，这里要进行记录
			if (!typeCheckOnly) {
				markBeanAsCreated(beanName);
			}

			try {
				RootBeanDefinition mbd = getMergedLocalBeanDefinition(beanName);
				checkMergedBeanDefinition(mbd, beanName, args);

				// Guarantee initialization of beans that the current bean depends on.
				String[] dependsOn = mbd.getDependsOn();
                //若当前创建的bean存在依赖则需要递归实例化依赖的bean
				if (dependsOn != null) {
					for (String dep : dependsOn) {
						if (isDependent(beanName, dep)) {
							throw new BeanCreationException(mbd.getResourceDescription(), beanName,
									"Circular depends-on relationship between '" + beanName + "' and '" + dep + "'");
						}
                        //循环依赖调用
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

				// 实例化依赖后便可以实例化mdb了，也就是RootBeanDefinition，这里是单例模式
				if (mbd.isSingleton()) {
                    //实例化bean 普通的单例bean，会在这个方法里面生成最终的bean
					sharedInstance = getSingleton(beanName, () -> {
						try {
                            //也就是这里
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
					bean = getObjectForBeanInstance(sharedInstance, name, beanName, mbd);
				}

				else if (mbd.isPrototype()) {
                    //采用原型模型实例化
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
                        //这里才是真正生成bean的地方，之前都是ObjectFactory
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

		// 检查需要的类型是否符合bean的实际类型
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

上面的代码比较复杂。分为如下步骤：

1. 转换对应的beanName
2. 尝试从缓存中加载bean
   * **这里获取到的bean可能是最终的bean，也可能是未初始化的bean，通过earlySingletonObjects提前暴露出来了**
3. 实例化bean，如果从缓存中获取到的是未初始化的bean，那么调用getObjectForBeanInstance方法实例化。
   * **注**：只有FactoryBean才会使用getObjectForBeanInstance方法进行实例化，正常的bean走到里面就return了。:sob:书上关于这里迷惑性也太强了吧。
4. 原型模式的依赖检查
5. 检测parentBeanFactory
6. 将存储XML配置文件的GernericBeanDefinition转换为RootBeanDefintion
7. 寻找依赖
8. 针对不同的scope进行真正bean的创建
9. 类型转换



## 从缓存中获取Bean

```java
//	DefaultSingletonBeanRegistry.java
public Object getSingleton(String beanName) {
		return getSingleton(beanName, true);
	}
	protected Object getSingleton(String beanName, boolean allowEarlyReference) {
        //singletonObjects是一个ConcurrentHashMap 它缓存了bean的ObjectFactory
		Object singletonObject = this.singletonObjects.get(beanName);
        //isSingletonCurrentlyInCreation方法判断当前bean是否正在创建
		if (singletonObject == null && isSingletonCurrentlyInCreation(beanName)) {
			synchronized (this.singletonObjects) {
				singletonObject = this.earlySingletonObjects.get(beanName);
                //如果这个bean正在加载则不做处理 allowEarlyReference为true说明要处理循环依赖
				if (singletonObject == null && allowEarlyReference) {
					ObjectFactory<?> singletonFactory = this.singletonFactories.get(beanName);
					if (singletonFactory != null) {
						singletonObject = singletonFactory.getObject();
						this.earlySingletonObjects.put(beanName, singletonObject);
						this.singletonFactories.remove(beanName);
					}
				}
			}
		}
		return singletonObject;
	}
```

上面的代码其实比较绕，就为了从缓存中拿到ObjectFactory为什么会需要这么多Map？这其实是为了解决循环依赖的问题。先说下几个map的作用，再来分析上面的步骤。

* singletonObjects：保存了beanName与bean之间的关系
* singletonFactories：保存了beanName与它对于ObjectFactory之前的关系
* earlySingletonObjects：用来保存beanName与bean之间的关系。和singletonObjects不同在于，当一个bean在创建过程中就可以被getBean给获取到了，其目的就是用来检测循环依赖。

现在就可以分析上面的步骤了：

1. 首先从singletonObjects中获取bean
2. 上一步没有获取到bean且bean正在创建过程中，就进行下一步（处理循环依赖的问题）
3. 从earlySingletonObjects中获取，没有获取到就进行下一步
4. 利用singletonFactories创建未初始化的bean，最后放入到earlySingletonObjects中去。
   * **这里为什么没有放入到singletonObjects中去？**
   * **答：**放入到singletonObjects中应该是在后面的getObjectForBeanInstance方法去做，因为走到这一步是为了解决循环依赖问题，需要将未初始化的bean通过earlySingletonObjects提前暴露出去。

## 从Bean的实例中获取对象

说明下这个getObjectForBeanInstance方法的作用，如果beanInstance是普通bean实例，那么会直接return，如果是FactoryBean且状态还是未初始化状态，那么会进行这个方法接下来的操作，也就是调用FactoryBean的getObject方法进行FactoryBean 的初始化。

```java
//	AbstractAutowireCapableBeanFactory.java
@Override
	protected Object getObjectForBeanInstance(
			Object beanInstance, String name, String beanName, @Nullable RootBeanDefinition mbd) {

		String currentlyCreatedBean = this.currentlyCreatedBean.get();
        //疑惑： 后面这里再看下
		if (currentlyCreatedBean != null) {
			registerDependentBean(beanName, currentlyCreatedBean);
		}

		return super.getObjectForBeanInstance(beanInstance, name, beanName, mbd);
	}
	protected Object getObjectForBeanInstance(
			Object beanInstance, String name, String beanName, @Nullable RootBeanDefinition mbd) {

		// 如果知道的name是工厂相关（以&为前缀）且beanInstance又不是FactoryBean类型则校验不通过
		if (BeanFactoryUtils.isFactoryDereference(name)) {
			if (beanInstance instanceof NullBean) {
				return beanInstance;
			}
			if (!(beanInstance instanceof FactoryBean)) {
				throw new BeanIsNotAFactoryException(beanName, beanInstance.getClass());
			}
		}
		//到目去为止，我们获得的beanInstance可能会是正常的bean/FactoryBean，如果是FactoryBean我们会使用它创建实例，但是如果用户想要直接获取工厂实例而不是工厂的getObject方法对应的实例，那么传入的name应该加上前缀&
		if (!(beanInstance instanceof FactoryBean) || BeanFactoryUtils.isFactoryDereference(name)) {
			return beanInstance;
		}
		//加载FactoryBean 
		Object object = null;
		if (mbd == null) {
            //尝试从缓存中加载
			object = getCachedObjectForFactoryBean(beanName);
		}
		if (object == null) {
            //到了这里可以明确的知道传进来的beanInstance一定是FactoryBean类型
			FactoryBean<?> factory = (FactoryBean<?>) beanInstance;
			// Caches object obtained from FactoryBean if it is a singleton.
			if (mbd == null && containsBeanDefinition(beanName)) {
				mbd = getMergedLocalBeanDefinition(beanName);
			}
			boolean synthetic = (mbd != null && mbd.isSynthetic());
			object = getObjectFromFactoryBean(factory, beanName, !synthetic);
		}
		return object;
	}
```

总结下上面的代码：

1. 对FactoryBean正确性的验证
2. 对非FactoryBean不做任何处理
3. 对bean进行转换
4. 将从Factory中解析bean的工作委托给getObjectFromFactoryBean。追进去这个方法可以看到调用了FactoryBean的getObject方法。

## 获取单例

前面的代码是从缓存中获取单例的过程，如果缓存中没有已经加载的bean，那就需要从头开始bean的加载过程了。

```java
//	DefaultSingletonBeanRegistry.java
public Object getSingleton(String beanName, ObjectFactory<?> singletonFactory) {
		Assert.notNull(beanName, "Bean name must not be null");
		synchronized (this.singletonObjects) {
            //首先检测对于的bean是否已经加载过
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
                //在创建之前要做的事 这里面其实就是记录加载状态
				beforeSingletonCreation(beanName);
				boolean newSingleton = false;
				boolean recordSuppressedExceptions = (this.suppressedExceptions == null);
				if (recordSuppressedExceptions) {
					this.suppressedExceptions = new LinkedHashSet<>();
				}
				try {
                    //真正核心的代码，初始化bean
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
					afterSingletonCreation(beanName);
				}
				if (newSingleton) {
                    //加入到缓存中去
					addSingleton(beanName, singletonObject);
				}
			}
			return singletonObject;
		}
	}
```

上面代码的逻辑分析如下：

1. 检查缓存是否已经加载过
2. 若没有加载，则记录beanName的正在加载状态，beforeSingletonCreation方法中实现的
   * **beforeSingletonCreation方法看代码只有几行，但其实非常重要，它这里将正在加载的bean加入到了singletonsCurrentlyInCreation(Map)中去，以后就可以通过这个对循环依赖进行检测**
   * 前面从缓存中获取bean的**getSingleton(String beanName, boolean allowEarlyReference)**方法里面有个判断是要判断当前bean是否在加载中，其实就是判断singletonsCurrentlyInCreation中是否存在这个bean
3. 加载单例前记录状态
4. 通过调用参数传入的ObjectFactory的个体Object方法实例化bean
5. 加载单例后的处理方法。afterSingletonCreation方法，将bean从singletonsCurrentlyInCreation中移除。
6. 将结果记录至缓存并删除加载bean过程中记录的各种辅助状态
7. 处理返回结果

## 准备创建bean

这里其实就是接着上面的代码，也就是调用了ObjectFactory的getObject方法，我们再往回看一层，其实可以看到真正创建bean的代码是在createBean方法里面的。

```java
//AbstractBeanFactory.java
protected <T> T doGetBean(....省略){
    //...省略
    sharedInstance = getSingleton(beanName, () -> {
        try {
            //重点在这里
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
    //...省略
}
//AbstractAutowireCapableBeanFactory.java
	protected Object createBean(String beanName, RootBeanDefinition mbd, @Nullable Object[] args)
			throws BeanCreationException {

		if (logger.isDebugEnabled()) {
			logger.debug("Creating instance of bean '" + beanName + "'");
		}
		RootBeanDefinition mbdToUse = mbd;
		//根据涉及的class熟悉或者根据className来解析Class。	
		//这里就是确保是否真的解析了bean类 
		Class<?> resolvedClass = resolveBeanClass(mbd, beanName);
		if (resolvedClass != null && !mbd.hasBeanClass() && mbd.getBeanClassName() != null) {
			mbdToUse = new RootBeanDefinition(mbd);
			mbdToUse.setBeanClass(resolvedClass);
		}

		// Prepare method overrides.
		try {
			//验证以及准备覆盖的方法
			mbdToUse.prepareMethodOverrides();
		}
		catch (BeanDefinitionValidationException ex) {
			throw new BeanDefinitionStoreException(mbdToUse.getResourceDescription(),
					beanName, "Validation of method overrides failed", ex);
		}

		try {
			// 给BeanPostProcessor一个机会来返回代理来替代真正的实例
			Object bean = resolveBeforeInstantiation(beanName, mbdToUse);
			if (bean != null) {
				return bean;
			}
		}
		catch (Throwable ex) {
			throw new BeanCreationException(mbdToUse.getResourceDescription(), beanName,
					"BeanPostProcessor before instantiation of bean failed", ex);
		}

		try {
            //看到do开头的方法就知道了这里才是真正做事的方法
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

上面代码具体步骤功能如下：

1. 根据涉及的class熟悉或者根据className来解析Class。
2. 对override属性进行标记及验证。就是**mbdToUse.prepareMethodOverrides();**这行代码，感兴趣可以点进去看。
   * Spring是没有override-method这样的配置，这里是为了处理lookup-method和replace-method的，Spring会将这两个配置统一放在BeanDefinnition中的methodOverrides属性中。
3. 应用初始化前的后处理器，解析指定bean是否存在初始化前的短路操作。resolveBeforeInstantiation方法。
   * 短路判断存在于resolveBeforeInstantiation方法中，看下一小节
4. 创建bean

### 实例化的前置处理

接着上面的resolveBeforeInstantiation方法。

```java
//AbstractAutowireCapableBeanFactory.java
	protected Object resolveBeforeInstantiation(String beanName, RootBeanDefinition mbd) {
		Object bean = null;
		if (!Boolean.FALSE.equals(mbd.beforeInstantiationResolved)) {
			// 如果此时bean尚未被解析
			if (!mbd.isSynthetic() && hasInstantiationAwareBeanPostProcessors()) {
				Class<?> targetType = determineTargetType(beanName, mbd);
				if (targetType != null) {
                    //这里源码比较简单，对所有InstantiationAwareBeanPostProcessor类型的后处理器进行postProcessBeforeInstantiation方法
					bean = applyBeanPostProcessorsBeforeInstantiation(targetType, beanName);
					if (bean != null) {
                        //和上面类型 ，对所有InstantiationAwareBeanPostProcessor类型的后处理器进行postProcessAfterInitialization方法调用
						bean = applyBeanPostProcessorsAfterInitialization(bean, beanName);
					}
				}
			}
			mbd.beforeInstantiationResolved = (bean != null);
		}
		return bean;
	}
```

**resolveBeforeInstantiation方法**给了程序一次修改BeanDefinition的机会，当经过这个方法后，bean可能不是我们任务的那个bean了。applyBeanPostProcessorsBeforeInstantiation和applyBeanPostProcessorsAfterInitialization方法会给Bean加上一些前置处理器和后置处理器。是不是觉得这个逻辑有点熟悉，没错，AOP就是在这里实现的！

## 循环依赖

循环依赖分为构造器依赖和setter循环依赖。

### 构造器循环依赖

对于这种依赖，是无法解决的，只能通过抛出BeanCurrentlyInCreationException异常表示循环依赖。

Spring容易将每一个正在创建的bean标识符放在一个**正在创建**的池子中，bean标识符将会在创建过程中一直在这个池子中，因此如果创建bean的过程中发现自己以及在这个池子中，那么就会抛出BeanCurrentlyInCreationException异常表示循环依赖。

### setter循环依赖

对于setter注入造成的依赖，可以通过前面源码中的提前暴露方案来解决。Spring可以提前暴露刚刚完成构造器注入，但还没有完成其它步骤的bean，而且只能解决单例作用域下的循环依赖。通过提前暴露一个单例工厂方法，从而使其它bean能引用到该bean。

假设A,B,C出现了setter循环依赖，Spring 的处理步骤如下：

1. Spring创建单例“A”Bean，首先根据无参构造方法创建bean，并暴露一个ObjectFactory，用于返回一个提前暴露一个创建中的bean，并将A放入到**正在创建池**中，再接着进行setter注入B。
2. Spring创建单例“B”Bean，首先根据无参构造方法创建bean，并暴露一个ObjectFactory，用于返回一个提前暴露一个创建中的bean，并将B放入到**正在创建池**中，再接着进行setter注入C。
3. Spring创建单例“C”Bean，首先根据无参构造方法创建bean，并暴露一个ObjectFactory，用于返回一个提前暴露一个创建中的bean，并将C放入到**正在创建池**中，再接着进行注入A，注入A的时候由于提前暴露了ObjectFactory工厂，所以可以获得到一个创建中的bean。
4. 最后再依赖注入B和C，完成setter注入。

问：为什么Spring只能解决单例的setter循环依赖。

答：因为Spring容器不缓存非单例bean，因此无法提前暴露出一个正在创建中的bean。

## Bean的创建

经历了前面resolveBeforeInstantiation方法后，程序有两个选择：

1. 如果创建了代理并在postProcessBeforeInstantiation中改变了bean，那么直接返回就可以了
2. 否则进行正常bean的创建，正常bean的创建就在doCreateBean方法中

```java
//AbstractAutowireCapableBeanFactory.java
protected Object doCreateBean(String beanName, RootBeanDefinition mbd, @Nullable Object[] args)
			throws BeanCreationException {

		// Instantiate the bean.
		BeanWrapper instanceWrapper = null;
        //清缓存
		if (mbd.isSingleton()) {
			instanceWrapper = this.factoryBeanInstanceCache.remove(beanName);
		}
		if (instanceWrapper == null) {
            //根据bean指定的策略创建新的实例：如工厂方法，构造函数自动注入，简单初始化
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

		//是否需要提早曝光
		boolean earlySingletonExposure = (mbd.isSingleton() && this.allowCircularReferences &&
				isSingletonCurrentlyInCreation(beanName));
		if (earlySingletonExposure) {
			if (logger.isDebugEnabled()) {
				logger.debug("Eagerly caching bean '" + beanName +
						"' to allow for resolving potential circular references");
			}
            //为避免后期循环依赖，可以在bean初始化完成前将创建实例的ObjectFactory加入工厂
            //我们熟知的AOP就是在getEarlyBeanReference方法中实现的，在这里将advice织入到bean中
            //疑惑：前面创建bean前不是处理过AOP么？难道不是在那里处理的？
			addSingletonFactory(beanName, () -> getEarlyBeanReference(beanName, mbd, bean));
		}

		// Initialize the bean instance.
		Object exposedObject = bean;
		try {
            //对bean进行填充，将各个属性值注入，其中，可能存在依赖于其它bean的属性，会递归创建
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
            //只有检测到有循环依赖的情况下才不会为空
			if (earlySingletonReference != null) {
                //如果exposedObject没有在初始化中被改变，也就是没有被增强
				if (exposedObject == bean) {
					exposedObject = earlySingletonReference;
				}
				else if (!this.allowRawInjectionDespiteWrapping && hasDependentBean(beanName)) {
					String[] dependentBeans = getDependentBeans(beanName);
					Set<String> actualDependentBeans = new LinkedHashSet<>(dependentBeans.length);
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

上面整个函数的思路：

1. 如果是单例则需要首先清除缓存。
   * 我理解这里说的缓存和前面说的提前暴露的缓存不是一个东西。这里还有疑惑。
2. 实例化bean，将BeanDefinnition转换为BeanWrapper。这里非常的复杂。
3. MergedBeanDefinitionPostProcessors的应用。
   * bean合并后的处理，Autowired注解正式通过这个方法来实现注入类型的预解析
4. 依赖处理
5. 属性填充
6. 循环依赖检查
7. 注册DisposableBean
8. 完成创建并返回

### 创建bean的实例

```java
//AbstractAutowireCapableBeanFactory.java
	protected BeanWrapper createBeanInstance(String beanName, RootBeanDefinition mbd, @Nullable Object[] args) {
		// 解析出Class对象
		Class<?> beanClass = resolveBeanClass(mbd, beanName);

		if (beanClass != null && !Modifier.isPublic(beanClass.getModifiers()) && !mbd.isNonPublicAccessAllowed()) {
			throw new BeanCreationException(mbd.getResourceDescription(), beanName,
					"Bean class isn't public, and non-public access not allowed: " + beanClass.getName());
		}

		Supplier<?> instanceSupplier = mbd.getInstanceSupplier();
		if (instanceSupplier != null) {
			return obtainFromSupplier(instanceSupplier, beanName);
		}
		//工厂方法不为空就使用工厂方法进行初始化
		if (mbd.getFactoryMethodName() != null) {
			return instantiateUsingFactoryMethod(beanName, mbd, args);
		}

		// Shortcut when re-creating the same bean...
		boolean resolved = false;
		boolean autowireNecessary = false;
		if (args == null) {
			synchronized (mbd.constructorArgumentLock) {
                //根据参数锁定构造函数对应的工厂方法
				if (mbd.resolvedConstructorOrFactoryMethod != null) {
					resolved = true;
					autowireNecessary = mbd.constructorArgumentsResolved;
				}
			}
		}
        //如果已经解析好后则不需要再锁定
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
            //构造函数自动注入
			return autowireConstructor(beanName, mbd, ctors, args);
		}

		// 使用默认构造函数构造
		return instantiateBean(beanName, mbd);
	}
```

上面逻辑：

1. 如果在RootBeanDefinition中存在factoryMethodName属性，那么Spring会尝试使用FactoryMethod方法生成bean的实例。
2. 解析构造函数并进行构造函数的实例化

**注：**autowireConstructor方法非常复杂，暂时不往里面分析

简单分析下autowireConstructor的作用：

1. 构造方法参数的确定：通过getBean的explicitArgs参数，缓存中，配置文件中获取。
2. 构造函数的确定
3. 根据确定的构造函数转换对应的参数类型
4. 构造函数不确定性的验证
5. 根据实例化策略以及得到的构造函数及构造函数参数实例化Bean。（这里实例化策略会根据是否需要动态扩展选择使用反射或cglib）

### 记录创建bean的ObjectFactory

直接看ObjectFactory的getObject()是如何处理循环依赖的。

```java
//AbstractAutowireCapableBeanFactory.java docreate方法中的一部分代码
		boolean earlySingletonExposure = (mbd.isSingleton() && this.allowCircularReferences &&
				isSingletonCurrentlyInCreation(beanName));
		if (earlySingletonExposure) {
			if (logger.isDebugEnabled()) {
				logger.debug("Eagerly caching bean '" + beanName +
						"' to allow for resolving potential circular references");
			}
            //() -> getEarlyBeanReference(beanName, mbd, bean)就是getObject的实现
			addSingletonFactory(beanName, () -> getEarlyBeanReference(beanName, mbd, bean));
		}
	protected Object getEarlyBeanReference(String beanName, RootBeanDefinition mbd, Object bean) {
		Object exposedObject = bean;
		if (!mbd.isSynthetic() && hasInstantiationAwareBeanPostProcessors()) {
			for (BeanPostProcessor bp : getBeanPostProcessors()) {
				if (bp instanceof SmartInstantiationAwareBeanPostProcessor) {
					SmartInstantiationAwareBeanPostProcessor ibp = (SmartInstantiationAwareBeanPostProcessor) bp;
					exposedObject = ibp.getEarlyBeanReference(exposedObject, beanName);
				}
			}
		}
		return exposedObject;
	}
```

分析结果如下（疑惑：没看明白书里怎么分析出来的）：

Spring处理循环依赖的方法，在B中创建依赖A时通过ObjectFactory提供的实例化方法来中断A的属性填充（其实就是递归创建依赖的bean去了），使B中持有的A仅仅是刚刚初始化并没有填充任何属性的A，而这正初始化的A的步骤还是在最开始创建A的时候进行的，但是因为A与B中的A所表示的属性地址是一样的，所以在A中创建好的属性填充自然可以通过B中的A获取，这样就解决的循环依赖的问题。

### 属性注入

也就是popilateBean这个方法（doCreateBean里调用），暂时先不放源码了，直接说结论。

1. InstantiationAwareBeanPostProcessor处理器的postProcessAfterInstantiation函数的应用，此函数可以控制程序是否继续进行属性填充
2. 根据注入类型，提取依赖的bean，并统一存入PropertyValues中
3. 应用InstantiationAwareBeanPosttProcessor处理器的postProcessPropertyValues方法，对属性获取完毕填充前对属性的再次处理。
4. 将所有PropertyValues中你的属性填充至BeanWrapper中

### 初始化bean

这一步主要是针对配置bean时配置的init-method的属性。除此之外还有几个作用。

1. 激活Aware方法。Spring提供了一些Aware相关接口，bean可以实现这个接口去使用它。
2. 处理器的应用。例如BeanPostProcessor的使用就是在这里。
3. 激活自定义的init方法

### 注册DisposableBean

也就是销毁方法的扩展入口，对于销毁方法的扩展除了比较常用的destroy-method方法外，用户还可以注册后处理器DestructionAwareBeanPostProcessor来统一处理bean的销毁方法。



