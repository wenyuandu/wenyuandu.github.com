---
layout:     post
title:      "如何通过Spring Profile进行多环境配置"
date:       2017-05-15 19:12:00
author:     "Wenyuan Du"
catalog: 	true
tags:
    - spring
---

在系统开发，测试，到最终上线发布的过程中，切换环境是一个很常见的需求，不同环境中的参数（例如数据库配置、日志系统的级别）并不相同，如果每次切环境都要手动修改配置，实在称不上高效优雅，这就使得如何低成本地切换环境成为了一个非常实际的问题。

我在上一篇文章[如何通过Maven Profile进行多环境配置](https://wenyuandu.github.io/2017/05/14/如何通过Maven-Profile进行多环境配置/)中，介绍了一种低成本切换环境的解决方案，但是每次切换环境都必须通过Maven重新构建，不但稍嫌麻烦而且有引入bug的可能，所以这次我会介绍另一种更加灵活的解决方案，即Spring Profile。

其实Spring Profile与Maven Profile在解决多环境配置的思路上并没有太大区别，同样是在资源文件中事先设置好不同环境中用到的参数，然后根据环境选择使用哪一个资源文件。唯一的区别是Spring不会在构建的时候选择资源文件，而是在运行的时候选择，所以打出的war包能够直接用于不同的运行环境，而不需要重新构建。下面我依旧举一个简单的例子方便大家理解。

###### 1. 资源文件
在资源文件中设置好不同环境所需的参数
![资源文件列表](http://upload-images.jianshu.io/upload_images/3478473-bdd88be9e87042f8.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)
![开发环境参数](http://upload-images.jianshu.io/upload_images/3478473-f2adfc8d51e5a4fd.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)
![生产环境参数](http://upload-images.jianshu.io/upload_images/3478473-8930d0fc77813954.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

###### 2. 配置Spring Profile
```xml
    <!--开发环境-->
    <beans profile="development">
        <util:properties id="config" location="classpath:properties/development/properties"/>
    </beans>

    <!--生产环境-->
    <beans profile="production">
        <util:properties id="config" location="classpath:properties/production/properties"/>
    </beans>

    <!--其他Bean-->
    <beans>
        <bean id="demoBean" class="com.demo.DemoBean" ></bean>
    </beans>
```
Spring会根据Profile的激活状态创建相应的Bean，而Profile的激活状态取决于两个属性，即spring.profiles.active和spring.profiles.default。
如果设置了spring.profiles.active属性，Spring就会根据它判断哪些Profile处于激活状态，如果没有设置spring.profiles.active，Spring则会退而求其次根据spring.profiles.default判断Profile的激活状态。

###### 3. Bean赋值
```java
@Component
public class DemoBean {

    @Value("#{config.name}")
    private String name;

    public String getName() {
        return name;
    }

    public void setName(String name) {
        this.name = name;
    }
}

```

###### 4. 激活Profile
```java
@RunWith(SpringJUnit4ClassRunner.class)
@ContextConfiguration(locations = { "classpath:applicationContext" })
@ActiveProfiles("development")
public class DemoBeanService {

    @Autowired
    private DemoBean demoBean;

    @Test
    public void main() {
        System.out.println("Current environment: "+demoBean.getName());
    }
}
```

###### 5.结果

![](http://upload-images.jianshu.io/upload_images/3478473-8ad1ae09a54cb465.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)


####常用的Profile激活方式
Spring提供了多种激活Profile的方式，大家可以灵活使用以下任意一种方式激活Profile：
- 作为DispatcherServlet的初始化参数
- 作为Web应用的上下文参数
- 作为JNDI条目
- 作为环境变量
- 作为JVM的系统属性
- 在集成测试类上，使用@ActiveProfiles注解设置

当然在实际工作中比较常用的激活方式还是使用环境变量或JVM的系统属性，接下来我们以JVM属性为例，去掉代码中的@ActiveProfiles("development")，并尝试在IDEA的JVM参数设置中激活production Profile：
```java
@RunWith(SpringJUnit4ClassRunner.class)
@ContextConfiguration(locations = { "classpath:applicationContext" })
//@ActiveProfiles("development")
public class DemoBeanService {

    @Autowired
    private DemoBean demoBean;

    @Test
    public void main() {
        System.out.println("Current environment: "+demoBean.getName());
    }
}
```

![](http://upload-images.jianshu.io/upload_images/3478473-13ef6680fffe46db.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)
结果和预期的一样：


![](http://upload-images.jianshu.io/upload_images/3478473-7102e3f9ddf5aea5.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)
