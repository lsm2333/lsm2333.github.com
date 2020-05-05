---
layout: post
category: Java
title: 加快你的SpringTest
tagline: by Snail
tags: 
  - Java
published: true
date: 2020-03-16 22:34:09 +08:00
---	

2020年了，时隔两年又想起这个小东西。。问：那下次是几年后？
<!--more-->

今天（年？）想记录下之前本地调试遇到的问题。
项目里有很多thrift服务，每次注册/发现服务都需要io开销。这导致了本地执行ut十分不便。最终的解决方案是只加载需要的bean，避免使用@SpringBootTest注解。代码如下：
```
@Import({DruidConfiguration.class})
@PropertySource("/application.properties")
@ComponentScan({"com.modules.*.dao"})
@EnableConfigurationProperties
public class BasicDAOTestConfig {
}
```
```
@RunWith(SpringRunner.class)
@ContextConfiguration(classes = {BasicDAOTestConfig.class})
public class DAOTest {
    @Autowired
    SomeDAO someDAO;
    @Test
    public void testDao() {
        someDAO.select(0l);
    }
}
```
这样就只会加载dao相关，如果只是想测试基本的crud的话，速度很快。
