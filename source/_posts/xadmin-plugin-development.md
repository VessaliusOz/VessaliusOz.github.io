---
title: xadmin插件开发
date: 2018-02-23 09:18:44
tags:
  - xadmin
  - django
---

# 概述
由于项目需要，最近一直在鼓捣xadmin，xadmin与原有的admin相比，有非常多的优点，不仅界面样式变得更加好看，系统还预置了非常好用的插件。而且插件还是可以自己开发的，总而言之，就是与原有的admin相比，更加好看，功能也更多。

xadmin提供了许多与原有admin相同的API,所以上手并不会那么的难。但还是有一些改变的地方。需要注意。

# 预备知识
xadmin官方文档对插件系统的描述如下:
> 想要了解 Xadmin 的插件系统架构首先需要了解 Xadmin AdminView 的概念。 简单来说，就是 Xadmin 系统中每一个页面都是一个 AdminView 对象返回的 HttpResponse 结果。Xadmin 的插件系统做的事情其实就是在 AdminView 运行过程中改变其执行的逻辑， 或是改变其返回的结果，起到修改或增强原有功能的效果。

在实际运用的过程中，我们可以制作插件来给页面添加新的布局、按钮等，来为页面添加更多自定义功能。

<!--more-->
# 内置插件
### Action
`action`的概念在admin中已经存在了，主要用来对批量操作进行自定义功能。admin和xadmin只内置了一个批量删除的功能。所以想要实现更多的功能，我们可以自己开发实现相应的`action`。开发`action`主要需要指定三个属性，`action_name`用来唯一标识这个`action`, `description`用来表示显示在action菜单中的名称， `model_perm`表示该action所需要的权限。

而后再实现`do_action`方法，该方法接受`queryset`参数，用来表示已经选择的数据的`queryset`。
```
from xadmin.plugins.actions import BaseActionView


class StatusReverseAction(BaseActionView):
    action_name = u"reverse_active"
    description = u'将激活状态变为相反'
    model_perm = 'change'

    def do_action(self, queryset):
        for obj in queryset:
            obj.active = not obj.active
            obj.save()


class StatusOffAction(BaseActionView):
    action_name = u"turn_active_to_false"
    description = u"所有状态未激活"
    model_perm = 'change'

    def do_action(self, queryset):
        queryset.update(active=False)


class StatusOnAction(BaseActionView):
    action_name = u'turn_active_to_true'
    description = u'所有状态激活'
    model_perm = 'change'

    def do_action(self, queryset):
        queryset.update(active=True)
```
然后将这些action通过`actions`参数注册到相应的`model option class`中即可：
```
# admin.py

class ActivityAdmin(object):
    actions = [StatusReverseAction, StatusOffAction, StatusOnAction]
    ...

xadmin.site.register(Activity, ActivityAdmin)
```

### data fliter
用于对表级实例进行数据查询
```
class ActivityAdmin(object):
    list_filter = ('title', 'start_date', 'end_date', 'active')
    search_fields = ('title', 'start_date', 'end_date', 'active')
```
只需要在相应的`model Option class`中指定`list_filter`和`search_fields`属性就可以了。`list_filter`用来指定需要过滤的字段，`search_fields`用来指定可以在搜索框中被模糊匹配的字段。

### chat plugin
用于数据的可视化图标展示

只需要给相应的`model option class`设置`data_charts`属性，这是一个字典。`key`用来表示图标的名称，值用来表示相应的配置。例如用来展示浏览次数的变化图:

```
class ArticleAdmin(object):
    list_editable = ['status', 'post_time', 'title', 'view_num']
    data_charts = {
        u'浏览次数': {
            'title': 'view_num', 'x-field': 'online_time', 'y-field': ('view_num',),
            },
        }
xadmin.site.register(Article, ArticleAdmin)
```
其中`y-field`可以设置多个值，用来在一张图中展示多个变化曲线。

### 细节展示
用来在`ListAdminView`中展示属性的细节。只需要在`model option class`中设置`show_detail_fields`属性即可。
要注意的是`show_detail_fields`的值应该是`list_display`的子集。


### 即时编辑
用来在`ListAdminView`中即时编辑属性。只需要在`model option class`中设置`list_editable`属性即可。
要注意的是`list_editable`的值应该是`list_display`的子集。

### book mark
用来记录经过特定的排序或者过滤处理的表，实质上是对表的一种查询处理，类似于视图。
如果需要使用，需要设置`model option class`的`show_bookmarks`属性和`list_bookmarks`属性。比如显示快到期的活动：
```
class ActivityAdmin(object):
    show_bookmarks = True
    list_bookmarks = [{
        'title': u'快到期的活动',
        'query': {'active': True},
        'order': ('end_date',),
        'cols': ('title', 'end_date', 'desc')
    }]

# list_bookmarks的设置规则如下
'''
list_bookmarks = [{
        'title': "xxx",         # 书签的名称, 显示在书签菜单中
        'query': {'active': True}, # 过滤参数, 是标准的 queryset 过滤
        'order': ('-age'),         # 排序参数
        'cols': ('first_name', 'age', 'phones'),  # 显示的列
        'search': ''    # 搜索参数, 指定搜索的内容
        }, {...}
    ]
'''    
```

# 自定义插件开发

### 基本知识
编写xadmin插件的流程其实不是很难，我们还是先来重温一下官网上面对xadmin插件系统架构的解释:
> 想要了解 Xadmin 的插件系统架构首先需要了解 Xadmin AdminView 的概念。 简单来说，就是 Xadmin 系统中每一个页面都是一个 AdminView 对象返回的 HttpResponse 结果。Xadmin 的插件系统做的事情其实就是在 AdminView 运行过程中改变其执行的逻辑， 或是改变其返回的结果，起到修改或增强原有功能的效果。

所以我们需要做的就是编写xadmin的插件，在将其注册启用即可：
```
from xadmin.views import BaseAdminPlugin

class HelloWorldPlugin(BaseAdminPlugin):
    ...

xadmin.site.register_plugin(HelloWorldPlugin, ListAdminView)
```
#### xadmin启用属性的方法
当将插件注册到特定的`AdminView`里面的之后，xadmin创建相应的`AdminView`实例的时候，会把该插件放入到实例的`plugins`属性中，当`AdminView`在处理请求的时候，会逐个调用`plugins`中插件的**`init_request()`**方法，插件在该方法中一般进行初始化的操作并且返回一个 `Boolean` 值告诉`AdminView`是否需要加载该插件。

我们可以先看看`xadmin`自带的即时编辑插件的源码：
```
# editable.py
import Xadmin

class EditablePlugin(BaseAdminPlugin):

    list_editable = []

    def __init__(self, admin_view):
        super(EditablePlugin, self).__init__(admin_view)
        self.editable_need_fields = {}

    def init_request(self, *args, **kwargs):
        active = bool(self.request.method == 'GET' and self.admin_view.has_change_permission() and self.list_editable)
        if active:
            self.model_form = self.get_model_view(ModelFormAdminUtil, self.model).form_obj
        return active

xadmin.site.register_plugin(EditablePlugin, ListAdminView)

# admin.py
class ActivityAdmin(object):
    list_display = ('title', 'start_date', 'end_date', 'img', 'active', 'go_to')
    actions = [StatusReverseAction, StatusOffAction, StatusOnAction]
    list_editable = ['title', 'active', 'img']

xadmin.site.register(Activity, ActivityAdmin)

```
Xadmin在创建插件实例的时候会将`OptionClass`的同名属性替换插件的属性。例如在本例中，`ActivityAdmin`在被注册以后，它的`list_editable = ['title', 'active', 'img']`属性会被用来替换掉`AdminView`实例中的`list_editable = []`属性，这样，在调用`init_request()`方法的时候，`active`的值变为`True`， 此时加载该插件，而没有在`OptionClass`中设置`list_editable`属性时，`active`的值变为`False`，就不加载该插件,以此来达到是否加载特定插件的效果。

#### xadmin插件作用流程
在`AdminView`的执行过程中，可以被插件截获或修改的方法都被`filter_hook()`装饰器装饰，使用 `filter_hook()`装饰的方法在执行过程中会根据一定原则执行插件中的同名方法。执行使用了该装饰器的方法时，会按照以下过程执行:

1.首先将实例的` plugins `属性取出，取出含有同样方法名的插件

2.按照插件方法的`priority`属性排序

3.顺序执行插件方法，执行插件方法的规则:

   * 如果插件方法没有参数, `AdminView`方法的返回结果不为空则抛出异常

   * 如果插件方法的第一个参数为`__`，则`AdminView`方法将作为第一个参数传入，注意，这时还未执行该方法， 在插件中可以通过`__()`执行，这样就可以实现插件在`AdminView`方法执行前实现一些自己的逻辑，例如:
```
def get_context(self, __):
    c = {'key': 'value'}
    c.update(__())
    return c
```
   * 如果插件方法的第一个参数不为 `__` ，则执行`AdminView`方法，将结果作为第一个参数传入

4.最终将插件顺序执行的结果返回

例如，我们还是可以看看xadmin系统自带的即时编辑插件的例子：
```
class EditablePlugin(BaseAdminPlugin):

    list_editable = []

    def __init__(self, admin_view):
        super(EditablePlugin, self).__init__(admin_view)
        self.editable_need_fields = {}

    def init_request(self, *args, **kwargs):
        active = bool(self.request.method == 'GET' and self.admin_view.has_change_permission() and self.list_editable)
        if active:
            self.model_form = self.get_model_view(ModelFormAdminUtil, self.model).form_obj
        return active

    def result_item(self, item, obj, field_name, row):
        if self.list_editable and item.field and item.field.editable and (field_name in self.list_editable):            
            pk = getattr(obj, obj._meta.pk.attname)
            field_label = label_for_field(field_name, obj,
                                          model_admin=self.admin_view,
                                          return_attr=False
                                          )

            item.wraps.insert(0, '<span class="editable-field">%s</span>')
            item.btns.append((
                '<a class="editable-handler" title="%s" data-editable-field="%s" data-editable-loadurl="%s">' +
                '<i class="fa fa-edit"></i></a>') %
                (_(u"Enter %s") % field_label, field_name, self.admin_view.model_admin_url('patch', pk) +
                 '?fields=' + field_name))

            if field_name not in self.editable_need_fields:
                self.editable_need_fields[field_name] = item.field
        return item

    # Media
    def get_media(self, media):
        if self.editable_need_fields:
            media = media + self.model_form.media + \
                self.vendor(
                    'xadmin.plugin.editable.js', 'xadmin.widget.editable.css')
        return media

xadmin.site.register_plugin(EditablePlugin, ListAdminView)
```
在这个插件中，重写了`ListAdminView`的`result_item()`方法以及`get_media()`方法。 `ListAdminView`本身的`result_item()`的函数签名为`result_item(self, obj, field_name, row)`, 而在这个插件中，函数签名为`result_item(self, item, obj, field_name, row)`, 多了一个`item`参数，这里就对应了上述步骤3的第三个规则，当第一个参数不为`__`的时候，将`AdminView`的执行结果作为第一个参数传递给插件中的重新实现的方法，`get_media`方法同理。


##### block_xxx(self, context, node)
在插件编写的过程中，还有一些方法需要注意。这些方法都是以`block_`开头，他们的作用主要是更改`templates`中
HTML片段的加载。
1. context 即为 TemplateContext， nodes 参数包含了其他插件的返回内容。
你可以直接返回HTML片段，或是将内容加入到nodes参数中
2. Xadmin 中的模板代码中的view_block Tag 即为插件的 插入点 。插件可以在自己的插件类中使用 block_ + 插入点名称 方法将 HTML 片段插入到页面的这个位置

至于`block_xxx`所代表的位置在哪，请自行查看`xadmin`的[源码](https://github.com/sshwsfc/xadmin/tree/master/xadmin)
   使用例子如下：
   ```
    def block_nav_menu(self, context, nodes):
        context.update({
            'options': ['intall', 'unintall']
        })
        nodes.append(loader.render_to_string('path/to/render/template', context=context))
   ```
上面的这个实现，会在{ % view_block 'nav_menu' % } 所在的位置，插入用context渲染`path/to/render/template`后的内容。

以上就是最通用的xadmin插件编写的方法，被`filter_hook()`装饰的方法以及对应的用途可以参考官方文档里给出的[API](https://xadmin.readthedocs.io/en/latest/views_api.html#)

#### xadmin views
上面提到了xadmin通用的插件制作的方法，还有一个需要知道的概念是`AdminView`。Xadmin系统中每一个页面都是一个`AdminView`对象返回的`HttpResponse`结果。例如最常见的每个`Model`的实例表页面其实本质上就是一个`ListAdminView`返回的结果。因而我们需要知道每一个`AdminView`所代表的含义以及应该将插件注册到哪一个`AdminView`上。下面是xadmin中常用的`AdminView`:

* BaseAdminView

  所有`AdminView`的基类，注册在该View上的插件将会影响所有的`AdminView`

* CommonAdminView

  用户已经登陆后显示的`View`，也是所有登陆后`View`的基础类。该`View`主要作用是创建了 Xadmin 的通用元素，例如：系统菜单，用户信息等。插件可以通过注册该`View`来修改这些信息。

* ModelAdminView

  基于`Model`的`AdminView`的基类，注册的插件可以影响所有基于`Model`的`View`。

* ListAdminView

  `Model`列表页面 View。

* ModelFormAdminView

  `Model`编辑页面 View。

* CreateAdminView

  `Model`创建页面 View。

* UpdateAdminView

  `Model`修改页面 View。

* DeleteAdminView

  `Model`删除页面 View。

* DetailAdminView

  `Model`详情页面 View。


# 自定义插件实例
下面是关于xadmin的插件开发的实例，xadmin可以说功能是非常的强大了，但是它本身并没有集成富文本编辑器，而富文本编辑器在现代web应用中可以说是必不可少的东西，所以我们来实现xadmin集成Ueditor的插件。

根据以上的xadmin插件编写的方法，我们的插件代码如下:
```
# coding: utf-8
import xadmin
from xadmin.views import BaseAdminPlugin, CreateAdminView,  UpdateAdminView
from django.conf import settings
from DjangoUeditor.models import UEditorField
from DjangoUeditor.widgets import UEditorWidget


class XadminUeditorWidget(UEditorWidget):
    def __init__(self, **kwargs):
        self.ueditor_options = kwargs
        self.Media.js = None
        super(XadminUeditorWidget, self).__init__(kwargs)


class UeditorPlugin(BaseAdminPlugin):
    def get_field_style(self, attrs, db_field, style, **kwargs):
        if style == 'ueditor':
            if isinstance(db_field, UEditorField):
                widget = db_field.formfield().widget
                param = {}
                param.update(widget.ueditor_settings)
                param.update(widget.attrs)
                return {'widget': XadminUeditorWidget(**param)}
        return attrs

    def block_extrahead(self, context, nodes):
        js = '<script type="text/javascript" src="%s"></script>' % (settings.STATIC_URL + "ueditor/ueditor.config.js")
        js += '<script type="text/javascript" src="%s"></script>' % (settings.STATIC_URL + "ueditor/ueditor.all.js")
        nodes.append(js)


xadmin.site.register_plugin(UeditorPlugin, UpdateAdminView)
xadmin.site.register_plugin(UeditorPlugin, CreateAdminView)
```
这里重写了`ModelFormAdminView`的`get_field_style()`方法(`UpdateAdminView`和`CreateAdminView`都继承自`ModelFormAdminView`)，`get_style_form`方法用来返回`model`编辑页面的样式，当`style`参数为`ueditor`的时候，
就使用`UEditorField`中的`widget`来替换xadmin自带的`widget`。然后还重写了`block_extrahead()`方法。这里会在所有{ % view_block 'extrahead' % }中插入`ueditor`文件夹下的JS文件。

然后只需要再将`UeditorPlugin`注册到xadmin的插件中去就好了。
