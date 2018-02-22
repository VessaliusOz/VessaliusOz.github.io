---
title: 从请求到响应django都做了哪些处理
date: 2017-12-10 09:47:51
tags: 
  - Django
---
最近学长在面试今日头条的时候被问到了一个问题，就是`request.user`里面的user是怎样得到的，这个问题他当时也没有回答上来，他问我的时候我也不知道，可以说是非常的尴尬，所以赶快查了一些资料，看了一些源码，特地来总结一下这个问题。

感觉要想回答为什么可以直接通过`request.user`得到请求的用户，应该先来看看请求被处理以及如何返回响应的流程。今天先总结一下`django`从请求到响应都进行了哪些过程。
<!--more-->

## django的中间件以及请求响应循环
以下将进入正题，参考资料在最后会列出来。

### WSGI
当客户端发送一次请求后，最先处理请求的实际上是web服务器就是我们经常说的`nginx`、`Apache`这类的web服务器，而`WSGI`的作用就是把web服务器和web框架(Django)连接起来。`WSGI`被分为了两个部分:服务端和应用端。为了处理一个`WSGI`的响应，服务端执行应用程序并向应用端提供一个回调函数，应用端处理请求并使用提供的回调将响应返回给服务端。本质上来讲，我觉得`WSGI`就是web服务器和django应用之间的一个联系人。

### 数据流
当用户向你的应用发送一个请求的时候，一个`WSGI handler`将会被初始化，它会完成以下工作:
1. 导入`settings.py`和`django`的异常类
2. 使用`load_middleware`方法加载`settings.py`中`MIDDLEWARE_CLASSES`或者`MIDDLEWARES`元组中所用的`middleware classes`.
3. 创建四个列表(`_request_middleware`,`_view_middleware`, `_response_middleware`, `_exception_middleware`)，里面分别包含处理`request`,`view`,`response`和`exception`的方法。
4. `WSGI Handler`将实例化一个`django.http.HTTPRequest`对象的子类，`django.core.handlers.wsgi.WSGIRequest`.
5. 循环遍历处理`request`的方法(`_request_middleware`列表)，并按照顺序调用他们
6. 解析请求的`url`
7. 循环遍历每个处理`view`的方法(`_view_middleware`列表)
8. 如果找的到的话，就调用视图函数
9. 处理任何异常的方法(`_exception_middleware`列表)
10. 循环遍历每个处理响应的方法(`_response_middleware`列表)，(从内向外，与请求中间件的顺序相反)
11. 最后得到一个响应，并调用`web server`提供的回调函数

### 中间件
中间件被用在了django的许多关键功能中:例如，使用CSRF中间键来防止跨站请求伪造攻击。它们也被用来处理会话数据，身份认证和授权同样是由中间件来完成的。我们也可以自己编写中间件来调整或者(短路)通过应用程序的数据流。

### process_request
django的中间件至少含有以下四个方法中的一个:`process_request`, `process_response`, `process_view`, `process_exception`。这些方法会被`WSGI handler`收集并按照顺序调用。我们可以先来看看`django.contrib.auth.middleware.AuthenticationMiddleware`:
```python
def get_user(request):
    if not hasattr(request, '_cached_user'):
        request._cached_user = auth.get_user(request)
    return request._cached_user


class AuthenticationMiddleware(MiddlewareMixin):
    def process_request(self, request):
        assert hasattr(request, 'session'), (
              "The Django authentication middleware requires session middleware "
              "to be installed. Edit your MIDDLEWARE%s setting to insert "
              "'django.contrib.sessions.middleware.SessionMiddleware' before "
              "'django.contrib.auth.middleware.AuthenticationMiddleware'."
        ) % ("_CLASSES" if settings.MIDDLEWARE is None else "")
        request.user = SimpleLazyObject(lambda: get_user(request))
```
这里我们可以发现`request.user`这个属性是在`AuthenticationMiddleware`中产生的。这个我们稍后再说。

这里我们可以发现，这个中间件只有`process_request`，说明它只在`request`这一步处理流入和流出django应用的数据流。这个中间件会首先验证会话中间件是否被使用，然后通过调用`get_user`函数来设置用户。当`WSGI`处理程序迭代`process_request`方法列表的时候，它将会构建这个最终会被传递给视图函数的请求对象，并能够使你引用`request.user`。一些中间件没有`process_request`方法，在这个阶段，会被跳过。

`process_request`应该返回`None`或者`HTTPResponse`对象。当返回`None`时，`WSGI handler`会继续加载`process_request`里面的方法，但是后一种情况会`短路`处理过程并进入`process_response`循环。

### 解析url
当所有的`process_request`被调用完之后，我们就会得到一个将被传递给视图函数的`request`对象。当这个事件发生之前，`django`必须解析`url`并决定调用哪一个视图函数。这个过程非常简单，只需要使用正则匹配即可。`settings.py`中有一个`ROOT_URLCONF`键来指定根`url.py`，在这里会包含你所有app的`urls.py`文件。如果没有匹配成功，将会抛出一个异常` django.core.urlresolvers.Resolver404`,这是`django.http.HTTP404`的子类。

### process_view
到这一步之后`WSGI handler`知道了调用哪一个视图函数，以及传递哪些参数。它会再一次调用中间件列表里面的方法,这次是`_view_middleware`列表。所有`Django`中间件的`process_view`方法将会被这样声明:

`process_view(request, view_function, view_args, view_kwargs)`

和`process_request`一样，`process_view`函数必须返回`None`或者`HTTPResponse`对象，使得`WSGI handler`继续处理视图或者'短路'处理流程并返回一个响应。在`CSRF middleware`中存在一个`process_view`的方法。作用是当`CSRF cookies`出现时，`process_view`方法将会返回`None`,视图函数将会继续的执行。如果不是这样，请求将会被拒绝，处理流程将会被'短路'，会生成一个错误的信息。

### 进入视图函数
一个视图函数需要满足三个条件:
1. 必须是可以调用的。这可以是基于函数的视图或者是`class-based`的视图(继承自`View`并且使用`as_view()`方法来使它成为可调用的。这些方法的调用依赖`HTTP verb(GET, POST, etc)`)

2. 必须接受一个`HTTPRequest`对象作为第一个位置参数。这个`HTTPRequest`对象是被所有`process_request`和`process_view`中间件方法处理的结果。

3. 必须返回一个`HTTPResponse`对象，或者抛出一个异常。就是用这个`response`对象来开启`WSGI handler`的`process_view`循环。

### process_exception
如果视图函数抛出一个异常，`Handler`将会循环遍历`_exception_middleware`列表，这些方法按照相反的顺序执行，从`settings.py`里面列出来的最后一个中间件到第一个。如果一个异常被抛出，处理过程将会被短路，其他的`process_exception`将不会被执行。通常我们依赖`Djnago's BaseHandler`提供的异常处理程序，但是我们也可以使用自定义的异常处理中间件。

### process_response
在这个阶段，我们得到了一个`HTTPResponse`对象，这个对象可能是`process_view`返回的，也可能是视图函数返回的。现在我们将循环访问响应中间件。这是中间件调整数据的最后的机会。执行的顺序是从内向外执行。
以`cache middleware`的`process_response`为例：它依赖于你的`app`里面的不同的状态(缓存是否打开或者关闭，是否在处理一个数据流)，来决定是否缓存你的响应。

#### 注意
`django 1.10`和之前版本的区别：
在旧版本的`MIDDLEWARE_CLASSES`中，就算一个中间件"短路"了执行过程，所有的中间件都会调用它们的`process_response`方法。而在新的`MIDDLEWARES`版本中，只有这个中间件和在它之前执行的中间件才会调用`process_response`方法。

### 总结
以上就是`django`在处理一个请求的基本的过程，最后`django`的`WSGI Handler`会创建一个来自`HTTPResponse`的返回值，而且会调用回调函数把数据传递给`web server`,最后返回给用户。

以下是两个关键点：
1. 我们现在知道了视图函数是如何和url解析器匹配以及什么在调用它(`WSGI Handler`)
2. 有四个关键的地方可以让你挂钩到请求/响应周期：`process_request`, `process_response`, `process_view`, `process_exception`。请求中间件是从外部向内执行，最后抵达到视图函数，然后通过响应中间件从内向外返回。

### 参考资料

[https://medium.com/zeitcode/django-middlewares-and-the-request-response-cycle-fcbf8efb903f](https://medium.com/zeitcode/django-middlewares-and-the-request-response-cycle-fcbf8efb903f)

[http://www.b-list.org/weblog/2006/jun/13/how-django-processes-request/](http://www.b-list.org/weblog/2006/jun/13/how-django-processes-request/)
