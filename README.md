# Python 最佳实践

该编程手册为腾讯蓝鲸在日常编码中，总结沉淀的 Python 以及相关框架(Django\DRF等)的编程经验。内容将跟随项目发展与 Python 以及框架的更新不断改进。

## 目录

- [Python](#python)
  - [内置数据结构](#内置数据结构)
  - [内置模块](#内置模块)
  - [生成器与迭代器](#生成器与迭代器)
  - [函数](#函数)
  - [面向对象编程](#面向对象编程)
  - [异常处理](#异常处理)
  - [工具选择](#工具选择)
  - [2to3](#2to3)
- [Django](#django)
  - [DB建模](#db-建模)
  - [DB查询](#db-查询)
- [Django REST framework](#drf)




# Python

Python最佳实践、优化思路、工具选择 及 Python 2to3 经验



## 内置数据结构

#### 不要在代码中出现魔术数字

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



# 2to3

## Python 版本升级后的注意事项

### logging 无法写入中文编码字符，抛出异常 `UnicodeEncodeError`

- 原因

  在 Python3 需指定文件流编码

- 解决

  直接在 logging 增加属性 `'encoding': 'utf-8'`

### 在获取字符串 MD5 时，抛出异常 `TypeError: Unicode-objects must be encoded before hashing`

```python
cache_str = "url_{url}__params_{params}".format(
    url=self.build_actual_url(params), params=json.dumps(params)
)
hash_md5 = hashlib.new('md5')
hash_md5.update(cache_str)  # 此处抛出异常
cache_key = hash_md5.hexdigest()
```

- 原因

  在 Python3，此函数参数为 bytes，需要进行 encode

- 解决

```python
cache_str = "url_{url}__params_{params}".format(
    url=self.build_actual_url(params), params=json.dumps(params)
)
hash_md5 = hashlib.new('md5')
hash_md5.update(cache_str.encode('utf-8'))  # 增加encode
cache_key = hash_md5.hexdigest()
```

### 执行行语句 `'a' >= 2` ，抛出异常 `TypeError: '>=' not supported between instances of 'str' and 'int'`

- 原因

  在 Python3 中，不允许直接把字符串和数字直接进行大小比较

### `xrange` 函数未被 2to3 工具自动替换为 `range`，需要手动修改

### 执行语句以下语句，抛出异常 `TypeError: 'cmp' is an invalid keyword argument for this function`

```python
sorted(referenced_weight.items(), cmp=lambda x, y: cmp(x[1], y[1]), reverse=True)
```

- 原因

  Python 3 把 `sort()` 方法中的`cmp`参数废弃了

- 解决

```python
from functools import cmp_to_key
nums = [1, 3, 2, 4]
nums.sort(key=cmp_to_key(lambda a, b: a - b))
print(nums)  # [1, 2, 3, 4]
```

#### 6. bytes 转换字符串，抛出异常 `TypeError: string argument without an encoding`

```python
f.encrypt(bytes(node_id))
```

- 原因

  python 3 将字符串转换成 byte 需要指定编码

- 解决

```python
f.encrypt(bytes(node_id, encoding='utf8'))
```

### btytes 类型 json dumps 抛出异常 `Object of type 'bytes' is not JSON serializable`

- 原因

  在 python 3 中，json 仅支持序列化 `str` 类型的字符串，而不支持 `bytes` 类型

- 解决

  使用 `ujson` 替换掉原生的 `json`

```python
import ujson as json
```

### base64 编码时，抛出异常 `TypeError: a bytes-like object is required, not 'str'`

```python
file_data = base64.b64encode(json.dumps({
    'template_data': templates_data,
    'digest': digest
}, sort_keys=True))
```

- 原因

  在 python 3 中，base64 仅支持对 `bytes` 类型字符串的 encode

- 解决

  在 encode 之前，先对 `str` 类型的字符串转换为 `bytes`

```python
file_data = base64.b64encode(json.dumps({
    'template_data': templates_data,
    'digest': digest
}, sort_keys=True).encode('utf-8'))
```

### 捕获异常后打印 `e.message` ，抛出异常 `AttributeError: 'Exception' object has no attribute 'message'`

```python
try:
    task.modify_cron(cron, tz)
except Exception as e:
    return JsonResponse({
        'result': False,
        'message': e.message
    })
```

- 原因

  python3 `BaseException` 中去掉了 `message` 属性

- 解决

  使用 `str(e)` 进行类型转换

```python
try:
    task.modify_cron(cron, tz)
except Exception as e:
    return JsonResponse({
        'result': False,
        'message': str(e)
    })
```

### 中文字符串字面量前不需添加 `u` 前缀

> 注意：该建议只适用于 Python3 版本

在 Python3 中，默认的字符串已经是 unicode 编码。对于包含中文的字符串字面量，可以放心的把 `u` 前缀去掉：

```python
# 可接受，但是没有必要
hello_world = u'hello，你好'

# GOOD
hello_world = 'hello，你好'
```


# Django

Django最佳实践、优化思路



## DB 建模

### 1.1 字段设计

- 避免允许 null 值的字段，null 值难以查询优化且占用额外的索引空间
- 充分考虑每张表的数据规模，选取合适主键字段类型。比如数据创建比较频繁的表，主键建议使用 `BigIntegerField`
- 使用正确的字段类型，避免`TextField`代替`CharField`，`IntegerField`代替`BooleanField`等
- 如果字段的取值是一个有限集合，应使用 `choices` 选项声明枚举值

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

- 如果某个字段或某组字段被频繁用于过滤或排序查询，建议建立单字段索引或联合索引

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



## DB 查询

### 2.1 基本要求

- 了解 Django ORM 是如何缓存数据的
- 了解 Django ORM 何时会做查询
- 不要以牺牲代码可读度为代价做过度优化

### 2.2 实际应用

- 避免全表扫描。优先使用`exists`, `count`等方法

```python
# 获取 project 的数量
projects = Project.objects.filter(enable=True)

# Bad: 会强制将 projects 实例化，导致全表扫描
project_count = len(projects)  

# Good: 直接从数据库层面统计，避免全表扫描
project_count = projects.count()
```

- 避免 N + 1 查询。可使用`select_related`提前将关联表进行 join，一次性获取相关数据，many-to-many 的外键则使用`prefetch_related`

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

- 如果仅查询外键 ID，则无需进行连表操作。使用 `外键名_id` 可直接获取

```python
# 获取学生的班级ID
student = Student.objects.first()

# Bad: 会产生一次关联查询
cls_id = student.cls.id

# Good: 不产生新的查询
cls_id = student.cls_id
```

- 避免查询全部字段。可使用`values`, `values_list`, `only`, `defer`等方法进行过滤出需要使用的字段。

```python
# 仅获取学生姓名的列表

# Bad
students = Student.objects.all()
student_names = [student.name for student in students]

# Good
students = Student.objects.all().values_list('name', flat=True)
```

- 避免在循环中进行数据库操作。尽量使用 ORM 提供的批量方法，防止在数据量变大的时候产生大量数据库连接导致请求变慢

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

- 避免隐式的子查询

```python
# 查询符合条件的组别中的人员

# Bad: 将查询集作为下一个查询的过滤条件，因此产生了子查询。IN 语句中的子查询在外层查询的每一行中都会被执行一次，复杂度为 O(n^2)
groups = Group.objects.filter(type="typeA")
members = Member.objects.filter(group__in=groups)

# Good: 以确定的数据作为过滤条件，避免子查询
group_ids = Group.objects.filter(type="typeA").values_list('id', flat=True)
members = Member.objects.filter(group__id__in=list(group_ids))
```

- `update_or_create` 与 `get_or_create` 不是线程安全的。因此查询条件的字段必须要有唯一性约束

```python
# 为了保证逻辑的正确性，Host 表中的 ip 和 bk_cloud_id 字段必须设置为 unique_together。否则在高并发情况下，可能会创建出多条相同的记录，最终导致逻辑异常
host, is_created = Host.objects.get_or_create(
    ip="127.0.0.1",
    bk_cloud_id="0"
)
```

- 如果查询集只用于单次循环，建议使用 `iterator()` 保持连接查询。当查询结果有很多对象时，QuerySet 的缓存行为会导致使用大量内存。如果你需要对查询结果进行好几次循环，这种缓存是有意义的，但是对于 QuerySet 只循环一次的情况，缓存就没什么意义了。在这种情况下，`iterator()`可能是更好的选择

```python
# Bad
for task in Task.objects.all():
    # do something

# Good
for task in Task.objects.all().iterator():
    # do something
```

### 使用 values_list(flat=True) 来查询某个字段值

如果要查询模型里某字段的值，可以使用 `values_list(flat=True)` 方法，效率最高。

```python
# BAD
pks = [obj.id for obj in Foo.objects.all()]

# BAD
pks = Foo.objects.all().values_list('id')
pks = [val[0] for val in pks]

# GOOD
pks = list(Foo.objects.all().values_list('id', flat=True))
```

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



# DRF

[WIP] DRF相关实践经验...

