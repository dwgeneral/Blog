> 算法世界之LRU缓存淘汰

#### 缓存

缓存是一种提高数据读取性能的技术，在硬件设计、软件开发中有着广泛的应用。比如CPU缓存、数据库缓存、浏览器缓存、服务器缓存等

缓存技术是好的，但我们不能把所有数据都一直存在缓存里，也要考虑成本的问题。所以具体哪些数据应该在缓存中，哪些应该被清理来节约空间，所以就有了缓存淘汰策略。常见的策略有三种：先进先出策略(FIFO), 最少使用策略(LFU), 最近最少使用策略(LRU)。相信这些策略你看到名字基本也就猜到它们的特征了。

因为 LRU策略目前在业界应用是比较广泛的，所以本文将通过链表和哈希表来实现一个LRU策略的模块

#### 实现LRU

我的思路是这样的，我们维护一个双端链表，越靠近链表尾部的结点是越早访问的，链表头部的结点是最近刚被访问的，当有一个新的数据被访问时，我们有如下逻辑需要处理：
  - 如果此数据之前已经被访问过了，我们需要找到这个数据的结点，删除它，然后在链表头部重新插入该结点
  - 如果此数据没有被缓存过，则需要将其插入到链表头部，但插入之前，我们需要检验缓存容量是否还够
    - 如果缓存没满，那直接头插法插入就好了
    - 如果缓存满了，就将链表尾部的结点删除，然后再将新数据插入

我们都知道链表的查询操作是O(n)的复杂度，每次插入都有可能要遍历链表，这样性能是不够好的，所以这里我们需要用一个哈希表来记录链表中结点的位置，当然数组的查询也是O(1)的，所以也可以用数组来代替哈希表，这样我就用O(1)的哈希表查询替代了O(n)的链表遍历，即最终的LRU模型的插入，查询操作都是O(1)的时间复杂度

#### 具体实现

```javascript
/**
 * 使用 哈希表 + 双端链表 实现
 */
class LinkedNode {
  constructor(key = 0, val = 0) {
    this.key = key
    this.val = val
    this.prev = null
    this.next = null
  }
}

class LinkedList {
  constructor() {
    this.head = new LinkedNode()
    this.tail = new LinkedNode()
    this.head.next = this.tail
    this.tail.prev = this.head
  }

  insertFirst(node) {
    node.next = this.head.next
    node.prev = this.head
    this.head.next.prev = node
    this.head.next = node
  }

  remove(node) {
    node.prev.next = node.next
    node.next.prev = node.prev
  }

  removeLast() {
    if (this.tail.prev === this.head) return null
    let last = this.tail.prev
    this.remove(last)
    return last
  }
}

/**
 * @param {number} capacity
 */
var LRUCache = function(capacity) {
  this.capacity = capacity
  this.keyNodeMap = new Map()
  this.cacheLink = new LinkedList()
};

/** 
 * @param {number} key
 * @return {number}
 */
LRUCache.prototype.get = function(key) {
  if (!this.keyNodeMap.has(key)) return -1
  let val = this.keyNodeMap.get(key).val
  this.put(key, val)
  return val
};

/** 
 * @param {number} key 
 * @param {number} value
 * @return {void}
 */
LRUCache.prototype.put = function(key, value) {
  let size = this.keyNodeMap.size
  if (this.keyNodeMap.has(key)) { this.cacheLink.remove(this.keyNodeMap.get(key)); --size }
  if (size >= this.capacity) this.keyNodeMap.delete(this.cacheLink.removeLast().key)
  let node = new LinkedNode(key, value)
  this.keyNodeMap.set(key, node)
  this.cacheLink.insertFirst(node)
};

/**
 * Your LRUCache object will be instantiated and called as such:
 * var obj = new LRUCache(capacity)
 * var param_1 = obj.get(key)
 * obj.put(key,value)
 */
```