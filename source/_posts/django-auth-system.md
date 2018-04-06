---
title: django中session机制的实现
date: 2018-03-22 22:17:37
tags:
	- django
	- 源码分析
---

## 概述
感觉django的认证系统是一个功能非常强大的东西，提供了非常便捷的功能，但是实际的运用场景下，在某些程度上还是有所欠缺的。因为根据我实习的经历，很多的项目都是需要定制后端的管理站点的，所以最近准备开始学习看`django`的源码，以备不时之需。

自己主要感兴趣的地方一个是`Django`的认证系统，还有一个就是`Django`的ORM系统，所以主要会研究这两个系统的源码(强行立一个flag👀,希望自己能坚持下去🤨)。同时在研究源码的时候，会把相对应的web development中的一些关键问题做一些总结，比如今天要说的`session`机制等等。
<!--more-->

## session和cookie

### cookie
cookie主要分为两类，会话cookie和持久cookie，其中会话cookie在用户退出浏览器的时候，就被删除了。而持久cookie的生存周期更长一些，它们会被存储在硬盘上。通常会用持久cookie维护某个用户会周期性访问的站点的配置文件和登录名。而会话cookie和持久cookie的唯一区别就在于过期时间。如果设置了Discard参数或者没有设置Expires或Max-Age参数来说明扩展的过期时间，这个cookie就是一个会话cookie。

### cookie的设置过程
cookie的设置过程:

1. 客户端第一次请求某个站点
2. 服务器返回的HTTP头部中包含`Set-cookie`或者`Set-Cookie2`返回给客户端一些特殊的键值对(这些键值对通常用来标识用户的唯一性)。这些键值对是服务器为了进行跟踪而产生的独特的识别码。(也可以通过url来传输)
3. 之后客户端的每次请求都会带上`Cookie`字段，他的值就是用来唯一标识自己身份的值，例如sessionid=xxx等等。

![流程图](/assets/blogImg/cookie2.png)

### session
session，中文经常翻译为会话，其本来的含义是指有始有终的一系列动作/消息，比如打电话是从拿起电话拨号到挂断电话这中间的一系列过程可以称之为一个session。然而当session一词与网络协议相关联时，它又往往隐含了“面向连接”和/或“保持状态”这样两个含义。

session在Web开发环境下的语义又有了新的扩展，它的含义是指一类用来在客户端与服务器端之间保持状态的解决方案。有时候Session也用来指这种解决方案的存储结构。

session机制是一种服务器端的机制，服务器使用一种类似于散列表的结构(也可能就是使用散列表)来保存信息。

但程序需要为某个客户端的请求创建一个session的时候，服务器首先检查这个客户端的请求里是否包含了一个session标识－称为session id，如果已经包含一个session id则说明以前已经为此客户创建过session，服务器就按照session id把这个session检索出来使用(如果检索不到，可能会新建一个，这种情况可能出现在服务端已经删除了该用户对应的session对象，但用户人为地在请求的URL后面附加上一个JSESSION的参数)。如果客户请求不包含session id，则为此客户创建一个session并且同时生成一个与此session相关联的session id，这个session id将在本次响应中返回给客户端保存。

session机制本身并不复杂，然而其实现和配置上的灵活性却使得具体情况复杂多变。这也要求我们不能把仅仅某一次的经验或者某一个浏览器，服务器的经验当作普遍适用的。

![session的原理图](/assets/blogImg/session.png)

### cookie和session的区别与联
session和cookie的目的相同，都是为了克服http协议无状态的缺陷,用来维护客户端和服务端之间的特异的状态。但是session通常存储在服务端，而cookie通常存储在客户端(浏览器)。session通过cookie，在客户端保存session id，而将用户的其他会话消息保存在服务端的session对象中，与此相对的，cookie需要将所有信息都保存在客户端。因此cookie存在着一定的安全隐患，例如本地cookie中保存的用户名密码被破译，或cookie被其他网站收集（例如：1. appA主动设置域B cookie，让域B cookie获取；2. XSS，在appA上通过javascript获取document.cookie，并传递给自己的appB）。


## Django中session机制的实现

### session的存储
Django的session一般存储在数据库中。存储的位置是数据表`django_session`中

```
sqlite> .schema django_session
CREATE TABLE IF NOT EXISTS "django_session" ("session_key" varchar(40) NOT NULL PRIMARY KEY, "session_data" text NOT NULL, "expire_date" datetime NOT NULL);
CREATE INDEX "django_session_expire_date_a5c62663" ON "django_session" ("expire_date");

```
`django_session`表中共有三个字段:

1. `session_key`, 这个就是上文所说的需要由服务器分发的存储到浏览器cookie中的session_id
2. `session_data`, 这个是存储的会话数据
3. `expire_date`, 过期时间


下面是`session_data`的加密方式：

```python
# django.contrib.sessions.backend

def _hash(self, value):
    key_salt = "django.contrib.sessions" + self.__class__.__name__
    return salted_hmac(key_salt, value).hexdigest()

def encode(self, session_dict):
    "Return the given session dictionary serialized and encoded as a string."
    serialized = self.serializer().dumps(session_dict)
    # 假如随机值
    hash = self._hash(serialized)
    # 拼接字符串"随机值:序列化数据",然后用base64加密
    return base64.b64encode(hash.encode() + b":" + serialized).decode('ascii')

def decode(self, session_data):
    encoded_data = base64.b64decode(force_bytes(session_data))
    try:
        # could produce ValueError if there is no ':'
        hash, serialized = encoded_data.split(b':', 1)
        expected_hash = self._hash(serialized)
        if not constant_time_compare(hash.decode(), expected_hash):
            raise SuspiciousSession("Session data corrupted")
        else:
            return self.serializer().loads(serialized)
    except Exception as e:
        # ValueError, SuspiciousOperation, unpickling exceptions. If any of
        # these happen, just return an empty dictionary (an empty session).
        if isinstance(e, SuspiciousOperation):
            logger = logging.getLogger('django.security.%s' % e.__class__.__name__)
            logger.warning(str(e))
        return {}
```

下面是数据库中的一条数据

```
session_key                       session_data                                                                                                                                                                                                                                                  expire_date               
--------------------------------  --------------------------------------------------------------------------------------------------------------------                                                                                                                                          --------------------------
qa4iva6zqogx9flxqg7fdsi1b2n9nna6  YjExNDQyY2QzN2RiNDJjYzFmNDVhMmFmODgwMzcwNWY0NzA3ZDBmNDp7Il9hdXRoX3VzZXJfaWQiOiIyIiwiX2F1dGhfdXNlcl9iYWNrZW5kIjoiZGphbmdvLmNvbnRyaWIuYXV0aC5iYWNrZW5kcy5Nb2RlbEJhY2tlbmQiLCJfYXV0aF91c2VyX2hhc2giOiIxMjdjZWRiZTUxNzQwZDE2YmExMGJiN2U0ZmI1NGIxYmMyYzQ2YjRhIn0=  2018-04-07 02:33:32.921703
Run Time: real 0.000 user 0.000173 sys 0.000166
```

我们可以解析出来`sesssion_data`里面的信息

```python
In [3]: import json, base64

In [4]: session_data = "YjExNDQyY2QzN2RiNDJjYzFmNDVhMmFmODgwMzcwNWY0NzA3ZDBmNDp7
   ...: Il9hdXRoX3VzZXJfaWQiOiIyIiwiX2F1dGhfdXNlcl9iYWNrZW5kIjoiZGphbmdvLmNvbnRy
   ...: aWIuYXV0aC5iYWNrZW5kcy5Nb2RlbEJhY2tlbmQiLCJfYXV0aF91c2VyX2hhc2giOiIxMjdj
   ...: ZWRiZTUxNzQwZDE2YmExMGJiN2U0ZmI1NGIxYmMyYzQ2YjRhIn0="

In [5]: encoded_data = base64.b64decode(session_data)

In [6]: hash, serialized = encoded_data.split(b":", 1)

In [7]: json.loads(serialized)
Out[7]:
{'_auth_user_backend': 'django.contrib.auth.backends.ModelBackend',
 '_auth_user_hash': '127cedbe51740d16ba10bb7e4fb54b1bc2c46b4a',
 '_auth_user_id': '2'}

```
这里的信息包含了`_auth_user_backend`, `_auth_user_hash`, `_auth_user_id`等信息，这些信息有什么用呢，我们待会再说。


### SessionBase
SessionBase是所有的不同类型的session class的基类，在SessionBase中实现了一个session对象所具有的大部分功能。比如关于过期时间的处理，如何生成session等功能。但是关于具体session对象的create()，save()，delete()，load()等方法需要在相应的场景中的对应的不同对象(SessionStore)自己去实现。这里session可以存储在数据库中(通常采用的方法),也可以存储在缓存中。在base文件中还定义相应的错误类`CreateError`和`UpdateError`.

```python
# in django.contrib.sessions.backend.base
class CreateError(Exception):
    """
    Used internally as a consistent exception type to catch from save (see the
    docstring for SessionBase.save() for details).
    """
    pass


class UpdateError(Exception):
    """
    Occurs if Django tries to update a session that was deleted.
    """
    pass


class SessionBase:
    ...
```


### SessionStore in django.contrib.sessions.backend.db
这个类主要用来实现将session对象存储在数据库中。他的源码其实还是很简单的，结合`SessionBase`对象提供的接口来看，基本上都是可以看懂的。

```python
class SessionStore(SessionBase):
    """
    完成数据库session的存储
    """
    def __init__(self, session_key=None):
        super().__init__(session_key)

    @classmethod
    def get_model_class(cls):
        # 避免循环引入，并允许在django.contrib.sessions不在INSTALLED_APPS中时导入SessionStore。
        from django.contrib.sessions.models import Session
        return Session

    @cached_property
    def model(self):
        return self.get_model_class()

    def load(self):
        try:
            s = self.model.objects.get(
                session_key=self.session_key,
                expire_date__gt=timezone.now()
            )
            return self.decode(s.session_data)
        except (self.model.DoesNotExist, SuspiciousOperation) as e:
            if isinstance(e, SuspiciousOperation):
                logger = logging.getLogger('django.security.%s' % e.__class__.__name__)
                logger.warning(str(e))
            self._session_key = None
            return {}

    def exists(self, session_key):
        return self.model.objects.filter(session_key=session_key).exists()

    def create(self):
        while True:
        	  #这里的 _session_key是在SessionBase定义属性
        	  # _session_key = property(_get_session_key, _set_session_key)
            self._session_key = self._get_new_session_key()
            try:
                # 立即存储来保证我们有一个唯一的数据库入口
                self.save(must_create=True)
            except CreateError:
                # 键不是唯一的，再次尝试
                continue
            self.modified = True
            return

    def create_model_instance(self, data):
        """
        返回一个新的session model object对象的实例，对象代表了当前session的状态。被用来将session存储到数据库中。
        """
        return self.model(
            session_key=self._get_or_create_session_key(),
            session_data=self.encode(data),
            expire_date=self.get_expiry_date(),
        )

    def save(self, must_create=False):
        """
        将当前session数据存储到数据库中. 如果 'must_create' 为
        True，但是存储操作没有创建一个新的实例(和更新一个存在的实例相反)时，将会引发一个database error。
        """
        if self.session_key is None:
            return self.create()
        data = self._get_session(no_load=must_create)
        obj = self.create_model_instance(data)
        using = router.db_for_write(self.model, instance=obj)
        try:
            with transaction.atomic(using=using):
                obj.save(force_insert=must_create, force_update=not must_create, using=using)
        except IntegrityError:
            if must_create:
                raise CreateError
            raise
        except DatabaseError:
            if not must_create:
                raise UpdateError
            raise

    def delete(self, session_key=None):
        if session_key is None:
            if self.session_key is None:
                return
            session_key = self.session_key
        try:
            self.model.objects.get(session_key=session_key).delete()
        except self.model.DoesNotExist:
            pass

    @classmethod
    def clear_expired(cls):
   
       cls.get_model_class().objects.filter(expire_date__lt=timezone.now()).delete()

```

#### cached_property
这里倒有一个非常有趣的装饰器`cached_property
`值得我们注意，源码如下:

```python
class cached_property:
    """
	这个装饰器主要用来将一个带有单个self参数的方法转换成一个缓存在实例的属性，可选的`name`属性允许你将其他方法变成缓存的属性 (e.g.  url = cached_property(get_absolute_url, name='url') )
    """
    def __init__(self, func, name=None):
        self.func = func
        self.__doc__ = getattr(func, '__doc__')
        self.name = name or func.__name__

    def __get__(self, instance, cls=None):
        """
        调用这个函数并将返回值存储在instance.__dict__中，因此实例的后续的属性访问将会返回缓存的值而不是调用cached_property.__get__()
		"""
        if instance is None:
            return self
        # 将属性存储在实例的__dict__中
        res = instance.__dict__[self.name] = self.func(instance)
        return res
```
顾名思义，这个装饰器的主要的作用是把计算之后得到的属性直接写入到实例的`__dict__`属性中，这样就避免了每次访问都需要计算一次。看下面的例子你就可以一目了然了:

```python
import datetime

class cached_property:
    def __init__(self, func, name=None):
        self.func = func
        self.name = name or func.__name__
        self.__doc__ = getattr(func, '__doc__')

    def __get__(self, instance, cls=None):
        if instance is None:
            return self
        print(instance, cls)
        res = instance.__dict__[self.name] = self.func(instance)
        return res


class my_property:
    def __init__(self, func):
        self.func = func

    def __get__(self, instance, cls):
        if instance is None:
            return self
        print(instance, cls)
        return self.func(instance)


class Student:
    birth_date = 1997

    @my_property
    def age(self):
        return datetime.date.today().year - self.birth_date


class Student1:
    birth_date = 1997

    @cached_property
    def age(self):
        return datetime.date.today().year - self.birth_date

```
交互的结果如下:

```python
In [1]: student = Student()

In [2]: student.age
<__main__.Student object at 0x10cdac748> <class '__main__.Student'>
Out[2]: 21

In [3]: student.age
<__main__.Student object at 0x10cdac748> <class '__main__.Student'>
Out[3]: 21

In [4]: student.age
<__main__.Student object at 0x10cdac748> <class '__main__.Student'>
Out[4]: 21

In [5]: student1 = Student1()

In [6]: student1.age
<__main__.Student1 object at 0x10cd7d208> <class '__main__.Student1'>
Out[6]: 21

In [7]: student1.age
Out[7]: 21

In [8]: student1.age
Out[8]: 21

In [9]: student.__dict__
Out[9]: {}

In [10]: student1.__dict__
Out[10]: {'age': 21}
```

上面只是总结了`session`存储在数据库中的方法,其实在django中，`session`还可以存储在缓存和文件系统中，只要在`settings`设置不同的`session`后端就可以了。


### SessionMiddleware
session是如何在请求中起作用的呢？我们可以来看看`SessionMiddleware`的内容。

```python
# in django.utils.deprecation

class MiddlewareMixin:
    def __init__(self, get_response=None):
        self.get_response = get_response
        super().__init__()

    def __call__(self, request):
        response = None
        if hasattr(self, 'process_request'):
            response = self.process_request(request)
        if not response:
            response = self.get_response(request)
        if hasattr(self, 'process_response'):
            response = self.process_response(request, response)
        return response
        
        
# in django.contrib.sessions.middleware
class SessionMiddleware(MiddlewareMixin):
    def __init__(self, get_response=None):
        self.get_response = get_response
        engine = import_module(settings.SESSION_ENGINE)
        self.SessionStore = engine.SessionStore

    def process_request(self, request):
        session_key = request.COOKIES.get(settings.SESSION_COOKIE_NAME)
        request.session = self.SessionStore(session_key)

    def process_response(self, request, response):
        """
        如果request.session被修改了，或者设置是每次都要保存session,将会保存更改并设置会话cookie或者在session被清空的时候删除会话cookie
       """
        try:
            accessed = request.session.accessed
            modified = request.session.modified
            empty = request.session.is_empty()
        except AttributeError:
            pass
        else:
            # 首先检查我们是否需要删除这个cookie
            # 只有当session为空的时候，才需要删除删除这个session
            if settings.SESSION_COOKIE_NAME in request.COOKIES and empty:
                response.delete_cookie(
                    settings.SESSION_COOKIE_NAME,
                    path=settings.SESSION_COOKIE_PATH,
                    domain=settings.SESSION_COOKIE_DOMAIN,
                )
            else:
                if accessed:
                    patch_vary_headers(response, ('Cookie',))
                if (modified or settings.SESSION_SAVE_EVERY_REQUEST) and not empty:
                    if request.session.get_expire_at_browser_close():
                        max_age = None
                        expires = None
                    else:
                        max_age = request.session.get_expiry_age()
                        expires_time = time.time() + max_age
                        expires = cookie_date(expires_time)
                    # Save the session data and refresh the client cookie.
                    # Skip session save for 500 responses, refs #3881.
                    if response.status_code != 500:
                        try:
                            request.session.save()
                        except UpdateError:
                            raise SuspiciousOperation(
                                "The request's session was deleted before the "
                                "request completed. The user may have logged "
                                "out in a concurrent request, for example."
                            )
                        response.set_cookie(
                            settings.SESSION_COOKIE_NAME,
                            request.session.session_key, max_age=max_age,
                            expires=expires, domain=settings.SESSION_COOKIE_DOMAIN,
                            path=settings.SESSION_COOKIE_PATH,
                            secure=settings.SESSION_COOKIE_SECURE or None,
                            httponly=settings.SESSION_COOKIE_HTTPONLY or None,
                        )
        return response

```

在处理请求的时候，`process_request`方法会在请求中取出`session_key`，并把一个新的session对象赋给`request.session`。在处理响应的时候，首先会检查`session`是否为空，如果为空，就删除`session`,然后会检查是持久cookie还是会话cookie,如果是持久cookie，则会在每次响应的时候都更新一次过期时间。其中`settings.SESSION_COOKIE_NAME`的默认的值就是`session_id`.


### 认证中的session
我们先来看一下`AuthenticationMiddleware`中是如何利用`session`来处理`request.user`的。

```python
#django.contrib.auth.middleware
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
        
        
# django.contrib.auth.__init__

def get_user(request):
    """
    Return the user model instance associated with the given request session.
    If no user is retrieved, return an instance of `AnonymousUser`.
    """
    from .models import AnonymousUser
    user = None
    try:
        user_id = _get_user_session_key(request)
        backend_path = request.session[BACKEND_SESSION_KEY]
    except KeyError:
        pass
    else:
        if backend_path in settings.AUTHENTICATION_BACKENDS:
            backend = load_backend(backend_path)
            user = backend.get_user(user_id)
            # Verify the session
            if hasattr(user, 'get_session_auth_hash'):
                session_hash = request.session.get(HASH_SESSION_KEY)
                session_hash_verified = session_hash and constant_time_compare(
                    session_hash,
                    user.get_session_auth_hash()
                )
                if not session_hash_verified:
                    request.session.flush()
                    user = None

    return user or AnonymousUser()
    
```

其中`get_user`函数会检查session中是否存放了当前用户对应的`user_id`,如果有，并通过认证检查之后，会通过`user_id`得到相应的`user model`，否则，就会返回一个匿名用户的实例。`user_id`就是通过上文中提到的`session_data`得到的。

```python
{'_auth_user_backend': 'django.contrib.auth.backends.ModelBackend',
 '_auth_user_hash': '127cedbe51740d16ba10bb7e4fb54b1bc2c46b4a',
 '_auth_user_id': '2'}
```