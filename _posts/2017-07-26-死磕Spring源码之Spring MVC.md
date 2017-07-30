---
layout: post
title:  "死磕Spring源码-Spring MVC"
date:   2017-07-26
desc: "死磕Spring源码-Spring MVC"
keywords: "java,Spring,源码,Spring MVC"
categories: [Java]
tags: [java,Spring,源码,Spring MVC]
icon: icon-html
---

Spring IoC是Spring的核心，所以IoC也是Spring MVC的基础，Spring MVC 会建立起一个IOC容器体系，IoC的启动过程与web容器启动的过程是集成在一起的，它们的启动由ContextLoaderListener监听器负责完成。建立IOC容器体系后，DispatcherServlet作为Spring MVC处理web请求的转发器也建立起来，从而完成响应HTTP请求的准备工作。

**IOC容器的启动（ContextLoaderListener）**

IoC容器的启动过程就是建立上下文的过程，由ContextLoaderListener启动的上下文为根上下文，在根上下文的基础上，还有一个与Web MVC相关的上下文作为根上下文的子上下文来保存DispatcherServlet所需要的MVC对象，构成一个上下文体系。这个上下文体系的建立与初始化是由ContextLoader来完成的。

	public class ContextLoaderListener extends ContextLoader implements ServletContextListener {
	     public ContextLoaderListener() {
	     }
	     public ContextLoaderListener(WebApplicationContext context) {
	          super(context);
	     }
	     @Override
	     public void contextInitialized(ServletContextEvent event) {
	          initWebApplicationContext(event.getServletContext());
	     }
	     @Override
	     public void contextDestroyed(ServletContextEvent event) {
	          closeWebApplicationContext(event.getServletContext());
	          ContextCleanupListener.cleanupAttributes(event.getServletContext());
	     }
	}
	
在ContextLoaderListener中实现了ServletContextListener接口，这个接口里的函数会结合Web容器的生命周期被调用（contextInitialized，服务器启动时被调用；contextDestroyed，服务器关闭时被调用）。因为ServletContextListener是ServletContext的监听者，如果ServletContext发生变化，会触发预先设计好的动作，比如初始化和销毁。比如初始化：

	ContextLoaderListener.contextInitialized
	@Override
	public void contextInitialized(ServletContextEvent event) {
	    initWebApplicationContext(event.getServletContext());
	}
	//初始化WebApplicationContext根上下文
	public WebApplicationContext initWebApplicationContext(ServletContext servletContext) {
	    //根上下文作为唯一实例存在，用ROOT_WEB_APPLICATION_CONTEXT_ATTRIBUTE标志位来实现
	    if (servletContext.getAttribute(WebApplicationContext.ROOT_WEB_APPLICATION_CONTEXT_ATTRIBUTE) != null) {
	        throw new IllegalStateException(
	                "Cannot initialize context because there is already a root application context present - " +
	                "check whether you have multiple ContextLoader* definitions in your web.xml!");
	    }
	
	    Log logger = LogFactory.getLog(ContextLoader.class);
	    servletContext.log("Initializing Spring root WebApplicationContext");
	    if (logger.isInfoEnabled()) {
	        logger.info("Root WebApplicationContext: initialization started");
	    }
	    long startTime = System.currentTimeMillis();
	
	    try {
	        if (this.context == null) {
	            //创建根上下文
	            this.context = createWebApplicationContext(servletContext);
	        }
	        if (this.context instanceof ConfigurableWebApplicationContext) {
	            //转型为需要产生的IoC容器
	            ConfigurableWebApplicationContext cwac = (ConfigurableWebApplicationContext) this.context;
	            if (!cwac.isActive()) {
	                if (cwac.getParent() == null) {
	                    // 载入根上下文的双亲上下文
	                    ApplicationContext parent = loadParentContext(servletContext);
	                    cwac.setParent(parent);
	                }
	                configureAndRefreshWebApplicationContext(cwac, servletContext);
	            }
	        }
	        //设置ROOT_WEB_APPLICATION_CONTEXT_ATTRIBUTE标志位
	   servletContext.setAttribute(WebApplicationContext.ROOT_WEB_APPLICATION_CONTEXT_ATTRIBUTE, this.context);
	
	        ClassLoader ccl = Thread.currentThread().getContextClassLoader();
	        if (ccl == ContextLoader.class.getClassLoader()) {
	            currentContext = this.context;
	        }
	        else if (ccl != null) {
	            currentContextPerThread.put(ccl, this.context);
	        }
	
	        if (logger.isDebugEnabled()) {
	            logger.debug("Published root WebApplicationContext as ServletContext attribute with name [" +
	                    WebApplicationContext.ROOT_WEB_APPLICATION_CONTEXT_ATTRIBUTE + "]");
	        }
	        if (logger.isInfoEnabled()) {
	            long elapsedTime = System.currentTimeMillis() - startTime;
	            logger.info("Root WebApplicationContext: initialization completed in " + elapsedTime + " ms");
	        }
	
	        return this.context;
	    }
	    catch (RuntimeException ex) {
	        logger.error("Context initialization failed", ex);
	        servletContext.setAttribute(WebApplicationContext.ROOT_WEB_APPLICATION_CONTEXT_ATTRIBUTE, ex);
	        throw ex;
	    }
	    catch (Error err) {
	        logger.error("Context initialization failed", err);
	        servletContext.setAttribute(WebApplicationContext.ROOT_WEB_APPLICATION_CONTEXT_ATTRIBUTE, err);
	        throw err;
	    }
	}
	
	protected WebApplicationContext createWebApplicationContext(ServletContext sc) {
	    //判断使用什么样的类作为Web容器中的IoC容器
	    Class<?> contextClass = determineContextClass(sc);
	    if (!ConfigurableWebApplicationContext.class.isAssignableFrom(contextClass)) {
	        throw new ApplicationContextException("Custom context class [" + contextClass.getName() +
	                "] is not of type [" + ConfigurableWebApplicationContext.class.getName() + "]");
	    }
	    return (ConfigurableWebApplicationContext) BeanUtils.instantiateClass(contextClass);
	}

	protected Class<?> determineContextClass(ServletContext servletContext) {
	    String contextClassName = servletContext.getInitParameter(CONTEXT_CLASS_PARAM);
	    //如果在ServletContext中配置了CONTEXT_CLASS_PARAM，则使用这个class反射根上下文。
	    if (contextClassName != null) {
	        try {
	            return ClassUtils.forName(contextClassName, ClassUtils.getDefaultClassLoader());
	        }
	        catch (ClassNotFoundException ex) {
	            throw new ApplicationContextException(
	                    "Failed to load custom context class [" + contextClassName + "]", ex);
	        }
	    }
	    else {//如果没有配置CONTEXT_CLASS_PARAM，则使用默认的contextClass来反射根上下文
	        contextClassName = defaultStrategies.getProperty(WebApplicationContext.class.getName());
	        try {
	            return ClassUtils.forName(contextClassName, ContextLoader.class.getClassLoader());
	        }
	        catch (ClassNotFoundException ex) {
	            throw new ApplicationContextException(
	                    "Failed to load default context class [" + contextClassName + "]", ex);
	        }
	    }
	}

	//设置ServletContext以及配置各类参数，最终调用refresh方法来初始化容器
	protected void configureAndRefreshWebApplicationContext(ConfigurableWebApplicationContext wac, ServletContext sc) {
	    if (ObjectUtils.identityToString(wac).equals(wac.getId())) {
	        // The application context id is still set to its original default value
	        String idParam = sc.getInitParameter(CONTEXT_ID_PARAM);
	        if (idParam != null) {
	            wac.setId(idParam);
	        }
	        else {
	            // Generate default id...
	            wac.setId(ConfigurableWebApplicationContext.APPLICATION_CONTEXT_ID_PREFIX +
	                    ObjectUtils.getDisplayString(sc.getContextPath()));
	        }
	    }
	
	    wac.setServletContext(sc);
	    String configLocationParam = sc.getInitParameter(CONFIG_LOCATION_PARAM);
	    if (configLocationParam != null) {
	        wac.setConfigLocation(configLocationParam);
	    }
	
	    ConfigurableEnvironment env = wac.getEnvironment();
	    if (env instanceof ConfigurableWebEnvironment) {
	        ((ConfigurableWebEnvironment) env).initPropertySources(sc, null);
	    }
	
	    customizeContext(sc, wac);
	    wac.refresh();
	}
	
ContextLoaderListener初始化根上下文过程图

<img title="死磕Spring源码-SpringMVC" 图片1="" src="http://img.mukewang.com/597dfb49000101aa16560728.png" style="width:100%" alt="初始化根上下文过程图">

参考文献：

《Spring技术内幕 深入解析Spring架构与设计原理》

