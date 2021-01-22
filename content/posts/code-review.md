---
title: 一次改善既有代码的设计
date: 2020-07-18
description: 通过 Dataclass 尝试对数据清洗中复杂数据的处理逻辑进行优化
tags: ["技术"]
---


## 前言

作为一名数据清洗程序员，平时的主要工作是搬运和规范化数据，在 ETL 中处于 T(Transform) 和 L(Load) 的阶段。在我们的线上业务数据实时更新流程里，对于每张 `MySQL` 表都要事先对比新旧数据，只提交更新变化的字段，即所谓真实更新，然而因为祖传代码和同事之间开发规范不统一，对于真实更新的处理存在多份不同的实现，导致了大量重复代码和难以维护的问题，我决定封装这一业务逻辑，将其统一和规范化，解决以上暴露出的问题。


## 正文

### 设计思考

封装这一业务，需要考虑兼容性和可扩展性，其一要能够兼容所有数据维度（每个维度对应一张表），其二便于用户扩展添加模块的功能，以应对特殊业务处理逻辑需求的场景。以上两点是整个设计过程中必须记住的，否则又会是多了一份重复代码，使原有的问题更严重。

观察业务，处理的数据为 `JSON` 格式，映射到 `Python` 里就是一个字典结构，每次更新时都会拿到多条这样的新数据，再与数据库里的旧数据对比去重，并找出每条数据变化了的字段，最后请求 `API` 接口更新。原有的实现大同小异，都是直接处理 `Python` 字典数据，去重逻辑为遍历循环作对比，并且衍生了很多不够通用的辅助功能函数，晦涩难懂，不好测试，简直维护地狱。

有个概念叫 `数据类`， 每一种数据可以封装成一个类，便于对数据做复杂的逻辑处理，此次代码优化也是以此概念为基础，把处理 `Python` 字典转换成处理数据类，如此一来可以简化工作，这里推荐两篇文章

- [Data classes](https://typeclasses.com/python/data-classes) - Type Classes
- [Python 工匠：做一个精通规则的玩家](https://www.zlovezl.cn/articles/a-good-player-know-the-rules/)

在 `3.7` 之后的 `Python` 版本中，有个名为 `dataclasses` 的模块，它就是数据类在 `Python` 里的通用模块实现，非常便于处理数据。最开始是 `Python3.6` 时候的一个第三方库，因为太好用且使用广泛，在 `3.7` 之后被加入进了标准库里。以下官方文档和原项目的 `GitHub` 链接，感兴趣可以看下。

- 原项目地址: https://github.com/ericvsmith/dataclasses
- 官方文档: https://docs.python.org/3/library/dataclasses.html
- [PEP 557 -- Data Classes](https://www.python.org/dev/peps/pep-0557/)

PS: 我曾试图把该模块移植到 `Python2`，不过失败了 :(

 [失败的实现](https://github.com/Zhiwei1996/dataclasses)

### 实现

因为数据更新都是走接口，所以这里和直接连接数据库更新有所不同，最后更新的数据要构造成接口规范的格式，下面代码里可能会有些具体业务上的细节，这些都是细枝末节，可以忽略，了解大致思路即可。

#### 数据类

首先开始实现一个数据类雏形

```python
class DataNode(object):
    def __init__(self, data, uniq_keys, eid=None, primary_key=None):
        self.data = copy.deepcopy(data)
        # 这里因为业务上需要所以必须有 eid，忽略即可
        # `eid` required
        if not self.data.get('eid') and isinstance(eid, basestring) and len(eid) == 36:
            self.assign(self.data, eid=eid)
        elif self.data.get('eid'):
            pass
        else:
            raise DataNodeError(
                'construct DataNode failed due to without eid or invaild eid')
        self.uniq_keys = uniq_keys
        self.pk = primary_key
        self.__unique = tuple([self.data[k] for k in self.uniq_keys])

    @property
    def unique(self):
        return self.__unique
    
    def assign(self, data, *args, **kwargs):
        """ assign value into DataNode.data

        Example:
        >>> assign(data, eid='123', pk=123)
        >>> assign(data, {'eid': '123', 'pk': 123})
        >>> assign([data, (eid', '123'), ('pk', 123)])
        """
        for x in args:
            if isinstance(x, collections.Iterable):
                kwargs.update(dict(x))
        for k, v in kwargs.iteritems():
            data[k] = v
```

`data` 是处理的 `JSON` 数据，`uniq_keys` 对于表的唯一键字段，不一定是物理唯一，也可以是业务上的，即逻辑唯一，`primary_key` 是表的物理主键，可选，这里也是因为接口规范里的，有些维度数据更新时需要用主键。这里定义给数据类型定义一个 `unique` 属性，在后面的去重时将会使用到这个属性值，它应该是不可变的。

到这里目的已经很明显了，把每条 `JSON` （字典）数据映射成一个数据类的实例，然后再去处理，这时候就可以用 `Python` 内置的集合去做数据去重，那么接下来实现对象的 `hash` 值。

```python
def __eq__(self, other):
  if type(self) is type(other):
    return self.unique == other.unique
  return False

def __ne__(self, other):
  return not __eq__(self, other)

def __hash__(self):
  return hash(self.unique)
```

除了定义了类的 `__hash__` 方法，还得有比较方法 `__eq__`，`Python2` 中另外需要 `__ne__` 方法，有了这些方法类就可以放到集合里去作处理，也有了可比较的属性。

以上就是数据类基础的实现，下面主要是业务上的一些方法的实现。

实现数据类实例之间的 `diff`

```python
def _diff(self, other, write_protect):
  keys = self.data.viewkeys() & other.data.viewkeys()
  fields = {
    k: self.data[k]
    for k in keys if not compare(self.data[k], other.data[k])
  }
  # protect value, not allow update as empty
  if write_protect:
    for k in keys:
      if not self.data[k] and k in write_protect:
        fields.pop(k, None)
      
  return fields
```

`write_protect` 参数是为了支持更新时的空保护，有些字段是不想被更新置空的

以下是接口规范的数据格式构造相关方法，这里生成接口更新时需要的 `key`，可以选择用物理主键还是唯一键字段

```python
def keygen(self, other=None):
    """
    :param other:
    :type other : [DataNode]

    """
    # unique keys are same between self and other
    # pk must be gained from other
    if not other:
      other = self

    if other.pk and other.data.get(other.pk):
        key = {other.pk: other.data.get(other.pk)}
    else:
        key = {k: other.data[k] for k in other.uniq_keys}
    # key without eid will cause "internal parse failed, errcode: -3"
    key['eid'] = other.data['eid']
    return key
```

下面同样是构造接口规范数据的方法，为增删改时的数据格式，不是重要的，这些方法可以被重写，以适应不同数据维度的处理逻辑

```python
    def get_insert(self, *args, **kwargs):
        # body without eid will cause "internal parse failed, errcode: -3"
        return {"body": self.data}

    def get_update(self, other, write_protect=None, exclude=None, *args, **kwargs):
        """
        :param other        : Node build of data fetched from mysql
        :param exclude      : fields exclued during post optimizing
        :param write_protect: fields need write-protecting, forbid to set empty

        """
        # unique keys are same between self and other
        # pk must from other
        key = self.keygen(other)

        # fields not to update
        if exclude:
            self._remove(exclude)

        body = self._diff(other, write_protect)
        if not body:
            logging.debug(
                '{}: update nothing due to no different fields'.format(self))
            return {}

        return {"key": key, "body": body}

    def get_delete(self, logic_del=True, *args, **kwargs):
        """
        :param logic_del: specify whether to use logic delete

        """
        # it is from mysql when executing delete
        key = self.keygen()

        if logic_del:
            # 逻辑删除
            # <u_tags> write-protect
            tag = int(self.data.get('u_tags') or 0)
            if tag == 1 or tag == 2:
                logging.debug(
                    'delete nothing due to u_tags is protected or hidden')
                return {}
            else:
                tag = 1

            return {"key": key, "body": {"u_tags": tag}}
        else:
            # 物理删除
            return {"key": key}
```

现在数据类的完整实现就做好了，每条待处理的 `JSON` 数据会被封装成一个数据类实例，可以用集合去重，对比找出每条数据真实更新了的字段，并调用相应的方法生成接口需要的数据格式。这个类是可以被继承并重写里面方法，对于特殊的维度数据实现其特殊的数据类，只需要作少量改动即可，并且也只局限在数据类内部的改动，不影响下面封装的统一处理数据类的代码逻辑，实现了数据与数据处理解耦。

#### 数据类处理

实现数据类实例的去重逻辑

```python
class BoostPost(object):
    def __init__(self, table_name, uniq_keys, primary_key=None, eid=None, datanode=DataNode):
        self.datanode = datanode
        self.table_name = table_name
        self.uniq_keys = uniq_keys
        self.primary_key = primary_key
        self.eid = eid
        self.nodes1 = None
        self.nodes2 = None
        self.insert = []
        self.update = []
        self.delete = []
        self.__post = {self.table_name: dict()}

    def spawn(self, records_new, records_old):
        """transform dict records to DataNode.
        NOTE: notice the order of params

        :param records_old         : old records from mysql
        :type  records_old         : [list]
        :param records_new         : new records to update
        :type  records_new         : [list]

        """
        if isinstance(self.nodes1, set) and isinstance(self.nodes2, set):
            raise BoostPostError('already spawned, do not spawn it again')

        try:
            self.nodes1 = {
                self.datanode(_,
                              self.uniq_keys,
                              eid=self.eid,
                              primary_key=self.primary_key)
                for _ in records_new
            }
            self.nodes2 = {
                self.datanode(_,
                              self.uniq_keys,
                              eid=self.eid,
                              primary_key=self.primary_key)
                for _ in records_old
            }
        except DataNodeError:
            raise BoostPostError(
                'data without eid or invaild eid')
```

`table_name` 是表名，即一个数据维度，`datanode` 对应数据维度的数据类，默认是上文实现的数据类基类，如果有特殊需求，那么就传入修改后的数据类。`self.nodes1` `self.nodes2` 是存放数据类实例的容器（集合），`self.insert` `self.update` `self.delete` 存放数据处理完后的接口所需的增删改数据。`spawn` 方法里把传入的所有 `JSON` 数据转换成了对应数据类实例。

找出新增数据，调用了 `DataNode` 的 `get_insert` 方法

```python
    def post_insert(self, *args, **kwargs):
        if len(self.insert) > 0:
            raise BoostPostError('already did it, do not invoke `post_insert` again')

        self.insert.extend([_.get_insert(*args, **kwargs) for _ in (self.nodes1 - self.nodes2)])
        self.insert = [_ for _ in self.insert if _]
        logging.debug('NODES TO INSERT: {}'.format(len(self.insert)))
        if len(self.insert) > 0:
            self.__post[self.table_name].setdefault('insert', []).extend(self.insert)
```

找出更新数据，调用了 `DataNode` 的 `get_update` 方法

```python
    def post_update(self, write_protect=None, exclude=None, *args, **kwargs):
        """
        :param write_protect: fields need write-protecting, forbid to set empty
        :param exclude      : fields exclued during post optimizing

        """
        # TODO: check `post_update` invoking
        if len(self.update) > 0:
            logging.warning('ensure it\'s the first to invoke `post_update`,'
                            'maybe cause wrong post data if not')

        # 计算出待更新的数据
        mine = [_ for _ in self.nodes1 if _ in self.nodes2]
        others = [_ for _ in self.nodes2 if _ in self.nodes1]
        if len(mine) == 0 and len(others) == 0:
            self.update = []
        else:
            # sort nodes in order to keep one-to-one correspondence
            self.update = [
                x.get_update(y, write_protect, exclude, *args, **kwargs)
                for x, y in zip(sorted(mine, key=lambda x: x.unique),
                                sorted(others, key=lambda x: x.unique))
            ]
        self.update = [_ for _ in self.update if _]
        logging.debug('NODES TO UPDATE: {}'.format(len(self.update)))
        if len(self.update) > 0:
            self.__post[self.table_name].setdefault('update', []).extend(self.update)
```

找出删除数据，同理

找出所有增删改数据，封装一下，简化步骤，一次调用全部
```python
    def post_all(self, *args, **kwargs):
        if len(self.insert) > 0 or len(self.update) > 0 or len(self.delete) > 0:
            raise BoostPostError('already posted, do not invoke `post_all`')

        # pop from kwargs avoid `TypeError: post_xxx() got multiple
        # values for keyword argument xxx`
        self.post_insert(self, *args, **kwargs)
        write_protect = kwargs.pop('write_protect', None)
        exclude = kwargs.pop('exclude', None)
        self.post_update(write_protect, exclude, *args, **kwargs)
        is_delete = kwargs.pop('is_delete', False)
        logic_del = kwargs.pop('logic_del', True)
        self.post_delete(is_delete, logic_del, *args, **kwargs)
```

返回完整的接口数据

```python
@property
def post(self):
    return self.__post if self.__post.get(self.table_name) else {}
```

做了以上封装之后我们就可以直接使用这个 `BoostPost` 模块去做真实更新处理，不必关心具体数据是什么样，各个维度数据放在类数据类中自定义和规范，数据被抽象出来作为一个广泛的数据类，数据处理逻辑不关心这个数据类内部的细节。

#### 使用示例

用一个简单的例子来看下封装的效果

```python
from boostpost import DataNode, BoostPost

boost = BoostPost('t_products', uniq_keys=['pid'])
boost.spawn(records_new, records_old)
boost.post_all(
    write_protect=['product_name'],
    exclude=['create_time'],
)
post_data = boost.post
```

## 结语

工作时我们会接手一堆祖传代码，可能难以阅读和维护，这时候就可以想办法考虑改善代码质量了，为了后面工作的效率和减少故障发生概率。

推荐两本书，讲解改善代码质量的

- [《重构 - 改善既有代码的设计》](https://book.douban.com/subject/4262627/)
- [《编写可读代码的艺术》](https://book.douban.com/subject/10797189/)
