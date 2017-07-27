---
layout: post
title:  "死磕Spring源码-AOP"
date:   2017-07-21
desc: "死磕Spring源码-AOP"
keywords: "java,Spring,源码,AOP"
categories: [Java]
tags: [java,Spring,源码,AOP]
icon: icon-html
---

在上一篇Spring源码的依赖注入总结中，我们可以清楚的知道DI有助于应用对象之间的解耦，这章我们来聊聊另外一种常见且有用的解耦方式，在比较大的系统中的日志管理、事务管理等功能，我们可以使用用面向切面编程的思想。在软件开发中，像日志这种散布在应用中多处的功能就是横切关注点，而AOP可以实现横切关注点与它们所影响的对象之间的解耦。

在面向切面编程中，我们只需在一个地方定义通用的功能，然后以声明的方式定义这个功能（Advice）何时（Before、After or Around）要以何种方式在何处（Pointcut）应用，并且我们无需改变使用这个通用功能的类。这样做有两个好处：第一是把每个关注点集中在一个地方而不是分散到项目的各个地方去；第二是代码结构极其简洁，非常好维护，这对于骨子里就有极简主义的程序猿来说真是一大幸事！不得不佩服Spring团队（迷弟脸）....

**AOP三个基本点**

Advice：通知，这定义了切入点的具体要干的事情，分为BeforeAdvice、AfterAdvice以及ThrowsAdvice。

`@Around("timeCost()”)//这时的timeCost()就是一个通知`

Pointcut：切点，这定义了切入点的位置，将需要增加的地方用某个正则表达式进行表示，或根据某个方法名来匹配。利用MethodMatcher中的matches方法来进行匹配判断，在JdkRegexpMethodPointcut中，matches方法实际上是通过JDK来实现正则表达式的匹配。

`@Pointcut("execution(public * com.intelligentler.shuitu.controller.*.*(..))”)`

Advisor：通知器，把具体操作（通知）和待增强的位置（切点）结合起来，定义在哪一个关注点使用哪一个通知。

	@Aspect
	@Component
	@Order(-5)
	public class TimeCostAspect {
	    …….
	}
这里的`TimeCostAspect`就不单单是一个POJO，而且还是一个切面。

**这里有个单例模式的彩蛋：**

	public class DefaultPointcutAdvisor extends AbstractGenericPointcutAdvisor implements Serializable {
	     private Pointcut pointcut = Pointcut.TRUE;
	     public DefaultPointcutAdvisor() {
	     }
	     public DefaultPointcutAdvisor(Advice advice) {
	          this(Pointcut.TRUE, advice);
	     }
	     public DefaultPointcutAdvisor(Pointcut pointcut, Advice advice) {
	          this.pointcut = pointcut;
	          setAdvice(advice);
	     }
	     public void setPointcut(Pointcut pointcut) {
	          this.pointcut = (pointcut != null ? pointcut : Pointcut.TRUE);
	     }
	     @Override
	     public Pointcut getPointcut() {
	          return this.pointcut;
	     }
	     @Override
	     public String toString() {
	          return getClass().getName() + ": pointcut [" + getPointcut() + "]; advice [" + getAdvice() + "]";
	     }
	}
	
在DefaultPointcutAdvisor类中Pointcut被设置为Pointcut.TRUE;Pointcut.TRUE在Pointcut的定义如下：

	//Pointcut.class
	Pointcut TRUE = TruePointcut.INSTANCE;
	//TruePointcut.class
	class TruePointcut implements Pointcut, Serializable {
	     public static final TruePointcut INSTANCE = new TruePointcut();
	     private TruePointcut() {
	     }
	     @Override
	     public ClassFilter getClassFilter() {
	          return ClassFilter.TRUE;
	     }
	     @Override
	     public MethodMatcher getMethodMatcher() {
	          return MethodMatcher.TRUE;
	     }
	     private Object readResolve() {
	          return INSTANCE;
	     }
	     @Override
	     public String toString() {
	          return "Pointcut.TRUE";
	     }
	}
	
重点来了：这里的TruePointcut INSTANCE是一个单例模式，使用static类变量来持有单例，再使用private私有构造函数来确保单例不会被再次创建和实例化。在这个类中的MethodMatcher.TRUE也是类似的单例实现。

**AOP的设计与实现**

Spring的AOP采用的是[动态代理模式](https://zhoum1118.github.io/java/2017/06/22/%E6%AD%BB%E7%A3%95%E8%AE%BE%E8%AE%A1%E6%A8%A1%E5%BC%8F.html)，Spring在代理类中包裹切面，在运行期间把切面织入到Bean中，也就是说Spring用代理类封装了目标类，同时拦截了被通知方法的调用，处理完通知（Advice）后，再把调用转发给真正的目标Bean，也真因为是动态代理，所以Spring的AOP只支持到方法连接点而无法提供字段和构造器接入点（AspectJ和JBoss可以），所以Spring无法创建细粒度的通知。

根据Spring AOP的动态代理的过程，我们可以把AOP的设计分为两大块：第一，需要为目标对象建立代理对象（如何生成代理对象？）；第二，需要启动代理对象的拦截器来完成各种横切面的织入（如何织入横切面同时如何拦截对目标对象方法的调用？）。

**什么是代理对象--AOP的动态代理**

Spring AOP的核心技术是动态代理，对于增强的对象方法，在用户调用这个对象方法（request）的时候其实是调用Spring AOP提前为其生成好的代理对象的相应方法，这个代理对象的方法实现就包含了preOperation—request—postOperation，通过对对象方法的这种拦截，增强了目标对象的方法操作，这种方式就是代理。

**如何生成代理对象？**

Spring的AspectJproxyFactory、ProxyFactoryBean和ProxyFactory封装了代理对象AopProxy的生成过程，代理对象的生成实现过程由JDK的Proxy和CGLIB第三方来实现。

**如何拦截对目标对象方法的调用？**

对于JDK的代理对象，拦截使用的是InvocationHandler的invoke回调入口；对于CGLIB的代理对象，拦截是有设置好的回调callback方法（intercept方法）来完成的。
不管用什么方法生成的代理对象，（在invoke或intercept方法中）对拦截器的调用都是通过proceed方法实现的。

首先会根据配置来对拦截器是否与当前的调用方法相匹配进行判断，如果当前的调用方法与配置的拦截器相匹配，那么相应的拦截器就开始发挥作用。这个过程是一个遍历的过程，它会遍历在代理对象中设置的拦截器链中所有的拦截器，被匹配的拦截器被逐一调用，直到所有的拦截器都被遍历完，才是对目标对象的方法调用，这样就完成了对目标对象方法调用的增强。

应该注意到，Advice通知不是直接对目标对象作用来完成增强的，而是对不同种类的通知通过AdviceAdapter适配器来实现的。




