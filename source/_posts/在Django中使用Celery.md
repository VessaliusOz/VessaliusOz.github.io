---
title: 在Django中使用Celery
date: 2017-10-19 21:01:40
tags:
  - Django
---
最近的那个传感器的项目的第一期可以说是完工了，主要功能已经实现了。但是有
些地方我觉得自己做的并不是很好，就比如对于某些可能会非常耗时的操作，我还是写的同步执行的代码。当读写操作非常多、或者因为某个非常耗时的任务导致响应阻塞的时候，同步操作的性能是非常的差的。

正好自己最近再看`Celery`，发现自己写的项目里面有好几个地方是可以用`Celery`来优化的。虽然这只是一个测试用的项目，瞬时的数据传输量其实不是很大，但是还是应该折腾一下，哈哈哈。
<!--more-->
就比如下面要讲到的一个场景：网关不停的给我的`socket`服务器发送数据，而每条数据都是要写入`Redis`里面的(主要是为了性能的考虑)，而我一开始的时候是按同步的方式执行的，就是说，来一条数据，我就写一条数据，直到写完之后，再来读下一条数据。相关的代码如下：
```python
# 简化过
def handle(self):
    time_interval = (datetime.datetime.now() - self.t).seconds
    while time_interval < 10 and not self.wantDisconnect:
        data = self.request.recv(1024)
        if data:
              data_handler = connect_to_redis(data)
              try:
                  data_handler.insert_into_redis()
              except:
                    break
              cur_thread = threading.current_thread()
              response = "{}: {}".format(cur_thread.name, data)
              self.request.sendall(response.encode("ascii"))
        else:
              print(threading.currentThread().name, " no data")
              break
```
其中`insert_into_redis()`就是一个将数据传感器数据写入`Redis`的操作，当传感器传入数据的速率过快、写入`Redis`的操作非常耗时的时候，在这种同步的处理下，服务器必须在写完一条数据之后才能继续读下一条数据，这样的性能是非常差的。所以这里完全可以把`insert_into_redis()`这个`task`交给`Celery`在后台异步的完成，这样子对于消息的接收就不会阻塞了(代码等我改完了之后在贴上来)。

一般来说，在常见的`web`服务中，我们都能遇到类似于用户注册发邮件的情况，在这种情况下，为了加快用户的响应时间，我们一般也都会采用异步任务的方式在后台执行这些任务，对于常见的请求执行时间较长的任务也同样可以采用这种方式。

而在这次的项目里面。也涉及到用户的注册和登录，所以对于发送邮件这件事情，也可以采用`Celery`在后台异步的执行。

发送邮件的博客以前记录过，这里不再详细说明，本篇博客权当记录如何在`Django`中使用`Celery`,用来备忘。官网上其实写的比较详细了，但是某些地方应该是有问题的(至少我按照官网上面的教程出了问题。。。)

这是我的文件结构
```
.
├── manage.py
├── MyAPP
│   ├── celery.py
│   ├── __init__.py
│   ├── settings.py
│   ├── urls.py
│   └── wsgi.py
└── test_celery
    ├── admin.py
    ├── apps.py
    ├── __init__.py
    ├── migrations
    ├── models.py
    ├── tasks.py
    ├── tests.py
    ├── urls.py
    └── views.py

```
`celery.py`是和`settings.py`在同一个文件夹里面的,它的配置信息如下:
```
from __future__ import absolute_import, unicode_literals
import os
from celery import Celery
from MyAPP import settings

os.environ.setdefault("DJANGO_SETTINGS_MODULE", "MyAPP.settings")

app = Celery("MyAPP")

# 若你的Celery版本为3.1则传入namespace参数会报错.
app.config_from_object("django.conf:settings", namespace="CELERY")

# 注意版本，如果你的Celery的版本为3.1则需要传入 lambda: settings.INSTALLED_APPS  参数
#否则会报错
app.autodiscover_tasks()

@app.task(bind=True)
def debug_task(self):
    print("Request: {0!r}".format(self.request))
```
其中`app.config_from_object("django.conf:settings", namespace="CELERY")`的`namespace`选项的作用是在settings里面，Celery的配置选项必须以大写而不是小写指定，并以CELERY_开头，这样有利于区分，当然还得注意你的`Celery`的版本的问题。

app.autodiscover_tasks()的作用是自动的发现每个app下面的`tasks.py`，这样有助于app的重用(文件名必须使用tasks.py)
```
- app1/
    - tasks.py
    - models.py
- app2/
    - tasks.py
    - models.py
```
再来看看`settngs`里面的配置，在末尾加入:
```
CELERY_BROKER_URL = "redis://localhost:6379"
CELERY_RESULT_BACKEND = "redis://localhost:6379"
```
这里我使用的是`Redis`作为`result backend`和`broker`.

然后你需要把这个`app`导入到` proj/proj/__init__.py`里面，这保证在django开始运行的时候会加载`app`, 而且`@shared_task`装饰器会使用这个app:
```
# MyAPP/__init__.py:
from __future__ import absolute_import, unicode_literals

# This will make sure the app is always imported when
# Django starts so that shared_task will use this app.

from .celery import app as celery_app

__all__ = ['celery_app']
```

再来说说app文件夹(这里是test_celery)下的`tasks.py`:
```
from __future__ import absolute_import, unicode_literals
from celery import shared_task
import time

@shared_task
def consume_time_task(number):
    time.sleep(int(number))
    print("I have sleep {}s ".format(number))
    return "I have sleep {}s ".format(number)
```
为什么要用`@shared_task`呢，而不是`@app.task`这个在装饰器呢。官方给出的解释是这样的。因为你编写的task可能会在可重复使用的app(指的是`Celery`的app)中使用，但是可重用的app(指的是`Celery`的app)并不依赖项目本身，所以你不能直接导入app的实例，@shared_task装饰器可以让你在无需任何具体的应用程序实例的情况下，创建任务。所以这里使用的是@shared_task,因为你有可能不只开启一个`Celery APP`的实例。

我们这里的`consume_time_task`就是一个简单函数，你传入多少时长，就阻塞多少秒。

我们再来看一下`views.py`里面的内容:
```
from django.http import HttpResponse
from test_celery.tasks import consume_time_task


def test_view(request, number=0):
    # 调用　consume_time_task　任务
    consume_time_task.delay(number)
    return HttpResponse("your input is {} ".format(number))

```
当在与`manage.py`的同级目录下运行`celery -A MyAPP worker -l info`命令后可以发现输出如下:
```
-------------- celery@vessalius-oz v4.1.0 (latentcall)
---- **** -----
--- * ***  * -- Linux-4.4.0-97-generic-x86_64-with-Ubuntu-16.04-xenial 2017-10-20 05:26:43
-- * - **** ---
- ** ---------- [config]
- ** ---------- .> app:         MyAPP:0x7f6851259828
- ** ---------- .> transport:   redis://localhost:6379//
- ** ---------- .> results:     redis://localhost:6379/
- *** --- * --- .> concurrency: 4 (prefork)
-- ******* ---- .> task events: OFF (enable -E to monitor tasks in this worker)
--- ***** -----
 -------------- [queues]
                .> celery           exchange=celery(direct) key=celery


[tasks]
  . MyAPP.celery.debug_task
  . test_celery.tasks.add
  . test_celery.tasks.consume_time_task
  . test_celery.tasks.mul
  . test_celery.tasks.xsum

[2017-10-20 05:26:43,574: INFO/MainProcess] Connected to redis://localhost:6379//
[2017-10-20 05:26:43,583: INFO/MainProcess] mingle: searching for neighbors
[2017-10-20 05:26:44,605: INFO/MainProcess] mingle: all alone
[2017-10-20 05:26:44,622: WARNING/MainProcess] /usr/local/lib/python3.5/dist-packages/celery/fixups/django.py:202: UserWarning: Using settings.DEBUG leads to a memory leak, never use this setting in production environments!
  warnings.warn('Using settings.DEBUG leads to a memory leak, never '
[2017-10-20 05:26:44,622: INFO/MainProcess] celery@vessalius-oz ready.

```
当我们连续两次访问`http://127.0.0.1:8000/seconds/10/`时,任务队列出现两个task:
```
[2017-10-20 05:29:44,570: INFO/MainProcess] Received task: test_celery.tasks.consume_time_task[cef9b40b-4ea8-41ba-ae92-7ab4088b523a]  
[2017-10-20 05:29:45,224: INFO/MainProcess] Received task: test_celery.tasks.consume_time_task[9641bb72-8125-435a-8367-a1013e98b3f9]  
```
而客户端就直接返回了。过了十秒钟以后我们可以发现:
```
[2017-10-20 05:29:54,582: WARNING/ForkPoolWorker-4] I have sleep 10s
[2017-10-20 05:29:54,583: INFO/ForkPoolWorker-4] Task test_celery.tasks.consume_time_task[cef9b40b-4ea8-41ba-ae92-7ab4088b523a] succeeded in 10.011570967999432s: 'I have sleep 10s '
[2017-10-20 05:29:55,234: WARNING/ForkPoolWorker-1] I have sleep 10s
[2017-10-20 05:29:55,236: INFO/ForkPoolWorker-1] Task test_celery.tasks.consume_time_task[9641bb72-8125-435a-8367-a1013e98b3f9] succeeded in 10.01056424700073s: 'I have sleep 10s '
```
这两个任务几乎是同时返回的，说明在任务队列里面，任务是并行执行的，但是并行执行的数量是有限的，默认并发数量是主机上面的CPU的数目。当所有这些进程正在忙着工作时，新任务将必须等待其中一个任务完成才能处理。

那对于发送邮件的任务可以这样写

```
# views.py

def register(request):
    ......
    send_email.delay(email, type="register")
    return HttpResponse({"code": 200, "msg": "send email successfully"})



# tasks.py

@shared_task
def send_email(email, type):
    ......
```
发送邮件的操作就可以交给后台异步处理啦。

`Celery`还有很多的功能，我也正在学，然后又发现了还有`django-celery`这个app,赶紧去学习一波，这次的总结就先到这儿啦。
