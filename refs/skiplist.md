## **SkipList简介**



SkipList是一个实现快速查找、增删数据的数据结构，可以做到O(logN)复杂度的增删查。从时间复杂度上来看，似乎和平衡树差不多，但是和平衡树比较起来，它的编码复杂度更低，实现起来更加简单。学过数据结构的同学应该都有了解，平衡树经常需要旋转操作来维护两边子树的平衡，不仅编码复杂，理解困难，而且debug也非常不方便。SkipList克服了这些缺点，原理简单，实现起来也非常方便。



## **原理**



SkipList的本质是List，也就是链表。我们都知道，链表是线性结构的，每次只能移动一个节点，这也是为什么链表获取元素和删除元素的复杂度都是O(n)。

![img](https://pic4.zhimg.com/80/v2-c73b47caa8053b17a4c44b6b76b9bb9f_720w.webp)

如果我们要优化这个问题，可以在当中一般的节点上增加一个指针，指向后面两个的元素。这样我们遍历的速度可以提升一倍，最快就可以在O(n/2)的时间内遍历完整个链表了。



![img](https://pic1.zhimg.com/80/v2-107adc9424c12f66a9d7800d337a8ff0_720w.webp)



同样的道理，如果我们继续增加节点上指针的个数，那么这个速度还可以进一步加快。理论上来说，如果我们设置log⁡n个指针，完全可以在log⁡n的时间内完成元素的查找，这也是SkipList的精髓。



但是有一个问题是我们光实现快速查找是不够的，我们还需要保证元素的有序性，否则查找也就无从谈起。但是元素添加的顺序并不一定是有序的，我们怎么保证节点分配到的指针数量合理呢？



为了解决这个问题，SkipList引入了随机深度的机制，也就是一个节点能够拥有的指针数量是随机的。同样这种策略来保证元素尽可能分散均匀，使得不会发生数据倾斜的情况。



![img](https://pic1.zhimg.com/80/v2-066a9c3461235fc1badd19c534e08b7c_720w.webp)



我觉得这个图放出来应该都能看懂，可以把每一个节点想象成一栋小楼。每个节点的多个指针可以看成是小楼的各个楼层，很显然，由于所有的小楼都排成一排，所以每栋楼的每一层都只能看到同样高度最近的一栋。



比如上图当中的2只有一层，那么它只能看到最近的一楼也就是3的位置。4有三层，它的第一层只能看到5，但是第二和第三层可以看到6。6也有三层，由于6之后没有节点有超过两层的，所以它的第三层可以直接看到结尾。



由于每个节点的高度是随机的，所以每个节点能看到的情况是分散的，可以防止数据聚集不平均等问题，从而可以保证运行效率。



```python
class Node:
    def __init__(self, key, value=None, depth=1):
        self._key = key
        self._value = value
        # 一开始全部赋值为None
        self._next = [None for _ in range(depth)]
        self._depth = depth

    @property
    def key(self):
        return self._key

    @key.setter
    def key(self, key):
        self._key = key

    @property
    def value(self):
        return self._value

    @value.setter
    def value(self, value):
        self._value = value

    # 为第k个后向指针赋值
    def set_forward_pos(self, k, node):
        self._next[k] = node

    # 获取指定深度的指针指向的节点的key
    def query_key_by_depth(self, depth):
        # 后向指针指向的内容有可能为空，并且深度可能超界
        # 我们默认链表从小到大排列，所以当不存在的时候返回无穷大作为key
        return math.inf if depth > self._depth or self._next[depth] is None else self._next[depth].key

    # 获取指定深度的指针指向的节点
    def forward_by_depth(self, depth):
        return None if depth > self._depth else self._next[depth]
    
    
class SkipList:
    def __init__(self, max_depth, rate=0.5):
        # head的key设置成负无穷，tail的key设置成正无穷
        self.root = Node(-math.inf, depth=max_depth)
        self.tail = Node(math.inf)
        self.rate = rate
        self.max_depth = max_depth
        self.depth = 1
        # 把head节点的所有后向指针全部指向tail
        for i in range(self.max_depth):
            self.root.set_forward_pos(i, self.tail)

    def random_depth(self):
        depth = 1
        while True:
            rd = random.random()
            # 如果随机值小于p或者已经到达最大深度，就返回
            if rd < self.rate or depth == self.max_depth:
                return depth
            depth += 1
            
    `def delete(self, key):
        # 记录下楼位置的数组
        heads = [None for _ in range(self.max_depth)]
        pnt = self.root
        for i in range(self.depth-1, -1, -1):
            while pnt.query_key_by_depth(i) < key:
                pnt = pnt.forward_by_depth(i)
            # 记录下楼位置
            heads[i] = pnt
        pnt = pnt.forward_by_depth(0)
        # 如果没找到，当然不存在删除
        if pnt.key == key:
            # 遍历所有下楼的位置
            for i in range(self.depth):
                # 由于是从低往高遍历，所以当看不到的时候，就说明已经超了，break
                if heads[i].forward_by_depth(i).key != key:
                    break
                # 将它看到的位置修改为删除节点同样楼层看到的位置
                heads[i].set_forward_pos(i, pnt.forward_by_depth(i))
            # 由于我们维护了skiplist当中的最高高度，所以要判断一下删除元素之后会不会出现高度降低的情况
            while self.depth > 1 and self.root.forward_by_depth(self.depth - 1) == self.tail:
                self.depth -= 1
        else:
            return False

    def insert(self, key, value):
        # 记录下楼的位置
        heads = [None for _ in range(self.max_depth)]
        pnt = self.root
        for i in range(self.depth-1, -1, -1):
            while pnt.query_key_by_depth(i) < key:
                pnt = pnt.forward_by_depth(i)
            heads[i] = pnt
        pnt = pnt.forward_by_depth(0)
        # 如果已经存在，直接修改
        if pnt.key == key:
            pnt.value = value
            return
        # 随机出楼层
        new_l = self.random_depth()
        # 如果楼层超过记录
        if new_l > self.depth:
            # 那么将头指针该高度指向它
            for i in range(self.depth, new_l):
                heads[i] = self.root
            # 更新高度
            self.depth = new_l

        # 创建节点
        new_node = Node(key, value, self.depth)
        for i in range(0, new_l):
            # x指向的位置定义成能看到x的位置指向的位置
            new_node.set_forward_pos(i, self.tail if heads[i] is None else heads[i].forward_by_depth(i))
            # 更新指向x的位置的指针
            if heads[i] is not None:
                heads[i].set_forward_pos(i, new_node)

```

原文：https://zhuanlan.zhihu.com/p/108386262