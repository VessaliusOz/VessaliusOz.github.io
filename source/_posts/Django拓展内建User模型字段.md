---
title: Django拓展内建User模型字段
date: 2017-07-18 22:38:59
tags: 
  - Django
---
最近准备自己从头到尾写一个有完善功能的、有一定规模的web应用，类似于论坛，又有点像知乎那样的应用。自己的脑海里已经有了这个应用的大概功能和样子，希望自己能在暑假结束的时候，基本完成它。

今天写完了关于用户注册、登录、注销的功能。在开发的过程中，纠结于是自己重新写User模块还是对Django内建的User模块进行字段扩展，最后还是选择了后者。考虑到用Django内建的User模块，在认证这一块已经比较成熟，而且非常方便和安全，它可以处理用户账户、权限、和基于cookie的用户会话。这些正是我需要的功能。不过在实际的扩展过程中，还是遇到了一些坑，所以写下这篇来作为总结。其中参考了许多地方，详细链接见文章最后。
<!--more-->
### 使用OneToOneField(使用Profile扩展User模块)
我见过的最常用的办法就是通过OneToOneField与现有的User模块进行一对一的关联，一般存储与现有用户模型相关而又与验证过程无关的额外信息时。我们通常称之为用户配置(User Profile)。这种方法会额外增加对相关信息的查询和检索。
```python
class UserProfile(models.Model):
    user = models.OneToOneField(User, verbose_name="账户", related_name='user_profile',
                                on_delete=models.SET_NULL, null=True, default=None, unique=True)
    sex = models.IntegerField('性别', choices=sexuality, default=0, null=True)
    logo = models.CharField('一句话介绍自己', default=None, null=True, max_length=200)
    photo = models.ImageField('头像', upload_to='images/user', default=None, null=True, blank=True)
    datetime = models.DateTimeField('加入时间', default=datetime.now)

    def __str__(self):
        return self.user.username
```
当我们在定义一些信号就可以使得当我们创建和更新用户实例时，UserProfile模块也可以被自动创建和更新。代码如下:
```python
@receiver(post_save, sender=User)
def create_user_profile(sender, instance, created, **kwargs):
    if created:
        UserProfile.objects.create(user=instance)


@receiver(post_save, sender=User)
def save_user_profile(sender, instance, **kwargs):
    if instance.user_profile:
        instance.user_profile.save()
```
关于信号的概念，我们下次在讨论。
如此以来，对于User的保存和创建等事件的发生，都会被信号连接到用户模型，这种信号被称为post_save.

还有关于数据库查询的优化问题，这里简单的说一下，Django关联为浅关联。也就是说当进入一个相关的属性时，Django所做的工作也只是查询数据库而已。有时候这会造成一些意想不到的后果，比如像开启成千上万的查询进程。这个问题可以使用select_related方法缓和一些。
当预知将会进入相关数据的查询时，可以使用简单的查询先将它取出来。
```python
users = User.objects.all().select_related('profile')
```
还有一些关于admin的配置，可以使的User与UserProfile一起进行内联编辑:
```python
# -*-encoding:utf8-*-
from django.contrib import admin
from forum.models import *
from django.contrib.auth.models import Group


class UserProfileInline(admin.StackedInline):
    model = UserProfile
    verbose_name = '用户信息'
    verbose_name_plural = '用户信息'
    max_num = 1
    can_delete = False


class UserProfileAdmin(admin.ModelAdmin):
    inlines = (UserProfileInline,)


admin.site.unregister(User)
admin.site.unregister(Group)
admin.site.register(User, UserProfileAdmin)
```
<strong>其中一定要先unregister(User)</strong>，否则会报User模块已经被注册的错误。因为我的项目里用不到Group，所以为了方便，也把Group给unregister了。



### 继承AbstractUser
这种方法相当的直接，因为django.contrib.auth.models.AbstractUser类提供了把默认用户作为抽象模型的整套的实现方案。
```python
from django.db import models
from django.contrib.auth.models import AbstractUser

class User(AbstractUser):
    bio = models.TextField(max_length=500, blank=True)
    location = models.CharField(max_length=30, blank=True)
    birth_date = models.DateField(null=True, blank=True)
```
然后必须更新settings.py,定义AUTH_USER_MODEL属性。
```python
AUTH_USER_MODEL=‘core.User’
```
这种方案的需要在项目开始时就考虑完成并需要额外的维护。它将会改变整个数据库的架构。如果创建外键导入用户模型设置 （from django.conf import settings）而且把引用settings.AUTH_USER_MODEL代替直接引用自定义的用户模型会更好。

详细全文链接见[这里](http://python.jobbole.com/86806/)

### 关于User.create_user() 和　User.create() 的区别和认证对应的表的内容
如果想要创建具有完整认证功能和会话功能的用户，我们应该使用create_user()这个方法
```python
>>>	from	django.contrib.auth.models	import	User
>>>	user = User.objects.create_user('john',	'lennon@thebeatles.c
om',	'johnpassword')
#	At this point, user is a User object that has already been saved
#	to the database. You can continue to change its attributes
# if you want to change other fields.
>>>	user.last_name = 'Lennon'
>>>	user.save()
```
而仅仅使用create()方法创建的时候，我们可以发现，在django admin中，他的密码是以明文的形式存储的，并不如我们用create_user()方法创建时的哈希值。这就是用create()方法创建的用户无法用authenticate()方法认证通过的原因。
这是sqlite里auth_user表的内容：
```sqlite
sqlite> select * from auth_user
   ...> ;
1|pbkdf2_sha256$36000$0NkfnNyK0SV0$5+7ucNUHJnELtn2WIjTww2DVldkJCHNoWclOLv5hZ9g=|2017-07-18 09:17:37.310208|1||||1|1|2017-07-14 14:26:00|vessalius
2|12345678||0|||111@qq.com|0|1|2017-07-17 09:23:00|xc
10|pbkdf2_sha256$36000$IadFEnx7LbTA$vdxLB/JV5Xn0g+WUeX6aLHbZSYY25+PO/ajJ//p3pkI=|2017-07-18 04:28:00|0||||0|1|2017-07-18 03:40:00|xz
```
其中xc是用create()方法创建的，它的密码为12345678，是直接明文写入password字段，而vessalius和xz的密码是加密的。

所以当你使用create()方法创建一个User对象的时候，密码是明文存储，此时可以直接使用get()方法得到这个对象,而用create_user()方法创建的对象无法用get()方法的得到,此时会报错，所以，我们要得到一个User对象，应该使用authenticate()，虽然看起来很复杂，但是它实际上是和数据库里的存储的值是统一的，Django的create_user()和authenticate()方法只是封装了加密解密的细节：
```python
>>>user = User.objects.get(username='xc', password='12345678')
>>>user
<User: xc>
>>>user = User.objects.get(username='vessalius',
password='xc19970113')
django.contrib.auth.models.DoesNotExist: User matching query does not exist.

>>>from django.contrib.auth import authenticate
>>>user = authenticate(username='vessalius', password='xc19970113')
user
<User: vessalius>

```
同样，因为密码的存储是哈希值，所以，我们在修改密码的时候，不能直接尝试修改用户的password属性，而是可以使用set_password()这个方法来更新密码，所以可以发现创建用户用create_user()辅助函数，修改密码用set_password()辅助函数：
```
>>>	from	django.contrib.auth.models	import	User
>>>	u	=	User.objects.get(username='john')
>>>	u.set_password('new	password')
>>>	u.save()
```
关于认证模块的细节我们以后再详细讨论。

### 关于Form
如果我们的表单认证中有关于文件上传的操作时，例子如下：
```python
class UserProfile(models.Model):
    user = models.OneToOneField(User, verbose_name="账户", related_name='user_profile',
                                on_delete=models.SET_NULL, null=True, default=None, unique=True)
    sex = models.IntegerField('性别', choices=sexuality, default=0, null=True)
    logo = models.CharField('一句话介绍自己', default=None, null=True, max_length=200)
    photo = models.ImageField('头像', upload_to='images/user', default=None, null=True, blank=True)
    datetime = models.DateTimeField('加入时间', default=datetime.now)

    def __str__(self):
        return self.user.username


class ModifyProfileForm(forms.ModelForm):
    class Meta:
        model = UserProfile
        fields = ['sex', 'logo', 'photo']


```
若要用form来上传photo字段的图片，需要在表单中添加如下代码,否则request.POST字典为空：
```html
<form method="post" enctype="multipart/form-data">
```
而相应的View为：
```python
@method_decorator(login_required, name='dispatch')
class ModifyProfileView(TemplateView):
    template_name = 'forum/modify.html'
    form_class = ModifyProfileForm

    def get_context_data(self, **kwargs):
        context = super(ModifyProfileView, self).get_context_data(**kwargs)
        context['modify_form'] = self.form_class
        return context

    def get(self, request, *args, **kwargs):
        return render(request, 'forum/modify.html', {'modify_form': self.form_class})

    def post(self, request):
        modify_form = self.form_class(request.POST, request.FILES, instance=request.user.user_profile)
        if modify_form.is_valid():
            print(request.FILES)
            modify_form.save()
            return HttpResponse("modify successfully")
        else:
            return render(request, 'forum/modify.html', {'modify_form': self.form_class})
```
