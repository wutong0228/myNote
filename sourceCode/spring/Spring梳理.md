## Spring

![image-20200821023900052](C:\Users\吴桐\AppData\Roaming\Typora\typora-user-images\image-20200821023900052.png)

### Spring启动流程

```java
	AnnotationConfigApplicationContext context = new AnnotationConfigApplicationContext();
	context.register(AppConfig.class);
	context.refresh();
```

等价于

```java
	AnnotationConfigApplicationContext context = new AnnotationConfigApplicationContext(AppConfig.class);
	
	->
	public AnnotationConfigApplicationContext(Class<?>... componentClasses) {
		this();
		register(componentClasses);
		refresh();
	}
```

#### 构造方法

先调用顶级父类`DefaultResourceLoader`的构造方法

```java
	public DefaultResourceLoader() {
        this.classLoader = ClassUtils.getDefaultClassLoader();
	}
```

再调用`AbstractApplicationContext`的构造方法

```java
	public AbstractApplicationContext() {
		this.resourcePatternResolver = getResourcePatternResolver();
	}
```

再调用`GenericApplicationContext`的构造方法

```java
	public GenericApplicationContext() {
        // 实例化一个beanFactory
		this.beanFactory = new DefaultListableBeanFactory();
	}
```

最后执行自己的构造方法

```java
	public AnnotationConfigApplicationContext() {
        // 实例化一个reader，可以将一个加了注解的类转化成beanDefinition
		this.reader = new AnnotatedBeanDefinitionReader(this);
        
	        ->
        	public AnnotatedBeanDefinitionReader(BeanDefinitionRegistry registry) {
				this(registry, getOrCreateEnvironment(registry));
			}
        
        		->
                public AnnotatedBeanDefinitionReader(BeanDefinitionRegistry registry, Environment environment) {
                    Assert.notNull(registry, "BeanDefinitionRegistry must not be null");
                    Assert.notNull(environment, "Environment must not be null");
                    this.registry = registry;
                    this.conditionEvaluator = new ConditionEvaluator(registry, environment, null);
                    AnnotationConfigUtils.registerAnnotationConfigProcessors(this.registry);
                }
        
        			->AnnotationConfigUtils.registerAnnotationConfigProcessors()
                    public static Set<BeanDefinitionHolder> registerAnnotationConfigProcessors(
                            BeanDefinitionRegistry registry, @Nullable Object source) {
						// 从registry里获得beanFactory
                        DefaultListableBeanFactory beanFactory = unwrapDefaultListableBeanFactory(registry);
                        if (beanFactory != null) {
                            if (!(beanFactory.getDependencyComparator() instanceof AnnotationAwareOrderComparator)) {
                                // 给beanFactory设置依赖比较器
                                beanFactory.setDependencyComparator(AnnotationAwareOrderComparator.INSTANCE);
                            }
                            if (!(beanFactory.getAutowireCandidateResolver() instanceof ContextAnnotationAutowireCandidateResolver)) {
                                // 给beanFactory设置自动注入解析器
                                beanFactory.setAutowireCandidateResolver(new ContextAnnotationAutowireCandidateResolver());
                            }
                        }

                        Set<BeanDefinitionHolder> beanDefs = new LinkedHashSet<>(8);
						// 如果不包含"org.springframework.context.annotation.internalConfigurationAnnotationProcessor"
                        // 则使用ConfigurationClassPostProcessor生成bd
                        // 实现了BeanDefinitionRegistryPostProcessor,
						// PriorityOrdered, ResourceLoaderAware, BeanClassLoaderAware, EnvironmentAware接口
                        // 核心是实现了BeanFactoryPostProcessor接口
                        // 重写了postProcessBeanFactory()方法
                        // 1、使用CGLIB增强配置类
                        // 2、beanFactory.addBeanPostProcessor(new ImportAwareBeanPostProcessor(beanFactory));
                        // 这个处理器的作用是如果bean实现了ImportAware，则给这个bean设置import元信息
                        // 实现了bdrpp接口，重写了postProcessBeanDefinitionRegistry方法
                        // 则生成一个bd注册进bf
                        if (!registry.containsBeanDefinition(CONFIGURATION_ANNOTATION_PROCESSOR_BEAN_NAME)) {
                            RootBeanDefinition def = new RootBeanDefinition(ConfigurationClassPostProcessor.class);
                            def.setSource(source);
                            beanDefs.add(registerPostProcessor(registry, def, CONFIGURATION_ANNOTATION_PROCESSOR_BEAN_NAME));
                            
                            ->DefaultListableBeanFactory.registerBeanDefinition()
                            // 将bd注册进bf的核心方法
                        }
						
                        // 如果不包含"org.springframework.context.annotation.internalAutowiredAnnotationProcessor"
                        // 则使用AutowiredAnnotationBeanPostProcessor生成bd
                        // 继承了InstantiationAwareBeanPostProcessorAdapter
                        // 实现了MergedBeanDefinitionPostProcessor, PriorityOrdered, BeanFactoryAware接口
                        // 核心是实现了BeanPostProcessor接口,但是没有重写方法
                        // 则生成一个bd注册进bf
                        if (!registry.containsBeanDefinition(AUTOWIRED_ANNOTATION_PROCESSOR_BEAN_NAME)) {
                            RootBeanDefinition def = new RootBeanDefinition(AutowiredAnnotationBeanPostProcessor.class);
                            def.setSource(source);
                            beanDefs.add(registerPostProcessor(registry, def, AUTOWIRED_ANNOTATION_PROCESSOR_BEAN_NAME));
                                                        
                            ->DefaultListableBeanFactory.registerBeanDefinition()
                            // 将bd注册进bf的核心方法
                        }

                        // Check for JSR-250 support, and if present add the CommonAnnotationBeanPostProcessor.
                        // 继承了InitDestroyAnnotationBeanPostProcessor
                        // 实现了InstantiationAwareBeanPostProcessor, BeanFactoryAware, Serializable接口
                        // 核心是实现了BeanPostProcessor接口
                        // 重写了postProcessBeforeInitialization()方法
                        // LifecycleMetadata metadata = findLifecycleMetadata(bean.getClass());
						// metadata.invokeInitMethods(bean, beanName);
                        if (jsr250Present && !registry.containsBeanDefinition(COMMON_ANNOTATION_PROCESSOR_BEAN_NAME)) {
                            RootBeanDefinition def = new RootBeanDefinition(CommonAnnotationBeanPostProcessor.class);
                            def.setSource(source);
                            beanDefs.add(registerPostProcessor(registry, def, COMMON_ANNOTATION_PROCESSOR_BEAN_NAME));
                        }

                        // Check for JPA support, and if present add the PersistenceAnnotationBeanPostProcessor.
                        // 实现了InstantiationAwareBeanPostProcessor, DestructionAwareBeanPostProcessor,
						// MergedBeanDefinitionPostProcessor, PriorityOrdered, BeanFactoryAware, Serializable接口
                        // 核心是实现了BeanPostProcessor，但是没有实现这两个方法
                        if (jpaPresent && !registry.containsBeanDefinition(PERSISTENCE_ANNOTATION_PROCESSOR_BEAN_NAME)) {
                            RootBeanDefinition def = new RootBeanDefinition();
                            try {
                                def.setBeanClass(ClassUtils.forName(PERSISTENCE_ANNOTATION_PROCESSOR_CLASS_NAME,
                                        AnnotationConfigUtils.class.getClassLoader()));
                            }
                            catch (ClassNotFoundException ex) {
                                throw new IllegalStateException(
                                        "Cannot load optional framework class: " + PERSISTENCE_ANNOTATION_PROCESSOR_CLASS_NAME, ex);
                            }
                            def.setSource(source);
                            beanDefs.add(registerPostProcessor(registry, def, PERSISTENCE_ANNOTATION_PROCESSOR_BEAN_NAME));
                        }
                        
						// 如果不包含"org.springframework.context.event.internalEventListenerProcessor"
                        // 实现了SmartInitializingSingleton、ApplicationContextAware、BeanFactoryPostProcessor接口
                        // 核心是实现了BeanFactoryPostProcessor
                        // 从工厂里获得EventListenerFactory.class
                        // 取出值并依据AnnotationAwareOrderComparator排序后设置给this.eventListenerFactories
                        // 则生成一个bd注册进bf
                        if (!registry.containsBeanDefinition(EVENT_LISTENER_PROCESSOR_BEAN_NAME)) {
                            RootBeanDefinition def = new RootBeanDefinition(EventListenerMethodProcessor.class);
                            def.setSource(source);
                            beanDefs.add(registerPostProcessor(registry, def, EVENT_LISTENER_PROCESSOR_BEAN_NAME));
                        }

                        // 如果不包含"org.springframework.context.event.internalEventListenerFactory"
                        // 实现了EventListenerFactory接口
                        // 则生成一个bd注册进bf
                        if (!registry.containsBeanDefinition(EVENT_LISTENER_FACTORY_BEAN_NAME)) {
                            RootBeanDefinition def = new RootBeanDefinition(DefaultEventListenerFactory.class);
                            def.setSource(source);
                            beanDefs.add(registerPostProcessor(registry, def, EVENT_LISTENER_FACTORY_BEAN_NAME));
                        }

                        return beanDefs;
                    }
        // 实例化一个scanner，可以将类或者包下的类转化成bd
		this.scanner = new ClassPathBeanDefinitionScanner(this);
	}
```

#### register()方法

```java
	@Override
	public void register(Class<?>... componentClasses) {
		Assert.notEmpty(componentClasses, "At least one component class must be specified");
		this.reader.register(componentClasses);
	}

	->AnnotatedBeanDefinitionReader.register()
	public void register(Class<?>... componentClasses) {
		for (Class<?> componentClass : componentClasses) {
			registerBean(componentClass);
		}
	}

	->this.registerBean()
	public void registerBean(Class<?> beanClass) {
		doRegisterBean(beanClass, null, null, null, null);
	}

	->this.doRegisterBean()
	private <T> void doRegisterBean(Class<T> beanClass, @Nullable String name,
			@Nullable Class<? extends Annotation>[] qualifiers, @Nullable Supplier<T> supplier,
			@Nullable BeanDefinitionCustomizer[] customizers) {
		// 通过beanClass生成ABD，即带注释的beanDefinition
		AnnotatedGenericBeanDefinition abd = new AnnotatedGenericBeanDefinition(beanClass);
		// 通过abd判断是否应该跳过
        if (this.conditionEvaluator.shouldSkip(abd.getMetadata())) {
			return;
		}
		// 设置实例提供者（null）
		abd.setInstanceSupplier(supplier);
        // 解析abd的scope元信息
		ScopeMetadata scopeMetadata = this.scopeMetadataResolver.resolveScopeMetadata(abd);
        // 给abd设置scope(Singleton、Prototype、session、request)
		abd.setScope(scopeMetadata.getScopeName());
		String beanName = (name != null ? name : this.beanNameGenerator.generateBeanName(abd, this.registry));
		// 处理公共注解（@Lazy、@Primary、@DependsOn、@Role、@Description）
		AnnotationConfigUtils.processCommonDefinitionAnnotations(abd);
        // qualifiers默认为null，不会走此处的逻辑
		if (qualifiers != null) {
			for (Class<? extends Annotation> qualifier : qualifiers) {
				if (Primary.class == qualifier) {
					abd.setPrimary(true);
				}
				else if (Lazy.class == qualifier) {
					abd.setLazyInit(true);
				}
				else {
					abd.addQualifier(new AutowireCandidateQualifier(qualifier));
				}
			}
		}
        // customizers默认为null，不会走此处的逻辑
		if (customizers != null) {
			for (BeanDefinitionCustomizer customizer : customizers) {
				customizer.customize(abd);
			}
		}
		// 通过abd和beanName生成BeanDefinitionHolder，可以理解为一个pair，key是beanName，value是abd
		BeanDefinitionHolder definitionHolder = new BeanDefinitionHolder(abd, beanName);
        // 判断scopedProxyMode，根据枚举类生成不同的BeanDefinitionHolder，如果是ScopedProxyMode.NO，则不作任何修改
		definitionHolder = AnnotationConfigUtils.applyScopedProxyMode(scopeMetadata, definitionHolder, this.registry);
        // 注册beanDefinition
		BeanDefinitionReaderUtils.registerBeanDefinition(definitionHolder, this.registry);
        
        	->BeanDefinitionReaderUtils.registerBeanDefinition()
            public static void registerBeanDefinition(
                BeanDefinitionHolder definitionHolder, BeanDefinitionRegistry registry)
                throws BeanDefinitionStoreException {

                // Register bean definition under primary name.
                String beanName = definitionHolder.getBeanName();
                // 将<beanName,BeanDefinition>放到beanDefinitionMap里
                // 将beanName放到beanDefinitionNames里
                // 将beanName从manualSingletonNames里移除
                registry.registerBeanDefinition(beanName, definitionHolder.getBeanDefinition());
				
                
                
                // Register aliases for bean name, if any.
                String[] aliases = definitionHolder.getAliases();
                if (aliases != null) {
                    for (String alias : aliases) {
                        registry.registerAlias(beanName, alias);
                    }
                }
            }
	}

```

#### refresh()方法

```java
	@Override
	public void refresh() throws BeansException, IllegalStateException {
		synchronized (this.startupShutdownMonitor) {
			// Prepare this context for refreshing.
            // 1、做一些准备工作
			prepareRefresh();

			// Tell the subclass to refresh the internal bean factory.
            // 2、获取bean工厂
			ConfigurableListableBeanFactory beanFactory = obtainFreshBeanFactory();
            
            	->
                protected ConfigurableListableBeanFactory obtainFreshBeanFactory() {
                    // AbstractRefreshableApplicationContext的实现会判断当前有没有工厂，有则销毁重新生成一个工厂，设置到this.beanFactory
                    // GenericApplicationContext的实现会CAS的设置refreshed为True
                    // 把环境的id设置给工厂的serializationId
                    refreshBeanFactory();
                    // 获取环境里的工厂
                    return getBeanFactory();
                }

			// Prepare the bean factory for use in this context.
            // 3、给工厂设置一些属性
			prepareBeanFactory(beanFactory);
            
                ->
                protected void prepareBeanFactory(ConfigurableListableBeanFactory beanFactory) {
                    // Tell the internal bean factory to use the context's class loader etc.
                    // 1、给工厂设置bean类加载器beanClassLoader
                    beanFactory.setBeanClassLoader(getClassLoader());
                    // 2、给工厂设置bean表达式解析器beanExpressionResolver
                    beanFactory.setBeanExpressionResolver(new StandardBeanExpressionResolver(beanFactory.getBeanClassLoader()));
                    beanFactory.addPropertyEditorRegistrar(new ResourceEditorRegistrar(this, getEnvironment()));

                    // Configure the bean factory with context callbacks.
                    // 3、添加了一个后置处理器ApplicationContextAwareProcessor
                    // 这个后置处理器的作用给实现了一些aware接口的bean回调对应的set方法
                    // EnvironmentAware、EmbeddedValueResolverAware、ResourceLoaderAware、ApplicationEventPublisherAware
                    // MessageSourceAware、ApplicationContextAware
                    beanFactory.addBeanPostProcessor(new ApplicationContextAwareProcessor(this));
                    // 4、添加到bf的已忽略依赖接口列表ignoredDependencyInterfaces
                    beanFactory.ignoreDependencyInterface(EnvironmentAware.class);
                    beanFactory.ignoreDependencyInterface(EmbeddedValueResolverAware.class);
                    beanFactory.ignoreDependencyInterface(ResourceLoaderAware.class);
                    beanFactory.ignoreDependencyInterface(ApplicationEventPublisherAware.class);
                    beanFactory.ignoreDependencyInterface(MessageSourceAware.class);
                    beanFactory.ignoreDependencyInterface(ApplicationContextAware.class);

                    // BeanFactory interface not registered as resolvable type in a plain factory.
                    // MessageSource registered (and found for autowiring) as a bean.
                    // 5、put进bf的resolvableDependencies，key是依赖的类型，value是主动注入的值
                    beanFactory.registerResolvableDependency(BeanFactory.class, beanFactory);
                    beanFactory.registerResolvableDependency(ResourceLoader.class, this);
                    beanFactory.registerResolvableDependency(ApplicationEventPublisher.class, this);
                    beanFactory.registerResolvableDependency(ApplicationContext.class, this);

                    // Register early post-processor for detecting inner beans as ApplicationListeners.
                    // 6、添加了一个后置处理器ApplicationListenerDetector应用监听探测器
                    // 作用是将单例的bean强转成ApplicationListener后添加到context的applicationListeners里
                    beanFactory.addBeanPostProcessor(new ApplicationListenerDetector(this));

                    // Detect a LoadTimeWeaver and prepare for weaving, if found.
                    // 7
                    if (beanFactory.containsBean(LOAD_TIME_WEAVER_BEAN_NAME)) {
                        beanFactory.addBeanPostProcessor(new LoadTimeWeaverAwareProcessor(beanFactory));
                        // Set a temporary ClassLoader for type matching.
                        beanFactory.setTempClassLoader(new ContextTypeMatchClassLoader(beanFactory.getBeanClassLoader()));
                    }

                    // Register default environment beans.
                    // 注册单例
                    if (!beanFactory.containsLocalBean(ENVIRONMENT_BEAN_NAME)) {
                        beanFactory.registerSingleton(ENVIRONMENT_BEAN_NAME, getEnvironment());

                        	->DefaultSingletonBeanRegistry.addSingleton()
                            this.singletonObjects.put(beanName, singletonObject);
                            this.singletonFactories.remove(beanName);
                            this.earlySingletonObjects.remove(beanName);
                            this.registeredSingletons.add(beanName);
                    }
                    if (!beanFactory.containsLocalBean(SYSTEM_PROPERTIES_BEAN_NAME)) {
                        beanFactory.registerSingleton(SYSTEM_PROPERTIES_BEAN_NAME, getEnvironment().getSystemProperties());
                    }
                    if (!beanFactory.containsLocalBean(SYSTEM_ENVIRONMENT_BEAN_NAME)) {
                        beanFactory.registerSingleton(SYSTEM_ENVIRONMENT_BEAN_NAME, getEnvironment().getSystemEnvironment());
                    }
                }


			try {
				// Allows post-processing of the bean factory in context subclasses.
                // 4、空方法，没有做任何事情
				postProcessBeanFactory(beanFactory);

				// Invoke factory processors registered as beans in the context.
                // 5、请求bfpp
                // 回调bdrpp的postProcessBeanDefinitionRegistry()
                // 回调bfpp的postProcessBeanFactory()
				invokeBeanFactoryPostProcessors(beanFactory);
                	
                	->
                    protected void invokeBeanFactoryPostProcessors(ConfigurableListableBeanFactory beanFactory) {
                        // 先依次请求bdrpp的postProcessBeanDefinitionRegistry()
                        // 再请求这些bdrpp的postProcessBeanFactory()
                        // 最后再依次请求未请求过的bfpp的postProcessBeanFactory()
                        PostProcessorRegistrationDelegate.invokeBeanFactoryPostProcessors(beanFactory, getBeanFactoryPostProcessors());
                        	
                        	->
                            // 获取bfpp列表
                            // 需要手动调用context的addBeanFactoryPostProcessor()方法的才能获取到
                            public List<BeanFactoryPostProcessor> getBeanFactoryPostProcessors() {
                                return this.beanFactoryPostProcessors;
                            }
                        
                        	->PostProcessorRegistrationDelegate.invokeBeanFactoryPostProcessors()
                            public static void invokeBeanFactoryPostProcessors(
                                    ConfigurableListableBeanFactory beanFactory, List<BeanFactoryPostProcessor> beanFactoryPostProcessors) {

                                // Invoke BeanDefinitionRegistryPostProcessors first, if any.
                                // 处理过的beanSet
                                Set<String> processedBeans = new HashSet<>();

                                // 如果实现了BeanDefinitionRegistry，True
                                if (beanFactory instanceof BeanDefinitionRegistry) {
                                    BeanDefinitionRegistry registry = (BeanDefinitionRegistry) beanFactory;
                                    // 规律的bfpp、List<bfpp>
                                    List<BeanFactoryPostProcessor> regularPostProcessors = new ArrayList<>();
                                    // bdrpp，注册器processor、List<bdrpp>
                                    List<BeanDefinitionRegistryPostProcessor> registryProcessors = new ArrayList<>();

                                    // 将获取到的bfpp分类存放在List<bfpp>或List<bdrpp>
                                    for (BeanFactoryPostProcessor postProcessor : beanFactoryPostProcessors) {
                                        if (postProcessor instanceof BeanDefinitionRegistryPostProcessor) {
                                            BeanDefinitionRegistryPostProcessor registryProcessor =
                                                    (BeanDefinitionRegistryPostProcessor) postProcessor;
                                            registryProcessor.postProcessBeanDefinitionRegistry(registry);
                                            registryProcessors.add(registryProcessor);
                                        }
                                        else {
                                            regularPostProcessors.add(postProcessor);
                                        }
                                    }

                                    // Do not initialize FactoryBeans here: We need to leave all regular beans
                                    // uninitialized to let the bean factory post-processors apply to them!
                                    // Separate between BeanDefinitionRegistryPostProcessors that implement
                                    // PriorityOrdered, Ordered, and the rest.
                                    // 当前注册器处理器
                                    List<BeanDefinitionRegistryPostProcessor> currentRegistryProcessors = new ArrayList<>();

                                    // First, invoke the BeanDefinitionRegistryPostProcessors that implement PriorityOrdered.
                                    // 首先，请求实现了PriorityOrdered的bdrpp
                                    // 获取实现了BeanDefinitionRegistryPostProcessor的beanNames
                                    String[] postProcessorNames =
                                            beanFactory.getBeanNamesForType(BeanDefinitionRegistryPostProcessor.class, true, false);
                                    for (String ppName : postProcessorNames) {
                                        // 获取实现了PriorityOrdered的beanName
                                        if (beanFactory.isTypeMatch(ppName, PriorityOrdered.class)) {
                                            // 根据beanName获取bean，保存到currentRegistryProcessors
                                            currentRegistryProcessors.add(beanFactory.getBean(ppName, BeanDefinitionRegistryPostProcessor.class));
                                            // 保存到处理过的beanSet里
                                            processedBeans.add(ppName);
                                        }
                                    }
                                    // 用工厂的排序器给currentRegistryProcessors排序
                                    sortPostProcessors(currentRegistryProcessors, beanFactory);
                                    // 把currentRegistryProcessors里的信息保存到registryProcessors
                                    registryProcessors.addAll(currentRegistryProcessors);
                                    // 回调currentRegistryProcessors中每个bdrpp的postProcessBeanDefinitionRegistry()方法
                                    invokeBeanDefinitionRegistryPostProcessors(currentRegistryProcessors, registry);
                                    // 清空currentRegistryProcessors
                                    currentRegistryProcessors.clear();

                                    // Next, invoke the BeanDefinitionRegistryPostProcessors that implement Ordered.
                                    // 接下来，请求实现了Ordered的berpp
                                    postProcessorNames = beanFactory.getBeanNamesForType(BeanDefinitionRegistryPostProcessor.class, true, false);
                                    for (String ppName : postProcessorNames) {
                                        if (!processedBeans.contains(ppName) && beanFactory.isTypeMatch(ppName, Ordered.class)) {
                                            currentRegistryProcessors.add(beanFactory.getBean(ppName, BeanDefinitionRegistryPostProcessor.class));
                                            processedBeans.add(ppName);
                                        }
                                    }
                                    sortPostProcessors(currentRegistryProcessors, beanFactory);
                                    registryProcessors.addAll(currentRegistryProcessors);
                                    invokeBeanDefinitionRegistryPostProcessors(currentRegistryProcessors, registry);
                                    currentRegistryProcessors.clear();

                                    // Finally, invoke all other BeanDefinitionRegistryPostProcessors until no further ones appear.
                                    // 最后，请求所有其他的bdrpp
                                    boolean reiterate = true;
                                    while (reiterate) {
                                        reiterate = false;
                                        postProcessorNames = beanFactory.getBeanNamesForType(BeanDefinitionRegistryPostProcessor.class, true, false);
                                        for (String ppName : postProcessorNames) {
                                            if (!processedBeans.contains(ppName)) {
                                                currentRegistryProcessors.add(beanFactory.getBean(ppName, BeanDefinitionRegistryPostProcessor.class));
                                                processedBeans.add(ppName);
                                                reiterate = true;
                                            }
                                        }
                                        sortPostProcessors(currentRegistryProcessors, beanFactory);
                                        registryProcessors.addAll(currentRegistryProcessors);
                                        invokeBeanDefinitionRegistryPostProcessors(currentRegistryProcessors, registry);
                                        currentRegistryProcessors.clear();
                                    }

                                    // Now, invoke the postProcessBeanFactory callback of all processors handled so far.
                                    // 现在、请求所有处理器的postProcessBeanFactory方法
                                    invokeBeanFactoryPostProcessors(registryProcessors, beanFactory);
                                    invokeBeanFactoryPostProcessors(regularPostProcessors, beanFactory);
                                }

                                else {
                                    // Invoke factory processors registered with the context instance.
                                    invokeBeanFactoryPostProcessors(beanFactoryPostProcessors, beanFactory);
                                }

                                // Do not initialize FactoryBeans here: We need to leave all regular beans
                                // uninitialized to let the bean factory post-processors apply to them!
                                String[] postProcessorNames =
                                        beanFactory.getBeanNamesForType(BeanFactoryPostProcessor.class, true, false);

                                // Separate between BeanFactoryPostProcessors that implement PriorityOrdered,
                                // Ordered, and the rest.
                                List<BeanFactoryPostProcessor> priorityOrderedPostProcessors = new ArrayList<>();
                                List<String> orderedPostProcessorNames = new ArrayList<>();
                                List<String> nonOrderedPostProcessorNames = new ArrayList<>();
                                for (String ppName : postProcessorNames) {
                                    if (processedBeans.contains(ppName)) {
                                        // skip - already processed in first phase above
                                    }
                                    else if (beanFactory.isTypeMatch(ppName, PriorityOrdered.class)) {
                                        priorityOrderedPostProcessors.add(beanFactory.getBean(ppName, BeanFactoryPostProcessor.class));
                                    }
                                    else if (beanFactory.isTypeMatch(ppName, Ordered.class)) {
                                        orderedPostProcessorNames.add(ppName);
                                    }
                                    else {
                                        nonOrderedPostProcessorNames.add(ppName);
                                    }
                                }

                                // First, invoke the BeanFactoryPostProcessors that implement PriorityOrdered.
                                // 首先、请求实现了PriorityOrdered的bfpp
                                sortPostProcessors(priorityOrderedPostProcessors, beanFactory);
                                invokeBeanFactoryPostProcessors(priorityOrderedPostProcessors, beanFactory);

                                // Next, invoke the BeanFactoryPostProcessors that implement Ordered.
                                // 其次，请求实现了Ordered的bfpp
                                List<BeanFactoryPostProcessor> orderedPostProcessors = new ArrayList<>(orderedPostProcessorNames.size());
                                for (String postProcessorName : orderedPostProcessorNames) {
                                    orderedPostProcessors.add(beanFactory.getBean(postProcessorName, BeanFactoryPostProcessor.class));
                                }
                                sortPostProcessors(orderedPostProcessors, beanFactory);
                                invokeBeanFactoryPostProcessors(orderedPostProcessors, beanFactory);

                                // Finally, invoke all other BeanFactoryPostProcessors.
                                // 最后，请求所有其他的bfpp
                                List<BeanFactoryPostProcessor> nonOrderedPostProcessors = new ArrayList<>(nonOrderedPostProcessorNames.size());
                                for (String postProcessorName : nonOrderedPostProcessorNames) {
                                    nonOrderedPostProcessors.add(beanFactory.getBean(postProcessorName, BeanFactoryPostProcessor.class));
                                }
                                invokeBeanFactoryPostProcessors(nonOrderedPostProcessors, beanFactory);

                                // Clear cached merged bean definitions since the post-processors might have
                                // modified the original metadata, e.g. replacing placeholders in values...
                                // 清理工厂元数据缓存
                                beanFactory.clearMetadataCache();
                            }

                        // Detect a LoadTimeWeaver and prepare for weaving, if found in the meantime
                        // (e.g. through an @Bean method registered by ConfigurationClassPostProcessor)
                        if (beanFactory.getTempClassLoader() == null && beanFactory.containsBean(LOAD_TIME_WEAVER_BEAN_NAME)) {
                            beanFactory.addBeanPostProcessor(new LoadTimeWeaverAwareProcessor(beanFactory));
                            beanFactory.setTempClassLoader(new ContextTypeMatchClassLoader(beanFactory.getBeanClassLoader()));
                        }
                    }

				// Register bean processors that intercept bean creation.
                // 6、注册beanPostProcessors
				registerBeanPostProcessors(beanFactory);

				// Initialize message source for this context.
                // 7、初始化消息资源
				initMessageSource();

				// Initialize event multicaster for this context.
                // 8、初始化应用事件广播器
				initApplicationEventMulticaster();

				// Initialize other special beans in specific context subclasses.
                // 9、刷新
				onRefresh();

				// Check for listener beans and register them.
                // 10、注册监听器
				registerListeners();

				// Instantiate all remaining (non-lazy-init) singletons.
                // 11、完成bf初始化
				finishBeanFactoryInitialization(beanFactory);

				// Last step: publish corresponding event.
                // 12、完成刷新
				finishRefresh();
			}

			catch (BeansException ex) {
				if (logger.isWarnEnabled()) {
					logger.warn("Exception encountered during context initialization - " +
							"cancelling refresh attempt: " + ex);
				}

				// Destroy already created singletons to avoid dangling resources.
				destroyBeans();

				// Reset 'active' flag.
				cancelRefresh(ex);

				// Propagate exception to caller.
				throw ex;
			}

			finally {
				// Reset common introspection caches in Spring's core, since we
				// might not ever need metadata for singleton beans anymore...
				resetCommonCaches();
			}
		}
	}

```

