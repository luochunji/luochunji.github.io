---
title: Spring事务详解(二)
date: 2018-06-02 21:03:44
tags:
- Spring
- 事务
- 事务注解
categories:
- Spring
- Spring事务
---

上一篇主要介绍了Spring事务的基本概念，包括基本流程、传播属性和隔离级别等，接下来将详细分析Spring事务源码，一步步了解Spring事务是如何实现的。

<!-- more -->

## 事务配置入口

从事务的配置入手

```xml
<tx:annotation-driven transaction-manager="transactionManager"/>

<bean id="transactionManager" class="org.springframework.jdbc.datasource.DataSourceTransactionManager">
<property name="dataSource" ref="dataSource"/>
</bean>
```

开启声明式事务，需要配置两个地方：

1. 数据源dataSource，任何ORM的实现都要实现dataSource接口，因为必须拿到Connection对象；
2. 事务管理器transactionManager，由Spring提供，管理所有的事务操作。

打开DataSourceTransactionManager

```java
public class DataSourceTransactionManager extends AbstractPlatformTransactionManager
      implements ResourceTransactionManager, InitializingBean {
      //省略
      }
```

DataSourceTransactionManager是JDBC默认的事务管理器实现，它继承自AbstractPlatformTransactionManager，继续向上查找，发现其实现了PlatformTransactionManager接口

```java
public abstract class AbstractPlatformTransactionManager implements PlatformTransactionManager, Serializable {
    //省略
}
```

PlatformTransactionManager是啥？它是Spring事务的管理接口，除了jdbc的实现DataSourceTransactionManager外，Spring还提供了其他几种ORM的实现，如下：

![](PlatformTransactionManager.png)

顺着这个配置，可以找到定义tx:annotation-driven的xsd文件spring-tx.xsd，再通过其命名空间，从spring.handlers中找到对应的实现类TxNamespaceHandler。

```properties
http\://www.springframework.org/schema/tx=org.springframework.transaction.config.TxNamespaceHandler
```

## TxNamespaceHandler

TxNamespaceHandler是Spring事务管理的核心功能，它允许使用xml或注解来配置声明式事务

```java
public class TxNamespaceHandler extends NamespaceHandlerSupport {

   //tx标签的属性名称
   static final String TRANSACTION_MANAGER_ATTRIBUTE = "transaction-manager";
   //默认的属性管理器 
   static final String DEFAULT_TRANSACTION_MANAGER_BEAN_NAME = "transactionManager";


   //获取事务管理器的beanName 
   static String getTransactionManagerName(Element element) {
       //如果配置了 transaction-manager，取配置的beanName，否则取默认值
       //DEFAULT_TRANSACTION_MANAGER_BEAN_NAME
      return (element.hasAttribute(TRANSACTION_MANAGER_ATTRIBUTE) ?
            element.getAttribute(TRANSACTION_MANAGER_ATTRIBUTE) : DEFAULT_TRANSACTION_MANAGER_BEAN_NAME);
   }


   @Override
   public void init() {
      //注册<tx:advice/>的解析类TxAdviceBeanDefinitionParser 
      registerBeanDefinitionParser("advice", new TxAdviceBeanDefinitionParser());
      //注册<tx:annotation-driven/>的解析类AnnotationDrivenBeanDefinitionParser 
      registerBeanDefinitionParser("annotation-driven", new AnnotationDrivenBeanDefinitionParser());
      //注册<tx:jta-transaction-manager/>的解析类
      //JtaTransactionManagerBeanDefinitionParser 
      registerBeanDefinitionParser("jta-transaction-manager", new JtaTransactionManagerBeanDefinitionParser());
   }

}
```

Spring初始化时，会先加载init方法将解析器注册到parsers中，在解析xml时，根据标签从parsers中找到对应的解析器，使用委派模式进行解析

1. 初始化时注册解析器到parsers中

   ```java
   private final Map<String, BeanDefinitionParser> parsers = new HashMap<>();
   //...省略
   protected final void registerBeanDefinitionParser(String elementName, BeanDefinitionParser parser) {
      this.parsers.put(elementName, parser);
   }
   ```

2. 解析xml配置文件时，根据标签查找解析器

   ```java
   public BeanDefinition parse(Element element, ParserContext parserContext) {
      //查找解析器
       BeanDefinitionParser parser = findParserForElement(element, parserContext);
      return (parser != null ? parser.parse(element, parserContext) : null);
   }
   ```

   ```java
   private BeanDefinitionParser findParserForElement(Element element, ParserContext parserContext) {
      String localName = parserContext.getDelegate().getLocalName(element);
      BeanDefinitionParser parser = this.parsers.get(localName);
      if (parser == null) {
         parserContext.getReaderContext().fatal(
               "Cannot locate BeanDefinitionParser for element [" + localName + "]", element);
      }
      return parser;
   }
   ```

3. 具体的解析实现委派给对应的解析器

### xml解析

上文中，TxNamespaceHandler中支持两种事务配置方式：xml和注解，xml配置的范例如下，它对应的解析器为TxAdviceBeanDefinitionParser，用来处理事务的通知规则

```xml
<tx:advice id="transactionAdvice" transaction-manager="transactionManager">
	<tx:attributes>
		<tx:method name="add*" propagation="REQUIRED" rollback-for="RuntimeException"/>
		<tx:method name="remove*" propagation="REQUIRED" rollback-for="RuntimeException"/>
		<tx:method name="update*" propagation="REQUIRED" rollback-for="RuntimeException"/>
		<tx:method name="query*" read-only="true"/>
	</tx:attributes>
</tx:advice>
```

```java
//事务通知规则解析器
class TxAdviceBeanDefinitionParser extends AbstractSingleBeanDefinitionParser {
   //method元素,切面的方法配置
   private static final String METHOD_ELEMENT = "method";
   //方法名称
   private static final String METHOD_NAME_ATTRIBUTE = "name";
   //attributes元素
   private static final String ATTRIBUTES_ELEMENT = "attributes";
   //超时时间，获取超时时间配置
   private static final String TIMEOUT_ATTRIBUTE = "timeout";
   //事务只读属性
   private static final String READ_ONLY_ATTRIBUTE = "read-only";
   //事务传播属性
   private static final String PROPAGATION_ATTRIBUTE = "propagation";
   //事务隔离级别
   private static final String ISOLATION_ATTRIBUTE = "isolation";
   //回滚配置
   private static final String ROLLBACK_FOR_ATTRIBUTE = "rollback-for";
   //不回滚配置
   private static final String NO_ROLLBACK_FOR_ATTRIBUTE = "no-rollback-for";


   @Override
   protected Class<?> getBeanClass(Element element) {
      return TransactionInterceptor.class;
   }

    //标签解析实现，在父类AbstractSingleBeanDefinitionParser的parseInternal中调用
   @Override
   protected void doParse(Element element, ParserContext parserContext, BeanDefinitionBuilder builder) {
       //获取事务管理器
      builder.addPropertyReference("transactionManager", TxNamespaceHandler.getTransactionManagerName(element));

       //遍历<tx:attributes/>下的配置
      List<Element> txAttributes = DomUtils.getChildElementsByTagName(element, ATTRIBUTES_ELEMENT);
       
      if (txAttributes.size() > 1) {
          //如果有多个<tx:attributes/>标签，记录解析异常（只允许有一个<tx:attributes/>标签）
         parserContext.getReaderContext().error(
               "Element <attributes> is allowed at most once inside element <advice>", element);
      }
      else if (txAttributes.size() == 1) {
         //仅有一个<tx:attributes/>标签，继续解析
         Element attributeSourceElement = txAttributes.get(0);
          //解析<tx:attributes/>标签下的子标签
         RootBeanDefinition attributeSourceDefinition = parseAttributeSource(attributeSourceElement, parserContext);
          //解析结果存入propertyValueList中
         builder.addPropertyValue("transactionAttributeSource", attributeSourceDefinition);
      }
      else {
         // 如果没有<tx:attributes/>标签，先假设使用的是注解配置
         builder.addPropertyValue("transactionAttributeSource",
               new RootBeanDefinition("org.springframework.transaction.annotation.AnnotationTransactionAttributeSource"));
      }
   }

   private RootBeanDefinition parseAttributeSource(Element attrEle, ParserContext parserContext) {
       //将<tx:attributes/>下的method配置解析成list集合
      List<Element> methods = DomUtils.getChildElementsByTagName(attrEle, METHOD_ELEMENT);
      ManagedMap<TypedStringValue, RuleBasedTransactionAttribute> transactionAttributeMap = new ManagedMap<>(methods.size());
      transactionAttributeMap.setSource(parserContext.extractSource(attrEle));

      for (Element methodEle : methods) {
          //解析方法名称
         String name = methodEle.getAttribute(METHOD_NAME_ATTRIBUTE);
          //以方法名定义TypedStringValue对象
         TypedStringValue nameHolder = new TypedStringValue(name);
         nameHolder.setSource(parserContext.extractSource(methodEle));

          //事务规则对象，包含回滚规则、传播规则、只读属性、超时时间和隔离级别规则
         RuleBasedTransactionAttribute attribute = new RuleBasedTransactionAttribute();
          //分别解析传播属性、隔离级别、超时时间、只读属性、回滚和不回滚配置
         String propagation = methodEle.getAttribute(PROPAGATION_ATTRIBUTE);
         String isolation = methodEle.getAttribute(ISOLATION_ATTRIBUTE);
         String timeout = methodEle.getAttribute(TIMEOUT_ATTRIBUTE);
         String readOnly = methodEle.getAttribute(READ_ONLY_ATTRIBUTE);
         if (StringUtils.hasText(propagation)) {
            attribute.setPropagationBehaviorName(RuleBasedTransactionAttribute.PREFIX_PROPAGATION + propagation);
         }
         if (StringUtils.hasText(isolation)) {
            attribute.setIsolationLevelName(RuleBasedTransactionAttribute.PREFIX_ISOLATION + isolation);
         }
         if (StringUtils.hasText(timeout)) {
            try {
               attribute.setTimeout(Integer.parseInt(timeout));
            }
            catch (NumberFormatException ex) {
               parserContext.getReaderContext().error("Timeout must be an integer value: [" + timeout + "]", methodEle);
            }
         }
         if (StringUtils.hasText(readOnly)) {
            attribute.setReadOnly(Boolean.valueOf(methodEle.getAttribute(READ_ONLY_ATTRIBUTE)));
         }

          //回滚规则解析
         List<RollbackRuleAttribute> rollbackRules = new LinkedList<>();
         if (methodEle.hasAttribute(ROLLBACK_FOR_ATTRIBUTE)) {
            String rollbackForValue = methodEle.getAttribute(ROLLBACK_FOR_ATTRIBUTE);
             //解析回滚配置
            addRollbackRuleAttributesTo(rollbackRules,rollbackForValue);
         }
         if (methodEle.hasAttribute(NO_ROLLBACK_FOR_ATTRIBUTE)) {
            String noRollbackForValue = methodEle.getAttribute(NO_ROLLBACK_FOR_ATTRIBUTE);
             //解析不回滚配置
            addNoRollbackRuleAttributesTo(rollbackRules,noRollbackForValue);
         }
         attribute.setRollbackRules(rollbackRules);
          //将解析的结果放入方法名对象nameHolder中
         transactionAttributeMap.put(nameHolder, attribute);
      }

      RootBeanDefinition attributeSourceDefinition = new RootBeanDefinition(NameMatchTransactionAttributeSource.class);
      attributeSourceDefinition.setSource(parserContext.extractSource(attrEle));
      attributeSourceDefinition.getPropertyValues().add("nameMap", transactionAttributeMap);
      return attributeSourceDefinition;
   }

    //解析回滚配置，可以用","配置多种回滚异常，解析成list
   private void addRollbackRuleAttributesTo(List<RollbackRuleAttribute> rollbackRules, String rollbackForValue) {
      String[] exceptionTypeNames = StringUtils.commaDelimitedListToStringArray(rollbackForValue);
      for (String typeName : exceptionTypeNames) {
         rollbackRules.add(new RollbackRuleAttribute(StringUtils.trimWhitespace(typeName)));
      }
   }

   private void addNoRollbackRuleAttributesTo(List<RollbackRuleAttribute> rollbackRules, String noRollbackForValue) {
      String[] exceptionTypeNames = StringUtils.commaDelimitedListToStringArray(noRollbackForValue);
      for (String typeName : exceptionTypeNames) {
         rollbackRules.add(new NoRollbackRuleAttribute(StringUtils.trimWhitespace(typeName)));
      }
   }

}
```

以上是解析xml事务通知配置的代码，接下来看注释方式的解析代码

### 注解解析

注解方式的配置，需要AOP的支持，AOP默认采用jdk动态代理实现，如果类没有实现接口，无法生成代理类，如果遇到此类情况，需要开启cglib代理的支持

```xml
<tx:annotation-driven transaction-manager="transactionManager" proxy-target-class="true"/>
```

注解解析器的实现为AnnotationDrivenBeanDefinitionParser

```java
class AnnotationDrivenBeanDefinitionParser implements BeanDefinitionParser {

    //解析器实现
   @Override
   @Nullable
   public BeanDefinition parse(Element element, ParserContext parserContext) {
       //注册TransactionalEventListenerFactory
      registerTransactionalEventListenerFactory(parserContext);
       //解析mode属性
      String mode = element.getAttribute("mode");
      if ("aspectj".equals(mode)) {
         // mode="aspectj"
         registerTransactionAspect(element, parserContext);
      }
      else {
         // mode="proxy"
         AopAutoProxyConfigurer.configureAutoProxyCreator(element, parserContext);
      }
      return null;
   }
```

首先注册了TransactionalEventListenerFactory，然后获取mode属性，根据mode属性做进一步解析，首先分析proxy模式

```java
//静态内部类，aop代理配置
private static class AopAutoProxyConfigurer {

		public static void configureAutoProxyCreator(Element element, ParserContext parserContext) {
			//1、注册InfrastructureAdvisorAutoProxyCreator
			AopNamespaceUtils.registerAutoProxyCreatorIfNecessary(parserContext, element);

			String txAdvisorBeanName = TransactionManagementConfigUtils.TRANSACTION_ADVISOR_BEAN_NAME;
			if (!parserContext.getRegistry().containsBeanDefinition(txAdvisorBeanName)) {
				Object eleSource = parserContext.extractSource(element);

				//2、创建TransactionAttributeSource definition，之前xml的分析中，如果没有<tx:attributes/>标签，和这里设置的一样
				RootBeanDefinition sourceDef = new RootBeanDefinition(
						"org.springframework.transaction.annotation.AnnotationTransactionAttributeSource");
				sourceDef.setSource(eleSource);
				//设置role属性，标记为内部bean
				sourceDef.setRole(BeanDefinition.ROLE_INFRASTRUCTURE);
				//如果@Component/@Repository/@Service/@Controller注解中配置了value属性，value值作为bean的名称，否则用默认名称
				String sourceName = parserContext.getReaderContext().registerWithGeneratedName(sourceDef);

				//3、创建TransactionInterceptor definition.
				RootBeanDefinition interceptorDef = new RootBeanDefinition(TransactionInterceptor.class);
				interceptorDef.setSource(eleSource);
				//也标记为内部bean
				interceptorDef.setRole(BeanDefinition.ROLE_INFRASTRUCTURE);
				//注册事务管理器
				registerTransactionManager(element, interceptorDef);
				//将sourceName对应的bean注入TransactionInterceptor.transactionAttributeSource
				interceptorDef.getPropertyValues().add("transactionAttributeSource", new RuntimeBeanReference(sourceName));
				//获取bean的名称
				String interceptorName = parserContext.getReaderContext().registerWithGeneratedName(interceptorDef);

				//4、创建TransactionAttributeSourceAdvisor definition.
				RootBeanDefinition advisorDef = new RootBeanDefinition(BeanFactoryTransactionAttributeSourceAdvisor.class);
				advisorDef.setSource(eleSource);
				//也标记为内部bean
				advisorDef.setRole(BeanDefinition.ROLE_INFRASTRUCTURE);
				//将sourceName对应的bean注入BeanFactoryTransactionAttributeSourceAdvisor.transactionAttributeSource
				advisorDef.getPropertyValues().add("transactionAttributeSource", new RuntimeBeanReference(sourceName));
				//将interceptorName对应的bean注入BeanFactoryTransactionAttributeSourceAdvisor.adviceBeanName
				advisorDef.getPropertyValues().add("adviceBeanName", interceptorName);
				//解析order属性
				if (element.hasAttribute("order")) {
					advisorDef.getPropertyValues().add("order", element.getAttribute("order"));
				}
				//将TransactionAttributeSourceAdvisor以txAdvisorBeanName为名称注册到IOC容器中
				parserContext.getRegistry().registerBeanDefinition(txAdvisorBeanName, advisorDef);

				//5、创建并注册CompositeComponentDefinition
				CompositeComponentDefinition compositeDef = new CompositeComponentDefinition(element.getTagName(), eleSource);
				compositeDef.addNestedComponent(new BeanComponentDefinition(sourceDef, sourceName));
				compositeDef.addNestedComponent(new BeanComponentDefinition(interceptorDef, interceptorName));
				compositeDef.addNestedComponent(new BeanComponentDefinition(advisorDef, txAdvisorBeanName));
				parserContext.registerComponent(compositeDef);
			}
		}
	}
```

以上这段代码，就是Spring声明式事务的核心，整体看了个大概，接下来就分析细节。

#### InfrastructureAdvisorAutoProxyCreator

它是做什么的？先看它的UML关系图

![](InfrastructureAdvisorAutoProxyCreator_uml.png)

不难发现它间接实现了BeanPostProcessor接口，因此在bean实例化前后分别会调用postProcessBeforeInitialization和postProcessAfterInitialization方法

```java
@Override
public Object postProcessBeforeInitialization(Object bean, String beanName) {
    //返回当前bean
   return bean;
}
```

postProcessBeforeInitialization没有太多的操作，直接返回当前bean。

```java
//如果bean的子类标记了bean需要代理，使用配置的拦截器创建代理
@Override
public Object postProcessAfterInitialization(@Nullable Object bean, String beanName) throws BeansException {
   if (bean != null) {
       //根据beanName和beanClass创建cacheKey
      Object cacheKey = getCacheKey(bean.getClass(), beanName);
       //判断当前类是否已经被代理
      if (!this.earlyProxyReferences.contains(cacheKey)) {
          //装饰器包装bean
         return wrapIfNecessary(bean, beanName, cacheKey);
      }
   }
   return bean;
}
```

```java
protected Object wrapIfNecessary(Object bean, String beanName, Object cacheKey) {
		//如果已经处理过，直接返回bean
		if (StringUtils.hasLength(beanName) && this.targetSourcedBeans.contains(beanName)) {
			return bean;
		}
		//如果bean无需代理，直接返回
		if (Boolean.FALSE.equals(this.advisedBeans.get(cacheKey))) {
			return bean;
		}
		//判断是否是Spring内部bean，内部bean无需代理
		if (isInfrastructureClass(bean.getClass()) || shouldSkip(bean.getClass(), beanName)) {
			this.advisedBeans.put(cacheKey, Boolean.FALSE);
			return bean;
		}

		// 获取当前bean的Advisor[]
		Object[] specificInterceptors = getAdvicesAndAdvisorsForBean(bean.getClass(), beanName, null);
		if (specificInterceptors != DO_NOT_PROXY) {
			//把bean标记为已代理放入缓存中
			this.advisedBeans.put(cacheKey, Boolean.TRUE);
			//创建bean的代理类
			Object proxy = createProxy(
					bean.getClass(), beanName, specificInterceptors, new SingletonTargetSource(bean));
			this.proxyTypes.put(cacheKey, proxy.getClass());
			return proxy;
		}

		this.advisedBeans.put(cacheKey, Boolean.FALSE);
		return bean;
	}
```

postProcessAfterInitialization主要做了以下处理

1. 判断类是否已经被代理，如果已被代理，返回bean，否则继续下一步；
2. 获取当前bean的Advisor，如果当前bean无需代理，返回bean；
3. 生成代理bean并返回；

以上的代码其实就是Spring中AOP对bean的代理增强，后续将分析如何提取事务注解并使事务生效的