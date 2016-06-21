# test-nginx
用nssm来管理nginx
# 软件介绍
* NSSM 简介
NSSM 全称为：the Non-Sucking Service Manager。
从名称上来看，功能应该跟Forever类似，会自动帮你重启，保证你的进程不会挂掉，NSSM 所采用的方式是把你的Nodejs应用安装为一个windows service，并开机自动运行；如果需要重启应用，那么重启对应的windows服务就可以了。
* nginx 简介
反向代理（Reverse Proxy）方式是指以代理服务器来接受internet上的连接请求，然后将请求转发给内部网络上的服务器，并将从服务器上得到的结果返回给internet上请求连接的客户端，此时代理服务器对外就表现为一个服务器。 
反向代理方式实际上就是一台负责转发的代理服务器，貌似充当了真正服务器的功能，但实际上并不是，代理服务器只是充当了转发的作用，并且从真正的服务器那里取得返回的数据。这样说，其实nginx完成的就是这样的工作。我们让nginx监听一个端口，譬如80端口，但实际上我们转发给在8080端口的tomcat，由它来处理真正的请求，当请求完成后，tomcat返回，但数据此时没直接返回，而是直接给nginx，由nginx进行返回，这里，我们会以为是nginx进行了处理，但实际上进行处理的是tomcat。
 说到上面的方式，也许很多人又会想起来，这样可以把静态文件交由nginx来进行处理。对，很多用到nginx的地方都是作为静态伺服器，这样可以方便缓存那些静态文件，比如CSS，JS，html，htm等文件。

# 了解nginx配置项

**[your_nginx_path]\conf\nginx.conf**

```
server {
        listen       80;
        server_name  localhost;

        #charset koi8-r;

        #access_log  logs/host.access.log  main;

        location / {
            root   html;
            index  index.html index.htm;
        }

        #error_page  404              /404.html;

        # redirect server error pages to the static page /50x.html
        #
        error_page   500 502 503 504  /50x.html;
        location = /50x.html {
            root   html;
        }

        # proxy the PHP scripts to Apache listening on 127.0.0.1:80
        #
        #location ~ \.php$ {
        #    proxy_pass   http://127.0.0.1;
        #}
    }
```

这里面有几个需要了解的地方：
* listen：表示当前的代理服务器监听的端口，默认的是监听80端口。注意，如果我们配置了多个server，这个listen要配置不一样，不然就不能确定转到哪里去了。
* server_name：表示监听到之后需要转到哪里去，这时我们直接转到本地，这时是直接到nginx文件夹内。
* location：表示匹配的路径，这时配置了/表示所有请求都被匹配到这里
* root：里面配置了root这时表示当匹配这个请求的路径时，将会在这个文件夹内寻找相应的文件，这里对我们之后的静态文件伺服很有用。
* index：当没有指定主页时，默认会选择这个指定的文件，它可以有多个，并按顺序来加载，如果第一个不存在，则找第二个，依此类推。
* error_page是代表错误的页面。

# 配置实现

这里我们用nginx实现对tomcat的真实请求，也就是来自于所有来自外部的请求，进入nginx的８０端口进行监听，如果静态文件(js,css,image等等)，由nginx直接返回，如果是别的请求将由tomcat响应，tomcat只对nginx可见．

## 软件准备

[nssm-2.24.zip](http://nssm.cc/release/nssm-2.24.zip)
[nginx-1.11.1.zip windows版本](http://220.112.193.202/files/51980000019FAC95/nginx.org/download/nginx-1.11.1.zip) 

## 加入到环境变量

path=F:\server\nssm-2.24\win64;F:\server\nginx-1.11.1;

准备一个nginx配置文件，静态文件由nginx直接处理，动态文件由8080端口处理

**F:\server\for-tomcat-nginx.conf**

```
#user  nobody;
worker_processes  1;

events {
    worker_connections  1024;
}


http {
    include       mime.types;
    default_type  application/octet-stream;

    log_format  main  '$remote_addr - $remote_user [$time_local] "$request" '
                      '$status $body_bytes_sent "$http_referer" '
                      '"$http_user_agent" "$http_x_forwarded_for"';

    #access_log  logs/access.log  main;

    sendfile        on;
    #tcp_nopush     on;

    #keepalive_timeout  0;
    keepalive_timeout  65;

    gzip  on;
	# 转发的服务器，upstream 为负载均衡做准备
	 upstream tomcat_server{ 
			server localhost:8080; 
	 } 

    server {
        listen       80;
        server_name  localhost;
		index index.html index.htm index.php;  
		root  F:/data/wwwtest/test-nginx/target/classes/static;  

        charset UTF-8;

		# 动态请求的转发
		location ~ .*.action$ { 
			proxy_pass http://tomcat_server; 
			proxy_set_header Host $host; 
		} 
		# 静态请求直接读取
		location ~ .*\.(gif|jpg|jpeg|png|bmp|swf|css|js)$ { 
			expires      30d; 
		}
        #error_page  404              /404.html;

        # redirect server error pages to the static page /50x.html
        #
        error_page   500 502 503 504  /50x.html;
        location = /50x.html {
            root   html;
        }
    }
}
```

##配置

命令行
`nssm install`
会弹出下面的配置界面:

![Paste_Image.png](http://upload-images.jianshu.io/upload_images/2026576-cf7edd4d2681e6b3.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

##开始服务

`nssm start test-nginx`
同时在开启你的tomcat服务器


![Paste_Image.png](http://upload-images.jianshu.io/upload_images/2026576-f2d88d6b3bf3d839.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

访问/index.action

![Paste_Image.png](http://upload-images.jianshu.io/upload_images/2026576-f8c5beea2a8d8ed4.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

#本文件的demo文件

spring boot+web
**test-nginx\src\resources\templates\application.properties**
```
spring.mvc.view.suffix=action
```
**test-nginx\src\test\java\com\example\DemoApplicationTests.java**
```
package com.example;

import org.springframework.boot.SpringApplication;
import org.springframework.boot.autoconfigure.SpringBootApplication;
import org.springframework.stereotype.Controller;
import org.springframework.web.bind.annotation.RequestMapping;
import org.springframework.web.servlet.ModelAndView;

import java.util.HashMap;
import java.util.Map;

@SpringBootApplication
@Controller
public class DemoApplication {

	public static void main(String[] args) {
		SpringApplication.run(DemoApplication.class, args);
	}
	@RequestMapping("/index")
	public ModelAndView main(){
		Map map=new HashMap();
		map.put("message","你好，tomLuo!");
		return new ModelAndView("index",map);
	}
}
```
**test-nginx\src\main\resources\template\index.ftl**
```
<!DOCTYPE html>
<html>
<head>
    <meta charset="utf-8">
    <meta name="viewport" content="width=device-width, initial-scale=1.0">
    <title>test nginx</title>
</head>
<body>
<p>这是动态页，来自实际的8080端口响应．</p>
<p>${message}</p>
</body>
</html>
```
**test-nginx\src\main\resources\static\index.html**
```
<!DOCTYPE html>
<html>
<head>
    <meta charset="utf-8">
    <meta name="viewport" content="width=device-width, initial-scale=1.0">


    <title>test nginx</title>

    <!-- Bootstrap core CSS -->
    <link href="css/bootstrap.css" rel="stylesheet">


</head>

<body>

<nav class="navbar navbar-default navbar-fixed-top" role="navigation">
    <div class="container">
        <div class="navbar-header">
            <button type="button" class="navbar-toggle" data-toggle="collapse" data-target=".navbar-ex1-collapse">
                <span class="sr-only">Toggle navigation</span>
                <span class="icon-bar"></span>
                <span class="icon-bar"></span>
                <span class="icon-bar"></span>
            </button>
            <a class="navbar-brand" href="/index.action">访问/index.action页</a>
        </div>


    </div><!-- /.container -->
</nav>

<p>test nginx</p>
<img src="image/1.jpg"/>
<img src="image/2.jpg"/>

<script src="js/jquery-1.10.2.js"></script>


</body>
</html>
```
