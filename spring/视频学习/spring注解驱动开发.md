### 03.@ComponentScan

### 05.@Scope设置Bean作用域

4种作用域

- ConfigurableBeanFactory#SCOPE_PROTOTYPE 多实例
- ConfigurableBeanFactory#SCOPE_SINGLETON 单例(默认)
- org.springframework.web.context.WebApplicationContext#SCOPE_REQUEST 同一个请求创建一个实例
- org.springframework.web.context.WebApplicationContext#SCOPE_SESSION 同一个session创建一个实例

Bean创建时机

- 单例 容器启动时创建对象
- 多例 容器启动时不创建 每次获取对象时创建

### 06.@Lazy 懒加载

@Lazy **单例情况下** 当容器启动时不实例bean，当获取bean时实例

### 07.@Conditional 条件注册

符合条件的 才会注册到容器，可以做到不同情况注册不同bean

![](/Users/network/Documents/note/pic/conditional.png)

- org.springframework.context.annotation.Condition 自己定义一个实现Condition接口的类 实现matches方法，matches返回true则将bean注册

### 08.@Import 组件注册

组件注入方式

- @Component、@Service、@Controller、@Repository 需要在类上加注解
- @Bean 需要注解在方法上，在方法中自己实例化对象，方法名为bean的id
- @Import 需要在@Configuration的类上加注解，写明需要哪些类的bean，spring会自动生产bean，更加简便，bean的id是全限定名
  - @Import(A.class) 注册A类
  - ImportSelector
    - @Import(XXImportSelector.class) XXImportSelector实现了ImportSelector接口，重写selectImports 返回想要导入的bean的类的全限定类名数组
  - ImportBeanDefinitionRegistrar
    - @Import(XXImportBeanDefinitionRegistrar.class) XXImportBeanDefinitionRegistrar需要实现ImportBeanDefinitionRegistrar接口
    - 重写registerBeanDefinitions方法，手动向参数registry中注册bean——需要自己实例bean，并给出bean id

### 09.ImportSelector

见08

### 10.ImportBeanDefinitionRegistrar

见08

### 11.FactoryBean注册

- 定义一个实现FactoryBean接口的类

  ![](../../../pic/factoryBean.png)

  - getObject 实例T类的对象的方法
  - isSingleton 是否单例，如果单例的话 每次从容器getBean时返回对象为同一个

- 将XXFactoryBean 通过@Bean注册到容器——bean的id是方法名，例如XXFactoryBean

- 那么XXFactoryBean注册的实际是Factory.getObject的返回结果，而不是beanFactory本身

- 那么如何获取beanFactory本身——beanFactory加前缀&，用这个id getBean就可以获得factory本身

### 12.生命周期@Bean指定初始化、销毁方法

- xml配置可以配置init和destroy
- @Bean(initMethod="", destoryMethod="") 方法名为注册的类的方法
  - 单例scope下
    - 初始化方法在容器创建时**对象创建完成并赋值好属性后**执行
    - 销毁方法在容器.close方法时执行
  - 多例下
    - 初始化在get时**对象创建完成并赋值好属性后**执行
    - 销毁方法不会执行——多例下get之后生命周期由用户自己负责

### 13.InitializingBean、DisposableBean接口

bean需要实现两个接口，并重写

- afterPropertiesSet 在属性设置之后执行
- destroy 销毁之前执行

### 14.@PostContruct、@PreDestory

在bean的方法上加注解，被注解方法会在属性赋值之后、销毁之前执行

### 15.BeanPostProcessor

![](../../../pic/BeanPostProcessor.png)

实现接口，并注册到容器中

- postProcessBeforeInitialization 在bean构造之后 **属性设置之前执行(在上面的几种方式执行后执行)**
- postProcessAfterInitialization 在bean的属性设置之后(**在上面几种执行后执行**)

### 16.BeanPostProcessor原理

![](../../../pic/beanPostProcessor原理.png)

- bean是已实例化的对象
- invokeInitMethods执行初始化方法——12到14讲的方法
- 在对bean进行初始化之前(invokeInitMethods)，先调用applyBeanPostProcessorsBeforeInitialization——执行所有注册了的beanPostProcessor的beforeInitialization
- 在初始化之后调用applyBeanPostProcessorsAfterInitialization——执行所有注册了的beanPostProcessor的afterInitialization方法

### 17.BeanPostProcessor在Spring底层的使用

- **ApplicationContextAware.setApplicationContext**可以把容器注入到bean，使使用者可以操作容器本身，而不只是容器中注册的组件
  - 通过ApplicationContextAwareProcessor实现
  - **ApplicationContextAwareProcessor**重写了BeanProcessor的**postProcessBeforeInitialization**，可以在初始化bean之前判断当前实例化bean是否是**ApplicationContextAware**的实现类，如果是的话，调用bean的**setApplicationContext**方法
- InitDestroyAnnotationBeanPostProcessor
  - 在bean初始化前，会获取当前bean所有加@PostConstruct注解的方法，之后调用
  - 在bean销毁之前，会获取当前bean所有加@PreDestory注解的方法，之后调用
- AutowiredAnnotationBeanPostProcessor实现@Autowired

### 18.@Value赋值

@Value("") 可以写多种值

- 基本数值
- SpEL #{}
- 配置文件中的值(在运行环境变量中的值)

### 19.属性赋值@PropertySource

读取外部配置文件的k/v保存到运行的环境变量中

- @Value("${}")读取
- application.getEnvironment().getProperty("") 获取

### 20.自动装配 @Autowired/Qualifier/Primary

- 注解注入bean(@Component, @Service)，id默认是类名首字母小写
- @Autowired注入规则
  - 先按类型去容器中找对于类型的组件
  - 如果存在多个，以属性名为bean id注入组件
  - @Qualifier可以手动指定bean的id，而不是以默认的属性名
  - 默认情况下，@Autowired注入失败，会抛异常，可以设置@Autowired(required=false)，注入失败时不抛异常
  - 在@bean的方法上加@Primary注解，@Autowired注入时，不再默认使用id为属性名的bean，而是以加@Primary的bean为默认(除非用Qualifier手动指定)

### 21.@Inject @Resource

- @Resource按组件名注入。不能设置required=false，也不能和@Primary起作用
- @Inject 和@Autowired一样，按类型注入，不支持required=false,支持@Primary

### 22.方法、构造器位置的自动装配

#### Autowired

- 方法上,setter的参数会自动从容器中拿(@Bean 方法参数默认从容器中获取)
- 构造器上，参数从容器中拿(如果只有一个构造器且有参，那么这个构造器可以不加@Autowired注解，自动从容器中获取参数)

### 23.Aware注入spring底层组件&原理

#### ApplicationContextAware 将容器本身注入组件

#### BeanNameAware 将当前bean的name告诉组件

#### EmbeddedValueResolverAware 将@Value("xxx")的解析器传入组件

- 功能都是由xxxAwareProcessor实现的
- 创建bean之后，执行初始化方法时，先执行aware方法，将spring底层组件注入到组件中

### 24.@Profile 配置文件

根据环境选择不同的组件

- @PropertySource引入配置文件
- @Value("${}") 从配置中取配置

### 25.@Profile 根据环境注册bean

- @Profile指定bean在哪个环境下被注册，和@Bean注解一起使用
- 默认时default环境(不加@Profile或者加@Profile("default")) 在任何环境下都会注册
- spring.profiles.active=xxx 设置当前环境
- application.getEnvironment().setActiveProfiles 设置当前环境

- @Profile 加到配置类上，那么整个配置类只有在对应环境才起作用

### 26.IOC总结

### 27.AOP功能测试

- 定义一个切点——一个想要切片处理的目标方法
- 切面方法——在切点周围执行

### 28.AOP原理 @EnableAspectJAutoProxy

EnableAspectJAutoProxy代码![](../../../pic/EnableAspectJAutoProxy.png)

- 会注册AspectJAutoProxyRegistrar(spring-context包中)组件

- 这个注册器会注册AnnotationAwareAspectJAutoProxyCreator

  ![](../../../pic/AnnotationAwareAspectJAutoProxyCreator.png)

- 注册的creator是一个postProcessor

### 29.AnnotationAwareAspectJAutoProxyCreator分析

实现的接口及接口定义的方法

- AbstractAutoProxyCreator

  - setBeanFactory BeanFactoryAware接口
  - 后置处理器 SmartInstantiationAwareBeanPostProcessor接口

- AbstractAdvisorAutoProxyCreator

  - 重写了父类AbstractAutoProxyCreator的setBeanFactory，调用了initBeanFactory

    ![](../../../pic/AbstractAdvisorAutoProxyCreator.png)

- AnnotationAwareAspectJAutoProxyCreator

  - 重写了initBeanFactory

### 30.AnnotationAwareAspectJAutoProxyCreator注册

1. 传入配置类，创建ico容器
2. 注册配置类，refresh刷新容器——重新注册bean
3. registerPostProcessor
   1. 获取注册的postProcessor
   2. 给容器加postProcessor
   3. 优先注册实现了PriorityOrdered的postProcessor
   4. 再是实现了Ordered的postProcessor
      1. AnnotationAwareAspectJAutoProxyCreator实现了Ordered接口，所以在这开始注册
      2. 创建bean，之后初始化
         1. 先实例化
         2. 执行初始化
            1. aware方法——setBeanFactory
            2. 执行postProcessor的初始化前处理方法
            3. 执行初始化方法
            4. 调用后置处理器的实例化后处理方法
   5. 最后是普通的
4. finishBeanFactoryInitialization 实例化所有其他的bean
   1. preInstantiateSingletons
   2. getBean
   3. doGetBean
      1. 先从缓存getSingleton
      2. 获取不到会getSingleton(多一个singletonFactory 和上一点的getSingleton不同)
         1. beforeSingletonCreation 将当前beanName加到singletonsCurrentlyInCreation中表明当前bean正在创建
         2. getObject =>> createBean
            1. resolveBeforeInstantiation 调用InstantiationAwareBeanPostProcessor的**postProcessBeforeInstantiation**
            2. 如果上一个处理结果为null则doCreateBean
               1. 先实例化createBeanInstance
               2. 填充bean populateBean
                  1. 执行InstantiationAwareBeanPostProcessor.postProcessAfterInstantiation
               3. 执行初始化 initializeBean
                  1. invokeAwareMethods 如果当前bean实现了Aware接口，执行对于Aware接口定义的方法
                  2. 执行postProcessor的初始化前处理方法applyBeanPostProcessorsBeforeInitialization
                  3. 执行初始化方法invokeInitMethods
                  4. 调用后置处理器的实例化后处理方法applyBeanPostProcessorsAfterInitialization

### 31.AnnotationAwareAspectJAutoProxyCreator执行时机

- ApplicationContextAware的setApplicationCOntext
- SmartInstantiationAwareBeanPostProcessor的实例化前的处理postProcessBeforeInstantiation

### 32.创建AOP代理

SmartInstantiationAwareBeanPostProcessor

1. postProcessBeforeInstantiation
   1. 判断当前bean是否在advisedBeans(保存了需要增强的bean)中,如果已存在就return

   2. 判断当前bean是否是基础类型(Pointcut、Advisor、AopInfrastructureBean这种用来实现AOP的基础类)或者shouldSkip

      也return

   3. **getCustomTargetSource 如果结果不为null,创建代理？？？不懂getCustomTargetSource是干什么**

2. postProcessAfterInitialization => wrapIfNecessary
   1. 判断advisedBeans.get(key) 是不是false,如果是false不做处理
   2. 判断bean是不是基础类型或者应该跳过，如果是那么不做处理
   3. getAdvicesAndAdvisorsForBean=>findEligibleAdvisors 获取适合当前bean的所有增强器
      1. findCandidateAdvisors 获取当前beanFactory的所有实现Advisor接口的bean
         1. 先从缓存拿advisorNames
         2. 拿不到就beanNamesForTypeIncludingAncestors
         3. 通过advisorNames,挨个从容器中通过name和类型(Advisor.class)获取bean
      2. findAdvisorsThatCanApply=>findAdvisorsThatCanApply 从上一步的返回——所有增强器中获取适合当前bean的增强器集合
         1. 遍历增强器，假如当前增强器是IntroductionAdvisor且增强器的类目标对象包含bean，那么就是适合当前bean的增强器
         2. 假如当前增强器是PointcutAdvisor，则判断类切点是否匹配当前bean
      3. 对适合的增强器进行排序
   4. createProxy创建代理对象并返回——这样容器中的
      1. new 一个ProxyFactory 将增强器添加进ProxyFactory
      2. proxyFactory.getProxy
         1. createAopProxy=>createAopProxy 创建代理对象
         2. spring决定是返回**ObjenesisCglibAopProxy**还是**JdkDynamicAopProxy**
         3. getProxy获取代理对象

### 33.获取拦截器链MethodInterceptor

代理对象的执行

- 容器中保存了组件的代理对象(cglib增强后的对象)，对象里保存了详细信息

- 对象方法执行

- org.springframework.aop.framework.CglibAopProxy.DynamicAdvisedInterceptor#intercept

  - getInterceptorsAndDynamicInterceptionAdvice 获取拦截器和增强器

    - 获取所有增强器getAdvisors

    - 获取每个advisor的拦截器getInterceptors(advisor)，合并并返回

      ![](../../../pic/getInterceptors.png)

  - 如果没拦截器链直接执行目标方法

  - 如果有，创建CglibMethodInvocation并调用proceed

### 34.链式调用通知方法

CglibMethodInvocation的proceed实现

- 每次获取一个方法拦截器，并调用
- 每个拦截器会调用CglibMethodInvocation的proceed——实现外递归，**CglibMethodInvocation由proceed调用的时候传入**
  - 前置拦截器会先执行前置通知，之后调用CglibMethodInvocation的proceed
  - 后置通知在递归返回之后调用

### 35.AOP原理总结

### 36.声明式事务-环境搭建

- dataSource
- jdbcTemplate

### 37.声明式事务测试

- @Transactional
- @EnableTransactionManagement
- 事务管理器 PlatformTransactionManager
  - DataSourceTransactionManager

### 38.声明式事务——源码分析

- @EnableTransactionManagement 导入selector

  - ```
    @Import(TransactionManagementConfigurationSelector.class)
    ```

- selector导入组件

  ![](../../../pic/TransactionManagementConfigurationSelector.png)
  - AdviceMode的值 是@EnableTransactionManagement的mode属性的值
  - 默认是proxy,会注入 两个组件
    - AutoProxyRegistrar
      - 注册InfrastructureAdvisorAutoProxyCreator组件
      - 利用后置处理器在对象创建后，返回一个代理对象
    - ProxyTransactionManagementConfiguration
      - 向容器注册组件
      - BeanFactoryTransactionAttributeSourceAdvisor 需要下面两个组件
      - TransactionAttributeSource
      - TransactionInterceptor 事务拦截器
        - 在目标方法执行时，执行拦截器链
        - 事务拦截器执行
          - createTransactionIfNecessary
          - invocation.proceedWithInvocation
          - 出现异常时completeTransactionAfterThrowing
          - 正确返回时commitTransactionAfterReturning
  - **aop和@Transactional如何转变为Advisor的？？？？**

### 39.扩展原理-BeanFactoryPostprocessor

在BeanDefinition注册完成后，bean实例化之前执行，允许添加或者修改BeanDefinition

- 执行时机beanFactory创建完成之后调用，执行

- 子类BeanDefinitionRegistryPostProcessor

### 40.BeanDefinitionRegistryPostProcessor

- 是BeanFactoryPostprocessor的子类

- 所以执行BeanFactoryPostprocessor时，也会执行BeanDefinitionRegistryPostProcessor
- 而且先于父类

### 41.ApplicationListener

接口：**onApplicationEvent**

接收**ApplicationEvent**事件

#### ApplicationEvent：

- ApplicationContextEvent
  - ContextRefreshedEvent 容器会自动发布
  -  ContextClosedEvent 容器会自动发布
  -  ContextStoppedEvent
  -  ContextStartedEvent
- PayloadApplicationEvent  当发布事件不是ApplicationEvent类型时，容器会自动用PayloadApplicationEvent包装

#### 事件如何发布

**ApplicationEventPublisher**接口定义方法**publishEvent**可以发布事件

- ApplicationContext接口继承ApplicationEventPublisher
- 所以所有applicationContext都可以发布事件
- 具体由容器的ApplicationEventMulticaster.multicastEvent广播事件
  - 当事件发生时，applicationContext——事件发布者发布具体的不同的Event，如：ContextRefreshEvent
  - ApplicationEventMulticaster统一的处理，无论什么类型的事件
  - 事件发布者会通过事件类型寻找适合的监听者并执行监听者方法

### 42.ApplicationListener原理

### 43.@EventListener和SmartInitializingSingleton

- @EventListener加在方法上，并且方法要有参数ApplicationEvent，当事件发生的时候，就会调用当前方法
  - 由EventListenerMethodProcessor实现
  - EventListenerMethodProcessor是一个SmartInitializingSingleton
- SmartInitializingSingleton 在所有bean被实例化之后被调用
  -  finishBeanFactoryInitialization中实例化所有单实例bean之后
  - 遍历所有bean，假如bean是SmartInitializingSingleton类型，则调用这个bean的afterSingletonsInstantiated方法

### 44.spring容器创建——BeanFactory预准备

- **容器的refresh(ConfigurableApplicationContext定义,AbstractApplicationContext实现)**
- refresh 12个步骤
  1. prepareRefresh 预处理
  
     - initPropertySources 初始化一些属性设置，子类自己实现
  
     - validateRequiredProperties 校验属性合法性
     - 初始化早期事件list，等待事件派发器实例化后，将事件派发出去
  
  2. obtainFreshBeanFactory 构建beanFactory
     - refreshBeanFactory
       -  GenericApplicationContext构造时会直接初始化beanFactory
       - AbstractRefreshableApplicationContext 会在调用refreshBeanFactory时才构造beanFactory，并加载beanDefinition
     - getBeanFactory 获取beanFactory
  3. prepareBeanFactory 预准备工作
  
  4. postProcessBeanFactory 子类可以重写，在beanFactory预准备完成之后调用
  
  5. invokeBeanFactoryPostProcessors 执行beanFactoryPostProcessor
     - invokeBeanFactoryPostProcessors
       - BeanDefinitionRegistryPostProcessor
         - 先查找实现PriorityOrdered接口的排序并执行postProcessor
         - 再执行 Ordered接口的
         - 最后执行其他未执行过的所有BeanDefinitionRegistryPostProcessor
         - 每次获取postProcessor(PriorityOrdered、Ordered、其他这三次)都要重新遍历所有bean获取——**因为有可能BeanFactoryPostProcessor会向容器注入新的postProcessor；而beanPostProcessor不需要**
       - BeanFactoryPostProcessor
         - 同上
  
  6. registerBeanPostProcessors注册beanPostProcessor
  
     - beanPostProcessor 类型——执行时机不同
       - DestructionAwareBeanPostProcessor
       - InstantiationAwareBeanPostProcessor
       - SmartInstantiationAwareBeanPostProcessor
       - MergedBeanDefinitionPostProcessor
     - 先注册PriorityOrdered，注册前排序
     - 再注册Ordered 注册前排序
     - 之后注册其他的，注册前排序
     - 最后添加internalPostProcessors——MergedBeanDefinitionPostProcessor ，注册前排序
  
  7. initMessageSource 实现国际化、消息绑定、消息解析
  
     - 判断是否有bean叫messageSource
     - 没有的话 就创建一个DelegatingMessageSource
     - 将bean赋值给容器的属性
  
  8. initApplicationEventMulticaster 初始化事件派发器
  
     - 同上一流程
  
  9. onRefresh 子类重写 在容器refresh是调
  
  10. registerListeners
  
      - 注册ApplicationListener
      - 注册ApplicationListenerBean
      - 将earlyApplicationEvents——之前产生的事件，派发出去
  
  11. finishBeanFactoryInitialization 初始化其他的单实例bean
  
      - preInstantiateSingletons 初始化bean
  
        1. 拿到所有beanName，获取beanDefinition
  
        2. 判断不是抽象的&&是单例&&不是懒加载
           1. 如果是factoryBean，用beanFactory.getBean
  
           2. 否则，getBean->doGetBean
              1. getSingleton(beanName) 从缓存拿，为空的时候进行创建
              2. markBeanAsCreated标记当前bean正在创建
              3. 获取beanDefinition
              4. 获取所有依赖bean，通过getBean获取
              5. 创建bean
                 1. getSingleton(beanName, beanFactory)->createBean
                 2. resolveBeforeInstantiation 创建bean前，beanPostProcessor执行，尝试返回一个代理对象
                 3. applyBeanPostProcessorsBeforeInstantiation 执行InstantiationAwareBeanPostProcessor.postProcessBeforeInstantiation
                 4. 如果返回不为空 applyBeanPostProcessorsAfterInitialization 执行postProcessAfterInitialization
                 5. 如果resolveBeforeInstantiation返回为空，进行创建->doCreateBean
                 6. createBeanInstance 创建bean实例
                 7. applyMergedBeanDefinitionPostProcessors 
                 8. populateBean 给bean赋值
                 9. InstantiationAwareBeanPostProcessor.postProcessAfterInstantiation、postProcessPropertyValues
                 10. applyPropertyValues为属性用setter等方法赋值
                 11. initializeBean初始化bean
                 12. invokeAwareMethods执行aware方法
                 13. applyBeanPostProcessorsBeforeInitialization 初始化前的方法
                 14. invokeInitMethods执行初始化方法
                 15. InitializingBean.afterPropertiesSet
                 16. 执行自定义的初始化方法
                 17. applyBeanPostProcessorsAfterInitialization 所有beanPostProcessor.after
                 18. registerDisposableBeanIfNecessary 注册销毁方法
           3. getSingleton返回调用addSingleton 将创建好的bean添加到容器
  
        3. bean创建完成之后，SmartInitializingSingleton.afterSingletonsInstantiated
  
  12. finishRefresh 完成刷新
  
      1. 
  
      