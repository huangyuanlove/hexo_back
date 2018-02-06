---
title: Jenkins安装与使用
date: 2017-06-28 17:30:42
tags: [jenkins,运维,服务器]
---
公司需求，业务越来越多，服务器越来越多，后台部署项目麻烦的要死，于是上了`jenkins`这货。
关于这货是干嘛的，请移步这里[https://jenkins.io/](https://jenkins.io/),下载请移步这里[https://jenkins.io/download/](https://jenkins.io/download/)
安装环境：ubuntu 16.04、tomcat7(这个是因为Jenkins是个war包)、maven(这个是因为后台的项目是maven工程)、jdk8(这个是因为需要tomcat)、gradle(构建Android工程)、SDK(构建Android工程)
<!--more-->
#### Jenkins 环境
安装JDK、Tomcat、MAVEN(如有需要)、gradle(构建Android工程)、SDK(构建Android工程)
#### 安装Jenkins
在上面的下载地址下载war，丢到tomcat的webapps文件夹下，然后启动tomcat，浏览器访问jenkin工程`127.0.0.1:8080/jenkins`如下图所示：
![jenkins首次启动](/image/jenkins/jenkins_start.png)
打开红字提示的文件，里面是首次进入时需要的密码,点击右下角continue。
##### jenkins安装插件
可以自定义也可以安装推荐的插件，我这里安装的是建议的插件.稍等一会，插件安装完成后会自动跳转到创建jenkins用户的界面
##### 创建jenkins用户
按照提示创建即可，点击右下角`Save and finish`
##### 配置Jenkins
以后再次访问jenkins的时候就不用再去使用初始密码，只需要使用上一步创建的账号即可，登录后首界面如下，
![jenkins首页](/image/jenkins/jenkins_index.png)
点击左侧的`系统管理`，在中间列出的工具里面点击`Global Tool Configuration`，指定JDK和MAVEN的路径，如下
![jenkins配置JDK](/image/jenkins/jenkins_jdk.png)
![jenkins配置MAVEN](/image/jenkins/jenkins_maven.png)
然后Save保存
#### 创建一个新的Maven工程
点击左侧菜单栏`新建`,输入工程名字，然后选择`构建一个只有风格的软件项目`,然后`ok`
![jenkins创建新工程](/image/jenkins/jenkins_create_new_project.png)
#### 配置工程
在新的界面可以配置项目的构建、源码管理等
我的工程是存放在git上面的，所以就选择git
![jenkins源码管理](/image/jenkins/jenkins_config_project.png)
构建触发器可以配置在什么时间配置，我们暂时没有，需要手动点击构建才行
构建环境没有配置
构建一项选择 `Invoke top-level Maven target`,当然根据需求可以选择其他构建方式
`Maven Version`是上面配置maven时选择的别名，`Goals`是需要执行的maven命令(前面不需要加maven)
![jenkins构建项目](/image/jenkins/jenkins_project_build.png)
点击左下角保存
#### 开始构建
点击左侧的立即构建，可以在构建历史中查看原来构建的个过程
![jenkins构建](/image/jenkins/jenkins_start_build_project.png)
点击构建历史列表里面对应构建历史的小圆点，可以查看控制台输出
![jenkins构建历史](/image/jenkins/jenkins_build_history.png)
![jenkins](/image/jenkins/jenkins_build_console_output.png)
ps：如果是普通用户启动的tomcat，使用git管理源码，则下载下来的工程源码在`/home/username/.jenkins/workspace`，如果是root用户，则在 `/root/.jenkins/workspace`下
pps:据说`jenkins`的包不需要tomcat也可以，执行`java -jar ***.war`即可