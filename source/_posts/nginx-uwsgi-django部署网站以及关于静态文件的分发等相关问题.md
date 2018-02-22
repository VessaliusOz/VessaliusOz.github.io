---
title: 一些关于nginx配置的问题
date: 2017-05-14 23:23:54
tags: 
  - nginx
---
在大致做完自己的第一个完整项目之后，拖了快两个月才开始总结，当时踩的坑有好多，现在基本都忘了，赶快趁着对一些坑还有印象的时候赶紧总结一下备忘。

以下是自己项目的nginx的配置文件，有一些是值得说明的地方：
<!--more-->
```
user  root;
worker_processes  1;

error_log  logs/error.log;
#error_log  logs/error.log  notice;
#error_log  logs/error.log  info;

pid        logs/nginx.pid;

events {
worker_connections  1024;
multi_accept on;
}

http {
include       mime.types;
default_type  application/octet-stream;
charset utf-8;
log_format  main  '$remote_addr - $remote_user [$time_local] "$request" '
                  '$status $body_bytes_sent "$http_referer" '
                  '"$http_user_agent" "$http_x_forwarded_for"';

#access_log  logs/access.log  main;

sendfile        on;
# tcp_nopush     on;

#keepalive_timeout  0;
keepalive_timeout  65;

client_max_body_size 10m;

gzip  on;
gzip_vary on;
gzip_comp_level 6;
gzip_buffers 16 8k;
gzip_min_length 0;
gzip_types application/json text/plain application/javascript application/x-javascript text/css application/xml text/javascript image/jpeg image/gif image/png ;
server {
    listen       80;
    server_name  xxxxxx;

    #charset koi8-r;

    access_log  logs/host.access.log  main;

root /home/tuanwei/xxshproject/xxsh/;
   # index index.html index.htm;


location ^~ /api/ {
  include /usr/local/nginx/conf/uwsgi_params;
  uwsgi_pass 127.0.0.1:8888;
  include /usr/local/nginx/conf/mime.types;

}


location ^~ /xadmin/ {
 include /usr/local/nginx/conf/uwsgi_params;
       uwsgi_pass 127.0.0.1:8888;
       include /usr/local/nginx/conf/mime.types;
}

location ^~ /upload_media/ {
include /usr/local/nginx/conf/uwsgi_params;
    uwsgi_pass 127.0.0.1:8888;
    include /usr/local/nginx/conf/uwsgi_params;

}

location / {
        try_files $uri /dist/index.html;
    }

location ~* \.*(gif|png|jpg|jpeg|css|js|ttf|less)$ {
    root /home/tuanwei/xxshproject/xxsh;
    include /usr/local/nginx/conf/mime.types;
}



}

```
### Gzip
笔者当时以为只要设置了`gzip on;`这个选项，静态文件就可以被压缩，后来发现自己还是太navie.只是设置`gzip on;`时，nginx服务器只默认对mime-type为text/html类型的页面数据进行压缩。当时一个六百多k的js文件加载了三十多秒，等的真是绝望。以下是笔者整理的关于gzip的设置:
##### client_max_body_size

其中`client_max_body_size`用来设置上传文件最大尺寸，默认情况下使用nginx反向代理上传大于2M的文件时，会报错413 Request Entity Too Large,想要更改时，只需设置
`client_max_body_size 10m;`
将上传文件的最大限度改成自己想要的大小，笔者设置的是10M.



###### gzip_comp_level
该指令用来设置Gzip的压缩程度，包括级别1~9，级别１表示压缩程度最低，而相应的压缩效率最高，级别9表示压缩程度最高，压缩效率最低，最费时间。其语法结构为：
`gzip_comp_level level;`
###### gzip_buffers
该指令用于设置Gzip压缩文件使用缓存空间的大小，语法结构为:
`gzip_buffers number size;`
其中number用来指定Nginx服务器需要向系统申请缓存空间的个数，size指定每个缓存空间的大小。默认情况下number*size的值为128，其中size的值取系统内存页一页的大小，为4kB或者8kB，如：
`gzip_buffers 32 4k | 16 8k;`
###### gzip_min_length
当需要根据响应页面的大小，选择性的开启Gzip的功能时，该指令设置页面的字节数，当响应页面的大小大于该值时，就启用Gzip的功能。响应页面的大小通过HTTP响应头部中的Content-Length指令来获取，但是如果使用了chunk编码动态压缩，或者不含有Content-Length时，该指令则不起作用。
建议设在1kB以上：
gzip_min_length 1024;
###### gzip_types
顾名思义，这个指令用来设置需要压缩的文件类型,若不指定，则只是对mime-type为text/html的页面数据进行压缩。其语法结构如下：
`gzip_types mime-type ...;`

例如笔者设置的
`gzip_types application/json text/plain application/javascript application/x-javascript text/css application/xml text/javascript image/jpeg image/gif image/png ;`

你也可以使用通配符
`gzip_types *;`
来选择对所有的MIME类型的页面数据进行压缩。
###### gzip_vary
该指令用于设置在使用Gzip功能时是否发送带有"Vary: Accept-Encoding:gzip"的响应首部。　on 为设置，off为不设置。

---
因为这个项目是前后端分离的，前端是学长用React写的，现在要部署在同一台服务器上面，后台只是用来提供相应的API，但是前端在处理时有自己的URI。而nginx服务器是根据URI来匹配相应的请求，所以要区分前端的URI和后台的URI.
这就要用到locations的进行相应的匹配。

### locations　优先级
[参考原文](http://www.cnblogs.com/lidabo/p/4169396.html)

语法　locations = [=|~|~*|^~|@] /uri/ {...}

语法的意思是可以使用"=","~","~*","^~","@"符号作为前缀，或者也可以没有前缀。紧接者的是/uri/,再接着是{...}指令块。

上述各类location可以分为两大类，分别是"普通的locations"和"正则location"，其中"普通location"是以"="或者"^~"为前缀或者没有任何前缀的/uri/;"正则uri"是以"~"或者"~*"为前缀的/uri/。

nginx的匹配规则是"正则location"让步"普通location"的严格精确匹配结果;但是覆盖"普通location"的最大前缀匹配结果。nginx 其实是“先匹配普通 location ，再匹配正则 location ”，但是普通 location 的匹配结果又分两种：一种是“严格精确匹配”，官方英文说法是“ exact match ”；另一种是“最大前缀匹配”，来看看如下配置：
```
server {

       listen       8000;

       server_name  localhost;

        location / {

           root   html;

           index  index.html index.htm;

       }

       location ~ \.html$ {

           allow all;

       }

}
```
当我们访问`http://localhost:8000/`时，匹配的是/，因为这是精确匹配，当访问`http://localhost:8000/a.html`时，它其实先匹配的是"\"，但这个是最大前缀匹配，所以会继续搜索正则location，然后匹配到了"\.html"
当访问`http://localhost:8000/a.txt时`，它其实先匹配的是"\"，但这个是最大前缀匹配，所以会继续搜索正则location，但是没有匹配到任何结果，所以最后还是由"/"匹配到了。

这里我们小结下“普通 location”与“正则 location ”的匹配规则：先匹配普通 location ，再匹配正则 location ，但是如果普通 location 的匹配结果恰好是“严格精确（ exact match ）”的，则 nginx 不再尝试后面的正则 location ；如果普通 location 的匹配结果是“最大前缀”，则正则 location 的匹配覆盖普通 location 的匹配。也就是前面说的“正则 location 让步普通location 的严格精确匹配结果，但覆盖普通 location 的最大前缀匹配结果”。


其实普通 location 也可以加前缀：“ ^~ ”和“ = ”。其中“ ^~”的意思是“非正则，不需要继续正则匹配”，也就是通常我们的普通 location ，还会继续搜索正则 location （恰好严格精确匹配除外），但是 nginx 很人性化允许配置人员告诉 nginx 某条普通 location ，无论最大前缀匹配，还是严格精确匹配都终止继续搜索正则 location ；而“ = ”则表达的是普通 location 不允许“最大前缀”匹配结果，必须严格等于，严格精确匹配。

也就是说，对于普通 location 指令，匹配规则是：最大前缀匹配（与顺序无关），如果恰好是严格精确匹配结果或者加有前缀“ ^~ ”或“ = ”（符号“ = ”只能严格匹配，不能前缀匹配），则停止搜索正则 location ；但对于正则 location 的匹配规则是：按编辑顺序逐个匹配（与顺序有关），只要匹配上，就立即停止后面的搜索。

总的来说：
"="执行的是精确匹配，也就是完全匹配，优先级最高，一旦匹配成功
"~^"执行的是非正则表达式的匹配，也就是说无论最大前缀匹配，还是严格精确匹配都终止继续搜索正则 location ，优先级第二
"~/~*"等正则表达式的匹配的优先级次之，如果有多个location的正则能匹配时，则匹配第一个匹配到的结果
"/uri/"的优先级最低

所以，关于我的项目里面的locations配置应该是这样的：
```
location ^~ /api/ {
  include /usr/local/nginx/conf/uwsgi_params;
  uwsgi_pass 127.0.0.1:8888;
  include /usr/local/nginx/conf/mime.types;

}


location ^~ /xadmin/ {
 include /usr/local/nginx/conf/uwsgi_params;
       uwsgi_pass 127.0.0.1:8888;
       include /usr/local/nginx/conf/mime.types;
}

location ^~ /upload_media/ {
include /usr/local/nginx/conf/uwsgi_params;
    uwsgi_pass 127.0.0.1:8888;
    include /usr/local/nginx/conf/uwsgi_params;

}

location / {
        try_files $uri /dist/index.html;
    }

location ~* \.*(gif|png|jpg|jpeg|css|js|ttf|less)$ {
    root /home/tuanwei/xxshproject/xxsh;
    include /usr/local/nginx/conf/mime.types;
}

```
当需要走API请求，管理页面或者上传文件时会先匹配'/',之后在分别匹配/api/,/xadmin/和/upload_media/，最后由uwsgi来处理.


其他的前端的自己URL会全部有"/"来匹配。而对静态文件的处理也由nginx来分发。

对了，在设置root之后，这个root文件夹就相当与linux下的根目录，所以在locations时里面配置时，"/uri/"一定要在前面加"/"，不然匹配不到。

### try_files
语法　`try_files file ... uri` 或者　`try_files ...=code`

默认值：无

作用域： server location

其作用是按照顺序检查文件是否存在，返回第一个找到的文件或者文件夹(结尾加斜线表示文件夹)，如果所有的文件或者文件夹都找不到，会调用一个内部重定向到最后一个参数。只有最后一个参数可以引发重定向，之前的参数只是设置内部url的指向。最后的一个参数是回调的URI，而且必须存在。否则会触发一个内部错误。命名的locations可以使用在最后一个参数中。不像rewrite,如果回调URI不是一个命名的location,那么$args不会自动保留，如果想保留$args，必须明确声明。
