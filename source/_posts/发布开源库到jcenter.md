---
title: 发布开源库到jcenter
tags: [jcenter]
photos:
  - /image/jcenter/jcenter.png
date: 2019-11-04 22:24:26
keywords: [jcenter,开源库到jcenter,jitpack]
---

最近学习了一下Annotation和APT，简单的写了个库，想要发布到公共仓库供大家使用(虽然没人用，但就是想尝试一下)，最简单的是通过<https://jitpack.io/>直接从github上抓取release代码打包，并且目前已经支持<https://gitee.com/>。但是，发布简单的纯java库或者Android Application库都比较简单，在github仓库中打个tag或者发布一下release，在jitpack上抓取一下就好，教程在这里<https://jitpack.io/docs/#publishing-on-jitpack>，对于有多个依赖的Android Lib,抱歉，我是真的没搞定，于是转战[https://bintray.com](https://bintray.com/)

<!--more-->

####  准备

1. 首先注册个账号，打开官网首页，点击白色的`Sign Up Here`而不是那个大绿色的按钮(START YOUR FREE TRIAL)
2. 填完信息后到邮箱激活一下账号、登录。
3. 创建一个仓库，仓库名随意，Type选择Maven，Licenses和Description选填。
4. 复制自己的api key。
![api key](/image/jcenter/get_jfrog_api_key.png)
在首页点击"edit"，在新页面左侧`API Key`,输入密码，复制一下

#### 上传到bintray

1. 在AndroidStudio工程和module中配置
	在工程的build.gradle中添加`classpath 'com.novoda:bintray-release:0.9.1'`
	在要上传的lib module中添加

	``` groovy
	apply plugin: 'java-library'
	apply plugin: 'com.novoda.bintray-release' //添加
	dependencies {
	    implementation fileTree(dir: 'libs', include: ['*.jar'])
	}
	...
	...
	
	//添加
	//其他人引用的格式为 groupId:artifactId:publishVersion
	publish {
	    userOrg = 'huangyuanlove' //JFrogBintray的用户名
	    repoName = 'AndroidAnnotation' //上面创建的仓库名
	    groupId = 'com.huangyuanlove' 
	    artifactId = 'view-inject-annotation'
	    publishVersion = '0.0.2'
	    desc = 'Make flow layouts simpler'
	    website = 'https://github.com/huangyuanlove/AndroidAnnotation'
	}
	```
	
	
	
2. 执行`./gradlew clean build bintrayUpload -PbintrayUser=username -PbintrayKey=apiKey -PdryRun=false`

3. 等待执行成功，提示successful

#### 发布到jcenter

![packages in repository](/image/jcenter/package_in_repository.png)

1. 在自己的bintray仓库中找到这个包，进入详情页

   ![packages in repository](/image/jcenter/add_to_jcenter.png)

2. 点击右上角`Actions`菜单，选择`Add to Jcenter`，在弹出框点击`send`

3. 审核结果会以站内信的形式通知你，不通过的话会告诉你原因。自己刚开始发布的版本含有`preview`，审核不通过，建议经过详细测试之后再提交

----
以上