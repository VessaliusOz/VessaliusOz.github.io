---
title: python使用smtplib库发送邮件
date: 2017-07-21 11:06:12
tags: 
  - python
---

今天准备在自己的论坛应用上实现密码找回的功能，主要方式就是用户点击忘了密码，然后后台会发一封电子邮件给用户注册时填写的email，然后实现找回密码的功能。

以前没有用python写过发邮件的功能，django的后台貌似自带了邮件发送的功能，只是自己还没有开始看。因为按照常理还应该去了解一下SMTP服务的具体内容和发送流程以及背后的一些流程，python的smtplib库已经对相应的方法做了很好的封装，有些语义还需要自己去了解。
<!--more-->
### 发送邮件实例
这里用了python的smtplib库和qq的smtp.qq.com服务。
首先，需要登录自己的qq邮箱，点击设置->账户->找到"POP3/IMAP/SMTP/Exchange/CardDAV/CalDAV服务"->打开POP3/SMTP服务,这时，qq会生成一个授权码，这个授权码就是之后代码里要填写的密码。

我们可以查到，qq邮箱的POP3和SMTP服务器的地址端口设置：

|        |  Mail  |      POP3       |     SMTP    |  SSL  |
|:------:| ------ |:--------------:|:------------:|:-----:|
|  ---   | qq.com |   pop.qq.com   | smtp.qq.com  |   --- |
|  PORT  |   ---  |       110      |      25      |   否  |
|  PORT  | ---    |       995      |  465 or 587  |   是  |

获得授权码以后，我们的代码如下：
```python
#! /user/bin/python3
# -*-encoding:utf8 -*-
from smtplib import SMTP_SSL
from email.header import Header
from email.mime.text import MIMEText


mail_info = {
    "from": "xxxxxx@qq.com", # 发件地址，必须和认证的user一样,就是下面的username
    "to": "xxxxxx@163.com", # 要发送的邮件地址
    "hostname": "smtp.qq.com",
    "username": "xxxxxx@qq.com", # 你的邮箱,和from的一致
    "password": "ckfqgefngpaphagd", # 就是qq生成的授权码
    "mail_subject": "test",　# 标题
    "mail_text": "hello,  this is a test email sended by Python",　# 内容
    "mail_encoding": "utf-8", # 编码
}


if __name__ == "__main__":
    smtp = SMTP_SSL(mail_info["hostname"]) # 创建一个SMTP_SSL()实例
    smtp.set_debuglevel(2) # 设置调试信息输出等级

    smtp.ehlo(mail_info["hostname"]) # 这与SMTP协议的初始化有关
    smtp.login(mail_info["username"], mail_info["password"])
    msg = MIMEText(mail_info["mail_text"], "plain",　#　传输文件的类型 mail_info["mail_encoding"])
    msg["Subject"] = Header(mail_info["mail_subject"], mail_info["mail_encoding"])
    msg["from"] = mail_info["from"]
    msg["to"] = mail_info["to"]

    smtp.sendmail(mail_info["from"], mail_info["to"], msg.as_string())

    smtp.quit()
```
以上代码通过了实际的验证，但是还是仅仅只能发送纯文本，更多更复杂的功能以后再说。

### smtplib文档翻译
下面是smtplib库的原文档的翻译，留作参考

smtplib模块定义了一个SMTP客户端会话对象，可用于向任何使用SMTP或ESMTP侦听器守护程序的Internet机器发送邮件。 有关SMTP和ESMTP操作的详细信息，请参阅RFC 821（简单邮件传输协议）和RFC 1869（SMTP服务扩展）。


#### class smtplib.SMTP(host='', port=0, local_hostname=None, [timeout, ]source_address=None)
SMTP实例封装SMTP连接。 它具有支持完整的SMTP和ESMTP操作的方法。 如果给出了可选的主机和端口参数，则在初始化期间使用这些参数调用SMTP connect（）方法。如果指定了local_hostname，则在HELO/EHLO指令中，local_hostname将被用作本地主机的FQDN。否则，则使用socket.getfqdn()来找到本地主机名。如果调用connect（）返回成功代码以外的任何东西，则会引发SMTPConnectError。可选的timeout参数将指定一个阻塞操作(连接请求等)的超时时长(seconds)，如果没有指定，则将使用全局默认超时设置。若超时，将会引发socket.timeout。可选的source_address参数允许再有多个网络接口的机器上或某个特定的原TCP端口上绑定某些特定的源地址。它需要2元组（主机，端口），以便在连接之前将套接字绑定为其源地址。如果省略，即(或者如果host和port分别或全部为0)，将会使用OS的默认行为。对于正常使用，您只需要初始化/连接，sendmail（）和quit（）方法。



#### class smtplib.SMTP_SSL(host='', port=0, local_hostname=None, keyfile=None, certfile=None, [timeout, ]context=None, source_address=None)

SMTP_SSL实例的行为与SMTP实例完全相同。SMTP_SSL适用于从连接开始就需要SSL并且使用starttls（）不合适的情况。如果host没有给出，就会使用本地主机。如果端口为零，则使用标准的SMTP-over-SSL端口（465）。其他的可选参数包括local_hostname,timeout,和source_address和SMTP实例里一样。可选参数context可以包含一个SSLContext,允许配置安全连接的各个方面。
keyfile和certfile是context的传统替代，并且可以指向用于SSL连接的PEM格式的私钥和证书链文件。


#### class smtplib.LMTP(host='', port=LMTP_PORT, local_hostname=None, source_address=None)
与ESMTP非常相似的LMTP协议主要基于标准SMTP客户端。 对于LMTP，通常使用Unix套接字，所以我们的connect()方法必须支持Unix套接字以及一个常规的host:port服务。可选参数local_hostname和source_address与SMTP类中的含义相同。为了指定一个Unix套接字,对于host,你必须使用绝对路径，用"/"开头。



#### 定义的异常：
###### smtplib.SMTPException
OSError的子类，是该模块提供的所有其他异常的基础异常类。

###### exception smtplib.SMTPServerDisconnected
当服务器意外断开或者在连接到一个服务之前尝试使用SMTP实例，会引发此异常。

###### exception smtplib.SMTPResponseException
包含SMTP错误代码的所有异常的基类。 某些情况下，SMTP服务器返回错误代码时会生成这些异常。 错误代码存储在错误的smtp_code属性中，smtp_error属性设置为错误消息。

###### exception smtplib.SMTPSenderRefused
发件人地址被拒。除了被SMTPResponseException异常设置的所有属性意外，这将会设置sender为SMTP服务拒绝的字符串。


###### exception smtplib.SMTPRecipientsRefused
所有收件人的地址都被拒绝。对于每个通过recipients属性得到的收件人的error，是一个与SMTP.sendmail()返回的完全相同排序的字典。


###### exception smtplib.SMTPDataError
SMTP服务器拒绝接收消息数据。

###### exception smtplib.SMTPConnectError
与服务器建立连接时发生错误。

###### exception smtplib.SMTPHeloError
服务器拒绝了我们的HELO消息。

###### exception smtplib.SMTPNotSupportedError
服务器不支持尝试的命令或选项。

###### exception smtplib.SMTPAuthenticationError
SMTP认证出错。 很可能服务器不接受提供的用户名/密码组合。


#### SMTP Objects
SMTP实例有如下方法
##### SMTP.set_debuglevel(level)
设置调试输出级别。设置值为1或True,将输出连接、发出的所有消息和得到的所有消息的调试信息。设置值为2将会输出带有时间戳的以上消息。

##### SMTP.docmd(cmd, args='')
发送命令cmd到服务器。 可选参数args简单地连接到命令，用空格分隔。这返回一个由数字响应代码和实际响应行组成的2元组（多行响应被连接成一条长行）。
在正常操作中，不需要显示调用这个方法。 它用于实现其他方法，可能有助于测试私有扩展。



##### SMTP.connect(host='localhost', port=0)
连接到给定端口上的主机。 默认值是在标准SMTP端口（25）连接到本地主机。 如果主机名以冒号（'：'）结尾，后跟一个数字，该后缀将被删除，并将该数字解释为要使用的端口号。 如果在实例化过程中指定了主机，则此方法由构造函数自动调用。 返回服务器在其连接响应中发送的响应代码和消息的2元组。

##### SMTP.helo(name='')
使用HELO来识别SMTP服务器。 hostname参数缺省为本地主机的完全限定域名。 服务器返回的消息存储为对象的helo_resp属性。



##### SMTP.ehlo(name='')
使用EHLO识别您的ESMTP服务器。 hostname参数缺省为本地主机的完全限定域名。 检查ESMTP选项的响应并存储它们以供has_extn（）使用。 还设置了几个信息属性：服务器返回的消息被存储为ehlo_resp属性，根据服务器是否支持ESMTP，将do_esmtp设置为true或false，而esmtp_features将是一个包含此服务器支持的SMTP服务扩展名称及其参数（如果有的话）的字典。


##### SMTP.ehlo_or_helo_if_needed()
如果没有以前的EHLO或HELO命令，此方法调用ehlo（）和/或helo（）。 它首先尝试ESMTP EHLO。

##### SMTP.has_extn(name)
如果名称位于服务器返回的SMTP服务扩展集中，则返回True，否则返回False。

##### SMTP.verify(address)
使用SMTP VRFY检查此服务器上的地址的有效性。 如果用户地址有效，则返回由代码250和完整的RFC 822地址（包括人名）组成的元组。 否则返回400或更大的SMTP错误代码和错误字符串。



##### SMTP.login(user, password, *, initial_response_ok=True)
登录需要身份验证的SMTP服务器。 参数是用于验证的用户名和密码。 如果以前没有EHLO或HELO命令本次会话，此方法首先尝试ESMTP EHLO。 如果身份验证成功，此方法将正常返回，或者可能会引发以下异常：
###### SMTPHeloError
服务器没有正确回复HELO问候语。
###### SMTPAuthenticationError
服务器不接受的用户名/密码组合。
###### SMTPNotSupportedError
服务器不支持AUTH命令。
###### SMTPException
没有找到合适的认证方法。

如果服务器支持smtplib支持的每个验证方法，都会依次尝试。有关支持的身份验证方法的列表，请参阅auth（）。 将initial_response_ok传递给auth（）。
可选关键字参数initial_response_ok指定，对于支持它的身份验证方法，如果是RFC 4954中规定的“initial response ”可以与AUTH命令一起发送的，就不需要询问/响应。



##### SMTP.auth(mechanism, authobject, *, initial_response_ok=True)
发出指定生们验证机制的smtp aut命令，并通过authproject处理查询响应
mechanism指定哪个认证机制用作AUTH命令的参数; 有效值是esmtp_features的auth元素中列出的值。authobject必须是一个可调用的对象，使用一个可选的单个参数：
data = authobject(challenge=None)

如果可选关键字参数initial_response_ok为true，那么将首先调用authobject（），而不用参数。它可以返回RFC 4954“initial response”字节，这些字节将使用AUTH命令编码和发送，如下所示。

如果authobject（）不支持初始响应（例如因为需要一个查询），则当使用challenge = None调用时，它应返回None。如果initial_response_ok为false，那么authobject（）不会首先用None调用。

如果初始响应检查返回None，或者如果initial_response_ok为false，将调用authobject（）来处理服务器的质询响应; 传递的质询参数将是一个字节。它应该返回将被base64编码并发送到服务器的字节数据。

SMTP类为CRAM-MD5，PLAIN和LOGIN机制提供authobjects; 它们分别命名为SMTP.auth_cram_md5，SMTP.auth_plain和SMTP.auth_login。 它们都要求将SMTP实例的用户和密码属性设置为适当的值。
用户代码通常不需要直接调用auth，而是可以调用login（）方法，这将依次列出所有上述机制。 auth暴露以方便执行没有（或尚未）被smtplib直接支持的身份验证方法。



##### SMTP.starttls(keyfile=None, certfile=None, context=None)
将SMTP连接置于TLS（传输层安全）模式。 以下所有SMTP命令将被加密。 之后你应该再次调用ehlo（）。

如果keyfile和certfile提供了，这将被传递给socket模块下的ssl()函数
可选的context参数是一个ssl.SSLContext对象; 这是使用密钥文件和证书文件的另一种方法，如果指定keyfile和certfile都应该为None。
如果以前没有EHLO或HELO命令本次会话，此方法首先尝试ESMTP EHLO。

###### SMTPHeloError
服务器没有正确回复HELO问候语。
###### SMTPNotSupportedError
服务器不支持STARTTLS扩展。
###### RuntimeError
您的Python解释器不支持SSL / TLS。



#### SMTP.sendmail(from_addr, to_addrs, msg, mail_options=[], rcpt_options=[])
发送邮件。
所需的参数是RFC 822 from-address字符串，RFC 822to-address字符串的列表（裸字符串将被视为具有1个地址的列表）和消息字符串。caller可以传递一个ESMTP选项列表（如8bitmime），以作为mail_options在MAIL FROM命令中使用。应与所有RCPT命令一起使用的ESMTP选项（如DSN命令）可以作为rcpt_options传递。 （如果需要对不同的收件人使用不同的ESMTP选项，则必须使用诸如mail（），rcpt（）和data（）之类的低级方法来发送消息。）

注意:from_addr和to_addrs参数用于构建传输代理使用的消息信封。 sendmail不会以任何方式修改消息头。

msg可能是包含ASCII范围内的字符或字符串的字符串。 使用ascii编解码器将字符串编码为字节，并将单个\\ r和\\ n字符转换为\\ r \\ n个字符。 字节字符串不被修改。

如果以前没有EHLO或HELO命令本次会话，此方法首先尝试ESMTP EHLO。 如果服务器执行ESMTP，则消息大小和每个指定的选项将被传递给它（如果该选项位于服务器通告的功能集中）。 如果EHLO发生故障，HELO将被尝试并且ESMTP选项被抑制。

如果至少有一个收件人接受了邮件，则此方法将正常返回，否则会引发异常。这意味着，如果这种方法没有引起异常，那么有人就应该接收到了你的邮件。如果这种方法没有引发异常，就会返回一个字典，有关于被拒绝的接收者的条目。每个条目都包含SMTP错误代码的元组和服务器发送的对应的错误信息。
可能会有如下异常：
###### SMTPRecipientsRefused
所有接收者都被拒绝。 没有人收到邮件。 异常对象的recipients属性是一个包含被拒绝的接收者信息的字典（如接受至少一个收件人时返回的收件人）。
###### SMTPHeloError
服务器没有正确回复HELO问候语。
###### SMTPSenderRefused
服务器不接受from_addr。    
###### SMTPDataError
服务器回复了一个意外的错误码（接收者的拒绝除外）。
###### SMTPNotSupportedError
SMTPUTF8已在mail_options中给出，但不受服务器支持。
除非另有说明，就算有异常出现，连接依然会在之后打开。





#### SMTP.send_message(msg, from_addr=None, to_addrs=None, mail_options=[], rcpt_options=[])
这是使用email.message.Message对象表示的消息调用sendmail（）的方便方法。 参数与sendmail（）具有相同的含义，除了msg是一个Message对象。

如果from_addr为None或to_addrs为None，则send_message将使用从RFC 5322中指定的msg头文件中提取的地址填充这些参数：

如果存在，则将from_addr设置为“Sender”字段，否则将设置为“From”字段。 to_addrs组合来自msg的To，Cc和Bcc字段的值（如果有）。

如果消息中只出现一组Resent- *标头，则会忽略常规标题，而使用Resent- *标头。

如果消息包含多个Resent- *标头集合，则会引发ValueError，因为无法清楚地检测最新的Resent标头集。

send_message使用带有\\ r \\ n的BytesGenerator串行化msg，并调用sendmail（）来传送结果消息。不管from_addr和to_addrs的值如何，send_message不会发送可能出现在msg中的任何密件抄送或Resent-Bcc头文件。 如果from_addr和to_addrs中的任何地址包含非ASCII字符，并且服务器不发布SMTPUTF8支持，则会引发SMTPNotSupported错误。否则，Message将会使用一个把utf8属性设置为True的克隆的策略使其序列化。并将SMTPUTF8和BODY = 8BITMIME添加到mail_options。

#### SMTP.quit()
终止SMTP会话并关闭连接。 返回SMTP QUIT命令的结果。
还支持与标准SMTP / ESMTP命令HELP，RSET，NOOP，MAIL，RCPT和DATA相对应的低级方法。 通常这些不需要直接调用，所以这里没有记录。 有关详细信息，请参阅模块代码。
