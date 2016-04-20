今天遇到一个filter的问题

涉案的两个Model大概是这样的：

```python

ModelA：

    event1_begined_at = models.DatetimeField()

    event2_ended_at = models.DatetimeField()

    ...



ModelB:

    ma = models.ForeignKey(ModelA)

    attr_a = xxx

    attr_b = yyy
```


我在做一个链式filter：

```python

something = ModelA.objects.filter(event1_begin_at__lte=today, event2_ended_at__gte=today).filter(modelb__attr_a=x)
```

我预想的是 筛选出那些今天还有效（第一个filter条件）的且以其为外键的ModelB实例中attr_a为x（第二个条件）的ModelA的实例。

它返回的确实也符合预期，然而，居然有重复。



看了一下原来是filter中使用了逆向外键的关系，它会去逐一考察那些外键指向(符合第一个filter条件的)ModelA的ModelB实例，如果attr_a符合条件，就添加到QuerySet中，因此，ModelA的实例a有多少个符合第二个filter条件的ModelB 实例，返回的结果中就包含多少个a。

-----

了解完这个后，又看到这个文档： [django docs](https://docs.djangoproject.com/en/dev/topics/db/queries/#spanning-multi-valued-relationships)

说的是有关多值外键的链式filter机制，看了之后很有收获。

这里有一个SO上的例子：[Chaining multiple filter](https://stackoverflow.com/questions/8164675/chaining-multiple-filter-in-django-is-this-a-bug) 

虽然答案跟我说的有些差别，它主要说的是链式filter跟同条件的单个filter的机制和可能的不同。我上面说的是对multi-valued-relationships(ManyToManyKey, reverse ForeignKey...)进行filter可能会产生重复对象。但原理上应该是一样的。
