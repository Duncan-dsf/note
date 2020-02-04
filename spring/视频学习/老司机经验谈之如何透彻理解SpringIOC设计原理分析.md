### spring平台组成技术体系

spring是一个分层框架，由7个体系模块组成，所有模块都构建在容器技术之上

![spring模块](../../../pic/spring模块.png)

顶层容器

- ClassPathXmlApplicationContext
- FileSystemXmlApplicationContext

初始化流程

```java
  public FileSystemXmlApplicationContext(String[] configLocations, boolean refresh, ApplicationContext parent)
        throws BeansException {
  
     super(parent);
     setConfigLocations(configLocations);
     if (refresh) {
        refresh();
     }
  }
```

  ```java
  public void refresh() throws BeansException, IllegalStateException {
     synchronized (this.startupShutdownMonitor) {
        // Prepare this context for refreshing.
        // 1.准备上下文
        prepareRefresh();
  
        // Tell the subclass to refresh the internal bean factory. 2.加载子类信息和参数
        ConfigurableListableBeanFactory beanFactory = obtainFreshBeanFactory();
  
        // Prepare the bean factory for use in this context.
        prepareBeanFactory(beanFactory);
  
        try {
           // Allows post-processing of the bean factory in context subclasses.
           postProcessBeanFactory(beanFactory);
  
           // Invoke factory processors registered as beans in the context.
           invokeBeanFactoryPostProcessors(beanFactory);
  
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

- 如何获取外部配置信息呢

- 为什么获取到beanFactory并处理完之后没有把beanFactory保存下来，岂不是白处理了

  - obtainFreshBeanFactory把创建beanFactory创建之后保存到this的属性中并返回，所以只对beanFactory处理，而没显式保存

  - obtainFreshBeanFactory![](../../../pic/obtainFreshBeanFactory.png)

  - refreshBeanFactory 

    - AbstractRefreshableApplicationContext

      ![](../../../pic/refreshBeanFactory.png)

      可刷新的上下文 refreshBeanFactory 重新创建

    - GenericApplicationContext 通用上下文 不可刷新 (AnnotationConfigApplicationContext) refreshBeanFactory不做操作

  - getBeanFactory ![](../../../pic/getBeanFactory.png)

  - 可以看到 refreshBeanFactory中创建beanFactory，并保存到this.beanFactory;而getBeanFactory就是获取this.beanFactory

- refreshBeanFactory会loadBeanDefinitions，加载bean定义

  - 通过配置文件的路径解析出Resource ![](../../../pic/loadBeanDefinitions.png)
  - 解析Resource——spring解析文件的模块 注册BeanDefinition

  