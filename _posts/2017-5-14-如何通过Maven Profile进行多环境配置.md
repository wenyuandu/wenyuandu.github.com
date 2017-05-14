---
layout:     post
title:      "如何通过Maven Profile进行多环境配置"
date:       2017-05-14 15:51:00
author:     "Wenyuan Du"
catalog: 	true
tags:
    - spring
---

在系统开发，测试，到最终上线发布的过程中，切换环境是一个很常见的需求，不同环境中的参数（例如数据库配置、日志系统的级别）并不相同，如果每次切环境都要手动修改配置，实在称不上高效优雅，这就使得如何低成本地切换环境成为了一个非常实际的问题。

一个解决思路是首先在资源文件中设置好不同环境中用到的参数，然后在构建阶段确定要将哪一个资源文件编译到应用包中。根据这个思路我们可以通过Maven Profile来完成多环境配置，下文举一个配置jdbc的简单例子方便大家理解。
</br>
######1. 资源文件
在资源文件中设置好jdbc连接的地址、密码等基本参数
![](http://upload-images.jianshu.io/upload_images/3478473-56ea3acba2efab7d.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

</br>
######2. 在pom.xml中配置profile
```maven
<profiles>
    <!--开发环境-->
    <profile>
        <id>development</id>
        <properties>
            <environment>development</environment>
        </properties>
        <!--默认选择这个配置-->
        <activation>
            <activeByDefault>true</activeByDefault>
        </activation>
    </profile>
    <!--生产环境-->
    <profile>
        <id>production</id>
        <properties>
            <environment>production</environment>
        </properties>
    </profile>
</profiles>
```
</br>
######3. 配置资源文件
```maven
<build>
    <resources>
        <resource>
            <directory>src/main/resources</directory>
            <excludes>
                <exclude>properties/*</exclude>
            </excludes>
        </resource>
        <resource>  
            <directory>src/main/resources/properties/${environment}</directory>  
        </resource>  
    </resources>
</build>
```
</br>
######4.Maven构建
```
mvn clean package -P development
mvn clean package -P production
```
分别构建出开发环境或生产环境的war包，若不设置-P参数，按照我们上文中pom.xml的配置，会默认选择开发环境的profile。

</br>
####缺点
通过Maven Profile实现多环境配置的一个缺点是每次切换环境都必须重新构建，而重新构建可能会引入bug，我准备在下一篇文章中介绍另一种更加灵活的多环境配置解决方案，即Spring Profile。
