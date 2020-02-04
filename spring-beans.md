## 核心类

### DefaultListableBeanFactory

#### 类图

![](../pic/DefaultListableBeanFactory类图.png)

#### 各类的用途

- AliasRegistry
  - 别名注册表：将一个name与一个alias关联
- SimpleAliasRegistry
  - AliasRegistry的实现
  - (name, alias)放在concurrentHashMap中
- SingletonBeanRegistry
  - 单例bean注册表：将beanName与Object关联
- DefaultSingletonBeanRegistry
  - SingletonBeanRegistry的实现
  - (beanName, Object)放在concurrentHashMap
- BeanDefinitionRegistry
  - bean定义注册表
  - 实现有：
    - SimpleBeanDefinitionRegistry
      - (beanName, beanDefinition)存储在concurrentHashMap中
    - DefaultListableBeanFactory
      - 加了一些逻辑判断
- BeanFactory
  - 通过beanName或类型获取对象
- HierarchicalBeanFactory