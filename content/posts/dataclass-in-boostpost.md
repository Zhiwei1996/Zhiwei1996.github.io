---
title: BoostPost - Data Class 最佳（误）实践
date: 2021-05-25
description: BoostPost 是一个辅助“更新优化”数据处理的工具组件，其旨在减轻为中间件写接口构造请求数据的负担。 输入新旧两批数据，输出完全适用于 API 接口的增、删、改数据，避免手动处理数据的繁琐。支持扩展，可以方便地添加额外的处理逻辑
tags: ["技术"]
---


## 前言

前一篇博客介绍了在数据处理工作中引入 `Data Class` 概念，当时的实现很不完善，只是灵机一动下花了半小时便写出了一个粗糙的原型库，在后续的工作中，我切合实际的需求场景为此进行了扩展和优化。现在，`BoostPost` 已经是一个能够帮助解决处理数据时大部分对比更新工作的较为完善的库了。

这里另起篇章深入介绍 `BoostPost` 的设计灵感和具体实现，以及在特定场景下数据处理工作中带来的效率提升。

## 正文

### 数据处理场景

有 API 接口定义如下：

* **URL**: /write/update_base
* **Method**: POST
* **Request Body**:

```json
{
  "t_people": {
    "insert": [
      {
        "body": {
          "gender": "Genderqueer",
          "first_name": "Tamma",
          "last_name": "Gledhall",
          "ip_address": "112.192.229.205",
          "email": "tgledhall2@oaic.gov.au"
        }
      }
    ],
    "update": [
      {
        "body": {
          "email": "bgraalman1@gmail.com"
        },
        "key": {
          "first_name": "Barr",
          "last_name": "Graalman"
        }
      }
    ],
    "delete": [
      {
        "key": {
          "first_name": "Delila",
          "last_name": "Coveny"
        }
      }
    ]
  }
}
```

需要从新旧两批数据生成以上更新数据格式，输入数据如下：

**对应 `MySQL` 表中展示**

新

| first_name | last_name | email                  | gender      | ip_address      |
| ---------- | --------- | ---------------------- | ----------- | --------------- |
| Barr       | Graalman  | bgraalman1@gmail.com   | Non-binary  | 66.234.0.116    |
| Tamma      | Gledhall  | tgledhall2@oaic.gov.au | Genderqueer | 112.192.229.205 |

旧

| id   | first_name | last_name | email                      | gender     | ip_address    |
| ---- | ---------- | --------- | -------------------------- | ---------- | ------------- |
| 1    | Delila     | Coveny    | dcoveny0@howstuffworks.com | Female     | 122.202.68.84 |
| 2    | Barr       | Graalman  | bgraalman1@digg.com        | Non-binary | 66.234.0.116  |

**对应 `JSON` 展示**

新

```json
[
  {
    "gender": "Non-binary",
    "first_name": "Barr",
    "last_name": "Graalman",
    "ip_address": "66.234.0.116",
    "email": "bgraalman1@gmail.com"
  },
  {
    "gender": "Genderqueer",
    "first_name": "Tamma",
    "last_name": "Gledhall",
    "ip_address": "112.192.229.205",
    "email": "tgledhall2@oaic.gov.au"
  }
]
```

旧

```json
[
  {
    "first_name": "Delila",
    "last_name": "Coveny",
    "gender": "Female",
    "email": "dcoveny0@howstuffworks.com",
    "ip_address": "122.202.68.84",
    "id": 1
  },
  {
    "first_name": "Barr",
    "last_name": "Graalman",
    "gender": "Non-binary",
    "email": "bgraalman1@digg.com",
    "ip_address": "66.234.0.116",
    "id": 2
  }
]
```

> 旧数据多了 `id` 字段，这是 `MySQL` 表的主键

如何构造一个更新请求的数据包，涉及 `Python` 程序中对 `list` 和 `dict` 对象的大量处理操作，也许你会写很多的 `for` 循环，还会有很多 `call-by-key` 和 `call-by-rank` 的取值赋值。能够想象到将会写出一堆多层嵌套的代码，最终只为了生成一条多层嵌套的 `JSON` 数据。

另外还需要考虑以下情况：

* 倘若换一个 API 接口定义，数据格式稍稍不一样
* 倘若当前维度只会增量更新，只有新增和更新的数据
* 倘若当前维度不会真的删除数据，而是通过标志位逻辑删除
* 倘若还有更新优先级，判断出某种来源的数据低优先级情况下不更新
* 倘若...

真实环境下会有很多特殊的业务逻辑，需要能够兼容和扩展。

### 不断重复的编码

再项目中的这些实现，每个人都有他自己的想法，每次都会从头至尾重新写一遍**更新优化**的处理流程。

梳理一下主要处理流程：

1. 取每条数据唯一键的值作为 `key`，整条数据作为 `value` 构造一个字典

2. `for ` 循环新数据，`key` 不在旧数据中的表示新增，在旧数据中的表示更新

3. `for` 循环旧数据，`key` 不在新数据中的表示删除

4. 对更新的数据项，逐个字段对比，只保留值有变化的字段

5. 对新增、更新、删除的所有数据项添加额外的元信息，合并生成接口所需的请求数据

    


观察项目中多处代码，总结有以上处理逻辑，每一处还有一点自己的特色。尽管大体逻辑类似，但并没有人将其抽离出来作为通用的处理模块，其实这是很容易便做成的事，些许细节可以不用考虑太多，后面可以不断优化迭代。

### Data Class 概念

分析上文中介绍的实现，取值为 `key`，再做交集、差集，这不就是集合思想嘛？

将数据项映射为 `key-value` 形式，是为了后续处理时方便地取完整数据和用 `key` 进行集合操作，这里 `key` 可以看做数据的索引，延伸一下，数据的索引好比 `Python` 对象的哈希值，`Python` 对象可以放进 `set`，直接利用现有的集合操作。

再延伸一下，每条数据对应一个 `Python` 类的实例，那么这个类自然而然就叫数据类。

**数据类**（Data Class）指拥有一些值域（fields），以及用于访问（读写〕这些值域的函数，除此之外一无长物。这样的classes只是一种「不会说话的数据容器」。

在 `Haskell` 中有关键词 `data`，可以定义新的数据类型，一定程度上可以看做是数据类。

在 `3.7` 之后的 `Python` 版本中，有个名为 `dataclasses` 的模块，它就是数据类概念在 `Python` 里的实现，对于数据处理非常方便。源于 `Python3.6` 时期的一个第三方库，因为太好用且使用广泛，在 `3.7` 之后被加入进了标准库里。以下是官方文档和原项目的 `GitHub` 链接

- 原项目地址: https://github.com/ericvsmith/dataclasses
- 官方文档: https://docs.python.org/3/library/dataclasses.html
- [PEP 557 – Data Classes](https://www.python.org/dev/peps/pep-0557/)

这里有一篇对比 `Data Class` 在 `Haskell` 和 `Python` 中的使用和差异的文章

- [Data classes](https://typeclasses.com/python/data-classes) - Type Classes

### 理想的功能模块

了解了数据类的概念，这时便可以大胆设想，数据项的处理都可以转换为对数据类的处理。数据项的唯一特征对应数据类的哈希值，两批数据类实例放进两个集合容器里找出交、差集，对于交集中的实例，就是待更新的数据项，在进行逐个字段值对比。

理想的情况下，仅需两三行代码便能完成一次“更新优化”处理，伪代码演示如下：

```python
boost = BoostPost(
	table='t_xxx',
    unique=['a', 'b']
)
boost.get_post(new_data, old_data)
print(boost.post_data)
```



这些只是最基础的场景，复杂场景下，还有前文中描述的各种特殊逻辑，需要支持。

通过一些具体的例子来描述：

**逻辑删除**

数据库表中有字段 `is_history` 标识数据项是否有效，置 `1` 时表示已删除。对应处理逻辑就是旧数据与新数据的差集，每一个数据类实例，生成相应的更新格式数据，其中只更新 `is_history` 为 `1`。

**来源优先级**

数据来源是分别是 A 和 ，其中 A 的优先级大于 B，在更新时，如果新数据是 B，旧数据是 A，那么这条数据项就直接跳过，不做更新。对应处理逻辑就是两个数据集合的交集，每一对新旧数据类实例都先比较来源优先级，其后再决定是否对比找出更新的字段。

**不同的接口数据格式**

不同于前文中定义的 API 接口，现有新的更新接口，其请求数据格式定义如下所示

```json
{
  "t_people": {
    "insert": [
      {
        "body": {
          "gender": "Genderqueer",
          "first_name": "Tamma",
          "last_name": "Gledhall",
          "ip_address": "112.192.229.205",
          "email": "tgledhall2@oaic.gov.au"
        },
        "split_params": "112.192.229.205"
      }
    ],
    "update": [
      {
        "body": {
          "email": "bgraalman1@gmail.com"
        },
        "split_params": "66.234.0.116",
        "key": {
          "first_name": "Barr",
          "last_name": "Graalman"
        }
      }
    ],
    "delete": [
      {
        "split_params": "122.202.68.84",
        "key": {
          "first_name": "Delila",
          "last_name": "Coveny"
        }
      }
    ]
  }
}
```

这里只是简单地添加了 `split_params` 参数，也许其它 API 接口的改动会很大。

要到达的效果是，对于以上的特殊场景都能够支持，可以是原生支持，可以是简单地只改动少数几行代码来支持，这里便需要对外提供扩展接口，方便二次开发。



### 动手实现

> 这里的代码隐去了一些无关紧要的细节和特殊的功能，了解大概设计思路即可

定义数据类基类

```python
class BaseDataNode(object):
    def __init__(self, data, uniq_keys, primary_key=None, **kwargs):
        self.data = data
        self.uniq_keys = uniq_keys
        self.pk = primary_key
        self.__unique = tuple([self.data[k] for k in self.uniq_keys])
	
    # 数据项的唯一键字段的值，可以视作索引
    @property
    def unique(self):
        return self.__unique

    def __eq__(self, other):
        if type(self) is type(other):
            return self.unique == other.unique
        return False

    # 数据类的哈希值
    def __hash__(self):
        return hash(self.unique)

    def _remove(self, fields):
        """
        :param fields: fields to remove form data
        :type fields : [list]

        """
        for x in fields:
            self.data.pop(x, None)
	
    def _diff(self, other, write_protect=None):
        # write_protect 参数是为了支持更新时的空保护，有些字段是不想被更新置空的
        keys = self.data.keys() & other.data.keys()
        fields = {
            k: self.data[k] for k in keys if not compare(self.data[k], other.data[k])
        }
        # protect values, prevent set empty
        if write_protect:
            for k in keys:
                if not self.data[k] and k in write_protect:
                    fields.pop(k, None)
        return fields

    def keygen(self, other=None):
        """
        :param other:
        :type other:

        """
        # unique keys between self and other are identical
        # pk must be getted from other
        if not other:
            other = self

        if other.pk and other.data.get(other.pk):
            key = {other.pk: other.data.get(other.pk)}
        else:
            key = {k: other.data[k] for k in other.uniq_keys}
        return key
	
    # 这是辅助方法，为了能自动序列化和反序列化 JSON 格式的字段数据
    def datagen(self, data, json_keys=None, encode=False):
        """
        :param data:
        :param json_keys: fields that values are JSON in data
        :param encode: use `json.dumps` to process data if True else `json.loads`

        """
        if not json_keys:
            return data

        ret = copy.deepcopy(data)
        for k in json_keys:
            if is_json(data.get(k), encode):
                if encode:
                    ret[k] = json.dumps(data[k], ensure_ascii=False)
                else:
                    ret[k] = json.loads(data[k])
        return ret

    # 这是数据类子类必须要实现的方法，对应生成增、删、改的更新数据
    def post_insert(self, *args, **kwargs):
        raise NotImplementedError

    def post_update(self, other, write_protect=None, exclude=None, *args, **kwargs):
        """
        :param other: Node build of data fetched from mysql
        :param exclude: fields exclued during post optimizing
        :param write_protect: fields need write-protecting, forbid to set empty

        """
        raise NotImplementedError

    def post_delete(self, *args, **kwargs):
        raise NotImplementedError

```

为 mysql proxy API 接口定义数据类

```python
class MyApiDataNode(BaseDataNode):
    """
    :param data:
    :param uniq_keys:
    :param primary_key: [optional]
    :param split_param: [optional]
    :param json_keys: [optional] auto loads/dumps json values

    """

    def __init__(
        self,
        data,
        uniq_keys,
        primary_key=None,
        json_keys=None,
        split_param=None,
        **kwargs
    ):
        self.json_keys = json_keys
        self.split_param = split_param
        super(MyApiDataNode, self).__init__(
            self.datagen(data, self.json_keys), uniq_keys, primary_key, **kwargs
        )

    def post_insert(self, *args, **kwargs):
        return {
            "body": self.datagen(self.data, self.json_keys, encode=True),
            "split_param": self.data.get(self.split_param) or "",
        }

    def post_update(self, other, write_protect=None, exclude=None, *args, **kwargs):
        """
        :param other        : Node build of data fetched from mysql
        :param exclude      : fields exclued during post optimizing
        :param write_protect: fields need write-protecting, forbid to set empty

        """
        key = self.keygen(other)

        # fields not to update
        if exclude:
            self._remove(exclude)

        body = self._diff(other, write_protect)
        if not body:
            log.debug("{}: update nothing due to no different fields".format(self))
            return dict()

        return {
            "key": key,
            "body": self.datagen(body, json_keys=self.json_keys, encode=True),
            "split_param": self.data.get(self.split_param) or "",
        }

    def post_delete(self, del_flag=0, *args, **kwargs):
        """
        :param del_flag: 0: not delete, 1: logical delete, 2: physical delete

        """
        # it is from mysql when executing delete
        key = self.keygen()

        if del_flag == 1:
            # 逻辑删除
            # <u_tags> write-protect
            tag = int(self.data.get("u_tags") or 0)
            if tag == 1 or tag == 2:
                log.debug("delete nothing due to u_tags is protected or hidden")
                return dict()
            else:
                tag = 1
            return {
                "key": key,
                "body": {"u_tags": tag},
                "split_param": self.data.get(self.split_param) or "",
            }

        elif del_flag == 2:
            # 物理删除
            return {"key": key, "split_param": self.data.get(self.split_param) or ""}

        else:
            return dict()
```

更新优化主模块

```python
class BoostPost(object):
    """Generate post data of business proxy API, It's convenient and efficient,
    can save you from a mess
	"""
    def __init__(self, table_name, uniq_keys, datanode, primary_key=None, **kwargs):
        self.table_name = table_name
        self.uniq_keys = uniq_keys  # 表的唯一键字段
        self.datanode = datanode  # 指定数据类
        self.primary_key = primary_key  # 表的主键，可以在更新和删除时代替 uniq_keys 生成的 key
        self.datanode_kwargs = kwargs
        self.nodes1 = None
        self.nodes2 = None
        self.insert = []
        self.update = []
        self.delete = []
        self.status = {"insert": False, "update": False, "delete": False}
        self.__post = defaultdict(dict)
	
    @property
    def post(self):
        return self.__post if self.__post.get(self.table_name) else {}
	
    # 该方法将数据项转换成数据类实例
    def spawn(self, records_new, records_old):
        """spwan instances of datanode from records

        :param records_new         : old records read from mysql
        :type  records_new         : [list]
        :param records_old         : new records to update
        :type  records_old         : [list]

        """
        if isinstance(self.nodes1, set) and isinstance(self.nodes2, set):
            raise BoostPostError("already spawned, can not spawn twice")

        self.nodes1 = {
            self.datanode(
                _, self.uniq_keys, primary_key=self.primary_key, **self.datanode_kwargs
            )
            for _ in records_new
        }
        self.nodes2 = {
            self.datanode(
                _, self.uniq_keys, primary_key=self.primary_key, **self.datanode_kwargs
            )
            for _ in records_old
        }
	
    # 这里对应数据类的 `post_xx` 方法
    def post_insert(self, *args, **kwargs):
        if self.status["insert"]:
            raise BoostPostError("already executed `post_insert`")

        self.status["insert"] = True
        self.insert.extend(
            [node.post_insert(*args, **kwargs) for node in (self.nodes1 - self.nodes2)]
        )
        self.insert = [_ for _ in self.insert if _]
        if len(self.insert) > 0:
            self.__post[self.table_name].setdefault("insert", []).extend(self.insert)

    def post_update(self, write_protect=None, exclude=None, *args, **kwargs):
        """
        :param write_protect: fields need write-protecting, forbid to set empty
        :param exclude: fields exclued during post optimizing

        """
        if self.status["update"]:
            raise BoostPostError("already executed `post_update`")

        self.status["update"] = True
        # select nodes to update
        mine = [node for node in self.nodes1 if node in self.nodes2]
        others = [node for node in self.nodes2 if node in self.nodes1]
        if len(mine) == 0 and len(others) == 0:
            self.update = []
        else:
            # sort nodes in order to keep one-to-one correspondence
            self.update = [
                x.post_update(
                    y, write_protect=write_protect, exclude=exclude, *args, **kwargs
                )
                for x, y in zip(
                    sorted(mine, key=lambda x: x.unique),
                    sorted(others, key=lambda x: x.unique),
                )
            ]
        self.update = [_ for _ in self.update if _]
        if len(self.update) > 0:
            self.__post[self.table_name].setdefault("update", []).extend(self.update)

    def post_delete(self, del_flag=2, *args, **kwargs):
        """
        :param del_flag: 0: not delete, 1: logical delete, 2: physical delete

        """
        if self.status["delete"]:
            raise BoostPostError("already executed `post_delete`")

        self.status["delete"] = True
        if del_flag == 0:
            return

        self.delete.extend(
            [
                node.post_delete(del_flag=del_flag, *args, **kwargs)
                for node in (self.nodes2 - self.nodes1)
            ]
        )
        self.delete = [_ for _ in self.delete if _]
        if not self.delete:
            return

        if del_flag == 1:
            self.__post[self.table_name].setdefault("update", []).extend(self.delete)
            self.delete *= 0
        elif del_flag == 2:
            self.__post[self.table_name].setdefault("delete", []).extend(self.delete)
        else:
            raise BoostPostError("unrecongnized del_flag: {}".format(del_flag))
```

#### 使用示例

```python
>>> from boostpost.nodes import MyApiDataNode
>>> from boostpost import BoostPost
>>> boost = BoostPost(
>>>     't_people',
>>>     uniq_keys=['first_name', 'last_name'],
>>>     datanode=MyApiDataNode,
>>>     split_param='ip_address'
>>> )
>>> boost.spawn(new_data, old_data)
>>> boost.post_insert()
>>> boost.post_update(exclude=['create_time'])
>>> boost.post_delete(del_flag=1)
```

#### 扩展示例

更新来源优先级

```python
class Node(MyApiDataNode):
    @property
    def priority(self):
        return 5 if self.data["source"] == "001" else 1

    def post_update(self, other, write_protect=None, exclude=None, *args, **kwargs):
        if self.priority < other.priority:
            return
        return super(Node, self).post_update(other, write_protect=write_protect, exclude=exclude, *args, **kwargs)
```

通过 `is_history` 逻辑删除

```python
class Node(MyApiDataNode):
    def post_delete(self, other, del_flag=1, *args, **kwargs):
        flag = other.data["is_history"]
        if flag == 1:
            return
        return {
            "body": {"is_history": 1},
            "key": {...}
        }
```



## 结语

项目代码质量是在持续不断的重构中提高的，需要多阅读优秀源码，借鉴其中好的设计，重构中很重要的一点是需要考虑“高内聚、低耦合”。

再次推荐：

- [《重构 - 改善既有代码的设计》](https://book.douban.com/subject/4262627/)
- [《编写可读代码的艺术》](https://book.douban.com/subject/10797189/)

