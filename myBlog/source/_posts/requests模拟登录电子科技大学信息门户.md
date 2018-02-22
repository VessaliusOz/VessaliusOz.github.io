---
title: requests模拟登录电子科技大学信息门户
date: 2017-10-05 17:33:56
tags: 
  - python
---
今天闲来无事，发现自己好像还一直没有做过关于模拟登录的东西。趁着十一休息，就尝试着来模拟登录一下电子科技大学(没错，就是我的大学)的信息门户。在尝试的过程中遇到了一些问题，所以特地来总结一下，还学到了关于HTTP协议的一些东西。感觉还是有些收获的。


<!--more-->
### HTTP流程分析
为了弄清楚整个http请求的流程，我先把浏览器的缓存和cookies清空。然后具体的响应流程如下，
#### 第一次请求登录页面
首先我们用get方式来请求登录页面，当我用firebug可以发现，第一次关于html页面的http请求头如下：

```
Host: idas.uestc.edu.cn
User-Agent: Mozilla/5.0 (X11; Ubuntu; Linux x86_64; rv:55.0) Gecko/20100101 Firefox/55.0
Accept: text/html,application/xhtml+xml,application/xml;q=0.9,*/*;q=0.8
Accept-Language: zh-CN,zh;q=0.8,en-US;q=0.5,en;q=0.3
Accept-Encoding: gzip, deflate
Connection: keep-alive
Upgrade-Insecure-Requests: 1
```
这里没有什么特别的，我们来看看响应头的信息：
```
Server: openresty
Date: Thu, 05 Oct 2017 12:50:07 GMT
Content-Type: text/html; charset=UTF-8
Content-Length: 7338
Connection: keep-alive
Pragma: no-cache
Expires: Thu, 01 Jan 1970 00:00:00 GMT
Cache-Control: no-cache, no-store
Content-Language: en-US
Set-Cookie: route=79410db11950631816595971e536cafe
JSESSIONID_ids2=0001M_p72CMi9tHlgrt5ILP8Cbc:-1SP0UUR; Path=/
```
然后在这次的响应中我们可以发现其中有`Set-Cookie`字段,具体的cookie我们稍后再说。

然后我们发现在随后的静态资源的请求中，所有的请求头里都有了`cookie`字段：
```
Host: idas.uestc.edu.cn
User-Agent: Mozilla/5.0 (X11; Ubuntu; Linux x86_64; rv:56.0) Gecko/20100101 Firefox/56.0
Accept: text/css,*/*;q=0.1
Accept-Language: zh-CN,zh;q=0.8,en-US;q=0.5,en;q=0.3
Accept-Encoding: gzip, deflate
Referer: http://idas.uestc.edu.cn/authserver/login?service=http://portal.uestc.edu.cn/index.portal
Cookie: route=79410db11950631816595971e536cafe; JSESSIONID_ids2=0001M_p72CMi9tHlgrt5ILP8Cbc:-1SP0UUR
Connection: keep-alive
```
而请求头里所带有的`cookie`的信息，和第一次响应头里的`Set-Cookie`里的信息是一样。

#### POST提交表单
接下来我们在登录表单里填入了学号和密码以后，点击submit按钮，会发现最开始的时候有两个重定向，第一个是POST方法，第二个是GET方法。
我们先来看看这个POST请求做了那些哪些事情：

+ 提交表单到`http://idas.uestc.edu.cn/authserver/login?service=http://portal.uestc.edu.cn/index.portal`这个url
我们来看看提交了哪些内容：
```
username	  xxxxxxx
password	  xxxxxxx
lt	     　　LT-338386-fI3husmUHWNeTe5D7r40…Xk6Olq31507177713997-qNsN-cas
dllt	　　   userNamePasswordLogin
execution	  e1s1
_eventId	  submit
rmShown	    1
```
我们可以发现，这次的form提交过程中，不仅仅提交了`username`和`password`字段，还提交了一些其他的字段`dllt`,`execution`,`_eventId`,`rmShown`和`lt`。
那么这些字段是哪里来的呢，我们回过头来看看审查`form`表单里的`input`元素，可以发现有如下元素：
```html
<input name="lt" value="LT-338386-fI3husmUHWNeTe5D7r40FQxXk6Olq31507177713997-qNsN-cas" type="hidden">
<input name="dllt" value="userNamePasswordLogin" type="hidden">
<input name="execution" value="e1s1" type="hidden">
<input name="_eventId" value="submit" type="hidden">
<input name="rmShown" value="1" type="hidden">
```
原来这是隐藏的表单。其中只有`lt`在每次请求时都会改变，这是为了防止`csrf`([什么是csrf](http://www.cnblogs.com/hyddd/archive/2009/04/09/1432744.html))攻击。而其他的不变的值可能只是一些事件的信息。所以，当你模拟登录时，只是提交`username`和`password`，是永远不可能成功的。
这一次的响应头如下：
```
Server: openresty
Date: Thu, 05 Oct 2017 04:32:21 GMT
Content-Length: 0
Connection: keep-alive
Pragma: no-cache
Expires: Thu, 01 Jan 1970 00:00:00 GMT
Cache-Control: no-cache, no-store
Location: http://portal.uestc.edu.cn/?ticket=ST-90243-j1gAuBb6dD6WioPIHZAq1507177941379-iVs1-cas
Content-Language: en-US
Set-Cookie: CASPRIVACY=""; Expires=Thu, 01-Dec-94 16:00:00 GMT; Path=/authserver/
iPlanetDirectoryPro=AQIC5wM2LY4SfcwbTApm%2BRJenSML1WX1YbW1ambOYsDS5rI%3D%40AAJTSQACMDE%3D%23; Path=/; Domain=.uestc.edu.cn
CASTGC=TGT-34220-baZa6T6w0cYhtFQmlQMkkhRGRbZbeRBxVI0tEfVyDSVAw6iokR1507177941262-r7bN-cas; Path=/authserver/; HttpOnly
Upgrade-Insecure-Requests: 1
```
其中又有`Set-Cookie`字段，这是用来设置新的`cookie`的信息，而我们来看看其中的`location`字段，这里指向的是重定向的地址，`Location`首部给出的是客户端用来临时定位的资源，为`http://portal.uestc.edu.cn/?ticket=ST-90243-j1gAuBb6dD6WioPIHZAq1507177941379-iVs1-cas`

+ 于是，接下来又是一个重定向，这就是GET方法的重定向，这次请求的url为`http://portal.uestc.edu.cn/?ticket=ST-90243-j1gAuBb6dD6WioPIHZAq1507177941379-iVs1-cas`.
我们看看这次的请求发生了什么。
这一次的请求头为
```
Host: portal.uestc.edu.cn
User-Agent: Mozilla/5.0 (X11; Ubuntu; Linux x86_64; rv:55.0) Gecko/20100101 Firefox/55.0
Accept: text/html,application/xhtml+xml,application/xml;q=0.9,*/*;q=0.8
Accept-Language: zh-CN,zh;q=0.8,en-US;q=0.5,en;q=0.3
Accept-Encoding: gzip, deflate
Referer: http://idas.uestc.edu.cn/authserver/login?service=http%3A%2F%2Fportal.uestc.edu.cn%2F
Cookie: iPlanetDirectoryPro=AQIC5wM2LY4SfcwbTApm%2BRJenSML1WX1YbW1ambOYsDS5rI%3D%40AAJTSQACMDE%3D%23
Connection: keep-alive
Upgrade-Insecure-Requests: 1
```
我们发现这一次，他的`cookie`值变成了上一次响应中`Set-Cookie`的值。而这一次的响应头为:
```
Server: openresty
Date: Thu, 05 Oct 2017 04:32:21 GMT
Content-Type: text/html
Content-Length: 154
Connection: keep-alive
Set-Cookie: MOD_AUTH_CAS=MOD_AUTH_ST-90243-j1gAuBb6dD6WioPIHZAq1507177941379-iVs1-cas; path=/; Httponly
Location: http://portal.uestc.edu.cn/
```

+ 接下来就是请求实际的信息门户的地址,请求头为:
```
Host: portal.uestc.edu.cn
User-Agent: Mozilla/5.0 (X11; Ubuntu; Linux x86_64; rv:55.0) Gecko/20100101 Firefox/55.0
Accept: text/html,application/xhtml+xml,application/xml;q=0.9,*/*;q=0.8
Accept-Language: zh-CN,zh;q=0.8,en-US;q=0.5,en;q=0.3
Accept-Encoding: gzip, deflate
Referer: http://idas.uestc.edu.cn/authserver/login?service=http%3A%2F%2Fportal.uestc.edu.cn%2F
Cookie: iPlanetDirectoryPro=AQIC5wM2LY4SfcwbTApm%2BRJenSML1WX1YbW1ambOYsDS5rI%3D%40AAJTSQACMDE%3D%23; MOD_AUTH_CAS=MOD_AUTH_ST-90243-j1gAuBb6dD6WioPIHZAq1507177941379-iVs1-cas
Connection: keep-alive
Upgrade-Insecure-Requests: 1
```
响应头为:
```
Server: openresty
Date: Thu, 05 Oct 2017 04:32:21 GMT
Content-Type: text/html;charset=UTF-8
Transfer-Encoding: chunked
Connection: keep-alive
Cache-Control: no-cache
Pragma: no-cache
Expires: Thu, 01 Jan 1970 00:00:00 GMT
Content-Language: zh-CN
Set-Cookie: route=bd484519ebc8081db0dadf641f700565
JSESSIONID=0000c1kMIqpEwBqT49FbsOjuZ3U:19sdu3g1i; Path=/
```
其中也带了`Set-Cookie`的信息，接下来请求静态文件等等，都是用的最后一次的`Set-Cookie`里的`cookie`的信息。说明此时就已经完成登录了。至此，整个登录的流程我们就已经知道的很清楚了。

### 代码实现
那写模拟登录的代码就非常轻松咯，这里用到的是`requests`库:

```
# !/usr/bin/python3
# -*- encoding:utf8-*-

import requests
from bs4 import BeautifulSoup

class ImitateLogin:

    def __init__(self,username, password,login_url, headers):
        self.login_url = login_url
        self.headers = headers
        # 用来保存每次请求的cookie
        self.all_cookies = None
        # 你的用户名和密码
        self.post_data = dict(username=username, password=password)

    # 解析所有的input标签
    def parse_input_tags(self, text):
        soup = BeautifulSoup(text, "lxml")
        input_list = soup.find_all("input")
        for input_tag in input_list[2:7]:
            self.post_data[input_tag.get("name")] = input_tag.get("value")

    # 登录页面
    def get_index(self):
        request = requests.get(self.login_url, headers=headers)
        self.all_cookies = request.cookies
        self.parse_input_tags(request.text)
        self.post()

    # 发post表单
    def post(self):
        # print(self.all_cookies)
        post = requests.post(login_url, data=self.post_data,
                            allow_redirects=False, cookies=self.all_cookies)
        post_cookie_dict = requests.utils.dict_from_cookiejar(post.cookies)
        requests.utils.add_dict_to_cookiejar(self.all_cookies, post_cookie_dict)
        # print(post.headers)
        location = post.headers["Location"]
        self.get_redirect_title(location)

    # 第二次重定向
    def get_redirect_title(self, location):
        get_title = requests.get(location, allow_redirects=False,
                                cookies=self.all_cookies)
        get_title_cookies_dict = requests.utils.dict_from_cookiejar(get_title.cookies)
        requests.utils.add_dict_to_cookiejar(self.all_cookies,
                                             get_title_cookies_dict)
        location = get_title.headers["Location"]
        self.get_finall(location)

    # 到主页面
    def get_finall(self, location):
        finall = requests.get(location, allow_redirects=False,
                             cookies=self.all_cookies)
        print(finall.text)


if __name__ == "__main__":
    login_url = "http://idas.uestc.edu.cn/authserver/login?service=http://portal.uestc.edu.cn/index.portal"

    headers = {
      "Host": "idas.uestc.edu.cn",
	    "User-Agent": "Mozilla/5.0 (X11; Ubuntu; Linux x86_64; rv:55.0) Gecko/20100101 Firefox/55.0",
	    "Accept": "text/html,application/xhtml+xml,application/xml;q=0.9,*/*;q=0.8",
	    "Accept-Language": "zh-CN,zh;q=0.8,en-US;q=0.5,en;q=0.3",
	    "Accept-Encoding": "gzip, deflate",
	    "Referer": "http://idas.uestc.edu.cn/authserver/login?service=http://portal.uestc.edu.cn/index.portal",
	    "Connection": "keep-alive",
	    "Upgrade-Insecure-Requests": "1",
	    "Cache-Control": "max-age=0"
}  
    # 传入你的username和password
    login = ImitateLogin(username, password, login_url, headers)
    login.get_index()

    # other_process.......
    do_something()
```
到这里，模拟登录就成功了，但是我们对每个请求都要手动存储一遍cookie是非常麻烦的，所以有更好的方法嘛？

还好`requests`提供了`Session`对象来供轻松地处理`cookie`,代码如下：
```
#! /usr/bin/python3
# -*- encoding:utf8-*-
import requests
from bs4 import BeautifulSoup


class Login:

    def __init__(self, username, password, login_url, headers):
        self.login_url = login_url
        self.headers = headers
        self.session = requests.Session()
        self.post_data = dict(username=username, password=password)

    def get_index(self):
        get_index = self.session.get(self.login_url, headers=headers)
        self.parse_input_tags(get_index.text)
        self.post()

    def parse_input_tags(self, text):
        soup = BeautifulSoup(text, "lxml")
        input_list = soup.find_all("input")
        for input_tag in input_list[2:7]:
            self.post_data[input_tag.get("name")] = input_tag.get("value")

    def post(self):
        post = self.session.post(self.login_url, data=self.post_data)
        print(post.text)


if __name__ == "__main__":

    loginUrl = "http://idas.uestc.edu.cn/authserver/login?service=http://portal.uestc.edu.cn/index.portal"

    headers = {
        "Host": "idas.uestc.edu.cn",
        "User-Agent": "Mozilla/5.0 (X11; Ubuntu; Linux x86_64; rv:55.0) Gecko/20100101 Firefox/55.0",
        "Accept": "text/html,application/xhtml+xml,application/xml;q=0.9,*/*;q=0.8",
        "Accept-Language": "zh-CN,zh;q=0.8,en-US;q=0.5,en;q=0.3",
        "Accept-Encoding": "gzip, deflate",
        "Referer": "http://idas.uestc.edu.cn/authserver/login?service=http://portal.uestc.edu.cn/index.portal",
        "Connection": "keep-alive",
        "Upgrade-Insecure-Requests": "1",
        "Cache-Control": "max-age=0"
    }
    username = "xxxxxx"
    password = "xxxxxx"
    login_obj = Login(username, password, loginUrl, headers)
    login_obj.get_index()

    #do_something_else
    do_something()
```
这就是以上使用`Session`对象的全部代码，是不是简洁多了。在代码中，你根本不用考虑`cookie`的事情。
