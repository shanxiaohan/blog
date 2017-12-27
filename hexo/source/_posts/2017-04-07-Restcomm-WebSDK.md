---
layout: post
title: 'Restcomm Web-SDK环境搭建'
date: 2017-04-07
tags: [Restcomm]
---

实验室的项目= =由于甲方购买了Restcomm的产品(Restcomm是一款云通讯平台，可以构建基于web的音视频通信和即时消息等应用)，希望我们在开发新需求时能够利用Restcomm的平台（p.s.项目背景是开发一个企业内部使用的融合服务器，包括音视频通信，即时消息通信，文件传输，地图服务等，这里的音视频通信可以直接使用Restcomm的Web-SDK，内部封装了和sip-servlet消息转换的部分）。于是在实验室服务器上自己搭了个环境，跑了个hello-world还是遇到一些问题的，先Mark一下~  

##### 环境准备  

是在自己的246服务器上从零开始的，主要需要node环境和restcomm-server.  

- 1. Restcomm Server  
  之前在owncloud上传过安装包，[链接](
http://10.109.247.139:8002/owncloud/index.php/apps/files/?dir=/RestComm/software&fileid=22) ，我选择的是Restcomm-JBoss-AS7-8.1.0.1145，解压后需要修改一下配置文件才能启动：  

```shell
  配置文件路径：  
  vim /home/shanxiaohan/node/restcomm/Restcomm-JBoss-AS7-8.1.0.1145/bin/restcomm/restcomm.conf  
  # Network configuration
  NET_INTERFACE='eth0'
  PRIVATE_IP='10.109.247.246'
  SUBNET_MASK='255.255.255.0'
  NETWORK='10.109.247.0'
  BROADCAST_ADDRESS='10.109.247.255'
  # PUBLIC IP ADDRESS
  STATIC_ADDRESS='10.109.247.246'
  #hostname of the server to be used at restcomm.xml. If not set the STATIC_ADDRESS will be used.
  RESTCOMM_HOSTNAME='10.109.247.246'
```

参考文档：  
[Restcomm Web SDK](http://documentation.telestax.com/connect/sdks/restcomm-client-web-sdk-quick-start.html#restcomm)  

启动Restcomm-server:  

```shell
cd /home/shanxiaohan/node/restcomm/Restcomm-JBoss-AS7-8.1.0.1145/bin/restcomm/  

./start-restcomm.sh
```

这时候启动restcomm-server会同时启动media-server，mediaserver启动可能会有报错，修改一下mediaserver的配置文件  

```shell
cd /home/shanxiaohan/node/restcomm/Restcomm-JBoss-AS7-8.1.0.1145/mediaserver  
#修改配置文件中的host等参数
vim mediaserver.conf
```

再次启动start-restcomm.sh,打印出log信息：  
>TelScale RestComm started running on standalone mode. Terminal session: restcomm.    
>Using IP Address: 10.109.247.246  

表示restcomm-server启动成功，可以浏览器打开8080端口（HTTP协议，若需要HTTPS是不同端口，可以查看restcomm的配置文件），出现默认登陆界面：  
`http://10.109.247.246:8080/olympus/#/`,使用默认账号`alice`, `1234`, 以及`5082`端口   
> 这里提一下两个常用的Linux命令：  
> ps aux | grep restcomm  #查看是否有正在运行的进程（便于kill杀死，第二列为PID）  
> #查看端口被什么进程占用，最后一列显示PID和运行的进程，如这里的26358/node  
> netstat -ap | grep 7080   


- 2. node环境  
  网上教程很多这里就不赘述啦~注意更换npm的registry，可以用淘宝镜像~  
  更新npm:    

    ```shell
      [sudo] npm install npm@latest -g
    ```

  注意安装node_modules的路径问题，否则在自己的工程下找不到模块= =  


- 3. Restcomm Web SDK  

  按照文档跑一个hello-world，发现有几个问题= =：  

  1) 直接跑`node server.js`,会报错找不到`node-static`模块，先查一下node_modules的安装路径：  

> [sudo] npm install node-static -g

> npm root -g  

> node  
> global.module.paths  

查看二者是否有交集  

 2) 






