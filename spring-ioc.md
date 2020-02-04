## Inversion of Control / Dependency Injection

### 概念

- 当需要一个对象的时候，只需要告诉容器你需要一个什么类型的对象，容器会帮我们找到并给我们
- 这样我们不需要知道这个对象的获取方式，只需要知道这个类的接口、方法，就可以直接使用
- 只要这个对象的类定义不变，就不会影响使用方
- 而当这个类所依赖的类改变时，只需要告诉容器自己需要什么依赖就可以解决——比如A类本来依赖B类实现功能，之后又改为依赖C类实现功能，那么只需要声明一个C类的成员变量就可以了

### spring容器

#### 容器类型

- BeanFactory
  - 基础IoC容器，提供完整的IoC服务支持
  - 不特殊设置的条件下，默认采用延迟初始化策略，当客户端对象需要访问容器中某个受管对象时，才对该受管对象进行初始化以及依赖注入操作
  - 所以，相对来说，BeanFactory容器启动初期速度较快，所需系统资源有限。
  - 对于系统资源有限，并且功能要求不是很严格的场景，适合使用BeanFactory
- ApplicationContext
  - ApplicationContext在beanFactory基础上构建，是相对比较高级的实现，在BeanFactory的基础上提供了其他高级特性：事件发布、国际化信息支持等
  - 受管对象默认全部初始化并绑定完成，所以相对BeanFactory来说，ApplicationContext需要更多系统资源，而且启动时间更长
  - 在系统资源充足并要求更多功能的场景中，ApplicationContext更合适
- ![](../pic/ApplicationContext类图.png)

#### BeanFactory

- BeanFactory负责bean的各个部分组装，使用者只需通过BeanFactory的接口获取
- 使用xml的情况下，根据xml，new一个ApplicationContext或BeanFactory，之后直接从容器中获取bean

##### 对象注册和依赖绑定方式

- 直接编码

  - 手动往beanFactory中绑定beanDefinition和参数

  - 

  - ```java
    public static void main(String[] args) {
    
            DefaultListableBeanFactory listableBeanFactory = new DefaultListableBeanFactory();
            BeanFactory beanFactory = bindViaCode(listableBeanFactory);
            ControllerA controllerA = (ControllerA) beanFactory.getBean("controllerA");
            controllerA.print();
        }
    
        public static BeanFactory bindViaCode(DefaultListableBeanFactory listableBeanFactory) {
    
            RootBeanDefinition controllerABeanDefinition = new RootBeanDefinition(ControllerA.class);
            RootBeanDefinition serviceABeanDefinition = new RootBeanDefinition(ServiceA.class);
            RootBeanDefinition serviceBBeanDefinition = new RootBeanDefinition(ServiceB.class);
    
            listableBeanFactory.registerBeanDefinition("controllerA", controllerABeanDefinition);
            listableBeanFactory.registerBeanDefinition("serviceA", serviceABeanDefinition);
            listableBeanFactory.registerBeanDefinition("serviceB", serviceBBeanDefinition);
    
            // 构造函数注入
            ConstructorArgumentValues argumentValues = new ConstructorArgumentValues();
            argumentValues.addIndexedArgumentValue(0, serviceABeanDefinition);
            argumentValues.addIndexedArgumentValue(1, serviceBBeanDefinition);
            controllerABeanDefinition.setConstructorArgumentValues(argumentValues);
    
            // 参数注入
            MutablePropertyValues mutablePropertyValues = new MutablePropertyValues();
            mutablePropertyValues.addPropertyValue(new PropertyValue("serviceA", serviceABeanDefinition));
            mutablePropertyValues.addPropertyValue(new PropertyValue("serviceB", serviceBBeanDefinition));
            controllerABeanDefinition.setPropertyValues(mutablePropertyValues);
    
            return listableBeanFactory;
        }
    ```

    

  - BeanDefinition

    - AbstractBeanDefinition类图![AbstractBeanDefinition类图](../pic/AbstractBeanDefinition类图.png)
      - 构造函数只有空和(设置bean构造函数参数和属性值)两种
      - 而如何标志beanDefinition是哪个class的呢，可以通过setBeanClass和setBeanClassName
    - AbstractBeanDefinition子类
      - RootBeanDefinition
        - 扩展了较多的功能
        - 但是没有parent，关于parent的实现
          - setParent入参只能是null，否则抛异常"root没有parent"
          - getParent 返回null
      - ChildBeanDefinition
        - 只覆盖实现了parent等基本操作
        - 必须有parent，parent为null报错
      - GenericBeanDefinition
        - 只实现了抽象类没实现的方法
        - 最少的功能
        - 可以有parent也可以没有