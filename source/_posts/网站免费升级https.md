---
title: 网站免费升级https
date: 2018-04-01 12:36:16
tags: [网站https,乱七八糟]
---
昨天跟着酷壳网左耳朵耗子的文章把自己在亚马逊主机上的网站变成https的安全访问了，证书不是自签名的，也不是花钱购买的。据说https的网站在搜索引擎中的rank值会比http的更高一些。升级完成后的浏览器截图如下：
 ![浏览器](/image/https/https1.png])
下面是这次升级的记录。
<!--more-->
为网站开启https安装证书非常简单，我用的是 [Let’s Encrypt ](https://letsencrypt.org/)这个免费的解决方案。
1. 打开[https://certbot.eff.org/](https://certbot.eff.org/)这个网页
2. 在Software 和 System选项里面选择你所使用的软件、系统，我用的nginx+ubuntu16.04
 ![网站截图](/image/https/https4.png])
3. 然后会跳转到一个新的网页，照着做就是了。
就以nginx+ubuntu16.04为例：
``` shell
$ sudo apt-get update
$ sudo apt-get install software-properties-common
$ sudo add-apt-repository ppa:certbot/certbot
$ sudo apt-get update
$ sudo apt-get install python-certbot-nginx 
```
安装成功后执行 `$ sudo certbot --nginx`
让你输入你的邮箱，然后是同意用户协议，然后是是否公开你的邮箱。
 ![邮箱、用户协议](/image/https/https2.png])
接着会列出来nginx下所有配置的服务名称，输入你想要开启https的服务名称所对应的编号，如果想为多个服务开启https，中间以空格分隔。然后nginx重新加载一个配置或者重启一下。
![开启https的服务名称](/image/https/https3.png])
我个人服务器上的nginx配置的`server_name`是`tomcat.huangyuanlove.com`,域名是在万网买的，然后在万网控制台添加一个A解析，把`tomcat.huangyuanlove.com`指向服务器的ip即可。
但是 **Let’s Encrypt 的证书90天就过期了**。所以还需要加上自动更新，使用`crontab -e`命令假如如下的定时作业(每个月强制更新一下)
``` sehll
0 0 1 * * /usr/bin/certbot renew --force-renewal
5 0 1 * * /usr/sbin/service nginx restart
```
需要注意的是，如果网站中有使用http的地方都要改成https,要不然一些资源文件如图片、js、css等非https的请求连接都会被ban掉。

----

关于ubuntu安装和配置nginx，我也是在官网找的教程，网址在这里[https://www.nginx.com/resources/wiki/start/index.html](https://www.nginx.com/resources/wiki/start/index.html),就知道你们不想看，安装命令如下：
``` shell
sudo apt-get update
sudo apt-get install nginx
```
安装完成后，nginx配置文件在`/etc/nginx/nginx.conf`,在http标签中添加server：
``` configuration
server{
        listen    80;
        server_name    localhost;
        location / {
            root    html;
            index index.html index.htm;
        }
        error_page 500 502 503 504 /50x.html;
        location = /50x.html{
            root    html;
        }
}
server{
        server_name    tomcat.huangyuanlove.com;
        location / {
            proxy_pass http://xxx.xxx.xxx/;
        }
}
```
给自己的添加https证书后，`Certbot`会修改你的nginx中的server配置，修改的内容如下：
```
 listen 443 ssl https2; # managed by Certbot
    ssl_certificate /etc/letsencrypt/live/tomcat.huangyuanlove.com/fullchain.pem; # managed by Certbot
    ssl_certificate_key /etc/letsencrypt/live/tomcat.huangyuanlove.com/privkey.pem; # managed by Certbot
    include /etc/letsencrypt/options-ssl-nginx.conf; # managed by Certbot
    ssl_dhparam /etc/letsencrypt/ssl-dhparams.pem; # managed by Certbot
```

```
server{
    if ($host = tomcat.huangyuanlove.com) {
        return 301 https://$host$request_uri;
    } # managed by Certbot
    listen    80;
    server_name    tomcat.huangyuanlove.com;
    return 404; # managed by Certbot
}
```
基本上就是这样了，没有别的了。

----

以上，嗯，对了，12306你什么时候按照这个教程做一下你的证书？

