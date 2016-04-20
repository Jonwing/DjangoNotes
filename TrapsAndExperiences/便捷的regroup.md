要把原先一个列表的页面修改，这个列表是把好几个model的内容整理到一行行显示的，像这样：



| type1| category1 | key1 | value1 | options|

----------------------------------------------

| type1| category2 | key1 | value1 | options|

----------------------------------------------

| type1| category2 | key2 | value2 | options|

----------------------------------------------

| type2| category2 | key1 | value1 | options|

......



想把row中重复的信息合起来，做成合并单元格的效果，便于分辨。

```python

Type:

    name = CharField()

    ...



Instance:

    type = ForeignKey(Type)

    categories = ManyToManyKey(AnotherModel)

    ...



Key:

    key = CharField()

    ...



Value:

    key = ForeignKey(Key)

    value = CharField()

    instance = ForeignKey(Instance)

    ...
```


这些单元来自不同的通过外键连起来的model：

用了regroup就方便很多。

传入template的value_list

```python

value_list = Value.objects.select_related('instance__type', 'instance__categories').order_by('instance__type', 'instance__categories')

```


template:

```html

<table class="table table-bordered">
    <thead>
        <tr>
          xxx
        </tr>
    </thead>
    <tbody>
        {% regroup value_list by instance.type.name as valuelist_by_type %}

        {% for value_type in valuelist_by_type %}
            <tr>
                <td rowspan="{{ value_type.list|length }}">{{ value_type.grouper }}</td>
                {% regroup value_type.list by instance.categories as valuelist_by_ctg %}
                {% for value_ctg in valuelist_by_ctg %}
                    <td rowspan="{{ value_ctg.list|length }}">
                    {% if value_ctg.grouper.all %}
                        {% for ctg in value_ctg.grouper.all %}
                            {{ ctg }}{% if not forloop.last %} , {% endif %}
                        {% endfor %}
                    {% else %}
                        ({% trans "Undefined"%})
                    {% endif %}
                    </td>
                    {% for value in value_ctg.list %}
                        <td>{{ value.key }}</td>
                        <td>{{ value.value }}</td>
                        <td>{{ option }}</td>
            </tr>
                    {% endfor %}
                {% endfor %}
            {% endfor %}
</tbody>
</table>

```

