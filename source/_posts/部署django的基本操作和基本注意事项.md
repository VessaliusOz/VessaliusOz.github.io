---
title: 部署django的基本操作和基本注意事项
date: 2018-03-14 23:05:47
tags: 
	- django
	- nginx
	- uwsgi
---

## 概述
这篇文章主要用来记录在使用`django-nginx-uwsgi`部署项目的时候遇到的一些问题，虽然没有什么技术难度，但是每次部署的时候都会要用一遍，时间长了就会忘了，又要重新去网上搜，非常浪费时间，所以就在这篇文章里把遇到的问题都集中一下，方便以后的查阅，节约时间。


<!--more-->
### UWSGI
uwsgi 主要的问题在于启动和停止，以及更新了项目文件之后的操作.`Django`的官网也给出了uwsgi的配置方法，详见👉[这里](https://docs.djangoproject.com/en/1.11/howto/deployment/wsgi/uwsgi/)

#### 配置文件
我们在项目根目录下新建一个名为`uwsgi.ini`的文件：

```
[uwsgi]
chdir=/root/weClass-env/WeClass-demo/
socket = 127.0.0.1:8888
module = WeClass.wsgi
process = 2
thread = 4
demonize =/root/weClass-env/WeClass-demo/uwsgi.ini
```

基本的部署只要这个就够了，这些配置的选项非常的易懂，在这里就不做过多的解释了。


#### 启动服务
直接在命令行运行:

```
$ nohup uwsgi --ini uwsgi.ini &
```
就好了，这里需要使用`nohup`命令，因为我们的`uwsgi`服务需要在服务器的后台运行，不然当你退出与服务器的会话窗口的时候，`uwsgi`的进程就会退出了。

#### 重启服务
直接在命令行里面运行

```
$killall -9 uwsgi
$nohup uwsgi --ini uwsgi.ini &
```
这两条命令就可以了，但是第一条的关闭命令太粗暴了，你也可以先列出所有的`uwsgi`进程，然后再通过`kill`关闭相应的某个`uwsgi`进程就好了。



### NGINX
#### 配置文件
关于`nginx`的配置稍微比`uwsgi`的要复杂一些,基本的的配置信息如下：

```
user  root;
worker_processes  4;

#error_log  logs/error.log;
#error_log  logs/error.log  notice;
#error_log  logs/error.log  info;

#pid        logs/nginx.pid;


events {
    worker_connections  1024;
    accept_mutex on;
    multi_accept on;
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

    gzip  on;  # 压缩文件
    gzip_comp_level 9;
    gzip_buffers 4 32k;
    gzip_min_length 0;
    gzip_types text/plain application/javascript application/x-javascript text/css application/xml text/javascript image/jpeg image/gif image/png;

		
	# http server
    server {
        listen       80;
        server_name  xxx;
	root /root/weClass-env/WeClass-demo/;

        #charset koi8-r;

        access_log  logs/host.access.log  main;
        error_log logs/host.error.log;

	location /  {
		include /usr/local/nginx/conf/uwsgi_params;
		uwsgi_pass 127.0.0.1:8888;
		include /usr/local/nginx/conf/mime.types;
	}

	location /static/ {
		alias /root/weClass-env/WeClass-demo/static/;
}

}

   # HTTPS server
       server {
        listen       443 ssl;
       server_name  sms.api.bz;

	root /root/weClass-env/WeClass-demo/;

	ssl on;
	#  证书所在路径
   ssl_certificate     /root/https_file/api.bz.crt;
   ssl_certificate_key  /root/https_file/api.bz_nopass.key;

	location /  {
                include /usr/local/nginx/conf/uwsgi_params;
                uwsgi_pass 127.0.0.1:8888;
                include /usr/local/nginx/conf/mime.types;
        }

	location /static/ {
                root /root/weClass-env/WeClass-demo/;
                include /usr/local/nginx/conf/mime.types;
        }
    }

}
```
这个基本是配置文件的模板。至于`location`的使用用法，可以自己去看`nginx`的官网哦🤗，非常的详细。


#### 启动命令
我们启动nginx的常用命令是,路径根据自己的实际情况来:

```
[root@localhost ~]# /usr/local/nginx/sbin/nginx -t
nginx: the configuration file /usr/local/nginx/conf/nginx.conf syntax is ok
nginx: configuration file /usr/local/nginx/conf/nginx.conf test is successful

[root@localhost ~]# /usr/local/nginx/sbin/nginx -c /usr/local/nginx/conf/nginx.conf
```

这个时候可能会遇到端口被占用的问题，比如会有以下输出:

```
Starting nginx: nginx: [emerg] bind() to 0.0.0.0:80 failed (98: Address already in use)

nginx: [emerg] bind() to 0.0.0.0:80 failed (98: Address already in use)
nginx: [emerg] bind() to 0.0.0.0:80 failed (98: Address already in use)
nginx: [emerg] bind() to 0.0.0.0:80 failed (98: Address already in use)
nginx: [emerg] bind() to 0.0.0.0:80 failed (98: Address already in use)
nginx: [emerg] still could not bind()
```

有时443端口也会变成这个样子，所以我们应该先把占用相关端口的进程关闭掉,先使用以下命令查看：

```
$lsof -i:80
$lsof -i:443
```
得到进程的PID,然后使用:

```
kill -9  <pid-number>
```

再重新启动`nginx`就好啦。

#### NGINX的SSL
有时候我们在编译`NGINX`的时候，可能会出现这样的错误

```
nginx: [emerg] the "ssl" parameter requires ngx_http_ssl_module in /usr/local/nginx/conf/nginx.conf:
```

原因是，nginx缺少http_ssl_module模块，我们应该在编译安装的时候带上`--with-http_ssl_module`配置就行了，但是现在的情况是我的nginx已经安装过了，怎么添加模块，其实也很简单，往下看： 做个说明：我的nginx的安装目录是/usr/local/nginx这个目录，我的源码包也还保留在/usr/local/src/nginx下。

1.查看nginx原有的模块

```
/usr/local/nginx/sbin/nginx -V
```
	
输出如下：
	
```
configure arguments: --prefix=/usr/local/nginx --with-http_stub_status_module --with-http_gzip_static_module
```

2.进入`nginx`的源码包下:

```
./configure --prefix=/usr/local/nginx --with-http_stub_status_module --with-http_gzip_static_module --with-http_ssl_module
```
运行上面的命令即可，等配置完
	
3.等配置完后运行`make`, 这里不要进行`make install`，否则就是覆盖安装.
	
4.备份原有已安装好的nginx
```
cp /usr/local/nginx/sbin/nginx /usr/local/nginx/sbin/nginx.bak	
```
	
5.将刚刚编译好的nginx覆盖掉原有的nginx（这个时候nginx要停止状态）
```
cp ./objs/nginx /usr/local/nginx/sbin/
```
	
6.启动nginx，仍可以通过命令查看是否已经加入成功:
```
/usr/local/nginx/sbin/nginx -V　
```

#### HTTPS和HTTP共存(补充)
HTTP和HTTPS在`nginx`是可以配在一起的，像下面这样，把`ssl on`去掉，ssl写在443端口后面。

```
server {
            listen 80 default backlog=2048;
            listen 443 ssl;
            server_name wosign.com;
            root /var/www/html;
  
            ssl_certificate /usr/local/Tengine/sslcrt/ wosign.com.crt;
            ssl_certificate_key /usr/local/Tengine/sslcrt/ wosign.com .Key;
        }
```
这样路由就只用写一遍了。


### emmmmmm
JJYY:  不能在这种重复的事情上浪费时间.