---
title: Spring泛型注入问题
tags:
  - Spring
category:
  - Java
date: 2021-05-12 10:14:57
---

下面代码如果对象是按照a->d->b的创建顺序则会启动失败，如果按照a->b->d的创建顺序会启动成功，如果没有d对象，也会启动成功，但是我们可以发现d对象和a，b对象并无任何直接的联系。

```java
@Configuration
public class Config {
    @Bean
    public Queue<String> a() {
        return new A<String>();
    }

    @Bean
    @DependsOn("c")
    public Object b(Queue<String> a) {
        return new Object();
    }
}

@Component
@DependsOn({"a"})
public class C {
    @Autowired(required = false)
    private List<Throwable> e;
}

public class A<E> implements List, Queue {
  //实现接口方法
}

@SpringBootApplication
public class DemoApplication {

    public static void main(String[] args) {
        ConfigurableApplicationContext run = SpringApplication.run(DemoApplication.class, args);
    }
    
}

//失败错误如下：
***************************
APPLICATION FAILED TO START
***************************

Description:

Parameter 0 of method b in com.example.demo.Config required a bean of type 'java.util.Queue' that could not be found.


Action:

Consider defining a bean of type 'java.util.Queue' in your configuration.
```

<!--more-->

这里出现问题主要和Spring的泛型注入有关，我们考虑创建D对象时的情况。D对象依赖了一个List<Throwable>类型的对象。

```java
//org.springframework.beans.factory.support.DefaultListableBeanFactory#doResolveDependency

//解析集合类型对象，如List,Map,Array等类型，如上例中的List<Throwable>，里面实际上会获取所有Throwable类型的bean集合并返回
Object multipleBeans = resolveMultipleBeans(descriptor, beanName, autowiredBeanNames, typeConverter);
if (multipleBeans != null) {
  return multipleBeans;
}
//否则当做普通对象处理，因为查询不到Throwable类型的bean，所以会当做普通类型处理，查询List类型的对象注入，因为会进行泛型处理，所以也不会错误注入
Map<String, Object> matchingBeans = findAutowireCandidates(beanName, type, descriptor);
if (matchingBeans.isEmpty()) {
  if (isRequired(descriptor)) {
    raiseNoMatchingBeanFound(type, descriptor.getResolvableType(), descriptor);
  }
  return null;
}
```

接下来我们看下泛型注入逻辑：

```java
//org.springframework.beans.factory.support.DefaultListableBeanFactory#findAutowireCandidates
protected Map<String, Object> findAutowireCandidates(
  @Nullable String beanName, Class<?> requiredType, DependencyDescriptor descriptor) {
	//获取目标类型bean名称，这里会获取所有List类型的bean对象
  String[] candidateNames = BeanFactoryUtils.beanNamesForTypeIncludingAncestors(
    this, requiredType, true, descriptor.isEager());
	//...
  for (String candidate : candidateNames) {
    //isAutowireCandidate方法内部会进行泛型处理
    if (!isSelfReference(beanName, candidate) && isAutowireCandidate(candidate, descriptor)) {
      addCandidateEntry(result, candidate, descriptor, requiredType);
    }
  }
  //...
}  
```

**首先我们思考一个问题，candidateNames集合中包不包含a。**

我们来看a对象的定义

```java
@Bean
public Queue<String> a() {
  return new A<String>();
}
public class A<E> implements List, Queue {
  //实现接口方法
}
```

首先我们目标类型是List，A是List的子类，但是bean定义时是Queue类型，Queue不是List的子类。

问题就出现在这里，如果你去观察代码的实现，如果bean对象已经被创建，则判断a是不是List的子类，否则使用BeanDefinition中的信息，也就是Queue，所以candidateNames是否包含a取决于a是否被创建，也就是bean的创建顺序。但无论那种情况，List<Throwable>都不会被注入，因为isAutowireCandidate方法内部会进行泛型处理，但这个方法会有`副作用`。接下来我们阅读下泛型的处理代码：

```java
protected boolean checkGenericTypeMatch(BeanDefinitionHolder bdHolder, DependencyDescriptor descriptor) {
  //目标类型，也就是List<Throwable>类型
  ResolvableType dependencyType = descriptor.getResolvableType();
  //如果目标类型不存在泛型则直接返回匹配
  if (dependencyType.getType() instanceof Class) {
    // No generic type -> we know it's a Class type-match, so no need to check again.
    return true;
  }
  //..
  //从创建方法中获取候选bean类型，这里是Queue<String>，如果和目标类型不匹配会返回null，这里会返回null
  targetType = getReturnTypeForFactoryMethod(rbd, descriptor);
  //如果targetType为空，则用下面方法获取对象类型，这里会丢失泛型信息，结果为A<?>
  Class<?> beanType = this.beanFactory.getType(bdHolder.getBeanName());
  targetType = ResolvableType.forClass(ClassUtils.getUserClass(beanType));
  //这里会把类型缓存起来，当b对象注入的时候，会发现targetType为A<?>，和目标Queue<String>不同，注入失败
  if (cacheType) {
    rbd.targetType = targetType;
  }
  //这里是处理通用泛型的情况,如上面的?，但是默认descriptor.fallbackMatchAllowed()返回false，这里不过多讨论
  if (descriptor.fallbackMatchAllowed() &&
      (targetType.hasUnresolvableGenerics() || targetType.resolve() == Properties.class)) {
    // Fallback matches allow unresolvable generics, e.g. plain HashMap to Map<String,String>;
    // and pragmatically also java.util.Properties to any Map (since despite formally being a
    // Map<Object,Object>, java.util.Properties is usually perceived as a Map<String,String>).
    return true;
  }
  //A<?>并不是List<Throwable>子类型，所以返回false。
  return dependencyType.isAssignableFrom(targetType);
}
```

通过上面的代码分析，可以看出问题出在缓存上，在第二次对b对象进行注入的时候，getReturnTypeForFactoryMethod(rbd, descriptor);并不会重新执行，导致注入失败。

不同的bean创建顺序，rbd.targetType的值可能为`A<?>`或者`Queue<String>`，也就导致是否存在Queue<String>类型的bean。