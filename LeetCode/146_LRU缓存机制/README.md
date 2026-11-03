# LRU 缓存

实现一个容量固定的缓存。`get(key)` 若命中则返回对应的值，并把这次访问记成「刚刚用过」；不命中返回 -1。`put(key, value)` 写入一对数据：键已存在就改值，并同样记成刚刚用过；键不存在就插入。插入时若已经放满，先丢掉最久没有被访问的那一对，再放入新数据。两种操作都希望是均摊常数时间。

最近最少使用，指的是淘汰「上一次被读或被写离现在最远」的键，不是淘汰数值最小的键，也不是先进先出。只要某个旧键被 `get` 或再次 `put`，它就变成最新，不会马上被挤掉。

## 为什么数组加哈希不够

哈希表能在常数时间里查到值，但「谁最旧」仍要一条能从两端改的顺序结构。若用普通数组保存键的新旧顺序，命中后要把键挪到末尾，删除中间元素要搬运后面所有项，已经是线性时间。容量一大，`get` 就不再是常数。

双向链表可以在已知结点的前提下，常数时间摘掉它、再挂到头部。哈希表则从键直接找到这个结点。两者合在一起：查找靠哈希，调整顺序靠链表。

## 结构设计

- 哈希表：键 → 链表结点。结点上同时放键和值。淘汰时要从结点拿回键，才能把哈希表里的记录删掉，所以结点不能只存值。
- 双向链表：靠近表头的是最新使用的，靠近表尾的是最久没用的。
- 两个哨兵结点 `head` 和 `tail` 始终占住两端，中间才是真实数据。这样插入、删除不必分「链表空不空」「是不是头结点」几种写法。

约定：`head.next` 是最新结点，`tail.prev` 是最旧结点。

辅助操作只有两个：

1. 把一个已存在的结点从链表上摘下（它的前驱和后继互相连接）。
2. 把一个结点插到 `head` 后面。

## get 与 put 的分支

`get`：

- 哈希表没有这个键，返回 -1，链表不动。
- 有这个键，把结点摘下再插到表头后面，返回结点里的值。

`put`：

- 键已经在表里：改值，再把结点挪到表头。容量不变，不会触发淘汰。即使当前已经放满，更新旧键也不该删别的键。
- 键不在表里，且还没放满：新建结点，放入哈希表，插到表头。
- 键不在表里，且已放满：先摘掉 `tail.prev`，按它的键删除哈希表记录，再插入新结点。

## 演算

容量为 3。表头在左，越往右越旧。

| 操作 | 返回 | 从新到旧的键 | 说明 |
| --- | --- | --- | --- |
| `put(7, 70)` | | 7 | 第一项 |
| `put(8, 80)` | | 8, 7 | 8 更新 |
| `get(7)` | 70 | 7, 8 | 7 被读过，升到最新 |
| `put(9, 90)` | | 9, 7, 8 | 刚好放满 |
| `put(6, 60)` | | 6, 9, 7 | 最旧的 8 被丢掉 |
| `get(8)` | -1 | 6, 9, 7 | 8 已经不在 |
| `put(9, 99)` | | 9, 6, 7 | 只改值，不淘汰 |
| `get(7)` | 70 | 7, 9, 6 | 7 再次变成最新 |
| `get(9)` | 99 | 9, 7, 6 | 值是更新后的 99 |

若有人把 `put(9, 99)` 误当成新键，就会先删掉 7 再插入，随后 `get(7)` 会错成 -1。

## 代码

```python
class _Entry:
    def __init__(self, key=0, val=0):
        self.key = key
        self.val = val
        self.prev = None
        self.nxt = None


class LRUCache:
    def __init__(self, capacity):
        self.limit = capacity
        self.locate = {}
        # 哨兵不计入容量
        self.newest = _Entry()
        self.oldest = _Entry()
        self.newest.nxt = self.oldest
        self.oldest.prev = self.newest

    def _detach(self, node):
        node.prev.nxt = node.nxt
        node.nxt.prev = node.prev

    def _as_newest(self, node):
        node.nxt = self.newest.nxt
        node.prev = self.newest
        self.newest.nxt.prev = node
        self.newest.nxt = node

    def get(self, key):
        node = self.locate.get(key)
        if node is None:
            return -1
        self._detach(node)
        self._as_newest(node)
        return node.val

    def put(self, key, value):
        node = self.locate.get(key)
        if node is not None:
            node.val = value
            self._detach(node)
            self._as_newest(node)
            return
        if len(self.locate) == self.limit:
            stale = self.oldest.prev
            self._detach(stale)
            del self.locate[stale.key]
        fresh = _Entry(key, value)
        self.locate[key] = fresh
        self._as_newest(fresh)
```

## 复杂度

哈希的均摊查找、链表的摘除和插入都是常数，所以 `get` 和 `put` 均摊 `O(1)`。结点个数不超过容量，空间 `O(capacity)`。

若改用数组保存顺序，每次命中都要线性查找并搬移，单次操作变成 `O(capacity)`。

## 边界

- 容量为 1：第二次写入不同键时，唯一的旧键被淘汰；对同一个键再次 `put` 只改值，缓存仍只有这一项。
- `get` 不存在的键返回 -1，不要把它插进链表。
- 值为 0 时仍然算命中。判断是否存在要用「键在不在表里」，不能写成「值是否为真」，否则值为 0 会被当成没有。
- 放满之后更新已有键，大小不变，`tail` 前的旧结点应留着。
- 哨兵的键不要放进哈希表，淘汰时只处理 `oldest.prev`，并且要确认链表非空（容量至少为 1 时，放满后这里一定有真实结点）。

## 容易写错的地方

把单向链表当成双向用，摘除结点时找不到前驱，只能从头扫描。忘记在结点里保存键，淘汰时就无法从哈希表删除，下次 `get` 会返回已经不在链表里的值。更新顺序时先插后删，若插入位置算错，结点会自己连成环。上面的哨兵写法把「插到最新」固定成插在 `newest` 之后，就不需要再讨论根结点是不是自己。

## 有序映射的等价实现

若语言的字典能记住插入顺序，并提供「挪到一端」的操作，可以不手写结点。读到已有键时把它挪到最新一端；写入新键时若超容量，就弹出最旧一端。下面用标准库的有序字典表达同一策略，容量仍为 3，操作序列缩短，方便对照。

```python
from collections import OrderedDict

class LRUCacheOD:
    def __init__(self, capacity):
        self.limit = capacity
        self.box = OrderedDict()

    def get(self, key):
        if key not in self.box:
            return -1
        self.box.move_to_end(key)
        return self.box[key]

    def put(self, key, value):
        if key in self.box:
            self.box.move_to_end(key)
            self.box[key] = value
            return
        if len(self.box) == self.limit:
            self.box.popitem(last=False)
        self.box[key] = value
```

`last=False` 弹出的是最久没有被 `move_to_end` 的键。`get` 和更新旧键都必须挪动，只在 `put` 新键时挪动会把「最近读过」的键提前淘汰。这个写法的常数更大一些，但复杂度仍是均摊 `O(1)`，适合用来核对链表版的淘汰顺序。

## 数组顺序表会慢在哪

用列表保存「从旧到新」的键，哈希只存值。容量 3，依次 `put(1,10)`、`put(2,20)`、`put(3,30)` 后，列表是 `[1,2,3]`。此时 `get(1)` 要先线性找到 1 的位置，删掉再追加到末尾，变成 `[2,3,1]`。列表中间删除会搬运后面的元素。接着 `put(4,40)` 还要弹出下标 0。容量为 `C` 时，一次命中最坏搬运 `C` 个键。题目若要求两次操作都是常数时间，这种搬运过不了关。链表版把「找到结点」交给哈希，把「改顺序」交给指针，才同时满足。

## 容量为 1 时的轨迹

`put(4, 0)`：缓存里只有键 4，值是 0。`get(4)` 必须返回 0，不能因为值是 0 就当成缺失。再 `put(4, 8)`：键没变，只把值改成 8，仍然只有一项。再 `put(5, 1)`：容量已满且键是新的，删掉 4，留下 5。随后 `get(4)` 为 -1，`get(5)` 为 1。这个轨迹能同时检查「值为 0」「更新不算新键」「满了才淘汰」三件事。

## 和另外两种淘汰策略

先进先出只看写入顺序，读操作不改变谁先被删。最近最少使用则把读也当成一次使用。最不经常使用看的是访问次数，次数相同还要再定平局规则，实现通常再配一个计数堆，不是一条双向链表能完成的。面试里如果只说「用堆按时间戳淘汰」而不在访问时更新时间戳，得到的是先进先出，不是本题。容量为 0 没有意义，题目保证容量至少为 1；若传入 0，第一次 `put` 会在淘汰时发现链表里没有真实结点，应在构造时直接拒绝。键的类型要能做哈希，值可以是任意对象；淘汰时删的是键，不要只删值。同一键连续 `get` 多次，只改变它的新旧位置，不会复制出第二份记录，容量计数也不会增加。
