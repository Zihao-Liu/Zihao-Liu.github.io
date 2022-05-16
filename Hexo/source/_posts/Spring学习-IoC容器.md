---
title: Spring学习-IoC容器
date: 2022-05-16 14:37:45
categories:
 - Spring Framework
 - 学习笔记
tags:
 - Spring Framework
 - 学习笔记
 - Spring
---
# 什么是IoC？
IoC全称是Inversion of Control，直译成中文就是`控制反转`，也被叫做`依赖注入`。

当一个实例被创建时，通过`构造方法参数`、`工厂类方法`或者`属性的set方法`将依赖注入到这个实例中，而不是这个实例主动去创建(比如直接的new)依赖的过程，就被称为IoC。

那么IoC容器呢？IoC容器就是用来负责管理和协调这些依赖和被依赖对象的框架。在Spring Framework中，IoC的容器主要由`beans`和`context`两个包实现。`BeanFactory`接口提供了管理这些对象的能力，`ApplicationContext`实现了`BeanFactory`并提供了额外的能力。

简单的来说，Spring通过IoC容器来管理对象的生命周期。这些被IoC容器管理的对象，被称作`bean`，用来维护bean之间关系的信息被称为`configuration meta`。

![IoC容器](https://docs.spring.io/spring-framework/docs/current/reference/html/images/container-magic.png)

> 抛开上面抽象的说法。个人认为，bean就像是食材，congiration meta（配置元系信息）就是食谱。而IoC容器就是一个厨师，通过食谱（configuration meta）将这些食材（bean）组合起来成为一道菜（application）。
<!-- more -->

# 配置元信息
正如食谱告诉厨师每样材料需要多少克，需要切成丝还是切成块。

配置元信息也告诉了IoC容器，如何初始化、配置和组装一个对象。IoC容器和元信息之间还需要一种协议（或者说形式）来理解这个过程，一般可以使用以下几种形式：
* xml：使用xml文件，比较传统的形式，接触Spring比较早的程序员应该都使用过。
* 注解：Spring从2.5开始支持使用基于注解的配置方式，如我们使用的`@Autowired`
* Java：Spring从3.0开始，可以使用Java代码的形式进行配置，需要配合`@Configuration`、`@Bean`等注解使用。

{% fold 下面看一个完整的xml方式的配置 %}
```xml
<!-- services.xml -->
<?xml version="1.0" encoding="UTF-8"?>
<beans xmlns="http://www.springframework.org/schema/beans"
  xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
  xsi:schemaLocation="http://www.springframework.org/schema/beans
  https://www.springframework.org/schema/beans/spring-beans.xsd">
  
  <bean id="petStore" class="org.xxx.PetStoreServiceImpl">
  <!-- id: 唯一标识一个bean定义 -->
  <!-- class: 使用类的全限定名标识bean的具体类型 -->
    <property name="accountDao" ref="accountDao"/>
    <!-- name: petStore中有个字段名为accountDao -->
    <!-- ref: 指向另一个id为accountDao的bean元素(参见下面dao.xml)-->
    <property name="itemDao" ref="itemDao"/>
    <!-- 其他配置 -->
  </bean>

</beans>
```
```xml
<!-- dao.xml -->
<?xml version="1.0" encoding="UTF-8"?>
<beans xmlns="http://www.springframework.org/schema/beans"
  xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
  xsi:schemaLocation="http://www.springframework.org/schema/beans
  https://www.springframework.org/schema/beans/spring-beans.xsd">
  
  <bean id="accountDao" class="org.xxx.JpaAccountDao">
  <!-- 其他配置 -->
  </bean>

  <bean id="itemDao" class="org.xxx.JpaItemDao">
  <!-- 其他配置 -->
  </bean>
  
</beans>
```
```xml
<!-- 也支持在一个bean元素中引入其他的xml文件 -->
<beans>
  <import resource="services.xml"/>
  <import resource="resources/messageSource.xml"/>
  <import resource="/resources/themeSource.xml"/>
  <bean id="bean1" class="..."/>
  <bean id="bean2" class="..."/>
</beans>
```
{% endfold %}

# Bean的初始化

1. 使用构造方法初始化

```xml
<bean id="exampleBean" class="examples.ExampleBean"/>
<bean name="anotherExample" class="examples.ExampleBeanTwo"/>
```

2. 使用static方法初始化
   
```xml
<bean id="clientService" class="examples.ClientService" factory-method="createInstance"/>
<!--  调用 createInstance方法初始化 -->
```
```java
public class ClientService {
    private static ClientService clientService = new ClientService();
    private ClientService() {}

    public static ClientService createInstance() {
        // 调用 createInstance方法初始化
        return clientService;
    }
}
```

3. 使用实例方法初始化

```xml
<!-- 工厂类，包含createClientServiceInstance方法 -->
<bean id="serviceLocator" class="examples.DefaultServiceLocator">
  <!-- inject any dependencies required by this locator bean -->
</bean>

<!-- 通过serviceLocator.createClientServiceInstance初始化clientService -->
<bean id="clientService" factory-bean="serviceLocator" factory-method="createClientServiceInstance"/>

<!-- 通过serviceLocator.createServerServiceInstance初始化serverService -->
<bean id="serverService" factory-bean="serviceLocator" factory-method="createServerServiceInstance"/>
```
```java
public class DefaultServiceLocator {
    private static ClientService clientService = new ClientServiceImpl();
    private static ServerService serverService = new ServerServiceImpl();
    
    public ClientService createClientServiceInstance() {
        return clientService;
    }
    public ServerService createServerServiceInstance() {
        return serverService;
    }
}
```
------
# 参考文档
> [Spring官方文档](https://docs.spring.io/spring-framework/docs/current/reference/html/core.html#beans)