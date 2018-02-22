---
title: django-channels初探
date: 2018-01-08 16:52:46
tags: 
  - Django
---
最近在倒腾`django-channels`，它刚刚出来的时候，我就倒腾过，但是可能是因为当时太浮躁了，就没有认真的学习下去(此处应该给自己一个耳光)。现在再重新来看的话，觉得它异常的牛逼。。。现在就一边来学习一边总结。

<!--more-->
# WebSocket
### 传统HTTP
我们知道，传统的`HTTP`是无状态协议，它不支持持久的连接(长连接和轮寻不算。)。在`HTTP1.0`的版本中,`HTTP`的生命周期是通过`Request`来界定的，客户端发起一个`Request`，服务端返回一个`Response`，一个`Request`,一个`Response`,这样，这次`HTTP`通信就结束了。比如说当你请求一个网页的时候，网页不仅有`HTML`文档,也有静态文件。那么在`HTTP1.0`中，你对每一个资源的请求，都会重新建立一个`HTTP`连接。比如你请求两张图片，就需要建立两次连接，因为每个连接里面只包含了一个`Request`和`Response`.

在`HTTP1.1`中进行了改进。`HTTP`头请求里面加入了`keep-alive`参数，也就是说，在一次`HTTP`通信中，可以发送多个`Request`,然后得到多个返回的`Response`。但是在这一次的通信中，每个`Request`对应且仅对应一个`Response`。还是上面的例子，你要请求两张图片，这次虽然有两个`Request`和`Response`,你只需要建立一次`HTTP`连接。

但是无论是`HTTP1.0`还是`HTTP1.1`,一个`Request`只能有一个`Response`,而且这个`Response`是被动的。不能主动发起。

### WebSocket
`WebSocket`是基于`HTTP`的。它借用了`HTTP`协议完成了
一部分握手。它解决了`HTTP`协议的几个难题。首先它解决了被动性的问题，从`WebSocket`的名字就可以看出，这是`Socket`,众所周知，`Socket`是一个全双工的协议，可以进行双向的通信。而在使用`WebSocket`之后，服务器就可以主动向客户端推送消息了。

`WebSocket`相比于'Ajax轮寻'以及`long pull长连接`而言，效率会高很多，而且也不会造成过多的资源浪费。

# channels

### channels的安装
emmmmmmm.安装没啥好说的。直接看官网就行了。但是需要注意的是`channels`支持的`django`的版本只有`1.8.x-1.10.x`，所以`channels`并不支持最新的`django2.0`。

### 基本的概念
`django-channels`中有几个基本的概念需要先说一下:

#### channel
`channel`是系统的核心，它是一个有序的，先进先出的队列，里面的消息有过期时间，而且最多一次只能传递给一个`listener`。你可以把`channel`想象为一个任务队列，`message`会被`producers`送到`channel`，然后只会被交给许多监听这个`channel`的`consumer`中的一个。其中`message`(可以被想象成是`view`里的`request`)必须是一个可序列化的对象，并且会有长度的限制。

`channel`使`Django`运行在`worker`模式，它会监听所有被部署了`consumer`的`channel`,当一个`message`到来的时候，它就会调用相应的`consumer`,所以，`channels`使得`Django`运行了三个不同的服务。

* interface servers
`Django`通过它来和外部打交道。它包含了一个`WSGI服务`和一个独立的`WebSocket`服务。
* the channel backend
主要负责传递`message`
* the workers
监听相关的`channel`,当消息来的时候，运行相应的`comsumer`。

`channels`处理的流程图
![](/img/django-channels.png)

#### channel types
主要有两种`channel`:

一种(大多数)是用来向`consumer`分发工作:当一个`message`被分发到一个`channel`时，任何一个`worker`可以使用`consumer`来处理`message`.

第二种主要用来回复消息。因为只有一个`interface server`在监听所有的`channel`，所以每个`reply channel`都是被单独命名的而且必须被路由回`interface server`。因此`response channel`必须把他们的`message`发送给他们正在监听的`channel server`。这样的话，每个`response channel`的命名都包含了`channel`的名字加上随机字符串。比如`http.response`的`reply_channel`的名字可能为`http.response!f5G3fE21f`。(其实写了这么多就是为了说明如何区分通过同一个`channel`连接后不同的`reply_channel`)

#### Groups
组这个名词我觉得大家应该都不陌生。`channel`可以通过`Group`来达到组播的效果。为了在连接建立或者关闭的时候将一个连接从组中添加或者移除，我们一般会这样写:
```
# Connected to websocket.connect
def ws_connect(message):
    # Add to reader group
    Group("liveblog").add(message.reply_channel)
    # Accept the connection request
    message.reply_channel.send({"accept": True})

# Connected to websocket.disconnect
def ws_disconnect(message):
    # Remove from reader group on clean disconnect
    Group("liveblog").discard(message.reply_channel)
```

今天就先写到这儿吧，感觉这样写博客其实挺没意思的，没有一点思考的痕迹。还是等在使用的时候，记录一些好玩的，值得记录的东西吧。

***

### 路由设置
在学习的时候有个地方当时没有想明白。。。感觉自己想得有点复杂了。在我看来，如果提供一个完整的`WebSocket`的服务，至少需要三个`channel`，分别对应`websocket.connect`,`websocket.receive`,`websocket.disconnect`以及相关的`consumer`，所以当时脑袋抽了，如果一个`app`里面需要提供多个独立的`websocket`的服务，那怎么提供呢?不可能共用三个`channel`啊？后来才发现是自己有点蠢。。。这个其实和`Django`里面的`url`的设置几乎是一样的。有多个独立的`WebSocket`服务的`url`应该是这样设置的.我觉得这样写很清晰而且一目了然。
```
from channels.routing import route, include
from chartroom.consumers import *

# 值得注意的是，这里的正则表达式包含了第一个"/"
chat_routings = [
    route('websocket.receive', ws_room_message, path=r'^/(?P<room_name>\w+)/$'),
    route('websocket.connect', ws_room_connect, path=r'^/(?P<room_name>\w+)/$'),
    route('websocket.disconnect', ws_room_disconnect, path=r'^/(?P<room_name>\w+)/$'),
]


normal_routings = [
    route('websocket.receive', ws_normal_message),
    route('websocket.connect', ws_normal_connect),
    route('websocket.disconnect', ws_normal_disconnect),
]


channel_routing = [
    include(chat_routings, path=r'/chat/'),
    # 同样，和Django一样，你也可以传递参数给包含的url,例如把prex传递给chat_routings
    include(chat_routings, path=r'^/(?P<prex>\w+)/'),
    include(normal_routings),
]
```

### 和model联系起来
还有一个我觉得很有趣的做法，是官网上面给出来的例子，可以参考一下:
```
# In models.py
class ChatMessage(models.Model):
    message = models.TextField(verbose_name='room_message', default=None, null=True)
    room = models.CharField(max_length=20, null=True, default=None)

# In consumers.py
from channels import Channel, Group
from channels.sessions import channel_session
from .models import ChatMessage

# Connected to chat-messages
def msg_consumer(message):
    # Save to model
    room = message.content['room']
    ChatMessage.objects.create(
        room=room,
        message=message.content['message'],
    )
    # Broadcast to listening sockets
    Group("chat-%s" % room).send({
        "text": message.content['message'],
    })

# Connected to websocket.connect
@channel_session
def ws_connect(message):
    # Work out room name from path (ignore slashes)
    room = message.content['path'].strip("/")
    # Save room in session and add us to the group
    message.channel_session['room'] = room
    Group("chat-%s" % room).add(message.reply_channel)
    # Accept the connection request
    message.reply_channel.send({"accept": True})

# Connected to websocket.receive
@channel_session
def ws_message(message):
    # Stick the message onto the processing queue
    Channel("chat-messages").send({
        "room": message.channel_session['room'],
        "message": message['text'],
    })

# Connected to websocket.disconnect
@channel_session
def ws_disconnect(message):
    Group("chat-%s" % message.channel_session['room']).discard(message.reply_channel)


# In routings.py
from channels.routing import route
from myapp.consumers import ws_connect, ws_message, ws_disconnect, msg_consumer

channel_routing = [
    route("websocket.connect", ws_connect),
    route("websocket.receive", ws_message),
    route("websocket.disconnect", ws_disconnect),
    # 这里指明了另外的的一个channel,但是这个channel对用户是不可见的，只是在程序的内部使用
    route("chat-messages", msg_consumer),
]
```
这里的例子是存储每个聊天室的所发送的消息，并且是一个异步的调用:这里把`send`事件和`model`的保存流绑定在一起，在写入数据库之后才把这个`message`广播到用户所在的组。在`ws_message consumer`中，我们只是把消息传递到`channel('chat-messages')`后就可以立即继续监听`channel('websocket.receive')`,而把数据的存储和广播(费时的操作)都交给了监听`channel('chat-messages')`的`consumer``msg_consumer`中完成。

而且我们可以在任何地方给`channel('chat-messages')`传递消息，比如`view`函数、在收到另外的`model`的`post_save`的信号后等等。


# else

今天学到了非常优雅的生成指定长度的随机字符串的方法,不需要使用非常丑陋的`+`来完成:
```
import random
import string

def generate_random_string(n):
    return ''.join(random.sample(string.ascii_letters, n))

```
