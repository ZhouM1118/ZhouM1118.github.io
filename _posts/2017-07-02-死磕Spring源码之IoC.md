---
layout: post
title:  "死磕Spring源码-IoC"
date:   2017-07-02
desc: "死磕Spring源码-IoC"
keywords: "java,Spring,源码,IoC"
categories: [Java]
tags: [java,Spring,源码,IoC]
icon: icon-html
---

我们都知道IoC（Inversion of control）是控制反转，控制反转的核心是依赖反转，那到底什么是“依赖反转”，“哪些方面的控制被反转了？”－－依赖对象的获得被反转了。通过依赖注入的方式获取类对象实例而不是传统的在类自身通过新建（new）类对象来获取。所以这种反转是“责任”的反转，传统的这种对对象的管理是由java类自身来管理，而Spring通过IoC容器来管理，这种对对象的依赖关系的管理被反转了，转到IoC容器来了。对象之间的相互依赖关系由IoC容器进行管理，并由IoC容器完成对象的注入。这种做法降低了类之间的耦合度，同时提高了代码的可测试性。

IoC容器主要有两个容器系列：BeanFactory和ApplicationContext。

**IoC容器主要的接口设计图：**

<img title="死磕Spring源码-IoC源码" 图片1="" src="http://img.mukewang.com/59744b1b0001c33d17101006.png" style="width:100%" alt="死磕Spring源码-IoC源码">

**一、BeanFactory**

接口类BeanFactory提供了Spring中所有IOC容器的最基本的功能规范，来看看**BeanFactory的组织结构图**：

<img title="死磕Spring源码-IoC源码" 图片2="" src="http://img.mukewang.com/597444f50001bb0106261194.png" style="width:100%" alt="死磕Spring源码-IoC源码">

	public interface BeanFactory {
	    //如果我们加了这个转义字符，则得到的是这个IOC容器本身，否则得到的是IOC容器的实例
	     String FACTORY_BEAN_PREFIX = “&”;
	     Object getBean(String name) throws BeansException;
	     <T> T getBean(String name, Class<T> requiredType) throws BeansException;
	     <T> T getBean(Class<T> requiredType) throws BeansException;
	     Object getBean(String name, Object... args) throws BeansException;
	     <T> T getBean(Class<T> requiredType, Object... args) throws BeansException;
	     boolean containsBean(String name);
	     boolean isSingleton(String name) throws NoSuchBeanDefinitionException;
	     boolean isPrototype(String name) throws NoSuchBeanDefinitionException;
	     boolean isTypeMatch(String name, ResolvableType typeToMatch) throws NoSuchBeanDefinitionException;
	     boolean isTypeMatch(String name, Class<?> typeToMatch) throws NoSuchBeanDefinitionException;
	     Class<?> getType(String name) throws NoSuchBeanDefinitionException;
	     String[] getAliases(String name);
	}

所有的IoC容器都需要满足BeanFactory这个基本的接口定义，Spring通过定义BeanDefinition来管理基于Spring的应用中的各种对象以及它们之间的相互依赖关系。对IoC容器来说，BeanDefinition抽象了对Bean的定义的一种数据类型。

**BeanDefinition的组织结构如下图：**

<img title="死磕Spring源码-IoC源码" 图片3="" src="http://img.mukewang.com/59744544000144cd06161220.png" style="width:100%" alt="死磕Spring源码-IoC源码">

BeanFactory是一个接口类，我们来看看一个BeanFactory的具体实现类XmlBeanFactory，从名字上就能看出来这是一个与xml相关的BeanFactory，它是一个可以读取以XML文件方式定义的BeanDefinition的IoC容器。

<img title="死磕Spring源码-IoC源码" 图片4="" src="http://img.mukewang.com/597445bc000120d818280524.png" style="width:100%" alt="死磕Spring源码-IoC源码">

	public class XmlBeanFactory extends DefaultListableBeanFactory {
	
	     private final XmlBeanDefinitionReader reader = new XmlBeanDefinitionReader(this);
	
	     public XmlBeanFactory(Resource resource) throws BeansException {
	          this(resource, null);
	     }
	
	     public XmlBeanFactory(Resource resource, BeanFactory parentBeanFactory) throws BeansException {
	          super(parentBeanFactory);
	          this.reader.loadBeanDefinitions(resource);
	     }
	
	}
	
在XmlBeanFactory这个IOC容器中，初始化了一个XmlBeanDefinitionReader，用这个XmlBeanDefinitionReader来处理XML中的BeanDefinition对象。

**手工创建IOC容器**

	public class XmlIOCTest {
	
	     public static void main(String[] args) {
	//定义BeanDefinition的信息来源，在XmlBeanFactory中作为构造函数的参数传给XmlBeanFactory     
			  ClassPathResource resource = new ClassPathResource("config\\beans.xml”);
	//创建一个BeanFactory的IoC容器
	          DefaultListableBeanFactory factory = new DefaultListableBeanFactory();
	//创建一个加载BeanDefinition的读取器，通过一个回调配置给BeanFactory
	          XmlBeanDefinitionReader reader = new XmlBeanDefinitionReader(factory);
	//从信息来源中加载BeanDefinition
	          reader.loadBeanDefinitions(resource);
	          Admin admin = (Admin)factory.getBean("admin");
	          admin.setName("ming.zhou");
	          admin.setPassword("123");
	          admin.setStatus(1);
	          System.out.println(admin);
	     }
	
	}
	
**输入结果：**

`Admin{id=null, name='ming.zhou', password='123', status=1}`

由上我们可以总结IoC容器的初始化过程（refresh()方法来启动）：（源码查看可以通过loadBeanDefinitions逐步查看）
Resource定位过程－－BeanDefinition的载入－－向IoC容器注册BeanDefinition（调用BeanDefinitionRegistry接口来实现，每个bean放在hashMap中存储）

**二、ApplicationContext**

接口类ApplicationContext是高级形态意义的IOC容器。

IoC容器的初始化是由refresh()方法来启动的，它标志这IoC容器的正式启动，启动过程包含BeanDefinition的Resource定位、载入和注册三个基本过程。

**2.1 Resource定位**

**Resource接口设计图：**

<img title="死磕Spring源码-IoC源码" 图片5="" src="http://img.mukewang.com/597450100001ace507340894.png" style="width:100%" alt="死磕Spring源码-IoC源码">

FileSystemApplicationContext，支持XML定义的BeanDefinition的ApplicationContext，可指定以文件形式的BeanDefinition的读入，文件放在本地文件系统中。

**FileSystemApplicationContext的继承关系图:**

<img title="死磕Spring源码-IoC源码" 图片6="" src="http://img.mukewang.com/597447b90001efaa07380916.png" style="width:100%" alt="死磕Spring源码-IoC源码">

**getResourceByPath()的调用栈：**

<img title="死磕Spring源码-IoC源码" 图片7="" src="http://img.mukewang.com/597447e10001335817380988.png" style="width:100%" alt="死磕Spring源码-IoC源码">

通过查看FileSystemApplicationContext中的getResourceByPath()的方法调用栈可以发现这个**IOC容器资源定位的实现过程：**

<img title="死磕Spring源码-IoC源码" 图片8="" src="http://img.mukewang.com/597447fa0001cf1e17800904.png" style="width:100%" alt="死磕Spring源码-IoC源码">

类AbstractRefreshableApplicationContext对容器初始化源码分析:

	@Override
	protected final void refreshBeanFactory() throws BeansException {
	//如果已经建立了BeanFactory，则销毁并关闭该BeanFactory，保证refresh以后使用的是新建立起来的IoC容器
	    if (hasBeanFactory()) {
	        destroyBeans();
	        closeBeanFactory();
	    }
	    try {
	        DefaultListableBeanFactory beanFactory = createBeanFactory();
	        beanFactory.setSerializationId(getId());
	        customizeBeanFactory(beanFactory);
	        loadBeanDefinitions(beanFactory);
	        synchronized (this.beanFactoryMonitor) {
	            this.beanFactory = beanFactory;
	        }
	    }
	    catch (IOException ex) {
	        throw new ApplicationContextException("I/O error parsing bean definition source for " + getDisplayName(), ex);
	    }
	}
	
	@Override
	public Resource getResource(String location) {
	    Assert.notNull(location, "Location must not be null");
	
	    for (ProtocolResolver protocolResolver : this.protocolResolvers) {
	        Resource resource = protocolResolver.resolve(location, this);
	        if (resource != null) {
	            return resource;
	        }
	    }
	
	    if (location.startsWith("/")) {
	        return getResourceByPath(location);
	    }
	    //处理带有classpath标识的Resource
	    else if (location.startsWith(CLASSPATH_URL_PREFIX)) {
	        return new ClassPathResource(location.substring(CLASSPATH_URL_PREFIX.length()), getClassLoader());
	    }
	    else {
	        try {
	            // 处理URL标识的Resource
	            URL url = new URL(location);
	            return new UrlResource(url);
	        }
	        catch (MalformedURLException ex) {
	            // No URL -> resolve as resource path.
	            return getResourceByPath(location);
	        }
	    }
	}
	
整个过程起源于FileSystemApplicationContext的构造函数的初始化，通过构造函数中的refresh()方法来启动整个应用，在AbstractRefreshableApplicationContext中的refreshBeanFactory()方法中我们可以看到使用了createBeanFactory构建了一个DefaultListableBeanFactory的IOC容器，同时启动资源载入（AbstractRefreshableApplicationContext中的loadBeanDefinitions()方法是抽象方法，因为载入的方式有很多种，具体的载入操作交由其子类去具体实现，这里的子类就是
XmlWebApplicationContext
）；具体的资源载入在XmlWebApplicationContext的loadBeanDefinitions()中读入BeanDefinition时完成（loadBeanDefinitions方法在方法调用栈中第一次出现是在XmlWebApplicationContext中的），载入的具体实现在XmlWebApplicationContext的基类AbstractBeanDefinitionReader中。

**2.2 BeanDefinition的载入与解析**

**loadBeanDefinitions的调用方法栈:**

<img title="死磕Spring源码-IoC源码" 图片9="" src="http://img.mukewang.com/597449790001262916520684.png" style="width:100%" alt="死磕Spring源码-IoC源码">

有了Resource定位对象后，开始BeanDefinition的载入与解析的工作，按照Spring的Bean定义规则来对这个XML的文档树进行解析了，解析工作是交给BeanDefinitionParserDelegate来完成的。BeanDefinition的载入分为两部分：首先是通过调用XML的解析器得到document对象，但这些document对象并没有按照Spring的Bean规则进行解析；然后在完成通用的XML解析后，才开始按照Spring的Bean规则进行解析。按照Spring的Bean规则进行解析过程是在documentReader中实现的，把Bean的id、name、aliase等属性元素读取出来后设置到生成的BeanDefinitionHolder中去。BeanDefinitionHolder是BeanDefinition的封装类，封装了BeanDefinition，Bean的名字和别名，用它来完成向IoC容器注册。

BeanDefinition的载入与解析过程源码解析如下，有点长，但不难，耐心看下去思路还是很清晰的。

	@Override
	//调用入口
	public int loadBeanDefinitions(Resource resource) throws BeanDefinitionStoreException {
	    return loadBeanDefinitions(new EncodedResource(resource));
	}
	//载入XML形式的BeanDefinition
	public int loadBeanDefinitions(EncodedResource encodedResource) throws BeanDefinitionStoreException {
	    Assert.notNull(encodedResource, "EncodedResource must not be null");
	    if (logger.isInfoEnabled()) {
	        logger.info("Loading XML bean definitions from " + encodedResource.getResource());
	    }
	
	    Set<EncodedResource> currentResources = this.resourcesCurrentlyBeingLoaded.get();
	    if (currentResources == null) {
	        currentResources = new HashSet<EncodedResource>(4);
	        this.resourcesCurrentlyBeingLoaded.set(currentResources);
	    }
	    if (!currentResources.add(encodedResource)) {
	        throw new BeanDefinitionStoreException(
	                "Detected cyclic loading of " + encodedResource + " - check your import definitions!");
	    }
	    //得到XML文件，并获取IO的InputSource准备进行读取
	    try {
	        InputStream inputStream = encodedResource.getResource().getInputStream();
	        try {
	            InputSource inputSource = new InputSource(inputStream);
	            if (encodedResource.getEncoding() != null) {
	                inputSource.setEncoding(encodedResource.getEncoding());
	            }
	            return doLoadBeanDefinitions(inputSource, encodedResource.getResource());
	        }
	        finally {
	            inputStream.close();
	        }
	    }
	    catch (IOException ex) {
	        throw new BeanDefinitionStoreException(
	                "IOException parsing XML document from " + encodedResource.getResource(), ex);
	    }
	    finally {
	        currentResources.remove(encodedResource);
	        if (currentResources.isEmpty()) {
	            this.resourcesCurrentlyBeingLoaded.remove();
	        }
	    }
	}
	//具体的读取过程
	protected int doLoadBeanDefinitions(InputSource inputSource, Resource resource)
	            throws BeanDefinitionStoreException {
	    try {
	        //调用XML的解析器得到document对象
	        Document doc = doLoadDocument(inputSource, resource);
	        //启动对BeanDefinition解析过程
	        return registerBeanDefinitions(doc, resource);
	    }
	    catch (BeanDefinitionStoreException ex) {
	        throw ex;
	    }
	    catch (SAXParseException ex) {
	        throw new XmlBeanDefinitionStoreException(resource.getDescription(),
	                "Line " + ex.getLineNumber() + " in XML document from " + resource + " is invalid", ex);
	    }
	    catch (SAXException ex) {
	        throw new XmlBeanDefinitionStoreException(resource.getDescription(),
	                "XML document from " + resource + " is invalid", ex);
	    }
	    catch (ParserConfigurationException ex) {
	        throw new BeanDefinitionStoreException(resource.getDescription(),
	                "Parser configuration exception parsing XML from " + resource, ex);
	    }
	    catch (IOException ex) {
	        throw new BeanDefinitionStoreException(resource.getDescription(),
	                "IOException parsing XML document from " + resource, ex);
	    }
	    catch (Throwable ex) {
	        throw new BeanDefinitionStoreException(resource.getDescription(),
	                "Unexpected exception parsing XML document from " + resource, ex);
	    }
	}
	public int registerBeanDefinitions(Document doc, Resource resource) throws BeanDefinitionStoreException {
	    //得到BeanDefinitionDocumentReader来对XML的BeanDefinition进行解析
	    BeanDefinitionDocumentReader documentReader = createBeanDefinitionDocumentReader();
	    int countBefore = getRegistry().getBeanDefinitionCount();
	    //调用BeanDefinitionDocumentReader的registerBeanDefinitions完成具体的解析过程
	    documentReader.registerBeanDefinitions(doc, createReaderContext(resource));
	    return getRegistry().getBeanDefinitionCount() - countBefore;
	}
	
BeanDefinitionDocumentReader类的registerBeanDefinitions方法

	@Override
	public void registerBeanDefinitions(Document doc, XmlReaderContext readerContext) {
	    this.readerContext = readerContext;
	    logger.debug("Loading bean definitions");
	    Element root = doc.getDocumentElement();
	    doRegisterBeanDefinitions(root);
	}
	protected void doRegisterBeanDefinitions(Element root) {
	    BeanDefinitionParserDelegate parent = this.delegate;
	    this.delegate = createDelegate(getReaderContext(), root, parent);
	
	    if (this.delegate.isDefaultNamespace(root)) {
	        String profileSpec = root.getAttribute(PROFILE_ATTRIBUTE);
	        if (StringUtils.hasText(profileSpec)) {
	            String[] specifiedProfiles = StringUtils.tokenizeToStringArray(
	                    profileSpec, BeanDefinitionParserDelegate.MULTI_VALUE_ATTRIBUTE_DELIMITERS);
	            if (!getReaderContext().getEnvironment().acceptsProfiles(specifiedProfiles)) {
	                if (logger.isInfoEnabled()) {
	                    logger.info("Skipped XML bean definition file due to specified profiles [" + profileSpec +
	                            "] not matching: " + getReaderContext().getResource());
	                }
	                return;
	            }
	        }
	    }
	
	    preProcessXml(root);
	    parseBeanDefinitions(root, this.delegate);
	    postProcessXml(root);
	
	    this.delegate = parent;
	}
	protected void processBeanDefinition(Element ele, BeanDefinitionParserDelegate delegate) {
	    //BeanDefinitionHolder是BeanDefinition的封装类，封装了BeanDefinition，Bean的名字和别名，用它来完成向IoC容器注册
	    BeanDefinitionHolder bdHolder = delegate.parseBeanDefinitionElement(ele);
	    if (bdHolder != null) {
	        bdHolder = delegate.decorateBeanDefinitionIfRequired(ele, bdHolder);
	        try {
	            //向IoC容器注册解析得到的BeanDefinition
	            BeanDefinitionReaderUtils.registerBeanDefinition(bdHolder, getReaderContext().getRegistry());
	        }
	        catch (BeanDefinitionStoreException ex) {
	            getReaderContext().error("Failed to register bean definition with name '" +
	                    bdHolder.getBeanName() + "'", ele, ex);
	        }
	        // Send registration event.
	        getReaderContext().fireComponentRegistered(new BeanComponentDefinition(bdHolder));
	    }
	}
	//BeanDefinitionParserDelegate类方法，具体的解析过程，把Bean的id、name、aliase等属性元素读取出来后设置到生成的BeanDefinitionHolder中去
	public BeanDefinitionHolder parseBeanDefinitionElement(Element ele, BeanDefinition containingBean) {
	    //取得<Bean>元素中的id、name、aliase属性的值
	    String id = ele.getAttribute(ID_ATTRIBUTE);
	    String nameAttr = ele.getAttribute(NAME_ATTRIBUTE);
	
	    List<String> aliases = new ArrayList<String>();
	    if (StringUtils.hasLength(nameAttr)) {
	        String[] nameArr = StringUtils.tokenizeToStringArray(nameAttr, MULTI_VALUE_ATTRIBUTE_DELIMITERS);
	        aliases.addAll(Arrays.asList(nameArr));
	    }
	
	    String beanName = id;
	    if (!StringUtils.hasText(beanName) && !aliases.isEmpty()) {
	        beanName = aliases.remove(0);
	        if (logger.isDebugEnabled()) {
	            logger.debug("No XML 'id' specified - using '" + beanName +
	                    "' as bean name and " + aliases + " as aliases");
	        }
	    }
	
	    if (containingBean == null) {
	        checkNameUniqueness(beanName, aliases, ele);
	    }
	    //对Bean元素的详细解析，比如class、parent、init-method、destroy-method等
	    AbstractBeanDefinition beanDefinition = parseBeanDefinitionElement(ele, beanName, containingBean);
	    if (beanDefinition != null) {
	        if (!StringUtils.hasText(beanName)) {
	            try {
	                if (containingBean != null) {
	                    beanName = BeanDefinitionReaderUtils.generateBeanName(
	                            beanDefinition, this.readerContext.getRegistry(), true);
	                }
	                else {
	                    beanName = this.readerContext.generateBeanName(beanDefinition);
	                    // Register an alias for the plain bean class name, if still possible,
	                    // if the generator returned the class name plus a suffix.
	                    // This is expected for Spring 1.2/2.0 backwards compatibility.
	                    String beanClassName = beanDefinition.getBeanClassName();
	                    if (beanClassName != null &&
	                            beanName.startsWith(beanClassName) && beanName.length() > beanClassName.length() &&
	                            !this.readerContext.getRegistry().isBeanNameInUse(beanClassName)) {
	                        aliases.add(beanClassName);
	                    }
	                }
	                if (logger.isDebugEnabled()) {
	                    logger.debug("Neither XML 'id' nor 'name' specified - " +
	                            "using generated bean name [" + beanName + "]");
	                }
	            }
	            catch (Exception ex) {
	                error(ex.getMessage(), ele);
	                return null;
	            }
	        }
	        String[] aliasesArray = StringUtils.toStringArray(aliases);
	        return new BeanDefinitionHolder(beanDefinition, beanName, aliasesArray);
	    }
	
	    return null;
	}
	
**载入与解析的方法调用流程图：**

<img title="死磕Spring源码-IoC源码" 图片10="" src="http://img.mukewang.com/59744a180001978917380986.png" style="width:100%" alt="死磕Spring源码-IoC源码">

经过以上的载入过程，我们大致完成了IoC容器的Bean对象的数据准备工作或初始化工作，但现在IoC容器BeanDefinition中还只是存在些静态的配置信息，还不能供IoC容器直接使用，要想让IoC容器发挥作用还需要进行BeanDefinition的注册。

**2.3 BeanDefinition的注册**

注册过程相对简单，这个过程是通过调用BeanDefinitionRegistry接口的实现来完成的，这个注册过程把载入过程中解析得到的BeanDefinition向IoC容器进行注册，通过源码分析，我们可以看到在IoC容器中是持有一个HashMap来装BeanDefinition数据的。

**registerBeanDefinition的调用方法栈:**

<img title="死磕Spring源码-IoC源码" 图片11="" src="http://img.mukewang.com/59744a4700014e0a17340788.png" style="width:100%" alt="死磕Spring源码-IoC源码">

	private final Map<String, BeanDefinition> beanDefinitionMap = new ConcurrentHashMap<String, BeanDefinition>(256);
	public static void registerBeanDefinition(
	        BeanDefinitionHolder definitionHolder, BeanDefinitionRegistry registry)
	        throws BeanDefinitionStoreException {
	
	    // Register bean definition under primary name.
	    String beanName = definitionHolder.getBeanName();
	    registry.registerBeanDefinition(beanName, definitionHolder.getBeanDefinition());
	
	    // Register aliases for bean name, if any.
	    String[] aliases = definitionHolder.getAliases();
	    if (aliases != null) {
	        for (String alias : aliases) {
	            registry.registerAlias(beanName, alias);
	        }
	    }
	}
	
	@Override
	public void registerBeanDefinition(String beanName, BeanDefinition beanDefinition)
	        throws BeanDefinitionStoreException {
	
	    Assert.hasText(beanName, "Bean name must not be empty");
	    Assert.notNull(beanDefinition, "BeanDefinition must not be null");
	
	    if (beanDefinition instanceof AbstractBeanDefinition) {
	        try {
	            ((AbstractBeanDefinition) beanDefinition).validate();
	        }
	        catch (BeanDefinitionValidationException ex) {
	            throw new BeanDefinitionStoreException(beanDefinition.getResourceDescription(), beanName,
	                    "Validation of bean definition failed", ex);
	        }
	    }
	
	    BeanDefinition oldBeanDefinition;
	
	    //检查是不是有同名字的BeanDefinition已经在IoC容器中注册了，如果存在且不允许覆盖则抛出异常
	    oldBeanDefinition = this.beanDefinitionMap.get(beanName);
	    if (oldBeanDefinition != null) {
	        if (!isAllowBeanDefinitionOverriding()) {
	            throw new BeanDefinitionStoreException(beanDefinition.getResourceDescription(), beanName,
	                    "Cannot register bean definition [" + beanDefinition + "] for bean '" + beanName +
	                    "': There is already [" + oldBeanDefinition + "] bound.");
	        }
	        else if (oldBeanDefinition.getRole() < beanDefinition.getRole()) {
	            // e.g. was ROLE_APPLICATION, now overriding with ROLE_SUPPORT or ROLE_INFRASTRUCTURE
	            if (this.logger.isWarnEnabled()) {
	                this.logger.warn("Overriding user-defined bean definition for bean '" + beanName +
	                        "' with a framework-generated bean definition: replacing [" +
	                        oldBeanDefinition + "] with [" + beanDefinition + "]");
	            }
	        }
	        else if (!beanDefinition.equals(oldBeanDefinition)) {
	            if (this.logger.isInfoEnabled()) {
	                this.logger.info("Overriding bean definition for bean '" + beanName +
	                        "' with a different definition: replacing [" + oldBeanDefinition +
	                        "] with [" + beanDefinition + "]");
	            }
	        }
	        else {
	            if (this.logger.isDebugEnabled()) {
	                this.logger.debug("Overriding bean definition for bean '" + beanName +
	                        "' with an equivalent definition: replacing [" + oldBeanDefinition +
	                        "] with [" + beanDefinition + "]");
	            }
	        }
	        this.beanDefinitionMap.put(beanName, beanDefinition);
	    }
	    else {
	        if (hasBeanCreationStarted()) {
	            //注册过程需要synchronized，保证数据的一致性
	            synchronized (this.beanDefinitionMap) {
	                this.beanDefinitionMap.put(beanName, beanDefinition);
	                List<String> updatedDefinitions = new ArrayList<String>(this.beanDefinitionNames.size() + 1);
	                updatedDefinitions.addAll(this.beanDefinitionNames);
	                updatedDefinitions.add(beanName);
	                this.beanDefinitionNames = updatedDefinitions;
	                if (this.manualSingletonNames.contains(beanName)) {
	                    Set<String> updatedSingletons = new LinkedHashSet<String>(this.manualSingletonNames);
	                    updatedSingletons.remove(beanName);
	                    this.manualSingletonNames = updatedSingletons;
	                }
	            }
	        }
	        else {
	            // Still in startup registration phase
	            this.beanDefinitionMap.put(beanName, beanDefinition);
	            this.beanDefinitionNames.add(beanName);
	            this.manualSingletonNames.remove(beanName);
	        }
	        this.frozenBeanDefinitionNames = null;
	    }
	
	    if (oldBeanDefinition != null || containsSingleton(beanName)) {
	        resetBeanDefinition(beanName);
	    }
	}

注册过程会检查是不是有同名字的BeanDefinition已经在IoC容器中注册了，如果存在且不允许覆盖则抛出异常，注册过程需要synchronized，保证数据的一致性。
完成了BeanDefinition的注册，IOC容器的初始化过程就结束了，在这个IOC容器中已经有了整个Bean的配置信息。











