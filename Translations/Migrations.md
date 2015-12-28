source https://docs.djangoproject.com/en/1.9/topics/migrations/

## Migrations

Migrations(迁移)是Django的一种将你对models的修改(增加字段，删除模型等)同步(传递)到数据库的方式。他大多数情况下是可以自动进行的，但你需要知道什么时候去生成migrations，以及什么时候跑migrations，还有一些可能遇到的普遍问题。



----



## Migrations命令

下列命令用于与migrations和Django的数据库处理程序进行交互：

> - migrate： 负责应用migrations和回滚，以及列出它们的状态。

> - makemigrations: 基于你对models的更改来创建新的migrations

> - sqlmigrate ：显示一个migration的语句

可以把migrations看作是数据库的一个版本控制系统，其中，**makemigrations**负责将你的models更改打包到单独的migrations文件——类似于commit——而**migrate**则是将这些更改应用到你的数据库中去。



每一个应用的migrations 文件都放在本应用下的名为migrations的文件夹里，从Django设计上来说，它也是该应用代码的一部分，所以，你应该在你的开发环境中生成这些migrations文件，然后，在你同事的环境，集成测试环境，以及生产环境都相同地跑这统一的 migrations文件



在相同的数据集下，migrations会以同样的方式运行，产出一致的结果。这意味着在同样的环境下，你在生产环境，集成测试环境得到的结果跟在生产环境是一样的。

Django会将所有对models的改动都反映到migrations文件中——即便是那些不会影响数据库的options——因为重建一个字段的正确方法只能是：记录所有的更改历史。而且，在后续的数据迁移中，你也可能需要这些options(比方说，你设置了一个自定义校验器)。



---

## 后台支持

支持Django 自带的后台，如果第三方自己通过SchemaEditor实现了对对数据库操作的支持，那也是可以的。



### PostgreSQL

PostgreSQL是所有支持的数据库里效果最好的，唯一需要注意的是新增一列带有默认值的字段会导致对整个表的覆写（增加默认值）, 耗时跟表的大小成正比。因此，推荐你在新增字段的时候设置null=True， 这样字段会马上添加到表中。



### MySQL

MySQL在进行数据库更改操作时缺少对事务的支持，意味着如果一个migration执行失败了，它不会自动回滚到执行前的点，需要你手动去撤销已经执行的那些部分，以便重新执行migration。

另外，几乎所有的数据表操作都会使MySQL完全覆写整个表，增加或删除列的耗时也与现有数据的规模成正比。在一些老(慢)机器上甚至可能低于百万行每分钟——也就是说，在有几百万行数据的表中增加一或几个字段可能会使你的网站挂起超过十分钟。



### SQLite

SQLite内建的数据库更改操作支持非常少，基于这个原因，Django通过一下的步骤来达到效仿(一些不被支持的操作)的目

+ 基于新的表结构创建新表

+ 将原表的数据复制过来

+ 删除旧表

+ 以旧表的名字重命名新表



一般情况下这个流程能正常工作，但可能会很慢或者碰到神出鬼没的bug。所以，除非你熟知它的限制和隐患，否则并不建议你在生产环境中的SQLite跑migrations。Django自带SQLite支持的设计初衷是让开发者在本地环境中开发简介的项目不用使用完全的数据库。



----

## 工作流

使用migrations非常简单。首先，更改你的models——比如，增加一个字段或删除一个model，然后，执行makemigrations：

**$ python manage.py makemigrations**
```python
Migrations for 'books':
  0003_auto.py:
    - Alter field author on book
```
这个操作会扫描你的models并将其与包含在migration文件中的现有版本进行比较，之后会得出一个新的migration集。有必要去读一下这个文件看看migration认为你更改了什么——它并不完全准确，改变比较复杂时候可能不会产出你期望的结果。

有了migration文件，你就可以执行migrate将它们应用到数据库了。注意要确保它们像你期望一样执行。

    **$ python manage.py migrate**

```shell

Operations to perform:

        Apply all migrations: books

    Running migrations:

         Rendering model states... DONE

        Applying books.0003_auto... OK

```

执行成功之后，就可以将此次的migration文件commit到你的版本控制系统，这样别的开发者checkout这支代码，他就既能得到models的改变，又能有相应的migration文件了。

想给migration文件起个有意义的名字？ 用 **--name** 参数吧。

**$ python manage.py makemigrations --name changed_my_model your_app_label**


---

## 版本控制

上面说产生的migration文件加入版本控制，那么很可能会碰到的一种情况是你跟别人的分支同时commit了相同app下的migration文件，他们拥有相同的序号(00**_***这种)

不要慌，这些序号只是方便开发者，Django只关心每个migration文件是不是有不同的名字。Migration其实是通过migration文件（包括之前的文件）的内容来判断这是migration文件是否有序的（主要是依赖链）。

出现上面这种情况时，Django会给出提示还有一些选项，如果它觉得足够安全的话，它会自动帮你线性化这两个migration文件。不自动也没有问题，自己手动修改migration文件也是不难的。会在后面的migration files中说明。



### 依赖项

Mifgration是单个app向的，有时很难一次性地实现你在model里定义的那些复杂的表和关系，例如，在books应用里增加一个指向author应用的外键，这样，当你运行books的makemigrations时，生成的migration文件里会有一个指向author的migration文件的依赖项。

依赖项的意思是，当你应用这些migration文件时，会先应用author里的migration来创建表，以供之后外键引用，然后才是应用books的migration文件，建立外键引用。不是这样的话migration会尝试建立指向不存在的表的引用，当然，你的数据库会报错了。

This dependency behavior affects most migration operations where you restrict to a single app. Restricting to a single app (either inmakemigrations or migrate) is a best-efforts promise, and not a guarantee; any other apps that need to be used to get dependencies correct will be.（这段不知道怎么翻译。。）



## Migration 文件

这里的Migration文件是指存储在磁盘上的migrations，它们其实就是普通的python文件。

基本的migration文件长得大概是这样的：

```python

from django.db import migrations, models

class Migration(migrations.Migration):

    dependencies = [("migrations", "0001_initial")]

    operations = [migrations.DeleteModel("Tribble"),migrations.AddField("Author", "rating", models.IntegerField(default=0)),]

```

当Django载入一个migration文件的时候主要是找一个名叫Migration的类(它是django.db.migrations.Migration的子类), 然后检查它的四个属性，其中两个是最常用到的：

+ **dependencies**：    一个依赖项的列表

+ **operations**：    一个定义了本migration需要做什么的Operations列表



operations是关键，它由一些告诉Django需要更改哪些数据表的陈述性语句组成。Django将这些operations扫描并载入到内存中，之后用它来生成最终改变数据库的SQL语句。

另外保存在内存中的结构体(structure)也用来比对当前的model状态；Django按顺序遍历这些内存结构体，得出自你最后一次跑makemigrations时的model状态，以之来与当前写在models.py中的model进行比较，从而得出更改了什么。

一般来说，你是不需要手动编辑migration文件的，不过必要时手动编辑也是完全可以的（一些复杂的不能自动检测的operations或者只能手动实现的migration）（个人经验，也有operations顺序错误需要手动排序的……），不要害怕，不难。



### 自定义字段

You can’t modify the number of positional arguments in an already migrated custom field without raising a TypeError. The old migration will call the modified __init__ method with the old signature. So if you need a new argument, please create a keyword argument and add something like assert 'argument_name' in kwargs in the constructor.



----

## Model Managers

可以把你的managers序列化到migration中，这样在operations中使用RunPython操作时就可以用上。方法是在manager类中定义use_in_migrations字段：

```python

class MyManager(models.Manager):

    use_in_migrations = True

class MyModel(models.Model):

    objects = MyManager()

```

如果manager类是用from_queryset()函数动态生成的，你需要从生成的类中继承以使manager类能够被导入：

```python

class MyManager(MyBaseManager.from_queryset(CustomQuerySet)):

    use_in_migrations = True

class MyModel(models.Model):

    objects = MyManager()

```



----
## 初始Migrations


顾名思义，就是第一个版本的migration文件，起到在数据库中建立对应model的表的作用。通常来说一个app只有一个初始migration文件，一些复杂的情况下可能有两个以上。

初始migration会在migration类中用 **initial = True** 标出。如果一个migration没有在migration类中找到initial属性，但是是该app的第一个migration文件（如没有到其他同app下的migration文件的依赖），也会被认作是initial。

有一个migrate 的参数是 **--fake-initial**, 当指定这个参数时，初始migration中的CreateModel、AddField等操作都会被认为是已经存在而不去执行，只是标记为已执行。没有 **--fake-initial**，初始migration与其他migration在执行上是一样的。



### 添加migrations到apps中

简单粗暴，它们已经预设好接受migrations的了，只需要model有改动时运行makemigrations就可以。

如果你的model已经以及数据库中对应的表已经存在（比如从老版本的django升级过来），只需要几个简单的步骤就可以使之与migrations接轨，如下：

    **$ python manage.py makemigraitons your_app_label**

上述命令的作用是为你指定的app创建初始migration，之后，运行 **python manage.py migrate --fake-initial** ，Django会检测初始migration和数据库中对应的表，将该初始migration标记为applied(其实是在数据库django_migrations中增加一条记录)。如果没有**--fake-initial**参数，会报table already existed的错误。

注意这仅仅在下面两者的基础上才会起作用：

    1. 在数据库表建立之后没有对model进行过改动。要使migrations正常运行，必须先执行初始migration，再对models更改。原因如前所述，migration检测改动是基于migration文件，而不是基于数据库，所以你的初始migration一定要与数据库匹配。

    2. 没有手动更改过数据库。Django无法侦测数据库的改动，如果你更改了，以后migration试图更改这些与models不匹配的部分时就很可能报错。

## 历史Models

当你跑migrations时，Django的工作是基于存储在migrations文件中的历史版本的。所以当你在migration中加入RunPython操作，或在数据库路由(database routers)中有allow_migrate方法，就会面临models的历史版本问题。

由于不可能序列化任意多的Python代码，所以这些历史models不会带有任何自定义的方法。它们只会有相同的fields， 关系，managers（那些设置了use_in_migrations = True的）以及Meta 选项。

注意：

    > 上述的意思是你在migration中是不会调用到自定义的save()方法的，也不会有任何自定义构造器或者实例方法。要计划好喔！



在field option中对函数的引用(如upload_to和limit_choices_to等)，还有设置了use_in_migrations = True的model manager 都是可以序列化到migrations里的，因此，那些被migrations引用的函数和类都是需要保留的。另外，自定义的model fields因为要在migrations中直接import，也是要保留。

还有， model的基类是以指针的方式保存的，如果有migration包含了对这些基类的引用，同样，这些基类也应该被保留。从好的方面说，这些基类的managers和方法可以自然地继承下来，嗯，如果你真的需要访问它们可以选择把它们都移到一个超类(superclass)。

~~/* 这段翻得挫 */~~



----

## 删除model字段时需要注意的

跟上述“对历史函数的引用”相似，移除一个在旧的migration里引用过的model字段也可能面临同样的问题。

为了解决这个问题，Django提供了一个model属性来通过system checks framework帮助处理那些过期的字段。

像下面一样，将system

[to be continue...]



----

## 数据迁移(DATA MIGRATIONS)

上面说的基本是数据表的迁移，你也可以用migration来做数据迁移，它甚至是可以跟表迁移(schema migration)同时进行的。

数据迁移的指令最好独立成自己的migration，以便与表迁移区分开来。

Django不会像表迁移一样自动帮你生成migration文件，需要你自己手动来写。这并不是一件很难的事情，毕竟migraiton是由operations组成的，你只需自己构造operations就好了。自造operations的方式通常是RunPython。

首先，创建一个空的migration文件

**$ python manage.py makemigrations --empty your_app_label**

生成的文件长这样：

```python

# -*- coding: utf-8 -*-

# Generated by Django A.B on YYYY-MM-DD HH:MM

from __future__ import unicode_literals

from django.db import migrations, models

class Migration(migrations.Migration):

    initial = True

    dependencies = [

        ('yourappname', '0001_initial'),

    ]

    operations = [

    ]

```

现在，你要做的是实现一些函数并传入RunPython中，RunPython接受可调用的参数，这个参数需要接受两个参数：第一个是app_registry, 拥有本app下所有model的历史版本; 第二个是SchemaEditor，你可以用它来做一些数据库的编辑工作（需要注意的是，这样做可能会影响autodetector的效果）。



是时候举个栗子了，现在我们的表新加入了字段name ， 以前有first_name 和 last_name，于是我们决定对已有的数据，用last_name+first_name来拼成name。代码如下：

```python

# -*- coding: utf-8 -*-

from __future__ import unicode_literals

from django.db import migrations, models



def combine_names(apps, schema_editor):

    # We can't import the Person model directly as it may be a newer

    # version than this migration expects. We use the historical version.

    Person = apps.get_model("yourappname", "Person")

    for person in Person.objects.all():

        person.name = "%s %s" % (person.first_name, person.last_name)

        person.save()



class Migration(migrations.Migration):

    initial = True

    dependencies = [

        ('yourappname', '0001_initial'),

    ]

    operations = [

        migrations.RunPython(combine_names),

    ]

```



以上完成后，就可以跑**python manage.py migrate**来执行数据迁移了。

还可以给RunPython传入第二个可调用的函数，它是用来当第一个函数执行失败的时候回退所执行的函数。这个参数没有指明的时候，migrate失败将会引发异常。
