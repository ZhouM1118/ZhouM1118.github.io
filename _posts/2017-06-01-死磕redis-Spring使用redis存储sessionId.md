---
layout: post
title:  "死磕redis-Spring使用redis存储sessionId"
date:   2017-06-01
desc: "死磕redis-Spring使用redis存储sessionId"
keywords: "redis, Spring, sessionId"
categories: [Database]
tags: [redis, Spring, sessionId]
icon: icon-html
---

**一、redis 储备知识**

Redis 是完全开源免费的一个高性能的key-value数据库。

Redis支持五种数据类型：string（字符串），hash（哈希），list（列表），set（集合）及zset(sorted set：有序集合)。

string类型是Redis最基本的数据类型，一个键最大能存储512MB。
> 127.0.0.1:6379> set database "redis"
> 
> OK
> 
> 127.0.0.1:6379> get database
> 
> "redis"

Redis hash是一个string类型的field和value的映射表，hash特别适合用于存储对象。每个 hash 可以存储 2^32 -1 键值对（40多亿）。

> 127.0.0.1:6379> hmset user:1 zhoum 24 man coder
> 
> OK
> 
> 127.0.0.1:6379> hgetall user:1
> 
> 1) "zhoum"
> 
> 2) "24"
> 
> 3) "man"
> 
> 4) "coder"

Redis List列表是简单的字符串列表，按照插入顺序排序。列表最多可存储 2^32 - 1 元素。

> 127.0.0.1:6379> lpush databases redis
> 
> (integer) 1
> 
> 127.0.0.1:6379> lpush databases mysql
> 
> (integer) 2
> 
> 127.0.0.1:6379> lpush databases sqlite
> 
> (integer) 3
> 
> 127.0.0.1:6379> lpush databases mongodb
> 
> (integer) 4
> 
> 127.0.0.1:6379> lrange databases 0 10
> 
> 1) "mongodb"
> 
> 2) "sqlite"
> 
> 3) "mysql"
> 
> 4) "redis"

Redis的Set是string类型的无序集合。

> 127.0.0.1:6379> sadd databases redis
> 
> (integer) 1
> 
> 127.0.0.1:6379> sadd databases mysql
> 
> (integer) 1
> 
> 127.0.0.1:6379> sadd databases sqlite
> 
> (integer) 1
> 
> 127.0.0.1:6379> sadd databases sqlite
> 
> (integer) 0
> 
> 127.0.0.1:6379> smembers databases
> 
> 1) "sqlite"
> 
> 2) "mysql"
> 
> 3) "redis"

Redis zset 和 set 一样也是string类型元素的有序集合,且不允许重复的成员。

> 127.0.0.1:6379> zadd databases 0 redis
> 
> (integer) 1
> 
> 127.0.0.1:6379> zadd databases 0 mysql
> 
> (integer) 1
> 
> 127.0.0.1:6379> zadd databases 0 sqlite
> 
> (integer) 1
> 
> 127.0.0.1:6379> zadd databases 0 sqlite
> 
> (integer) 0
> 
> 127.0.0.1:6379> ZRANGEBYSCORE databases 0 1000
> 
> 1) "redis"
> 
> 2) "mysql"
> 
> 3) “sqlite”

**二、Spring使用redis存储sessionid**

pom.xml中的redis部分

	<dependency>
	    <groupId>org.springframework.data</groupId>
	    <artifactId>spring-data-redis</artifactId>
	</dependency>
	
	<dependency>
	    <groupId>redis.clients</groupId>
	    <artifactId>jedis</artifactId>
	</dependency>

配置文件：redis.properties

	redis.maxIdle=100  
	redis.maxActive=300  
	redis.maxWait=1000  
	redis.timeout=100000  
	redis.maxTotal=1000  
	redis.minIdle=8  
	redis.testOnBorrow=true
	redis.host=120.27.35.26
	redis.port=7006
	
Spring redis配置文件：redis-context.xml

	<?xml version="1.0" encoding="UTF-8"?>
	<beans xmlns="http://www.springframework.org/schema/beans"
	    xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance" xmlns:p="http://www.springframework.org/schema/p"
	    xsi:schemaLocation="http://www.springframework.org/schema/beans http://www.springframework.org/schema/beans/spring-beans.xsd">
	
	    <bean id="jedisPoolConfig" class="redis.clients.jedis.JedisPoolConfig">
	        <property name="maxIdle" value="${redis.maxIdle}" />
	        <property name="maxTotal" value="${redis.maxTotal}" />
	        <property name="maxWaitMillis" value="${redis.maxWait}" />
	        <property name="testOnBorrow" value="${redis.testOnBorrow}" />
	    </bean>
	
	    <bean id="jedisConnectionFactory"
	        class="org.springframework.data.redis.connection.jedis.JedisConnectionFactory">
	        <property name="poolConfig" ref="jedisPoolConfig"/>
	        <property name="hostName" value="${redis.host}"/>
	        <property name="port" value="${redis.port}"/>
	    </bean>
	
	    <bean id="redisTemplate" class="org.springframework.data.redis.core.RedisTemplate">
	        <property name="connectionFactory" ref="jedisConnectionFactory" />
	        <property name="keySerializer">
	            <bean
	                class="org.springframework.data.redis.serializer.StringRedisSerializer" />
	        </property>
	        <property name="valueSerializer">
	            <bean
	                class="org.springframework.data.redis.serializer.JdkSerializationRedisSerializer" />
	        </property>
	        <property name="hashKeySerializer">
	            <bean
	                class="org.springframework.data.redis.serializer.StringRedisSerializer" />
	        </property>
	        <property name="hashValueSerializer">
	            <bean
	                class="org.springframework.data.redis.serializer.JdkSerializationRedisSerializer" />
	        </property>
	    </bean>
	</beans>

java代码

	import org.springframework.data.redis.core.RedisTemplate;
	import java.io.Serializable;
	
	@Autowired
	protected RedisTemplate<Serializable, Serializable> redisTemplate;
	
	public void addAdminRedis(String sessionId, Integer aid) {
	    if(null != sessionId && null != aid){
	        redisTemplate.opsForValue().set(sessionId+ADMIN_SUFFIX,aid);
	    }
	}
	
	public AdministratorVo getAdminRedis(String sessionId) {
	    if(null == sessionId){
	        return  null;
	    }
	    if(null == redisTemplate.opsForValue().get(sessionId+ADMIN_SUFFIX)){
	        return  null;
	    }
	    Integer aid = (Integer)redisTemplate.opsForValue().get(sessionId+ADMIN_SUFFIX);
	    AdministratorVo admin = adminMapper.selectAdminById(aid);
	    if(null != admin){
	        admin.setPassword(null);
	    }
	    return admin;
	}
	
	public void deleteAdminRedis(String sessionId) {
	    if(sessionId != null){
	        AdministratorVo vo = getAdminRedis(sessionId+ADMIN_SUFFIX);
	        if(null != vo){
	            redisTemplate.delete(ADMIN_POWER_PREFIX+vo.getId().toString());
	        }
	
	        redisTemplate.delete(sessionId+ADMIN_SUFFIX);
	    }
	}