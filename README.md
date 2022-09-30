# 蓝鲸最佳实践 <!-- omit in toc -->

该文档为[腾讯蓝鲸团队](https://bk.tencent.com/) 多年的编程最佳实践总结，包括 Python \ Golang 等多个语言及其相关领域。内容将跟随项目发展与语言/框架的更新不断改进。

# 目录 <!-- omit in toc -->
- [Python](#python)
  - [内置数据结构](#内置数据结构)
    - [避免魔术数字](#避免魔术数字)
    - [不要预计算字面量表达式](#不要预计算字面量表达式)
    - [优先使用列表推导或内联函数](#优先使用列表推导或内联函数)
  - [内置模块](#内置模块)
    - [使用 operator 模块替代简单 lambda 函数](#使用-operator-模块替代简单-lambda-函数)
      - [替代相乘函数](#替代相乘函数)
      - [替代索引获取函数](#替代索引获取函数)
      - [替代属性获取函数](#替代属性获取函数)
    - [logging 模块：尽量使用参数，而不是直接拼接字符串](#logging-模块尽量使用参数而不是直接拼接字符串)
  - [生成器与迭代器](#生成器与迭代器)
    - [未激活生成器的陷阱](#未激活生成器的陷阱)
    - [使用现代化字符串格式化方法](#使用现代化字符串格式化方法)
  - [函数](#函数)
    - [统一返回值类型](#统一返回值类型)
    - [增加类型注解](#增加类型注解)
    - [不要使用可变类型作为默认参数](#不要使用可变类型作为默认参数)
    - [优先使用异常替代错误编码返回](#优先使用异常替代错误编码返回)
  - [面向对象编程](#面向对象编程)
    - [使用 dataclass 定义数据类](#使用-dataclass-定义数据类)
      - [在数据量较大的场景下，需要在结构的便利性和性能中做平衡](#在数据量较大的场景下需要在结构的便利性和性能中做平衡)
  - [异常处理](#异常处理)
    - [避免含糊不清的异常捕获](#避免含糊不清的异常捕获)
  - [工具选择](#工具选择)
    - [使用 PyMySQL 连接 MySQL 数据库](#使用-pymysql-连接-mysql-数据库)
    - [使用 dogpile.cache 做缓存](#使用-dogpilecache-做缓存)
- [Django](#django)
  - [DB 建模](#db-建模)
    - [如果字段的取值是一个有限集合，应使用 `choices` 选项声明枚举值](#如果字段的取值是一个有限集合应使用-choices-选项声明枚举值)
    - [如果某个字段或某组字段被频繁用于过滤或排序查询，建议建立单字段索引或联合索引](#如果某个字段或某组字段被频繁用于过滤或排序查询建议建立单字段索引或联合索引)
    - [变更数据表时，新增字段尽量使用 `null=True` 而不是 `default`](#变更数据表时新增字段尽量使用-nulltrue-而不是-default)
  - [DB 查询](#db-查询)
    - [使用 .exists() 判断数据是否存在](#使用-exists-判断数据是否存在)
    - [使用 .count() 查询数据条目数](#使用-count-查询数据条目数)
    - [避免 N + 1 查询](#避免-n--1-查询)
    - [如果仅查询外键 ID，则无需进行连表操作。使用 `外键名_id` 可直接获取](#如果仅查询外键-id则无需进行连表操作使用-外键名_id-可直接获取)
    - [避免查询全部字段](#避免查询全部字段)
    - [避免在循环中进行数据库操作](#避免在循环中进行数据库操作)
    - [避免隐式的子查询](#避免隐式的子查询)
    - [`update_or_create` 与 `get_or_create` 通过 defaults 参数避免全表查询](#update_or_create-与-get_or_create-通过-defaults-参数避免全表查询)
    - [`update_or_create` 与 `get_or_create` 查询条件的字段必须要有唯一性约束](#update_or_create-与-get_or_create-查询条件的字段必须要有唯一性约束)
    - [如果查询集只用于单次循环，建议使用 `iterator()` 保持连接查询](#如果查询集只用于单次循环建议使用-iterator-保持连接查询)
    - [针对数据库字段更新尽量使用 `update_fields`](#针对数据库字段更新尽量使用-update_fields)
    - [使用 Django Extra 查询时，需要使用内置的字符串表达](#使用-django-extra-查询时需要使用内置的字符串表达)
    - [善用 bulk_create/bulk_update 减少批量数据库操作耗时](#善用-bulk_createbulk_update-减少批量数据库操作耗时)
    - [当 MySQL 版本较低时（<5.7)，谨慎使用 DateTimeField 进行排序](#当-mysql-版本较低时57谨慎使用-datetimefield-进行排序)
- [Golang](#golang)
    - [channel空间设定为1或者阻塞](#channel空间设定为1或者阻塞)
    - [除for循环以外，不要在代码块初始化中使用:=](#除for循环以外不要在代码块初始化中使用)
    - [channel接受使用两段式](#channel接受使用两段式)
    - [不能通过取出来的值来判断 key 是不是在 map 中](#不能通过取出来的值来判断-key-是不是在-map-中)
    - [变量接受使用两段式](#变量接受使用两段式)
    - [定义常量时，区分某些类型和标识](#定义常量时区分某些类型和标识)

# Python

Python 🐍 最佳实践、优化思路、工具选择。


## 内置数据结构

### 避免魔术数字

不要在代码中出现 [Magic Number](https://en.wikipedia.org/wiki/Magic_number_(programming))，常量应该使用 Enum 模块来替代。

```python
# BAD
if cluster_type == 1:
    pass


# GOOD
from enum import Enum

Class BCSType(Enum):
    K8S = 1
    Mesos = 2

if cluster_type == BCSType.K8S.value:
    pass
```


### 不要预计算字面量表达式

如果某个变量是通过简单算式得到的，应该保留算式内容。不要直接使用计算后的结果。

```python
# BAD
if delta_seconds > 950400:
    return

# GOOD
if delta_seconds > 11 * 24 * 3600:
    return
```

### 优先使用列表推导或内联函数

使用列表推导或内联函数能够清晰知道要生成一个列表，并且更简洁

```python
# BAD
list_two = []
for v in list_one:
    if v[0]:
        new_list.append(v[1])

# GOOD one
list_two = [v[1] for v in list_one if v[0]]

# GOOD two
list_two = list(filter(lambda x: x[0], list_one))
```

## 内置模块

### 使用 operator 模块替代简单 lambda 函数

在很多场景下，`lambda` 函数都可以用 `operator` 模块来替代，后者效率更高。

#### 替代相乘函数

```python
# BAD
product = reduce(lambda x, y: x * y, numbers, 1)

# GOOD
from operator import mul
product = reduce(mul, numbers, 1)
```

#### 替代索引获取函数

```python
# BAD
rows_sorted_by_city = sorted(rows, key=lambda row: row['city'])

# GOOD
from operator import itemgetter
rows_sorted_by_city = sorted(rows, key=itemgetter('city'))
```

#### 替代属性获取函数

```python
# BAD
products_by_quantity = sorted(products, key=lambda p: p.quantity)

# GOOD
from operator import attrgetter
products_by_quantity = sorted(products, key=attrgetter('quantity'))
```

### logging 模块：尽量使用参数，而不是直接拼接字符串

在使用 `logging` 模块打印日志时，请尽量 **不要** 在第一个参数内拼接好日志内容（不论是使用何种方式）。正确的做法是只在第一个参数提供模板，参数由后面传入。

在大规模循环打印日志时，这样做效率更高。

参考：https://docs.python.org/3/howto/logging.html#optimization

```python
# BAD
logging.warning("To iterate is %s, to recurse %s" % ("human", "divine"))

# BAD，但并非不可接受
logging.warning(f"To iterate is {human}, to recurse {divine}")

# GOOD
logging.warning("To iterate is %s, to recurse %s", "human", "divine")
```

## 生成器与迭代器

### 未激活生成器的陷阱

调用生成器函数后，拿到的对象是处于“未激活”状态的生成器对象。比如下面的 `get_something()` 函数，当你调用它时并不会抛出 `ZeroDivisionError` 异常：

```python
>>> def get_something():
...     yield 1 / 0
...
>>> get_something()
<generator object get_something at 0x10d2301b0>
```

如果对该对象使用布尔判断，将永远返回 `True`。

```python
>>> bool(get_something())
True
```

激活生成器可以使用下面这些方式：

```python
# 使用 list 内建函数
>>> list(get_something())
Traceback (most recent call last):
... ...
ZeroDivisionError: division by zero

# 使用 next 内建函数
>>> next(get_something())
Traceback (most recent call last):
... ...
ZeroDivisionError: division by zero

# 使用 for 循环
>>> for i in get_something(): pass
...
Traceback (most recent call last):
... ...
ZeroDivisionError: division by zero
```

### 使用现代化字符串格式化方法

在需要格式化字符串时，请使用 [str.format()](https://docs.python.org/3/library/stdtypes.html#str.format) 或 [f-strings](https://docs.python.org/3/reference/lexical_analysis.html#f-strings)。

```python
# BAD
metric_name = '%s_clsuter_id' % data_id

# GOOD
metric_name = '{}_cluster_id'.format(data_id)

# BEST
metric_name = f'{data_id}_cluster_id'
```

当需要重复键入格式化变量名时，使用 `f-strings`。

```python
# BAD
"{a}-{b}-{c}-{d}".format(a=a, b=b, c=c, d=d)

# GOOD
f"{a}-{b}-{c}-{d}"
```



## 函数

### 统一返回值类型

单个函数应该总是返回同一类数据。

```python
# BAD
# 可能返回 bool 或者 None
def is_odd(num):
    if num % 2 > 0:
        return True


# GOOD
def is_odd(num):
    if num % 2 > 0:
        return True
    return False
```

### 增加类型注解

对变量和函数的参数返回值类型做注解，有助于通过静态检查减少类型方面的错误。

```python
# BAD
def greeting(name):
    return 'Hello ' + name

# GOOD
def greeting(name: str) -> str:
    return 'Hello ' + name
```

### 不要使用可变类型作为默认参数

函数作为对象定义的时候就被执行，默认参数是函数的属性，它的值可能会随着函数被调用而改变。

```python
# BAD
def foo(li: list = []):
    li.append(1)
    print(li)

# GOOD
def foo(li : Optional[list] = None):
    li = li or []
    li.append(1)
    print(li)
```
调用两次foo函数后，不同的输出结果：
```python
# BAD
[1]
[1,1]

# GOOD
[1]
[1]
```

### 优先使用异常替代错误编码返回

当函数需要返回错误信息时，以抛出异常优先。

```python
# BAD
def disable_agent(ip):
    if not IP_DATA.get(ip):
        return {"code": ErrorCode.UnknownError, "message": "没有查询到 IP 对应主机"}


# GOOD
def disable_agent(ip):
    """
    :raises: 当找不到对应信息时，抛出 UNABLE_TO_DISABLE_AGENT_NO_MATCH_HOST
    """
    if not IP_DATA.get(ip):
        raise ErrorCode.UNABLE_TO_DISABLE_AGENT_NO_MATCH_HOST
```

## 面向对象编程


### 使用 dataclass 定义数据类

对于需要在初始化阶段设置很多属性的数据类，应该使用 dataclass 来简化代码。但同时注意不要滥用，比如一些实例化参数少、没有太多“数据”属性的类仍然应该使用传统 `__init__` 方法。

```python
# BAD
class BcsInfoProvider:
    def __init__(self, project_id, cluster_id, access_token , namespace_id, context):
        self.project_id = project_id
        self.cluster_id = cluster_id
        self.namespace_id = namespace_id
        self.context = context

# GOOD
from dataclasses import dataclass

@dataclass
class BcsInfoProvider:
    project_id: str
    cluster_id: str
    access_token: str
    namespace_id: int
    context: dict
```
#### 在数据量较大的场景下，需要在结构的便利性和性能中做平衡

很多场景下，我们会得到比较大的原始数据（比如数万个嵌套的 `dict`），为了更便利地操作这些数据，往往会选择通过 `class` 进行实例化，但基于 Python 孱弱的 CPU 计算性能，这一操作可能会耗时过于久。

所以需要在两方面做平衡：
- 保持原始数据，获得最好的性能，但是不方便操作
- 使用 `NamedTuple` 类似的结构，获得一定的结构便利性，但相较于原始数据，会牺牲一定性能
- 使用 `dataclass` 或者 `class` 等方式，保证最大的结构便利性，但会非常影响性能


## 异常处理

### 避免含糊不清的异常捕获

不要捕获过于基础的异常类，比如 Exception / BaseException，捕获这些会扩大处理的异常范围，容易隐藏其他本来不应该被捕获的问题。

尽量捕获预期内可能出现的异常。

```python
# BAD
try
    func_code()
except:
    # your code

# GOOD
try:
    func_code()
except CustomError as error:
    # your code
except Exception as error:
    logger.exception("some error: %s", error)
```



## 工具选择

### 使用 PyMySQL 连接 MySQL 数据库

建议使用纯 Python 实现的 [PyMySQL](https://github.com/PyMySQL/PyMySQL) 模块来连接 MySQL 数据库。如果需要在项目中完全替代 MySQL-python 模块，可以使用模块提供的猴子补丁功能：

```python
import pymysql
pymysql.install_as_MySQLdb()

# 如果版本号无法通过框架检查，可以在导入模块后动态修改
setattr(pymysql, 'version_info', (1, 2, 6, "final", 0))
```

### 使用 dogpile.cache 做缓存

`dogpile.caches` 扩展性强，提供了一套可以对接多中后端存储的缓存 API，推荐作为项目中缓存的基础库。

样例代码：

```python
from dogpile.cache import make_region

region = make_region().configure('dogpile.cache.redis')  # 其他参数官方文档


@region.cache_on_arguments(expiration_time=3600)
def get_application(username):
    # your code


@region.cache_on_arguments(expiration_time=3600, function_key_generator=ignore_access_token) # function_key_generator 参考官方文档
def get_application(access_token, username):
    # your code
```

# Django

Django最佳实践、优化思路。


## DB 建模

### 如果字段的取值是一个有限集合，应使用 `choices` 选项声明枚举值

```python
class Students(models.Model):
    class Gender(object):
        MALE = 'MALE'
        FEMALE = 'FEMALE'

    GENDER_CHOICES = (
        (Gender.MALE, "男"),
        (Gender.FEMALE, "女"),
    )

    gender = models.IntegerField("性别", choices=GENDER_CHOICES)
```

### 如果某个字段或某组字段被频繁用于过滤或排序查询，建议建立单字段索引或联合索引

```python
# 字段索引：使用 db_index=True 添加索引
title = models.CharField(max_length=255, db_index=True)

# 联合索引：将多个字段组合在一起建立索引
class Meta:
    index_together = ['field_name_1', 'field_name_2']

# 联合唯一索引：将多个组合在一起的索引，并且字段的组合值唯一
class Meta:
    unique_together = ('field_name_1', 'field_name_2')
```

### 变更数据表时，新增字段尽量使用 `null=True` 而不是 `default`

```python
# BAD
new_field = models.CharField(default="foo")

# GOOD
new_field = models.CharField(null=True)
```

前者将会在 `migrate` 操作时对已存在的数据批量刷新，对现有数据库带来不必要的影响。

参考：https://pankrat.github.io/2015/django-migrations-without-downtimes/


## DB 查询

### 使用 .exists() 判断数据是否存在

如果要查询记录是否存在，建议使用 `.exists()` 方法。该方法将会往数据库发起一条设置了 `LIMIT 1` 的查询语句，效率最佳。

```python
# BAD
# 将会查询表中所有结果，效率低
if Foo.objects.filter(name='test'):
    # Do something

# GOOD
if Foo.objects.filter(name='test').exists():
    # Do something
```

### 使用 .count() 查询数据条目数

如果要统计数据条目数，建议使用使用 `.count()` 方法。该方法将会往数据库发起一条 `SELECT count(*)` 查询语句。

```python
# BAD
# 将查询表中所有内容，耗费大量内存和 CPU
count = len(Foo.objects.all())

# GOOD
count = Foo.objects.count()
```

### 避免 N + 1 查询

可使用`select_related`提前将关联表进行 join，一次性获取相关数据，many-to-many 的外键则使用`prefetch_related`

```python
# select_related

# Bad
# 由于 ORM 的懒加载特性，在执行 filter 操作时，并不会将外键关联表的字段取出，而是在使用时，实时查询。这样会产生大量的数据库查询操作
students = Student.objects.all()
student_in_class = {student.name: student.cls.name for student in students}

# Good
# 使用 select_related 可以避免 N + 1 查询，一次性将外键字段取出
students = Student.objects.select_related('cls').all()
student_in_class = {student.name: student.cls.name for student in students}
# prefetch_related

# Bad
articles = Article.objects.filter(id__in=(1,2))
for item in articles:
    # 会产生新的数据库查询操作
    item.tags.all()

# Good
articles = Article.objects.prefetch_related("tags").filter(id__in=(1,2))
for item in articles:
    # 不会产生新的数据库查询操作
    item.tags.all()
```

### 如果仅查询外键 ID，则无需进行连表操作。使用 `外键名_id` 可直接获取

```python
# 获取学生的班级ID
student = Student.objects.first()

# Bad: 会产生一次关联查询
cls_id = student.cls.id

# Good: 不产生新的查询
cls_id = student.cls_id
```

### 避免查询全部字段
可使用`values`, `values_list`, `only`, `defer`等方法进行过滤出需要使用的字段。

```python
# 仅获取学生姓名的列表

# Bad
students = Student.objects.all()
student_names = [student.name for student in students]

# Good
students = Student.objects.all().values_list('name', flat=True)
```

### 避免在循环中进行数据库操作

尽量使用 ORM 提供的批量方法，防止在数据量变大的时候产生大量数据库连接导致请求变慢

```python
# 批量创建项目
project_names = ['ProjectA', 'ProjectB', 'ProjectC']

# Bad
for project_name in project_names:
    Project.objects.create(name=project_name)

# Good
projects = []
for project_name in project_names:
    project = Project(name=project_name)
    projects.append(project)
Project.objects.bulk_create(projects)
# 批量查询项目
project_names = ['ProjectA', 'ProjectB', 'ProjectC']

# Bad: 每次循环都产生一次新的查询
projects = []
for project_name in project_names:
    project = Project.objects.get(name=project_name)
    projects.append(project)

# Good：使用 in，只需一次数据库查询
projects = Project.objects.filter(name__in=project_names)
# 批量更新项目
project_names = ['ProjectA', 'ProjectB', 'ProjectC']
projects = Project.objects.filter(name__in=project_names)

# Bad: 每次循环都产生一次新的查询
for project in projects:
    project.enable = True
    project.save()

# Good：批量更新，只需一次数据库查询
projects.update(enable=True)
```

### 避免隐式的子查询

```python
# 查询符合条件的组别中的人员

# Bad: 将查询集作为下一个查询的过滤条件，因此产生了子查询。IN 语句中的子查询在外层查询的每一行中都会被执行一次，复杂度为 O(n^2)
groups = Group.objects.filter(type="typeA")
members = Member.objects.filter(group__in=groups)

# Good: 以确定的数据作为过滤条件，避免子查询
group_ids = Group.objects.filter(type="typeA").values_list('id', flat=True)
members = Member.objects.filter(group__id__in=list(group_ids))
```

### `update_or_create` 与 `get_or_create` 通过 defaults 参数避免全表查询

使用 `update_or_create` 与 `get_or_create` 时，需要将 **查询字段** 和 **更新字段** 做区分：

- 前者放在方法参数中，会被 Django 当作查询条件判断是否已有记录
- 后者应该被放入 `defaults` 参数中，否则将会被当作查询条件，容易触发全表查询


```python
# BAD
ModelA.objects.update_or_create(
    field_1="field_1",
    field_2="field_2",
    field_3="field_3",
)

# GOOD
ModelA.objects.update_or_create(
    field_1="field_1",
    defaults={
        "field_2": "field_2",
        "field_3": "field_3",
    }
```

### `update_or_create` 与 `get_or_create` 查询条件的字段必须要有唯一性约束


```python
# `update_or_create` 与 `get_or_create` 不是线程安全的。
# 为了保证逻辑的正确性，Host 表中的 ip 和 bk_cloud_id 字段必须设置为 unique_together。
# 否则在高并发情况下，可能会创建出多条相同的记录，最终导致逻辑异常
host, is_created = Host.objects.get_or_create(
    ip="127.0.0.1",
    bk_cloud_id="0"
)
```

### 如果查询集只用于单次循环，建议使用 `iterator()` 保持连接查询

当查询结果有很多对象时，QuerySet 的缓存行为会导致使用大量内存。如果你需要对查询结果进行好几次循环，这种缓存是有意义的，但是对于 QuerySet 只循环一次的情况，缓存就没什么意义了。在这种情况下，`iterator()`可能是更好的选择。

```python
# Bad
for task in Task.objects.all():
    # do something

# Good
for task in Task.objects.all().iterator():
    # do something
```

### 针对数据库字段更新尽量使用 `update_fields`  

如果要对数据库字段进行更新，使用 `update_fields` 避免并行 `save()` 产生数据冲突

```python
# BAD
foo_instance.bar_field = other_value
foo_instance.save()

# GOOD
foo_instance.bar_field = other_value
foo_instance.save(update_fields=["bar_field"])
```

同时需要注意的是，如果 `Model` 中包含 `auto_now` 字段时，需要在 `update_fields` 的列表中添加该字段，保证同时更新。

### 使用 Django Extra 查询时，需要使用内置的字符串表达

```python
# BAD
# 有注入风险, username 不会被转义，可以直接注入
Entry.objects.extra(where=[f"headline='{username}'"])

# GOOD
# 安全，Django 会将 username 内容转义
Entry.objects.extra(where=['headline=%s'], params=[username])
```

### 善用 bulk_create/bulk_update 减少批量数据库操作耗时

```python
# BAD
## 每次都执行commit，整体耗时较长(大约25s左右)
for num in range(10000):
    Record.objects.create(num=num)

# GOOD
## 统一提交数据库，耗时很短(1s以内)
inserted_list = []
for num in range(10000):
    inserted_list.append(Demo(num=num))

Record.objects.bulk_create(inserted_list)
```

同理，当 Django 版本 > 2.x 时，`bulk_update` 也可以加快批量修改。

```python
tasks = [
    Task.objects.create(name='task1', status='start', cost=1),
    Task.objects.create(name='task2', status='start', cost=1),
    ...
]

# BAD
for task in tasks:
    task.name = f'{task.pk}-{task.name}'
    task.save()

# GOOD
for task in tasks:
    task.name = f'{task.pk}-{task.name}'
Task.objects.bulk_update(tasks, ['name'])
```

同时还有一些需要额外注意: 
- bulk_create 方法只执行一次数据库交互，这样相当于创建时间一样，并且自定字段不会在返回数据中
- 当单次提交的对象可能过多时，可通过 `batch_size` 控制

### 当 MySQL 版本较低时（<5.7)，谨慎使用 DateTimeField 进行排序

当 MySQL 版本较低时，DATETIME 类型默认是不支持 milliseconds 的，当批量创建对象时，会导致大量记录的 `auto_now_add` 字段都在同一秒，此时根据该字段是无法获得稳定的排序结果的。

```python
# BAD
class Foo(models.Model):
    ...
    foo = models.DateTimeField(auto_now_add=True)
    ...

    class Meta:
        ordering = ["foo"]



# GOOD
class Foo(models.Model):
    ...
    foo = models.DateTimeField(auto_now_add=True)
    ...

    class Meta:
        # 使用自增 ID 或者其他能准确表明顺序的字段
        ordering = ["id"]
```

参考：
- https://stackoverflow.com/questions/13344994/mysql-5-6-datetime-doesnt-accept-milliseconds-microseconds


# Golang 

蓝鲸监控团队的Golang实践，持续补充中...

### channel空间设定为1或者阻塞

如果改为其他长度的channel，都需要很详细的评估设计，因此建议默认考虑长度为1或阻塞的channel

```go
// BAD
c := make(chan int, 100)

// GOOD
c := make(chan int)
```

### 除for循环以外，不要在代码块初始化中使用:=

如果在代码块中使用了新建变量，容易导致覆盖上层的变量而不会发现，容易引发bug

```go
// BAD 
if _, err := openFile("/path") {
   // do something
}

// GOOD 
var err error
if _, err = openFile("/path") {
   // do something
}
```

### channel接受使用两段式

由于读取已关闭的channel会导致panic，因此要求在读取channel的代码都使用二段式，可以避免channel已关闭的导致panic

```go
// BAD
value := <- ch

// GOOD
var (
	ok bool
)
if _, ok = <- ch; !ok {
	// do something when channel is closed.
}
```

### 不能通过取出来的值来判断 key 是不是在 map 中

go 会返回元素对应数据类型的零值，取值操作总有值返回，不能通过取出来的值来判断 key 是不是在 map 中

```go
// BAD
x := map[string]string{"demo1": "1", "demo2": "2"}
if v := x["demo3"]; v == "" {
  fmt.Println("demo3 is not exist")
}


// GOOD
x := map[string]string{"demo1": "1", "demo2": "2"}
if _, ok := x["demo3"]; !ok {
    fmt.Println("demo3 is not exist")
}
```

### 接口类型转换应使用两段式

由于当接口(interface)类型转换为实际类型时，如果类型不正确或接口为nil，会导致panic。因此应该使用二段式或switch的方式来避免panic

```go
var (
  a  interface{}
  b  int
  ok bool
)

// BAD
b = a.(int)

// GOOD
b, ok = a.(int)
// or
switch a.(type) {
case int:
    // do something when type is int
case float64:
    // do something when type is float64
default:
    // Ooops, trans failed.
}
```

### 定义常量时，使用自增的方式定义

定义常量时，应使用`itoa`的方式由编译器协助为各个常量赋值，降低后续维护的成本

```go
// BAD
const (
   Red = 0
   Gray  = 1
)

// GOOD
const (
   Red = iota
   Gray 
)
```