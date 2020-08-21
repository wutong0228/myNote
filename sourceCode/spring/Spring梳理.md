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
            // 做一些准备工作
			prepareRefresh();

			// Tell the subclass to refresh the internal bean factory.
            // 获取bean工厂
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
            // 给工厂设置一些属性
			prepareBeanFactory(beanFactory);

			try {
				// Allows post-processing of the bean factory in context subclasses.
                // 空方法，没有做任何事情
				postProcessBeanFactory(beanFactory);

				// Invoke factory processors registered as beans in the context.
				invokeBeanFactoryPostProcessors(beanFactory);
                	
                	->
                    protected void invokeBeanFactoryPostProcessors(ConfigurableListableBeanFactory beanFactory) {
                        // 请求bfpp
                        PostProcessorRegistrationDelegate.invokeBeanFactoryPostProcessors(beanFactory, getBeanFactoryPostProcessors());
                        	
                        	->
                            public List<BeanFactoryPostProcessor> getBeanFactoryPostProcessors() {
                                return this.beanFactoryPostProcessors;
                            }

                        // Detect a LoadTimeWeaver and prepare for weaving, if found in the meantime
                        // (e.g. through an @Bean method registered by ConfigurationClassPostProcessor)
                        if (beanFactory.getTempClassLoader() == null && beanFactory.containsBean(LOAD_TIME_WEAVER_BEAN_NAME)) {
                            beanFactory.addBeanPostProcessor(new LoadTimeWeaverAwareProcessor(beanFactory));
                            beanFactory.setTempClassLoader(new ContextTypeMatchClassLoader(beanFactory.getBeanClassLoader()));
                        }
                    }

				// Register bean processors that intercept bean creation.
				registerBeanPostProcessors(beanFactory);

				// Initialize message source for this context.
				initMessageSource();

				// Initialize event multicaster for this context.
				initApplicationEventMulticaster();

				// Initialize other special beans in specific context subclasses.
				onRefresh();

				// Check for listener beans and register them.
				registerListeners();

				// Instantiate all remaining (non-lazy-init) singletons.
				finishBeanFactoryInitialization(beanFactory);

				// Last step: publish corresponding event.
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

