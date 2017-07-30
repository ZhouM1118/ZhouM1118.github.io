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

Spring AOP的核心技术是动态代理，对于增强的对象方法，在用户调用这个对象方法（request）的时候其实是调用Spring AOP提前为其生成好的代理对象（Proxy）的相应方法，这个代理对象的方法实现就包含了preOperation—request—postOperation，通过对对象方法的这种拦截，增强了目标对象的方法操作，这种方式就是代理。

**如何生成代理对象？**

<img title="死磕Spring源码-AOP分析" 图片1="" src="http://img.mukewang.com/597dad5c00012f2713520784.png" style="width:100%" alt="死磕Spring源码-AOP分析">

通过继承ProxyConfig、AdvisedSupport和ProxyCreatorSupport等基类的功能实现，具体的AOP代理对象的生成，根据不同的需要，分别由AspectJProxyFactory、ProxyFactoryBean和ProxyFactory来完成。对于需要AspectJ的AOP应用，AspectJProxyFactory起到了集成Spring和AspectJ的作用；对于使用Spring AOP的应用，ProxyFactoryBean和ProxyFactory都提供了AOP功能的封装，其中对于ProxyFactoryBean，可以在IOC容器中完成声明配置，而对于ProxyFactory则需要编程式地使用Spring AOP的功能。Spring的AspectJProxyFactory、ProxyFactoryBean和ProxyFactory封装了代理对象AopProxy的生成过程，代理对象的生成实现过程由JDK的Proxy和CGLIB第三方来实现。

下面我们具体分析下ProxyFactoryBean，ProxyFactoryBean是一个FactoryBean，FactoryBean一般是如何生成Bean的呢？在调用getBean方法->doGetBean方法中，我们可以看到无论是直接取单例的bean，还是创建单例、多例、自定义生命周期的bean，都会经过bean = getObjectForBeanInstance(sharedInstance, name, beanName, null);这个方法，深入进去会发现最终会调用FactoryBean的getObject方法生产指定Bean的实例对象（Spring中具体的getObject的实现方法有70多个），也就是说对于FactoryBean而言，调用getBean(String BeanName)得到的是具体某个Bean对象（在getObject中具体实现的）而不是FactoryBean对象本身，如果想得到FactoryBean对象本身，只需要加上&符号即可。在FactoryBean的实现原理中我们可以发现工厂模式和装饰器模式的具体应用。

对于ProxyFactoryBean来说，生成代理对象，也是通过getObject方法封装（修饰）了对目标对象增加的增强处理。那么我们具体分析下ProxyFactoryBean中的getObject方法。

	@Override
	public Object getObject() throws BeansException {
	    //初始化通知器链
	    initializeAdvisorChain();
	    //区分singleton和prototype，生成对应的proxy代理对象
	    if (isSingleton()) {
	        return getSingletonInstance();
	    }
	    else {
	        if (this.targetName == null) {
	            logger.warn("Using non-singleton proxies with singleton targets is often undesirable. " +
	                    "Enable prototype proxies by setting the 'targetName' property.");
	        }
	        return newPrototypeInstance();
	    }
	}

这里的初始化通知器链只发生在第一次通过ProxyFactoryBean去获取代理对象的时候，在initializeAdvisorChain方法里有一个标志位advisorChainInitialized来保证这一点。

生成单例代理对象是在getSingletonInstance方法中完成的，它是生成AopProxy代理对象的调用入口。

	private synchronized Object getSingletonInstance() {
	    if (this.singletonInstance == null) {
	        this.targetSource = freshTargetSource();
	        if (this.autodetectInterfaces && getProxiedInterfaces().length == 0 && !isProxyTargetClass()) {
	            // Rely on AOP infrastructure to tell us what interfaces to proxy.
	            Class<?> targetClass = getTargetClass();
	            if (targetClass == null) {
	                throw new FactoryBeanNotInitializedException("Cannot determine target class for proxy");
	            }
	            setInterfaces(ClassUtils.getAllInterfacesForClass(targetClass, this.proxyClassLoader));
	        }
	        // Initialize the shared singleton instance.
	        super.setFrozen(this.freezeProxy);
	        //通过代理工厂来生成对应的代理对象
	        this.singletonInstance = getProxy(createAopProxy());
	    }
	    return this.singletonInstance;
	}
	
	protected Object getProxy(AopProxy aopProxy) {
	    return aopProxy.getProxy(this.proxyClassLoader);
	}
	
	protected final synchronized AopProxy createAopProxy() {
	    if (!this.active) {
	        activate();
	    }
	    //通过AopProxyFactory工厂类来获取Aop的代理对象AopProxy，生成什么样的代理对象由AdivisedSupport决定，AdivisedSupport作为参数传入createAopProxy方法中，这里表示为this。
	    return getAopProxyFactory().createAopProxy(this);
	}
	
	@Override
	public AopProxy createAopProxy(AdvisedSupport config) throws AopConfigException {
	    if (config.isOptimize() || config.isProxyTargetClass() || hasNoUserSuppliedProxyInterfaces(config)) {
	        //从AdvisedSupport中获取目标对象class
	        Class<?> targetClass = config.getTargetClass();
	        if (targetClass == null) {
	            throw new AopConfigException("TargetSource cannot determine target class: " +
	                    "Either an interface or a target is required for proxy creation.");
	        }
	        //如果targetClass是接口类则使用JDK来生成代理对象，否则使用CGLIB来生成。
	        if (targetClass.isInterface() || Proxy.isProxyClass(targetClass)) {
	            return new JdkDynamicAopProxy(config);
	        }
	        return new ObjenesisCglibAopProxy(config);
	    }
	    else {
	    	//默认使用JDK来生成代理对象
	        return new JdkDynamicAopProxy(config);
	    }
	}
	
<img title="死磕Spring源码-AOP分析" 图片2="" src="http://img.mukewang.com/597dadc80001f1ff09520680.png" style="width:100%" alt="死磕Spring源码-AOP分析">
	

**如何拦截对目标对象方法的调用？**

对于JDK的代理对象，拦截使用的是InvocationHandler的invoke回调入口；对于CGLIB的代理对象，拦截是有设置好的回调callback方法（intercept方法）来完成的。

	@Override
	public Object invoke(Object proxy, Method method, Object[] args) throws Throwable {
	    MethodInvocation invocation;
	    Object oldProxy = null;
	    boolean setProxyContext = false;
	
	    TargetSource targetSource = this.advised.targetSource;
	    Class<?> targetClass = null;
	    Object target = null;
	
	    try {
	        if (!this.equalsDefined && AopUtils.isEqualsMethod(method)) {
	            // The target does not implement the equals(Object) method itself.
	            return equals(args[0]);
	        }
	        else if (!this.hashCodeDefined && AopUtils.isHashCodeMethod(method)) {
	            // The target does not implement the hashCode() method itself.
	            return hashCode();
	        }
	        else if (method.getDeclaringClass() == DecoratingProxy.class) {
	            // There is only getDecoratedClass() declared -> dispatch to proxy config.
	            return AopProxyUtils.ultimateTargetClass(this.advised);
	        }
	        else if (!this.advised.opaque && method.getDeclaringClass().isInterface() &&
	                method.getDeclaringClass().isAssignableFrom(Advised.class)) {
	            // Service invocations on ProxyConfig with the proxy config...
	            return AopUtils.invokeJoinpointUsingReflection(this.advised, method, args);
	        }
	
	        Object retVal;
	
	        if (this.advised.exposeProxy) {
	            // Make invocation available if necessary.
	            oldProxy = AopContext.setCurrentProxy(proxy);
	            setProxyContext = true;
	        }
	
	        // 获取目标对象
	        target = targetSource.getTarget();
	        if (target != null) {
	            targetClass = target.getClass();
	        }
	
	        // 获取拦截器链
	        List<Object> chain = this.advised.getInterceptorsAndDynamicInterceptionAdvice(method, targetClass);
	
	        // 如果没有设定拦截器则直接调用目标对象的对应方法
	        if (chain.isEmpty()) {
	            Object[] argsToUse = AopProxyUtils.adaptArgumentsIfNecessary(method, args);
	            retVal = AopUtils.invokeJoinpointUsingReflection(target, method, argsToUse);
	        }
	        else {
	            // We need to create a method invocation...
	            invocation = new ReflectiveMethodInvocation(proxy, target, method, args, targetClass, chain);
	            // Proceed to the joinpoint through the interceptor chain.
	            retVal = invocation.proceed();
	        }
	
	        // Massage return value if necessary.
	        Class<?> returnType = method.getReturnType();
	        if (retVal != null && retVal == target && returnType.isInstance(proxy) &&
	                !RawTargetAccess.class.isAssignableFrom(method.getDeclaringClass())) {
	            // Special case: it returned "this" and the return type of the method
	            // is type-compatible. Note that we can't help if the target sets
	            // a reference to itself in another returned object.
	            retVal = proxy;
	        }
	        else if (retVal == null && returnType != Void.TYPE && returnType.isPrimitive()) {
	            throw new AopInvocationException(
	                    "Null return value from advice does not match primitive return type for: " + method);
	        }
	        return retVal;
	    }
	    finally {
	        if (target != null && !targetSource.isStatic()) {
	            // Must have come from TargetSource.
	            targetSource.releaseTarget(target);
	        }
	        if (setProxyContext) {
	            // Restore old proxy.
	            AopContext.setCurrentProxy(oldProxy);
	        }
	    }
	}
	
	@Override
	public Object intercept(Object proxy, Method method, Object[] args, MethodProxy methodProxy) throws Throwable {
	    Object oldProxy = null;
	    boolean setProxyContext = false;
	    Class<?> targetClass = null;
	    Object target = null;
	    try {
	        if (this.advised.exposeProxy) {
	            // Make invocation available if necessary.
	            oldProxy = AopContext.setCurrentProxy(proxy);
	            setProxyContext = true;
	        }
	        // May be null. Get as late as possible to minimize the time we
	        // "own" the target, in case it comes from a pool...
	        target = getTarget();
	        if (target != null) {
	            targetClass = target.getClass();
	        }
	        //获取AOP通知
	        List<Object> chain = this.advised.getInterceptorsAndDynamicInterceptionAdvice(method, targetClass);
	        Object retVal;
	        // 如果没有配置AOP通知，则直接调用目标对象的对应方法
	        if (chain.isEmpty() && Modifier.isPublic(method.getModifiers())) {
	            Object[] argsToUse = AopProxyUtils.adaptArgumentsIfNecessary(method, args);
	            retVal = methodProxy.invoke(target, argsToUse);
	        }
	        else {
	            // 启动AOP通知
	            retVal = new CglibMethodInvocation(proxy, target, method, args, targetClass, chain, methodProxy).proceed();
	        }
	        retVal = processReturnType(proxy, target, method, retVal);
	        return retVal;
	    }
	    finally {
	        if (target != null) {
	            releaseTarget(target);
	        }
	        if (setProxyContext) {
	            // Restore old proxy.
	            AopContext.setCurrentProxy(oldProxy);
	        }
	    }
	}

不管用什么方法生成的代理对象，（在invoke或intercept方法中）对拦截器的调用都是通过proceed方法实现的，在proceed方法中完成对目标对象的增强功能，这种实现是通过运行逐个具体的拦截器的拦截方法来完成的。

	@Override
	public Object proceed() throws Throwable {
	    // 从索引为-1的拦截器开始调用，并按序递增，拦截器方法调用完毕则开始调用目标对象的方法
	    if (this.currentInterceptorIndex == this.interceptorsAndDynamicMethodMatchers.size() - 1) {
	        return invokeJoinpoint();
	    }
	
	    Object interceptorOrInterceptionAdvice =
	            this.interceptorsAndDynamicMethodMatchers.get(++this.currentInterceptorIndex);
	    if (interceptorOrInterceptionAdvice instanceof InterceptorAndDynamicMethodMatcher) {
	        InterceptorAndDynamicMethodMatcher dm =
	                (InterceptorAndDynamicMethodMatcher) interceptorOrInterceptionAdvice;
	        // 判断拦截器的拦截方法是否匹配当前的调用方法，匹配则调用拦截器（invoke）方法方法
	        if (dm.methodMatcher.matches(this.method, this.targetClass, this.arguments)) {
	            return dm.interceptor.invoke(this);
	        }
	        else {
	            // 不匹配则递归调用proceed
	            return proceed();
	        }
	    }
	    else {
	        return ((MethodInterceptor) interceptorOrInterceptionAdvice).invoke(this);
	    }
	}


首先会根据配置来判断拦截器是否与当前的调用方法相匹配（matches），如果当前的调用方法与配置的拦截器相匹配，那么相应的拦截器就开始发挥作用。这个过程是一个递归遍历的过程，它会遍历在代理对象中设置的拦截器链中所有的拦截器，被匹配的拦截器被逐一调用，直到所有的拦截器都被遍历完，才是对目标对象的方法调用，这样就完成了对目标对象方法调用的增强。

应该注意到，Advice通知不是直接对目标对象作用来完成增强的，而是对不同种类的通知通过AdviceAdapter适配器来实现的。


参考文献：

《Spring实战》

《Spring技术内幕 深入解析Spring架构与设计原理》


