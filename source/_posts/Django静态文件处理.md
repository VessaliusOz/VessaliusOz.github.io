---
title: Django静态文件处理
date: 2017-07-27 12:35:58
tags: 
  - Django
---
今天是关于django静态文件处理的一些总结。早些时候，因为没有区分STATIC_URL、STATIC_ROOT、STATICFILES_DIR等相关的配置，走了一些不必要的弯路，耽误了开发的时间，所以今天决定好好总结一下，备忘。

我们把JS, CSS, images等文件称为静态文件，Django提供了django.contrib,staticfiles来帮助我们处理他们。
<!--more-->
### 开发环境下(DEBUG=True)
我们首先在INSTALLED_APPS里注册django.contrib.staticfiles，这是在你创建文件一开始就默认帮你注册好的，我们还可以在settings.py中发现多了一个STATIC_URL='/static/'，这个时候，我们只要在自己的app下建立文件夹，目录结构如下<font color=#00ffff >my_app/static/my_app/example.jpg</font>

其实，只要设置了这几步，运行python manage.py runserver 我们访问http://127.0.0.1:8000/static/my_app/example.jpg ,就可以访问到我们图片，urls此时还不需要任何的配置。这与我之前看到的许多博客上的描述不符。

我们可以在官方文档上找到这样的描述：
During development, if you use django.contrib.staticfiles, this will be done automatically by runserver when DEBUG is set to True (see django.contrib.staticfiles.views.serve()). 就是说
在开发环境中，如果使用了django.contrib.staticfiles,在DEBUG=True的时候，以上步骤将会由runserver自动完成，但这种方法低效和不安全，只建议在生产环境中使用。

#### 静态文件命名空间
我们可以将静态文件直接放在my_app / static /（而不是创建另一个my_app子目录）中，但这实际上是一个坏主意。 Django将使用它找到的名称匹配的第一个静态文件，如果你在不同的应用程序中有一个具有相同名称的静态文件，Django将无法区分它们。 我们需要能够将Django指向正确的方法，最简单的方法是通过命名来确定这一点。 也就是说，将这些静态文件放在为应用程序本身命名的另一个目录中。

#### STATICFILES_DIR
由以上我们可以发现，只要在相应的app下有static目录，我们就可以访问到静态文件，但是实际开发过程中，我们有一些静态文件并不是只属于一个特定的app，可能有许多app共用一些静态文件，这时，在所有的app下都保存一遍这些静态文件会有很大的空间开销。这时，我们就可以用STATICFILES_DIRS，这是一个包含其他文件目录的完整路径的列表，
例子如下：
```python
STATICFILES_DIRS = [
    "/home/special.polls.com/polls/static",
    "/home/polls.com/polls/static",
    "/opt/webfiles/common",
]
```
如果您想使用附加命名空间引用其中一个位置的文件，则可以选择提供前缀（前缀，路径）元组，例如：
```python
STATICFILES_DIRS = [
    # ...
    ("downloads", "/opt/webfiles/stats"),
]
```
这时，假设您将STATIC_URL设置为“/ static /”，则collectstatic管理命令将在STATIC_ROOT的“downloads”子目录中收集“stats”文件。
这将允许您在模板中使用“/static/downloads/polls_20101022.tar.gz”路径引用本地文件“/opt/webfiles/stats/polls_20101022.tar.gz”。

同样，只要设置了STATICFILES_DIRS, 例如我们这样设置：
```python
STATICFILES_DIRS =(
    os.path.join(BASE_DIR, 'static'),
)
```
那么这个文件夹下的文件也可以通过访问相应的路径得到，比如根目录下的static文件夹(不是各个app目录下的static)中，'/static/images/abgail.jpg'，我们可以通过　http://127.0.0.1:8000/static/images/abgail.jpg　访问到。此时urls中也无需任何配置。

#### STATICFILES_FINDERS
看到以上我们就有疑惑了，在这么多的app中的static子文件夹下，以及STATICFILES_DIRS中的文件目录下，django都可以查找到静态文件，那如果有重名的怎么办？查找顺序是什么呢？这时，我们就可以看看STATICFILES_FINDERS这个参数。

STATICFILES_FINDERS的默认值为：
```python
[
    'django.contrib.staticfiles.finders.FileSystemFinder',
    'django.contrib.staticfiles.finders.AppDirectoriesFinder',
]
```
这是一个包含了知道如何找到各处静态文件的查找后端的列表。
默认情况下，将找到存储在STATICFILES_DIRS(使用django.contrib.staticfiles.finders.FileSystemFinder)和每个app中'static'子目录下(使用django.contrib.staticfiles.finders.AppDirectoriesFinder)的静态文件。当有同名文件出现时，将会使用找到的第一个文件。所以查找顺序是先查找STATICFILES_DIRS中的文件，再查找app中static文件夹下的文件。

从以上我们可以发现，在开发模式下，我们是可以不使用STATIC_ROOT这个参数的。

#### STATIC_ROOT
##### 关于是否需要urls的配置
官方文档上是这样说的

"If you use django.contrib.staticfiles as explained above, runserver will do this automatically when DEBUG is set to True. If you don’t have django.contrib.staticfiles in INSTALLED_APPS, you can still manually serve static files using the django.views.static.serve() view."

意思是说，如果你用了django.contrib.staticfiles，在DEBUG=True时，runserver会自动帮你完成静态文件的路由、查找等工作。当你没有在INSTALLED_APPS中注册django.contrib.staticfiles时,您仍然可以使用django.views.static.serve（）视图手动提供静态文件。

当你注销django.contrib.staticfiles时，你再直接访问例如　http://127.0.0.1:8000/static/images/abgail.png 时，会发现报错:The current URL, static/images/beauty.png, didn't match any of these.就是没有匹配到这个url.

所以我们如果这样做了，我们还需要在urls.py配置相应的静态文件路由，例如：
```python
from django.conf import settings
from django.conf.urls.static import static

urlpatterns = [
    # ... the rest of your URLconf goes here ...
] + static(settings.STATIC_URL, document_root=settings.STATIC_ROOT)
```

注意

此帮助函数仅在调试模式下工作，并且仅在给定前缀是本地（例如/ static /）而不是URL（例如http://static.example.com/）时）。

此帮助函数仅用于实际的STATIC_ROOT文件夹; 它不像django.contrib.staticfiles那样有执行静态文件发现的过程。

如果使用这种方式来配置静态文件的话，我们就会使用STATIC_ROOT.

#### STATIC_ROOT
在实际的部署过程中，我们会把所有的静态文件都放在一个文件目录下，从而方便一些web服务诸如nginx、Apache来进行分发。django提供了一个命令collectstatic来方便我们把所有的静态文件都收集到STATIC_ROOT对应的目录中。

STATIC_ROOT的值是一个绝对路径指向的目录，它用来存储的是用collectstatic命令收集到的为部署做准备的所有静态文件。前提是只有在INSTALLED_APPS中注册了staticfiles，才能使用collectstatic命令。

需要注意的是STATIC_ROOT不能被包含在STATICFILES_DIRS中，STATIC_ROOT一个最好是一个空目录，这只是为了部署的方便而临时使用的一个用来存储静态文件的文件夹。

#### STATIC_URL
这是用来设置引用位于STATIC_ROOT中的静态文件时使用的URL。
例子:"/static/"和"http://static.example.com/" 如果不为None时，它必须以反斜线终止。
实例如下，我设置STATIC_URL="/static/", 那么访问时静态文件url应该是 http://127.0.0.1:8000/static/images/abgail.png 而当我设置为STATIC_URL="/static_files/"时，那么我的访问路径应该是　http://127.0.0.1:8000/static_files/images/abgail.png


### 生产环境下(DEBUG=False)
在生产环境下部署静态文件也很简单。当我们在运行python manage.py collectstatic 之后，我们只要在nginx中作出相应的配置就行了，将请求指向STATIC_ROOT所指向的文件夹,media文件就指向media所在的目录：
```

      location /static/ {
	alias /path_to_static/collect_static/;

      location /media/ {
  alias /path_to_static/media/;
}


```
我在看到有的博客中说明，需要在urls.py 中加入以下配置信息
```python
url(r'^static/(?P<path>.*)$', 'django.views.static.serve', {'document_root': settings.STATIC_ROOT}),
```
这是在DEBUG=False时，我们运行"python manage.py runserver　0.0.0.0:8000"，如果没有加入上面这一条语句，是访问不到静态文件的。我个人的理解是，当为DEBUG=False时，django认为你是在生产环境下，静态文件的分发应该由其他的web服务来处理。所以django.contrib.staticfiles的自动查找的作用失效了，django不在处理静态文件。所以你必须手动的配置静态文件的访问路径。

而在真正的生产环境中，如果你的nginx是像以上那样配置的，不管你的urls.py里面有没有如下语句
```python
url(r'^static/(?P<path>.*)$', 'django.views.static.serve', {'document_root': settings.STATIC_ROOT}),
```
它都是可以访问到的，因为'/static/'这个路由是由nginx来处理的，与django是没有关系的。



然而，如果你的nginx是这样配置的话
```
	location ~* \.*(gif|png|jpg|jpeg|js|ttf|less)$ {
		root /path_to_app/;
		include /usr/local/nginx/conf/mime.types;
	}
```
直接访问就不行，静态文件是加载不出来的。。。(这真是一个奇怪的问题，我之前一直是这样配置的)必须得加入
```python
# urls.py

url(r'^static/(?P<path>.*)$', 'django.views.static.serve', {'document_root': settings.STATIC_ROOT}),
url(r'^media/(?P<path>.*)$', 'django.views.static.serve',  {"document_root": settings.MEDIA_ROOT}),
```
才能访问的到。。。难道nginx里面我像这样写的话实际上还是由django来处理的嘛？我觉得可能和访问的优先级有关系。这个问题我有空再看看，其实，最后还是建议大家nginx使用第一种配置，同时也加上如下语句比较好，
```python
url(r'^static/(?P<path>.*)$', 'django.views.static.serve', {'document_root': settings.STATIC_ROOT}),
url(r'^media/(?P<path>.*)$', 'django.views.static.serve',  {"document_root": settings.MEDIA_ROOT}),
```
这样其实是便于调试的。

而且以上语句在django1.11中不是那样写的，应该如下：
```python
from django.views import static
    url(r'^static/(?P<path>.*)$', static.serve, {"document_root": settings.STATIC_ROOT}, name='static'),
    url(r'^media/(?P<path>.*)$', static.serve, {"document_root": settings.MEDIA_ROOT}, name='media'),

```

MEDIA_ROOT和MEDIA_URL的关系和STATIC_URL与STATIC_ROOT的关系差不多，就不在赘述了。
