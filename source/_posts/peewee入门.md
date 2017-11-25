---
title: peewee入门
date: 2017-10-24 9:39:00
categories: Python
tags:
- ORM
---

# peewee quick start

## 摘要

peewee是python ORM的一个框架。

<!--more-->

### 安装

* 使用pip直接安装最新版本

```
pip install peewee

```

* 使用git安装

项目的地址： https://github.com/coleifer/peewee 

```
git clone https://github.com/coleifer/peewee
cd peewee
python setup.py install

```

### 快速开始

本编文档为peewee主要功能做了一个简明的、高水准的概述，目录包含：

* 模块定义
* 存储数据
* 检索数据

#### 模块定义

Model classes, fields and model instances 都映射了数据库的概念：


Thing | Corresponds to database
---|---
Modelclasses | db table
fields | table column
model instance | Row in table

当使用peewee开始一个项目时，一贯最好的方式就是从数据模型开始，通过定义一个或多个模型类

```python
from peewee import *
db = SqliteDatabase('people.db')
clss Person(Model):
    name = CharFiled()
    birthday = DateFiled()
    is_relative = BooleanFiled()
    
    class Meta:
        database = db
        
```

这里提供了很多类型的field，以便适合存储不同类型的数据。peewee处理器可以将python里的数值类型转换为数据库中的数据类型，因此，您可以在代码中使用Python类型，而不必担心

当我们用外键给model之间建立起联系后，事情就开始变得有趣了。
在peewee中可以轻松是实现:

```python
class Pets(Model):
    owner = ForeignKeyField(Person,related_name='pets')
    name = CharField()
    animal_type = CharField()
    
    class Meta:
        database = db
        
```

既然我们有自己的model，那就让它们和数据库关联起来吧。虽然没有必要显式地打开连接，这是一个很好的做法，因为它会立即显示您的数据库连接的任何错误，与第一次查询执行后的任意时间不同。完成后关闭连接也是很好的--例如，一个web app当接收一个请求时要打开数据库连接，响应时要关闭连接。

```
db.connect()

```

首先，我们将在数据库中创建存储数据的表。这将创建具有适当列、索引、序列和外键约束的表。

```
db.create_tables([People,pets])

```

#### 数据存储

首先，我们用一些people对象填充数据库吧。可以使用save()和create()方法添加和更新people记录

```python
from datetime import date
uncle_bob = Person(name='Bob',birthday=date(1960,1,15),is_relative = True)
uncle_bob.save()

```

调用save()方法后，修改记录的记录数将被返回

你也可以使用create()方法添加一条person记录，它返回的是一个person实例

```python
grandma = Person.create(name='Grandma', birthday=date(1935, 3, 1), is_relative=True)
herb = Person.create(name='Herb', birthday=date(1950, 5, 5), is_relative=False)

```

更新，修改实例可以调用save()方法做持久化的修改，现在修改Grandma的名字，然后保存在数据库中

```python
grandma.name = 'Grandma L'
grandma.save()

```

现在有三条people记录在数据库中，那开始添加一个pets吧，Grandma不喜欢屋子里有动物，所以她不需要，单数Herb是一个动物爱好者

```python
bob_kitty = Pet.create(owner=uncle_bob, name='Kitty', animal_type='cat')
herb_fido = Pet.create(owner=herb, name='Fido', animal_type='dog')
herb_mittens = Pet.create(owner=herb, name='Mittens', animal_type='cat')
herb_mittens_jr = Pet.create(owner=herb, name='Mittens Jr', animal_type='cat')

```

经过一段漫长的生命周期，Mittens 生病了，然后就挂了。现在需要将它从数据库中删除。

```python
herb_mittents.delete_instance()

```

delete_instance()方法的返回值是数据库中被删除的记录数

#### 数据检索

数据库的好处就是它允许我们使用查询语句检索数据，关系型数据库非常适合做[即席查询](https://en.wikipedia.org/wiki/Ad_hoc)

##### 获取单条数据

在库里检索Grandma's的记录，为了获取一条记录，使用SelectQuery.get()

```
grandma = Person.select().where(Person.name == 'Grandma L').get()

```

我们也可以使用等效速记法Model.get()

```
grandma = Person.get(Person.name == 'Grandma L')

```

##### 获取多条记录

列出所有person的名字

```python
for person in Person.select():
   print person.name,person.relative 

```

列出所有的cat和它们owner的名字

```python
query = Pet.select().where(Pet.animal_type = 'cat')
for cat in query:
    print cat.name,cat.owner.name

```

上面的查询有一个很大的问题，因为我们访问pet.owner.name时，我们没有在原始查询中获取此值，pewwee就不得不再做一次查询，来获取pet的owner。这种行为被称作N+1,通常是要被避免的。

我们可以避免这种额外的查询，通过使用join同时查询pet和person

```python
query = Pet.select(Pet,Person).join(person).where(Pet.animal_type == 'cat')
for pet in query:
    print pet.name ,pet.owner.name

```

现在我们获取Bob的所有宠物

```python
for pet in Pet.select().join(Person).where(Person.name == 'Bob')
    print pet.name

```

我们可以在这里做另一件很酷的事来得到Bob的宠物。既然我们已经有一个对象来代表Bob，我们可以这样做

```python
for pet in Pet.select().where(Pet.ower == uncle_bob)
    print pet.name

```

为了让这些数据按字母排序，我们可以使用order_by()语句

```python
for pet in Pet.select().where(Pet.owner == uncle_bob).order_by(Pet.name):
    print pet.name

```

按年龄从小到大列出people

```python
for person in Person.select().order_by(Perso.birthday.desc()):
    print person.name, person.birthday

```

列出所有的people和他们的宠物信息

```python
for person in Person.select():
    print person.name, person.pets.count
    for pet in person.pets:
        print pet.name, pet.animal_type

```

我们又一次遇到一个n + 1的查询行为的经典例子。我们可以通过执行连接和聚合记录来避免这种情况。

```python
subquery = Pet.select(fn.COUNT(pet.id)).where(Pet.owner == Person.id)
query = (Person.select(Person, pet, subquery.alias('pet_count')).join(Pet, JOIN.LEFT_OUTER).order_by(Person.name))
for person in query.aggregate_rows():
    print person.name, person.pets.count
    for pet in person.pets:
        print pet.name, pet.animal_type

``` 

即使我们创建了分开的子查询，但是实际执行了一次查询。
最后，做一次复杂的查询吧，获取生日符合以下条件的people:

* before 1940 (grandma)
* after 1959 (bob)

```python
d1940 = date(1940, 1, 1)
d1960 = date(1960, 1, 1)
query = (Person.select().where((Person.birthday < d1940) | (Perosn.birthday > d1960)))
for person in query:
    print person.name, person.birthday

```

做一次相反的查询，获取在1940年和1960年之间的people

```python
d1940 = date(1940, 1, 1)
d1960 = date(1960, 1, 1)
query = (Person.select().where((Person.birthday > d1940) & (Perosn.birthday < d1960)))
for person in query:
    print person.name, person.birthday

```

做最后一次查询，这里将用到一个SQL函数获取名字开头是'g'|'G'的people

```python
expression = (fn.Lower(fn.Substr(Person.name, 1, 1)) == 'g')
for person in Person.select().where(experssion):
    print person.name

```

数据库操作完毕，需要关闭数据库连接

```
db.close()

```

这只是最基本的，你可以让你的查询更复杂。

其他SQL语句也可用，例如

* group_by()
* having()
* limit() and offset()

#### 与现有数据库协同工作

如果你已经有了数据库，你可以使用pwiz（一种模型生成器）自动生成peewee模型；例如，假设我有postgresql数据库，命名为charles_blog,我将执行:

```
python -m pwiz -e postgresql charles_blog > blog_models.py

```


