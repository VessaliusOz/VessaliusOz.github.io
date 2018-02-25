---
title: Django中_meta的运用
date: 2018-02-24 21:31:31
tags:
  - django
  - model
---

# 概要
最近写的项目里面有一个需求：django如何实现动态的给一个model添加字段，一开始
并不知道如何做，因为以前的项目里面model一般都是会在migrate之前就被定义好了的，所以并不涉及到实时的修改，当时还是有点伤脑经的。然后在提出了解决方案以后，我又遇到了一个问题，即如何访问这些动态添加的字段。

<!--more-->

# 动态的添加字段
直到突然地恍然大悟，我们其实可以给每一个django支持的field都建立一个model，然后通过外键关联的形式把每一个基于field的model关联到需要关联的model上面，这样就可以实现动态的给每一个model添加字段了。

其实网上早就有大神讨论了`django dynamic model fields`的情况，详见[django dynamic model field](https://stackoverflow.com/questions/7933596/django-dynamic-model-fields)

其实我觉得这种情况下最好地方案还是要使用`MongoDB`或者`JSON格式来存储`,当时考虑到如果使用这两种方案的话，那就不能使用`django admin`了，当时为了赶进度，以及自己前端技术薄弱(发誓一定好好学前端)，遂舍弃这两种方案。

上面的方案建立的模型如下:

```python
# coding: utf-8
from __future__ import unicode_literals
from django.db import models
from django.utils.datetime_safe import datetime, date


class CommonFramework(models.Model):
    class Meta:
        verbose_name = u'内容'
        verbose_name_plural = verbose_name
    weight = models.DecimalField(u'权重', max_digits=3, decimal_places=0, default=999,
                                 help_text=u'取值范围0-999，用于排序。权重越大排序越靠前.')
    create_time = models.DateTimeField(u'创建时间', default=datetime.now, null=True)
    until_time = models.DateTimeField(u'到期时间', default=datetime.now, null=True)
    title = models.CharField(u'标题', max_length=50, default=None, null=True)
    author = models.CharField(u'作者', max_length=10, default=None, null=True, blank=True)
    source = models.CharField(u'来源', max_length=10, default=None, null=True, blank=True)
    source_addr = models.URLField(u'来源地址', default=None, null=True, blank=True)
    content = UEditorField(u'内容', width=900, height=800, imagePath="contents/", filePath="contents/", default="")
    view_num = models.IntegerField(u'浏览次数', default=0, null=True)
    abstract = models.CharField(u'摘要', max_length=20, default=None, null=True, blank=True)
    template = models.ForeignKey(Template, verbose_name=u'模板', default=None, null=True, on_delete=models.SET_NULL)
    tag = models.ManyToManyField(Tag, verbose_name=u'标签', default=None, null=True)
    topic = models.ForeignKey(Topic, verbose_name=u'话题', default=None, null=True, on_delete=models.SET_NULL)
    category = models.ForeignKey(Category, default=None, null=True, verbose_name=u'目录', on_delete=models.SET_NULL)

    def __unicode__(self):
        return self.title


class StringModel(models.Model):
    class Meta:
        verbose_name = u'字符串框'
        verbose_name_plural = verbose_name
    label = models.CharField(u'标识', max_length=30, default=None, null=True, help_text='模板标识')
    content = models.CharField(u'内容', max_length=100, default=None, null=True, help_text='内容')
    common_framework = models.ForeignKey(CommonFramework, verbose_name='关联内容', default=None,
                                         null=True, on_delete=models.CASCADE)

    def __unicode__(self):
        return self.label
 

class IntModel(models.Model):
    class Meta:
        verbose_name = u'数字字段'
        verbose_name_plural = verbose_name
    label = models.CharField(u'标识', max_length=30, default=None, null=True, help_text='模板标识')
    content = models.IntegerField(u'内容', default=0)
    common_framework = models.ForeignKey(CommonFramework, verbose_name='关联内容', default=None,
                                         null=True, on_delete=models.CASCADE)

    def __unicode__(self):
        return self.label


class BooleanModel(models.Model):
    class Meta:
        verbose_name = u'布尔值字段'
        verbose_name_plural = verbose_name
    label = models.CharField(u'标识', max_length=30, default=None, null=True, help_text='模板标识')
    content = models.BooleanField(u'内容', default=True)
    common_framework = models.ForeignKey(CommonFramework, verbose_name='关联内容', default=None,
                                         null=True, on_delete=models.CASCADE)

    def __unicode__(self):
        return self.label


class DateModel(models.Model):
    class Meta:
        verbose_name = u'日期字段'
        verbose_name_plural = verbose_name

    label = models.CharField(u'标识', max_length=30, default=None, null=True, help_text='模板标识')
    content = models.DateField(u'日期', default=date.today)
    common_framework = models.ForeignKey(CommonFramework, verbose_name='关联内容', default=None,
                                         null=True, on_delete=models.CASCADE)

    def __unicode__(self):
        return self.label


class DatetimeModel(models.Model):
    class Meta:
        verbose_name = u'时间字段'
        verbose_name_plural = verbose_name

    label = models.CharField(u'标识', max_length=30, default=None, null=True, help_text='模板标识')
    content = models.DateTimeField(u'时间', default=datetime.now)
    common_framework = models.ForeignKey(CommonFramework, verbose_name='关联内容', default=None,
                                         null=True, on_delete=models.CASCADE)

    def __unicode__(self):
        return self.label
 
...
```
其中还省略了一些`model`。

# 访问动态添加的字段
因为这些外键的字段是人为的与`CommonFramework`关联的，所有我们事先就不能够得知有哪些字段会与`CommonFramework model`建立关联的关系，而且因为`CommonFramework model`本身就具有很多的属性，所以如果我们还像以前那样:

```python
# just an example 
def a_view(request, pk):
	common_content = get_object_or_404(CommonFramework, pk=int(pk))
    weight = common_content.weight
    title = common_content.title
    ...
    rel_string = common_content.stringmodel_set.all()
    rel_int = common_content.intmodel_set.all()
    ...
```
这样一个属性一个属性的去访问，去判断是否存在简直是不能忍受的，关键是这样写出的代码不仅代码量大，而且非常的丑陋。而且因为我们事先不知道到底有多少种外联的到`CommonFramework`模型的字段模型，所以我们并不能通过一般的方法再去访问这些`{model_field_name}_set`。

那么到底应该如何解决这个问题呢？我google了一下发现官网上果然提供了访问一个模型字段属性的API。(想着那些框架的开发者肯定会比我先认识到这个问题吧，不然只能一个属性一个属性去访问岂不是太傻了😂😂😂)

## 初始 _meta
这里就要先提出`_meta`这个概念了，官方文档上可是专门列出了一个专题来将`_meta`的用法。详情请戳这里👉 [Model _meta API](https://docs.djangoproject.com/en/2.0/ref/models/meta/)

我们先来看看使用`_meta`之后的代码吧:

```python
def render_content(request, c_type, pk):
    category = get_object_or_404(Category, label=c_type)
    try:
        content = category.commonframework_set.get(pk=int(pk))
    except CommonFramework.DoesNotExist:
        raise Http404("source doesn't exist")
    context = {}

    # 这里不包含多对多的field, 包含关联的外键
    field_names = [f_name.name for f_name in CommonFramework._meta.fields]
    for field in field_names:
        # 如果是外键
        if isinstance(getattr(content, field, None), models.Model):
            context[field] = getattr(content, field, None).name
        # 普通字段
        else:
            context[field] = getattr(content, field, None)

    # 处理多对多的tag
    context['tag'] = []
    for tag in content.tag.all():
        context['tag'].append(tag.name)

    # 处理外联到commonframework的各种类型的字段
    related_field = [m2m_field.name for m2m_field in content._meta.get_all_related_objects()]
    for rel_field in related_field:
        if hasattr(content, '{}_set'.format(rel_field)):
            rel_field_set = getattr(content, '{}_set'.format(rel_field)).all()
            if rel_field != 'imagemodel':
                for item in rel_field_set:
                    context[item.label] = item.content
            else:
                for item in rel_field_set:
                    context[item.label] = item.content.url

    # get the template
    template_src = 'CMSAPP/{}/index.html'.format(content.template.name)

    context.update(info_dict)
    return render(request, template_src, context)

```
我们先来看看第一段代码：

```python
field_names = [f_name.name for f_name in CommonFramework._meta.fields]
    for field in field_names:
        # 如果是外键
        if isinstance(getattr(content, field, None), models.Model):
            context[field] = getattr(content, field, None).name
        # 普通字段
        else:
            context[field] = getattr(content, field, None)
```
### 一般字段和外键字段的访问
这里我们通过`Model._meta.fields`来得到`CommonFramework`所有`field`属性的实例,包括`<class 'django.db.models.fields.XXXField'>`和`<django.db.models.fields.related.ForeignKey`(需要注意的是这里不包括多对多的字段)，然后通过`name`属性得到字段的名称。然后再结合`for`循环和`getattr()`函数就可以巧妙地遍历到字段的值。

因为在本例中`CommonFramework`外键关联的字段包括`Category`, `templates`等，而我们真正想得到的是这些这些外键关联属性的`name`属性，所以就加了一个判断:

```python
for field in field_names:
        # 如果是外键
        if isinstance(getattr(content, field, None), models.Model):
            context[field] = getattr(content, field, None).name
        # 普通字段
        else:
            context[field] = getattr(content, field, None)
```

以上完成了`model`的字段属性的访问，但是我们还有多对多字段和外键关联到`CommonFramework`的字段(may_to_one)没有访问。

### 多对多关系字段的访问
本例中因为多对多关系只有一个，所以就是按照最常规的方法来访问的。但是_meta也能提供更便捷的方法来访问。比如可以通过`many_to_many`属性来访问到所有多对多字段的实例(`<django.db.models.fields.related.ManyToManyField: xxx>`)，然后通过通过for循环遍历:

```python
content = MyModel.objects.get(pk=1)
for rel_m2m_model in  MyModel._meta.many_to_many:
    m2m_field_manager = getattr(content, rel_m2m_model.name)
    for item in m2m_field_manager.all():
        do_something()
```
### `<ManyToOneRel>`的访问(关联到CommonFramework上的model)
可以通过`get_all_related_objects()`来访问到所有`ManyToOneRel`的实例，然后通过`name`属性访问到对应的名称。

```python
    related_field = [m2m_field.name for m2m_field in content._meta.get_all_related_objects()]
    for rel_field in related_field:
        if hasattr(content, '{}_set'.format(rel_field)):
            rel_field_set = getattr(content, '{}_set'.format(rel_field)).all()
            if rel_field != 'imagemodel':
                for item in rel_field_set:
                    context[item.label] = item.content
            else:
                for item in rel_field_set:
                    context[item.label] = item.content.url
```
然后通过`format()`和`getattr()`访问到对应的集合。

## _meta office API
因为这个项目我用的是`django1.9`，其中`_meta`有些非官方的API将会自`django 1.10`舍弃掉(比如`get_all_related_objects()`)，所以还是总结一下`_meta`的官方的`API`吧。

### get_field(field\_name)
按名称检索模型的单个字段实例，field_name可以是模型上字段的名称，抽象或继承模型上的字段，或指向模型的另一个模型上定义的字段。 在后一种情况下，field_name将是由用户定义的related_name或由Django自身自动生成的名称。

```python
>>> from django.contrib.auth.models import User

# A field on the model
>>> User._meta.get_field('username')
<django.db.models.fields.CharField: username>

# A field from another model that has a relation with the current model
>>> User._meta.get_field('logentry')
<ManyToOneRel: admin.logentry>

# A non existent field
>>> User._meta.get_field('does_not_exist')
Traceback (most recent call last):
    ...
FieldDoesNotExist: User has no field named 'does_not_exist'
```

### get_fields(include\_parents=True, include\_hidden=False)
检索一个`model`的所有字段实例:

* include_parents

	默认情况下为True。 递归地包含在父类上定义的字段。 如果设置为False，get_fields（）将仅搜索直接在当前模型上声明的字段。 直接从抽象模型或代理类继承的模型字段被认为是本地的，而不是父类。
* include_hidden

	默认为False。 如果设置为True，get_fields（）将包含用于支持其他字段功能的字段。 这还将包括具有以“+”开头的related_name的任何字段（如ManyToManyField或ForeignKey）。
	
```
>>> pprint(CommonFramework._meta.get_fields())
(<ManyToOneRel: CMSAPP.stringmodel>,
 <ManyToOneRel: CMSAPP.textmodel>,
 <ManyToOneRel: CMSAPP.intmodel>,
 <ManyToOneRel: CMSAPP.booleanmodel>,
 <ManyToOneRel: CMSAPP.datemodel>,
 <ManyToOneRel: CMSAPP.datetimemodel>,
 <ManyToOneRel: CMSAPP.emailmodel>,
 <ManyToOneRel: CMSAPP.imagemodel>,
 <ManyToOneRel: CMSAPP.urlmodel>,
 <django.db.models.fields.AutoField: id>,
 <django.db.models.fields.DecimalField: weight>,
 <django.db.models.fields.DateTimeField: create_time>,
 <django.db.models.fields.DateTimeField: until_time>,
 <django.db.models.fields.CharField: title>,
 <django.db.models.fields.CharField: author>,
 <django.db.models.fields.CharField: source>,
 <django.db.models.fields.URLField: source_addr>,
 <DjangoUeditor.models.UEditorField: content>,
 <django.db.models.fields.IntegerField: view_num>,
 <django.db.models.fields.CharField: abstract>,
 <django.db.models.fields.related.ForeignKey: template>,
 <django.db.models.fields.related.ForeignKey: topic>,
 <django.db.models.fields.related.ForeignKey: category>,
 <django.db.models.fields.related.ManyToManyField: tag>,
 <django.contrib.contenttypes.fields.GenericRelation: eav_values>)
```

### 运用get_field()和get\_fields()来实现旧的API
虽然在`django1.9`里面的API被移除了，但是我们可以通过get_field以及get\_fields来实现之前的API

 * 通过get_fields()得到所有的字段之后，利用[字段属性](https://docs.djangoproject.com/en/2.0/ref/models/fields/)来过滤所需要的字段

下面举几个常用的例子，更多的可以去看👉[官方文档](https://docs.djangoproject.com/en/2.0/ref/models/meta/)

* __`MyModel._meta.get_fields_with_model()`__ 为:

```python
[
    (f, f.model if f.model != MyModel else None)
    for f in MyModel._meta.get_fields()
    if not f.is_relation
        or f.one_to_one
        or (f.many_to_one and f.related_model)
]
```

* __`MyModel._meta.get_concrete_fields_with_model()`__ 为:

```python
[
    (f, f.model if f.model != MyModel else None)
    for f in MyModel._meta.get_fields()
    if f.concrete and (
        not f.is_relation
        or f.one_to_one
        or (f.many_to_one and f.related_model)
    )
]
```

* __`MyModel._meta.get_m2m_with_model()`__  **有用！！！** :

```python
[
    (f, f.model if f.model != MyModel else None)
    for f in MyModel._meta.get_fields()
    if f.many_to_many and not f.auto_created
]
```

* __`MyModel._meta.get_all_related_objects()`__ __ **有用！！！** :

```python
[
    f for f in MyModel._meta.get_fields()
    if (f.one_to_many or f.one_to_one)
    and f.auto_created and not f.concrete
]
```

* __`MyModel._meta.get_all_related_objects_with_model()`__ **有用！！！** :

```python
[
    (f, f.model if f.model != MyModel else None)
    for f in MyModel._meta.get_fields()
    if (f.one_to_many or f.one_to_one)
    and f.auto_created and not f.concrete
]
```

* __`MyModel._meta.get_all_related_many_to_many_objects()`__ 为:

```python
[
    f for f in MyModel._meta.get_fields(include_hidden=True)
    if f.many_to_many and f.auto_created
]
```

* __`MyModel._meta.get_all_related_m2m_objects_with_model()`__ 为:

```python
[
    (f, f.model if f.model != MyModel else None)
    for f in MyModel._meta.get_fields(include_hidden=True)
    if f.many_to_many and f.auto_created
]
```

* __`MyModel._meta.get_all_field_names()`__ 为:

```python
from itertools import chain
list(set(chain.from_iterable(
    (field.name, field.attname) if hasattr(field, 'attname') else (field.name,)
    for field in MyModel._meta.get_fields()
    # For complete backwards compatibility, you may want to exclude
    # GenericForeignKey from the results.
    if not (field.many_to_one and field.related_model is None)
)))
```

# 总结
为自己以前写的一个属性一个属性去判断去访问的代码感到有点难过😑😑😑，所以赶紧用起来吧。



