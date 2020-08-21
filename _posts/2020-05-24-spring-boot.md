---
layout: post
category: Java
title: spring-boot那些事儿
tagline: by Snail
tags: 
  - Spring
  - Java
  - 开源框架
published: true
date: 2020-05-24 11:26:31 +08:00
---	
趁热打铁，来看看spring boot中需要关心的几个重点。
<!--more-->

## 自动配置原理
@SpringBootApplication是spring boot中重要的注解，我们使用它来标记应用的主入口，只要标注了，Spring Boot就会运行这个入口类中的main方法来启动应用。
而@SpringBootApplication注解是一个组合注解，其中涉及到的几个关键注解及其作用列举如下：
- @EnableAutoConfiguration：开启自动配置功能。
    - @AutoConfigurationPackage：添加该注解的类所在的package作为"自动配置package"进行管理。主要是@Import了Registrar.class。
        - Registrar.class：注册扫描的包路径。
        - @Import({AutoConfigurationImportSelector.class})：查找META-INF/spring.factories中的配置（都是@Configuration标记过的类），并加载到容器中。 

## 为什么要用spring boot
简化配置，实现了自动化配置。包括了以下几点：
- 不需要编写太多的xml配置文件
- 基于spring，可以利用spring的ioc等优势，同时方便集成spring的其他组件
- 内置tomcat，省去了打包成war再放入tomcat的步骤

## spring-boot-starter-web
快速搭建一个web应用的maven脚手架工程，它依赖了spring核心组件、spring mvc、jackson、tomcat、日志工具等web常用的依赖。

参考：
- [SpringBoot自动装配原理分析](https://blog.csdn.net/Dongguabai/article/details/80865599)
- [@AutoConfigurationPackage注解](https://blog.csdn.net/ttyy1112/article/details/101284541)
- [为什么要使用SpringBoot?使用SpringBoot的最大好处是什么？](http://www.imooc.com/article/287576)