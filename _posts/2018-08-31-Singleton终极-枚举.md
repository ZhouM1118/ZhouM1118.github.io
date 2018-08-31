---
layout: post
title:  "Singleton终极-枚举"
date:   2018-08-31
desc: "Singleton终极-枚举"
keywords: "Singleton,静态内部类,枚举"
categories: [Java]
tags: [Singleton,静态内部类,枚举]
icon: icon-html
---

Java的同学对写单例都是信手拈来，各大博客都有什么懒汉式，饿汉式，double-checked locking等等，但《Effective Java》推荐了两种更为精简高效的写法：静态内部类和枚举。

# **1 静态内部类**

这种方式能达到double-checked locking一样的功效，但实现更简单。饿汉式只要 Singleton 类被装载了，那么 instance 就会被实例化，并没有达到 lazy loading 效果，因为是static的：

	private static Singleton instance = new Singleton(); 

而静态内部类实现单例对静态域使用延迟初始化，代码如下：

	public class Singleton {  
	    private static class SingletonHolder {  
	    	private static final Singleton INSTANCE = new Singleton();  
	    }  
	    private Singleton (){}  
	    public static final Singleton getInstance() {  
	    	return SingletonHolder.INSTANCE;  
	    }  
	}

从以上代码可以看出，静态内部类SingletonHolder只有在getInstance()方法被调用时才会被加载，实现了Lazy 初始化。而且根据JVM本身机制，静态内部类的加载已经实现了线程安全。但值得注意的是，用户可通过反射机制，借助Java反射AccessibleObject类的setAccessible方法调用私有构造函数来创建新的实例，但不是所有的用户都有这种权限，默认情况下，内核API和扩展目录的代码具有该权限，而类路径或通过URLClassLoader加载的应用程序不拥有此权限。

# **2 枚举**

在JDK1.5后，枚举是实现单例模式的最佳方法。它更简洁，不仅能避免多线程同步问题，而且还自动支持序列化机制，防止反序列化重新创建新的对象，绝对防止多次实例化（用户无法通过 reflection attack 来调用私有构造方法）。代码如下：

	// 赋予单例可以执行的额外操作
	public interface MySingleton {
	    void doSomething();
	}
	
	public enum Singleton implements MySingleton {
	
	    //枚举类其实是省略了private类型的构造函数
	    //private Singleton(){}
	    
	    INSTANCE {
	        @Override
	        public void doSomething() {
	            System.out.println("complete singleton");
	        }
	    };
	
	    public static MySingleton getInstance() {
	        return Singleton.INSTANCE;
	    }
	}

为什么枚举可以实现单例？因为枚举其实省略了private类型的构造函数，而且枚举类的field起始就是枚举类的一个实例。