---
title: MongoEngine初探
date: 2017-07-14 14:30:07
tags: 
  - MongoEngine
  - Django
  - MongoDB
---
在实际的应用过程中，有一些场合需要我们使用NoSQL(即非关系型数据库)，而最常用的NoSQL包括键存储数据库。其中就有我们今天要使用的MongoDB.

众所周知，因为Django使用的是ORM，即对象关系映射，它用来把对象模型表示的对象映射到基于SQL的关系模型数据库结构中去。当我们对数据进行了抽象和封装以后，只需简单的操作对象的属性和方法，而不用再去和复杂的SQL语句打交道。ORM相当于一座桥梁，前台的对象型数据和数据库中的关系型的数据通过这个桥梁来相互转化。

当我们操作关系型数据库时，Django的ORM为我们提供了非常多的便利。而MongoEngine也提供了类似的API,它是MongoDB的一个ODM框架，它提供了类似于Django中ORM的语法来操作mongoDB数据库。当你有了之前操作关系型数据库的经历，使用MongoEngine还是非常容易上手的。

以下是我在学习时的一点笔记和对原英文文档的翻译。这篇文章又臭又长，权当参考吧。
<!--more-->
### 定义document

在MongoDB中，文档大致相当于RDBMS中的一行。 在使用关系数据库时，行存储在表中，这些行具有行遵循的严格模式。 MongoDB将文档存储在集合而不是表中 - 主要区别在于数据库级别不执行模式。

### 定义文档的模式
MongoEngine允许您为文档定义模式，因为这有助于减少编码错误，并允许在可能存在的字段上定义实用程序方法。
要定义文档的模式，请创建一个继承自Document的类。 通过将字段对象作为类属性添加到文档类来指定字段：

```
from mongoengine import *
import datetime

class Page(Document):
    title = StringField(max_length=200, required=True)
    date_modified = DateTimeField(default=datetime.datetime.now)
```

文档根据其字段顺序进行序列化。MongoDB的优点之一是集合的动态模式，在一些需要动态/扩展样式文档的场景中，数据应该被计划和组织（所有显式都比隐含的更好！）。

###### DynamicDocument
DynamicDocument文档以与Document相同的方式工作，但将保存为其设置的任何数据/属性

```
from mongoengine import *

class Page(DynamicDocument):
    title = StringField(max_length=200, required=True)

# Create a new page and add tags
>>> page = Page(title='Using MongoEngine')
>>> page.tags = ['mongodb', 'mongoengine']
>>> page.save()

>>> Page.objects(tags='mongoengine').count()
>>> 1
```

### Fields
默认情况下，不需要字段值。 要使字段成为强制性字段，请将字段的required关键字参数设置为True。 字段也可能具有可用的验证约束（例如上例中的max_length）。

#### field arguments
每个字段类型都可以通过关键字参数自定义。 可以在所有字段上设置以下关键字参数：
###### db_field (Default: None)
NongoDB的字段名
###### required(default:False)
是否为强制必须
###### default(Default=None)
设置默认值

定义默认值遵循python的一般规则，这意味着在处理可变对象的时候我们应该小心。(比如ListField和DictField)

```
class ExampleFirst(Document):
    # Default an empty list
    values = ListField(IntField(), default=list)

class ExampleSecond(Document):
    # Default a set of values
    values = ListField(IntField(), default=lambda: [1,2,3])

class ExampleDangerous(Document):
    # This can make an .append call to  add values to the default (and all the following objects),
    # instead to just an object
    values = ListField(IntField(), default=[1,2,3])
```
取消设置默认值的字段将恢复为默认值。

###### unique(default:False)
为True时，对于这个字段，集合中的所有文档都不会有相同的值
###### unique_with(None)
与该字段一起使用的字段名称（或字段名称列表）将不会在集合中具有相同值的两个文档。
###### primary_key(default:False)
当为True时，将此字段用作集合的主键。 DictField和Embedded Documents都支持文档作为主键。

###### choices (Default: None)
```
# An iterable (e.g. list, tuple or set) of choices to which the value of this field should be limited.
# Can be either be a nested tuples of value (stored in mongo) and a human readable key

SIZE = (('S', 'Small'),
        ('M', 'Medium'),
        ('L', 'Large'),
        ('XL', 'Extra Large'),
        ('XXL', 'Extra Extra Large'))


class Shirt(Document):
    size = StringField(max_length=3, choices=SIZE)

# Or a flat iterable just containing values

SIZE = ('S', 'M', 'L', 'XL', 'XXL')

class Shirt(Document):
    size = StringField(max_length=3, choices=SIZE)
```

###### **kwargs (Optional)
你可以提供额外的元数据作为任意的其他关键字参数。 但是，您不能覆盖现有属性。 常见的选择包括help_text和verbose_name，通常由窗体和窗口小部件库使用。

#### 以下是一些字段

###### Listfields
MongoDB允许存储项目列表。 要将项目列表添加到文档，请使用ListField字段类型。 ListField将另一个字段对象作为其第一个参数，它指定可以在列表中存储哪些类型的元素：
class Page(Document):
    tags = ListField(StringField(max_length=50))


###### Embedded documents(嵌套的文档)
MongoDB能够在其他文档中嵌入文档。 可以为这些嵌入式文档定义模式，就像常规文档一样。 要创建嵌入式文档，只需像往常一样定义一个文档，而是继承自EmbeddedDocument而不是Document：
```
class Comment(EmbeddedDocument):
    content = StringField()
```

要将文档嵌入到另一个文档中，请使用EmbeddedDocumentField字段类型，将嵌入的文档类作为第一个参数：
```
class Page(Document):
    comments = ListField(EmbeddedDocumentField(Comment))

comment1 = Comment(content='Good work!')
comment2 = Comment(content='Nice article!')
page = Page(comments=[comment1, comment2])
```

###### Dictionary Fields
通常，可以使用嵌入的文档来代替字典，因为字典不支持验证或自定义字段类型，所以推荐使用嵌入的文档。但是有时候你不知道你想要的存储结构，在这种情况下，DictField是合适的。
```
class SurveyResponse(Document):
    date = DateTimeField()
    user = ReferenceField(User)
    answers = DictField()

survey_response = SurveyResponse(date=datetime.now(), user=request.user)
response_form = ResponseForm(request.POST)
survey_response.answers = response_form.cleaned_data()
survey_response.save()
```
字典可以存储复杂的数据，其他字典，列表，对其他对象的引用，所以是可用的最灵活的字段类型。



###### Reference fields
引用可以使用ReferenceField存储在数据库中的其他文档中。 将另一个文档类作为构造函数的第一个参数传递，然后简单地将文档对象分配给该字段：
```
class User(Document):
    name = StringField()

class Page(Document):
    content = StringField()
    author = ReferenceField(User)

john = User(name="John Smith")
john.save()

post = Page(content="Test Page")
post.author = john
post.save()
```
User对象会自动转换成幕后的引用，并在检索到Page对象时解除引用。
给正在定义的文档对象添加一个引用自己的ReferenceField，使用'self'作为ReferenceField构造函数的参数来代替文档类。

和Django一样，要引用尚未定义的文档，请使用未定义文档的名称作为构造函数的参数：
```
class Employee(Document):
    name = StringField()
    boss = ReferenceField('self')
    profile_page = ReferenceField('ProfilePage')

class ProfilePage(Document):
    content = StringField()
```

###### One to Many with ListFields
如果你正在通过一个reference列表来表征一对多的关系，并且references作为DBRefs存储且需要查询，你需要传递一个对象的实例来查询。
```
class User(Document):
    name = StringField()

class Page(Document):
    content = StringField()
    authors = ListField(ReferenceField(User))

bob = User(name="Bob Jones").save()
john = User(name="John Smith").save()

Page(content="Test Page", authors=[bob, john]).save()
Page(content="Another Page", authors=[john]).save()

# Find all pages Bob authored
Page.objects(authors__in=[bob])

# Find all pages that both Bob and John have authored
Page.objects(authors__all=[bob, john])

# Remove Bob from the authors for a page.
Page.objects(id='...').update_one(pull__authors=bob)

# Add John to the authors for a page.
Page.objects(id='...').update_one(push__authors=john)
```

#### 处理引用文档的删除

默认情况下，MongoDB不检查数据的完整性，因此删除其他文档仍然保留引用的文档将导致一致性问题。
Mongoengine ReferenceField添加了一些功能来防止这些数据库完整性问题，为每个引用提供删除规则规范
通过在引用字段定义中提供reverse_delete_rule属性来指定删除规则，如下所示：
```
class ProfilePage(Document):
    ...
    employee = ReferenceField('Employee', reverse_delete_rule=mongoengine.CASCADE)
```
此示例中的声明意味着当删除Employee对象时，引用该employee的ProfilePage也被删除。 如果整批employee被删除，所有相关的profilepage也被删除。


###### mongoengine.DO_NOTHING
这是默认的，不会做任何事情。 删除速度很快，但可能导致数据库不一致或悬挂引用。

###### mongoengine.DENY
如果仍然存在对要删除的对象的引用，则将删除删除。
###### mongoengine.NULLIFY
任何仍然引用被删除的对象的对象字段被将删除（使用MongoDB的“unset”操作），有效地使关系无效。
###### mongoengine.PULL
从ListField（ReferenceField）的任何对象的字段中移除对对象的引用（使用MongoDB的“pull”操作）。


关于设置这些删除规则的安全说明，由于这些删除规则没有在数据库层被MongoDB自身记录，而是被MongoEngine在运行时，在内存中记录，所以在调用delete之前，加载声明关系的模块是非常重要的。
在Django中，请确保将所有具有删除规则声明的应用程序放在其在models.py中的INSTALLED_APPS元组中。


#### Generic reference fields
第二种参考字段也存在，GenericReferenceField。 这允许您引用任何类型的文档，因此不会将Document子类作为构造函数参数：
```
class Link(Document):
    url = StringField()

class Post(Document):
    title = StringField()

class Bookmark(Document):
    bookmark_object = GenericReferenceField()

link = Link(url='http://hmarr.com/mongoengine/')
link.save()

post = Post(title='Using MongoEngine')
post.save()

Bookmark(bookmark_object=link).save()
Bookmark(bookmark_object=post).save()
```
使用GenericReferenceFields的效率略低于标准的ReferenceFields，因此如果只引用一个文档类型，那么最好使用标准的ReferenceField。



#### Uniqueness constraints
MongoEngine允许您通过为Field的构造函数提供unique = True来指定字段在集合中是唯一的。 如果您尝试将与唯一字段具有相同值的文档作为已经在数据库中的文档保存，则会引发NotUniqueError。
```
class User(Document):
    username = StringField(unique=True)
    first_name = StringField()
    last_name = StringField(unique_with='first_name')

```

#### 跳过文档验证保存
您也可以通过在调用save（）方法时设置validate = False来跳过整个文档验证过程：
```
class Recipient(Document):
    name = StringField()
    email = EmailField()

recipient = Recipient(name='admin', email='root@localhost')
recipient.save()               # will raise a ValidationError while
recipient.save(validate=False) # won't
```


#### Document collections文档集合

从Document直接继承的文档类将在数据库中拥有自己的集合。
集合的名称是默认的类的名称，转换为小写（因此在上面的示例中，集合将被称为页面）。
如果需要更改集合的名称（例如，使用MongoEngine与现有数据库），则在文档中创建一个名为meta的类字典属性，并将collection设置为您希望文档类使用的集合的名称：
```
class Page(Document):
    title = StringField(max_length=200, required=True)
    meta = {'collection': 'cmsPage'}
```


#### 覆盖的集合
文档可以通过在meta字典中指定max_documents和max_size来使用Capped　collections。 max_documents是允许存储在集合中的最大文档数，max_size是集合的最大大小（以字节为单位）。
max_size 将会在MongoDB内部mongoengine之前四舍五入到下一个256的倍数。同样使用256的倍数来避免混淆。

如果未指定max_size，但是max_documents指定了，max_size默认为10485760字节（10MB）。以下示例显示一个日志文档，限制为1000个条目和2MB的磁盘空间：
```
class Log(Document):
    ip_address = StringField()
    meta = {'max_documents': 1000, 'max_size': 2000000}
```

#### 索引
通过在meta字典里创建一个名为indexs的索引规范的列表来完成，其中索引规范可以是单个字段名称，包含多个字段名称的元组或包含完整索引定义的字典。
```
class Page(Document):
    category = IntField()
    title = StringField()
    rating = StringField()
    created = DateTimeField()
    meta = {
        'indexes': [
            'title',
            '$title',  # text index
            '#title',  # hashed index
            ('title', '-rating'),
            ('category', '_cls'),
            {
                'fields': ['created'],
                'expireAfterSeconds': 3600
            }
        ]
    }
```

如果传递了一个字典，则可以使用以下选项：
###### fields (Default: None)
要索引的字段。 应该与上述相同的格式指定。

###### cls (Default: True)
如果具有继承并允许allow_inheritance的多态模型，则可以配置索引是否应将_cls字段自动添加到索引的开头。
###### sparse (Default: False)
索引是否应该是稀疏的。
###### expireAfterSeconds (Optional)
允许您通过设置使字段过期的时间（以秒为单位）来自动将集合里的数据过期。

###### Global index default options全局索引默认选项
所有可以设置的索引都有一些顶级默认值：

###### index_options (Optional)
设置任何默认索引选项 - 请参阅完整选项列表
###### index_background (Optional)
为索引是否应该在后台被索引设置默认值
###### index_cls (Optional)
关闭_cls的特定索引的方法



#### 复合索引和索引子文档
可以通过将嵌入字段或字典字段名称添加到索引定义来创建复合索引。



#### Comparing Indexes
使用mongoengine.Document.compare_indexes（）将数据库中的实际索引与您的文档定义定义的索引进行比较。 这对于维护目的很有用，并确保您的架构具有正确的索引。


#### Ordering排序
可以使用meta的ordering属性为您的QuerySet指定默认排序。 当QuerySet创建时，将应用ordering，并且可以通过后续调用order_by（）来覆盖排序。
```
from datetime import datetime

class BlogPost(Document):
    title = StringField()
    published_date = DateTimeField()

    meta = {
        'ordering': ['-published_date']
    }

blog_post_1 = BlogPost(title="Blog Post #1")
blog_post_1.published_date = datetime(2010, 1, 5, 0, 0 ,0)

blog_post_2 = BlogPost(title="Blog Post #2")
blog_post_2.published_date = datetime(2010, 1, 6, 0, 0 ,0)

blog_post_3 = BlogPost(title="Blog Post #3")
blog_post_3.published_date = datetime(2010, 1, 7, 0, 0 ,0)

blog_post_1.save()
blog_post_2.save()
blog_post_3.save()

# get the "first" BlogPost using default ordering
# from BlogPost.meta.ordering
latest_post = BlogPost.objects.first()
assert latest_post.title == "Blog Post #3"

# override default ordering, order BlogPosts by "published_date"
first_post = BlogPost.objects.order_by("+published_date").first()
assert first_post.title == "Blog Post #1"
```



#### Shard keys
如果您的集合被分片，那么您需要使用meta的shard_key属性将分片键指定为元组。 这可以确保在现有Document实例上调用save（）或update（）方法时，使用查询发送分片密钥：
```
class LogEntry(Document):
    machine = StringField()
    app = StringField()
    timestamp = DateTimeField()
    data = StringField()

    meta = {
        'shard_key': ('machine', 'timestamp',)
    }

```



#### Document inheritance文档继承
要创建您定义的文档的专门类型，您可以将其子类化，并添加您可能需要的任何额外的字段或方法。由于这是新类不是Document的直接子类，它不会存储在自己的集合中; 它将使用与其超类使用相同的集合。 这样可以更方便快捷地检索相关文件 - 所有您需要做的是在文档的meta中将allow_inheritance设置为True：


```
# Stored in a collection named 'page'
class Page(Document):
    title = StringField(max_length=200, required=True)

    meta = {'allow_inheritance': True}

# Also stored in the collection named 'page'
class DatedPage(Page):
    date = DateTimeField()
```


#### 字段参考
有如下field(具体用法请参见官方文档,很多都和Django中的ORM类似)
```


    BinaryField
    BooleanField
    ComplexDateTimeField
    DateTimeField
    DecimalField
    DictField
    DynamicField
    EmailField
    EmbeddedDocumentField
    EmbeddedDocumentListField
    FileField
    FloatField
    GenericEmbeddedDocumentField
    GenericReferenceField
    GeoPointField
    ImageField
    IntField
    ListField
    MapField
    ObjectIdField
    ReferenceField
    SequenceField
    SortedListField
    StringField
    URLField
    UUIDField
    PointField
    LineStringField
    PolygonField
    MultiPointField
    MultiLineStringField
    MultiPolygonField
```
