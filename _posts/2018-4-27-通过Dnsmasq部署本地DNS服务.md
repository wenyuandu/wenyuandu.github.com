---
layout:     post
title:      "通过Dnsmasq部署本地DNS服务"
date:       2018-4-27 10:18:00
author:     "Wenyuan Du"
catalog: 	true
tags:
    - DNS
---

在开发、测试和正式环境中，我们总希望通过同一个域名找到对应环境中的服务实例，简化配置流程，例如在测试环境中，让api.changjinglu.net关联到IP为192.168.1.34的测试服务器，而在正式环境中，让api.changjinglu.net关联到IP为47.96.51.143的正式服务器。

我们现在的解决方案是在本机的/etc/hosts文件中记录相应的域名IP映射关系，本机在尝试解析一个域名时，会先去/etc/hosts中查找该域名对应的IP，并访问相应IP的服务器。只有当/etc/hosts中没有该域名的记录时，本机才会去DNS服务器进行域名解析。本机解析域名的优先级为DNS缓存>/etc/hosts>DNS服务。

这个解决方案稍显繁琐，因为每台机器都必须在自己的/etc/hosts文件中配置正确的域名IP映射关系，一旦映射关系发生改变，所有机器又必须全部做相应的修改。一个更简洁的解决方案是构建一个本地DNS服务器，让路由器指向该本地DNS服务器，让它统一管理所有通用的域名IP映射，如果个别开发者有自己的特别需要，可以利用域名解析的优先级顺序，通过修改自己本机的/etc/hosts覆盖本地DNS服务的映射关系。使用这个新方案，当局域网中新增某个服务或某个原有服务改变IP地址时，只需要在本地DNS服务器上新增或修改映射配置，局域网中的所有机器无需做修改，就能享受到正确的映射关系了。

下面讲一讲如何通过Dnsmasq实现这个新方案。

###### 1. 安装Dnsmasq
我将本地DNS服务安装在了192.168.1.98上，因为该测试服务器的系统是ubuntu，使用自带的包管理器下载并安装Dnsmasq最简洁。
sudo apt-get install dnsmasq

###### 2. 配置Dnsmasq
Dnsmasq所有的配置都在/etc/dnsmasq.conf文件中完成，按照需要简单做了以下修改。
```conf
    #首先配置resolv-file，这个参数表示dnsmasq会从这个指定的文件中寻找上游DNS服务器

    resolv-file=/etc/resolv.dnsmasq.conf

    #单设置127.0.0.1为只能本机使用，单设置本机IP为只能内部全网使用而本机不能用，这里需要同时设置两者

    listen-address=127.0.0.1,192.168.1.98

    #dnsmasq缓存设置
    
    cache-size=1024
```

然后根据自己设置的resolv-file=/etc/resolv.dnsmasq.conf，配置/etc/resolv.dnsmasq.conf文件，指定上游DNS服务器
```conf
    nameserver 114.114.114.114
```

###### 3. 坑
按以上配置配置好Dnsmasq并启动后，会发现Dnsmasq无法正常解析域名，使用ps -ef | grep dnsmasq查看后发现如下信息
```conf
    dnsmasq  10384     1  0 15:16 ?        00:00:00 /usr/sbin/dnsmasq -x /var/run/dnsmasq/dnsmasq.pid -u dnsmasq -r /var/run/dnsmasq/resolv.conf -7 /etc/dnsmasq.d,.dpkg-dist,.dpkg-old,.dpkg-new --local-service --trust-anchor=.,19036,8,2,49AAC11D7B6F6446702E54A1607371607A1A41855200FD2CE1CDDE32F24E8FB5
```
其中dnsmasq -r /var/run/dnsmasq/resolv.conf说明Dnsmasq是从/var/run/dnsmasq/resolv.conf文件中获取上游DNS服务器，而非我们指定的resolv-file=/etc/resolv.dnsmasq.conf。
查阅了无数文档以后，发现在/etc/default/dnsmasq中有一个IGNORE_RESOLVCONF属性，说明如下
```conf
    # If the resolvconf package is installed, dnsmasq will use its output 
    # rather than the contents of /etc/resolv.conf to find upstream 
    # nameservers. Uncommenting this line inhibits this behaviour.
    # Note that including a "resolv-file=<filename>" line in 
    # /etc/dnsmasq.conf is not enough to override resolvconf if it is
    # installed: the line below must be uncommented.
    # IGNORE_RESOLVCONF=yes
```
这里必须取消IGNORE_RESOLVCONF=yes前的注释，才能让resolv-file=/etc/resolv.dnsmasq.conf生效。

###### 4. 启动Dnsmasq
大功告成，启动Dnsmasq。
sudo service dnsmasq start


###### 5. 设置路由器，将DNS服务指向本地DNS服务器
