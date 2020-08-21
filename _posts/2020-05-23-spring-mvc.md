---
layout: post
category: Java
title: spring-mvc那些事儿
tagline: by Snail
tags: 
  - Spring
  - Java
  - 开源框架
published: true
date: 2020-05-23 22:49:52 +08:00
---	
今天来回顾下Spring MVC中那些基础的概念和一些重要的组件。
<!--more-->

SpringMVC采用了"前端控制器"的设计模式，即采用了一个"前端控制器"来响应请求、调用controller、将model交给渲染器、接着返回响应。

## spring MVC流程
1. DispatcherServlet接受客户端http请求
2. DispatcherServlet向控制器映射HandlerMapping根据request获取对应的handlerExecutionChain
    - 包括了handler（controller)
    - interceptorList（拦截器列表）
3. DispatcherServlet通过适配器HandlerAdapter调用Handler（控制器Controller）
4. Handler处理业务逻辑，返回ModelAndView
5. DispatcherServlet得到Handler返回结果ModelAndView
6. DispatcherServlet把ModelAndView传给ViewResolver处理获得view
7. DispatcherServlet调用view.render对model渲染
8. 视图渲染完毕，返回response给客户端

## 拦截器
拦截器主要用于拦截用户请求，并作出相应的处理。目前项目中经常用拦截器来做http请求的日志记录、用户鉴权等工作。它有如下两种方式实现：

### 实现HandlerInterceptor接口
HandlerInterceptor是在spring.web.servlet包下的接口，它主要定义了3个方法，分别是preHandle, postHandle和 afterCompletion。
- preHandle: 该方法会在控制器方法前执行，如果为true则继续执行，否则中断。
    - 判断登陆：可以在该方法中获取到request的header，从而拿到token和userId来判断是否登陆。
    - 日志：也可以在该方法中使用ThreadLocal来记录当前请求的开始时间，从而统计一个请求处理的时长。
- postHandle: 该方法在控制器方法后调用之后，解析视图之前执行。
    - 日志：可以记录modelAndView中的viewName，来判断是否找到了正确的modelAndView。
- afterCompletion: 该方法在视图解析完成后执行。
    - 日志：计算处理时长
    - 资源：资源清理
    
### WebMvcConfigurer配置
HandlerInterceptor接口实现后，需要实现WebMvcConfigurer，并在其实现类中将已经实现好的HandlerInterceptor添加进去，同时可以使用excludePathPatterns或addPathPatterns方法来控制拦截的url路径。

### 执行顺序
整个Interceptor的执行流程是：Interceptor.preHandle->HandleAdapter.handle->Interceptor.postHandle->DispatcherServlet.render->Interceptor.afterCompletion。

对于多个Interceptor的情况，会按照配置的先后顺序来执行对应的方法。其中，preHandle与配置顺序相同，postHandle和afterCompletion与配置顺序相反。

注意和javax.servlet.filter使用时，filter是先执行，随后走interceptor。

## 常用注解和注意事项
- @Controller：控制器注解，以控制器形式被IoC容器管理
- @RequestMapping：用于拦截某个url到某个controller或方法上，配合method=RequestMethod.XXX，params=”param=xxx”
- @ResponseBody：返回体按照json格式返回，且不作为ModelAndView
- Controller方法入参可以定义HttpServletRequest作为参数，来获取request中的信息，比如request.getParameterMap获取参数map

参考：
- [Spring MVC-拦截器](https://www.cnblogs.com/black-spike/p/7813238.html)