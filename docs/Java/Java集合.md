## Collection

Collection接口没有直接的实现子类，是通过它的子接口Set、List和Queue实现的

![](Java\collection.jpeg)

### List

有序，可重复，支持索引，常用的有ArrayList，LinkedList，Vector

#### ArrayList

基本等同于Vector，效率高，但是**线程不安全**

底层是`Object[]` 数组



和Array区别？

1. 大小和自动扩容
2. 支持泛型
3. 存储对象
4. 集合功能



**扩容机制：**

以无参数构造方法创建 `ArrayList` 时，实际上初始化赋值的是一个空数组。当真正对数组进行添加元素操作时，才真正分配容量。即向数组中添加第一个元素时，数组容量扩为 10，到达当前容量后扩容为1.5倍。

如果使用指定大小的构造器，初始容量为指定大小，如果需要扩容则扩容为**1.5倍。**



#### Vector

底层是`Object[]` 数组。

线程同步的，即线程安全, 操作方法带 `synchronized`

|           | 底层结构 | 线程安全 效率  | 扩容机制                                                   |
| --------- | -------- | -------------- | ---------------------------------------------------------- |
| ArrayList | 可变数组 | 不安全，效率高 | 有参扩容1.5倍 <br>无参默认0，第一次扩容为10，后面扩容1.5倍 |
| Vector    | 可变数组 | 安全，效率不高 | 有参扩容2倍<br>无参默认是10，后面扩容2倍                   |



#### LinkedList

同时实现了 List、Queue 和 Deque 接⼝。底层是基于双向链表的。

线程不安全，需要用到 `LinkedList` 的场景几乎都可以使用 `ArrayList` 来代替，并且，性能通常会更好

|            | 底层结构 | 增删效率 | 改查效率 | 线程安全 | 随机访问 | 占用内存 |
| ---------- | -------- | -------- | -------- | -------- | -------- | -------- |
| ArrayList  | 可变数组 | 较低     | 较高     | 不安全   | 支持     | 小       |
| LinkedList | 双向链表 | 较高     | 较低     | 不安全   | 不支持   | 大       |



### Set

**Comparable 和 Comparator 的区别**

`Comparable` 接口和 `Comparator` 接口都是 Java 中用于排序的接口，它们在实现类对象之间比较大小、排序等方面发挥了重要作用：

- `Comparable` 接口实际上是出自`java.lang`包 它有一个 `compareTo(Object obj)`方法用来排序
- `Comparator`接口实际上是出自 `java.util` 包它有一个`compare(Object obj1, Object obj2)`方法用来排序



|               | 线程安全 | 底层数据结构 | 应用场景                 |
| ------------- | -------- | ------------ | ------------------------ |
| HashSet       | 不安全   | HashMap      | 无需保证插入取出顺序场景 |
| LinkedHashSet | 不安全   | 链表+哈希表  | 需要保证插入取出顺序场景 |
| TreeSet       | 不安全   | 红黑树       | 元素自定义排序场景       |



#### HashSet

无序，唯一

HashSet实际上是HashMap , HashMap底层是（数组+链表+红黑树）

HashSet如何检查键值重复？`HashSet`的`add()`方法直接调用`HashMap`的`put()`方法：先比较hashcode，如果发现有相同 `hashcode` 值的对象，这时会调用`equals()`方法来检查 `hashcode` 相等的对象是否真的相同。

```java
public HashSet() {
    map = new HashMap<>();
}
```

扩容机制

```
// 第一次添加时，table数组扩容到16，临界值是16*loadFactor(0.75)=12, 如果table数组使用到了临界值，就会扩容2倍，依次类推
1. 添加一个元素时，先得到hash值然后转换为索引值
2. 找到存储数据的table，看索引位置是否有元素
3. 如果没有，直接加入
4. 如果有，调用equals比较，如果相同放弃添加，如果不同添加到最后。equals不能简单的认为是比较内容或是地址，程序员可以进行重写 
5. 在java8中，如果一条链表的元素个数 >= TREEIFY_THRESHOLD(默认8)，并且table大小 >= MIN_TREEIFY_CAPACITY(默认64)，就会进行树化(红黑树)
```

| HashMap              | HashSet              |
| -------------------- | -------------------- |
| 实现Map接口          | 实现Set接口          |
| 存储键值对           | 存储对象             |
| put 添加元素         | add 添加元素         |
| 使用key计算 hashCode | 使用对象计算hashCode |



#### LinkedHashSet

HashSet的子类

底层是一个LinkedHashMap，底层维护了一个数组+双向链表

根据元素的hashCode值决定元素的存储位置，同时使用链表维护元素的次序，使得元素看起来是以插入顺序保存的

不允许元素重复



#### TreeSet

底层是TreeMap，红黑树

可以实现排序，构造器可以传入一个比较器（匿名内部类）对TreeSet进行排序



### Queue

`Queue` 是单端队列，`Deque` 是双端队列

| `Queue` 接口 | 抛出异常  | 返回特殊值 |
| ------------ | --------- | ---------- |
| 插入队尾     | add(E e)  | offer(E e) |
| 删除队首     | remove()  | poll()     |
| 查询队首元素 | element() | peek()     |

| `Deque` 接口 | 抛出异常      | 返回特殊值      |
| ------------ | ------------- | --------------- |
| 插入队首     | addFirst(E e) | offerFirst(E e) |
| 插入队尾     | addLast(E e)  | offerLast(E e)  |
| 删除队首     | removeFirst() | pollFirst()     |
| 删除队尾     | removeLast()  | pollLast()      |
| 查询队首元素 | getFirst()    | peekFirst()     |
| 查询队尾元素 | getLast()     | peekLast()      |



#### PriorityQueue

`Object[]` 数组来实现小顶堆。

线程不安全

当没有传入数组容量的时候，默认是11

如果容量小于64时，是按照oldCapacity的2倍方式扩容的；如果容量大于等于64，是按照oldCapacity的1.5倍方式扩容的



#### BlockingQueue

BlockingQueue 主要⽤于在多线程之间安全地传递数据，并提供了阻塞操作，以便在队列为空或队列已满时进⾏ 等待或阻塞

BlockingQueue的实现类：

`ArrayBlockingQueue`：基于数组实现的有界队列。

`LinkedBlockingQueue`：基于链表实现的有界或⽆界队列。

`PriorityBlockingQueue`：基于优先级的⽆界队列。

`DelayQueue`：⽤于实现延迟任务的⽆界队列。



#### DelayQueue

`DelayQueue` 底层是使用优先队列 `PriorityQueue` 来存储元素，而 `PriorityQueue` 采用二叉小顶堆的思想确保值小的元素排在最前面，这就使得 `DelayQueue` 对于延迟任务优先级的管理就变得十分方便。

`DelayQueue` 为了保证线程安全还用到了可重入锁 `ReentrantLock`,确保单位时间内只有一个线程可以操作延迟队列。

最后，为了实现多线程之间等待和唤醒的交互效率，`DelayQueue` 还用到了 `Condition`，通过 `Condition` 的 `await` 和 `signal` 方法完成多线程之间的等待唤醒。

#### ArrayDeque

基于动态数组的双端队列。底层使⽤循环数组实现

`ArrayDeque` 是基于可变长的数组和双指针来实现，而 `LinkedList` 则通过链表来实现。

## Map

![](E:\笔记\notes\Java\map.jpeg)

### HashMap

线程不安全，保证线程安全就选用 `ConcurrentHashMap`。

`HashMap` 可以存储 null 的 key 和 value，但 null 作为键只能有一个，null 作为值可以有多个

JDK1.8 之前 `HashMap` 由数组+链表组成的，数组是 `HashMap` 的主体，链表则是主要为了解决哈希冲突而存在的（“拉链法”解决冲突）。JDK1.8 以后在解决哈希冲突时有了较大的变化，当链表长度大于阈值（默认为 8）（将链表转换成红黑树前会判断，如果当前数组的长度小于 64，那么会选择先进行数组扩容，而不是转换为红黑树）时，将链表转化为红黑树，以减少搜索时间。

**`HashMap` 默认的初始化大小为 16。到达临界值（临界值是16*loadFactor(0.75)=12）之后，容量变为原来的 2 倍。并且， `HashMap` 总是使用 2 的幂作为哈希表的大小。**因为对长度取模的操作可以用位运算来替代（` hash%length==hash&(length-1)`），能够有效保留hashcode低位并且提高效率。

初始化传的不是2的幂时，会向上寻找离得近的2的幂作为初始化大小。

添加元素：

```
1. 添加一个元素时，先得到hash值然后转换为索引值( (n - 1) & hash )
2. 找到存储数据的table，看索引位置是否有元素
3. 如果没有，直接插入（该节点直接放在数组中）
4. 如果有，调用equals比较判断该元素与要存入的元素的 hash 值以及 key 是否相同，如果相同则覆盖，如果不同插入到链表末端。equals不能简单的认为是比较内容或是地址，程序员可以进行重写 
5. 在java8中，如果一条链表的元素个数 >= TREEIFY_THRESHOLD-1(默认8)，并且table大小 >= MIN_TREEIFY_CAPACITY(默认64)，就会进行树化(红黑树)
```



**`HashMap` 的长度是 2 的幂次方的原因：**

1. 位运算效率更高：位运算(&)比取余运算(%)更高效。当长度为 2 的幂次方时，`hash % length` 等价于 `hash & (length - 1)`。
2. 可以更好地保证哈希值的均匀分布：扩容之后，在旧数组元素 hash 值比较均匀的情况下，新数组元素也会被分配的比较均匀，最好的情况是会有一半在新数组的前半部分，一半在新数组后半部分。
3. 扩容机制变得简单和高效：扩容后只需检查哈希值高位的变化来决定元素的新位置，要么位置不变（高位为 0），要么就是移动到新位置（高位为 1，原索引位置+原容量）。



**HashMap为什么线程不安全？**

死循环和数据丢失

1. JDK1.7 及之前版本的 `HashMap` 在多线程环境下**扩容操作可能存在死循环问题**，这是由于当一个桶位中有多个元素需要进行扩容时，多个线程同时对链表进行操作，头插法可能会导致链表中的节点指向错误的位置，从而形成一个环形链表，进而使得查询元素的操作陷入**死循环**无法结束。[JDK 1.7 hashmap循环链表的产生（图文并茂，巨详细）_hashmap循环链表是如何产生的-CSDN博客](https://blog.csdn.net/qq_44833552/article/details/125575981)

2. 多个线程对 `HashMap` 的 `put` 操作会有**数据覆盖**的风险。并发环境下，推荐使用 `ConcurrentHashMap` 。



**HashMap遍历方式：**

1. 使用迭代器（Iterator）EntrySet 的方式进行遍历；
2. 使用迭代器（Iterator）KeySet 的方式进行遍历；
3. 使用 For Each EntrySet 的方式进行遍历；
4. 使用 For Each KeySet 的方式进行遍历；
5. 使用 Lambda 表达式的方式进行遍历；
6. 使用 Streams API 单线程的方式进行遍历；
7. 使用 Streams API 多线程的方式进行遍历。

`entrySet` 的性能比 `keySet` 的性能高出了一倍之多，因此我们应该尽量使用 `entrySet` 来实现 Map 集合的遍历。

`EntrySet` 的性能比 `KeySet` 的性能高出了一倍，因为 `KeySet` 相当于循环了两遍 Map 集合，而 `EntrySet` 只循环了一遍。



不能在遍历中使用集合 `map.remove()` 来删除数据，这是非安全的操作方式，但我们可以使用迭代器的 `iterator.remove()` 的方法来删除数据，这是安全的删除集合的方式。同样的我们也可以使用 Lambda 中的 `removeIf` 来提前删除数据，或者是使用 Stream 中的 `filter` 过滤掉要删除的数据进行循环，这样都是安全的，当然我们也可以在 `for` 循环前删除数据在遍历也是线程安全的。



### ConcurrentHashMap

Java 7 中 `ConcurrnetHashMap` 由很多个 `Segment` 组合，而每一个 `Segment` 是一个类似于 `HashMap` 的结构，所以每一个 `HashMap` 的内部可以进行扩容。但是 `Segment` 的个数一旦**初始化就不能改变**，默认 `Segment` 的个数是 16 个，你也可以认为 `ConcurrentHashMap` 默认支持最多 16 个线程并发。

Java 8 中 不再是之前的 **Segment 数组 + HashEntry 数组 + 链表**，而是 **Node 数组 + 链表 / 红黑树**。当冲突链表达到一定长度时，链表会转换成红黑树。



ConcurrentHashMap 为什么 key 和 value 不能为 null？

`ConcurrentHashMap` 的 key 和 value 不能为 null 主要是为了避免二义性。null 是一个特殊的值，表示没有对象或没有引用。如果你用 null 作为键，那么你就无法区分这个键是否存在于 `ConcurrentHashMap` 中，还是根本没有这个键。同样，如果你用 null 作为值，那么你就无法区分这个值是否是真正存储在 `ConcurrentHashMap` 中的，还是因为找不到对应的键而返回的。



### LinkedHashMap

`LinkedHashMap` 继承自 `HashMap`，所以它的底层仍然是基于拉链式散列结构即由数组和链表或红黑树组成。另外，`LinkedHashMap` 在上面结构的基础上，增加了一条双向链表，使得上面的结构可以保持键值对的插入顺序。同时通过对链表进行相应的操作，实现了访问顺序相关逻辑。

LRU缓存：

1. 继承 `LinkedHashMap`;

2. 构造方法中指定 `accessOrder` 为 true ，这样在访问元素时就会把该元素移动到链表尾部，链表首元素就是最近最少被访问的元素；

3. 重写`removeEldestEntry` 方法，该方法会返回一个 boolean 值，告知 `LinkedHashMap` 是否需要移除链表首元素（缓存容量有限）。

```java
public class LRUCache<K, V> extends LinkedHashMap<K, V> {
    private final int capacity;

    public LRUCache(int capacity) {
        super(capacity, 0.75f, true);
        this.capacity = capacity;
    }

    /**
     * 判断size超过容量时返回true，告知LinkedHashMap移除最老的缓存项(即链表的第一个元素)
     */
    @Override
    protected boolean removeEldestEntry(Map.Entry<K, V> eldest) {
        return size() > capacity;
    }
}
```

```java
LRUCache<Integer, String> cache = new LRUCache<>(3);
cache.put(1, "one");
cache.put(2, "two");
cache.put(3, "three");
cache.put(4, "four");
cache.put(5, "five");
for (int i = 1; i <= 5; i++) {
    System.out.println(cache.get(i));
}
```



### Hashtable

* 键和值都不能为空
* 使用方法基本和HashMap一样
* Hashtable是线程安全的，通过在每个⽅法上添加同步关键字来实现的，但这也可能 导致性能下降。

```
底层数组Hashtable$Entry[] 初始化大小 11
临界值 threshold = 8 (11*0.75)
扩容机制
```

|           | 线程安全   | 效率                   | 对null key/value的支持 | 扩容机制                               | 底层数据结构            |
| --------- | ---------- | ---------------------- | ---------------------- | -------------------------------------- | ----------------------- |
| HashMap   | 线程不安全 | 高                     | 允许                   | 默认初始化大小16，每次扩容为原来的2倍  | 数组+链表+红黑树        |
| HashTable | 线程安全   | 基本被淘汰，不建议使用 | 不允许                 | 默认初始化大小11，每次扩容为原来的2n+1 | 数组+链表，没有树化机制 |



**ConcurrentHashMap 和 Hashtable 的区别：**

1. 底层数据结构：ConcurrentHashMap和HashMap一样，使用数组+链表/红黑树，HashTable使用数组+链表，没有树化机制
2. 实现线程安全的方式：`ConcurrentHashMap` 取消了 `Segment` 分段锁，采用 `Node + CAS + synchronized` 来保证并发安全，`synchronized` 只锁定当前链表或红黑二叉树的首节点。HashTable使用 `synchronized` 来保证线程安全，效率非常低下

### TreeMap

基于红黑树数据结构的实现的

实现 `NavigableMap` 接口让 `TreeMap` 有了对集合内元素的搜索的能力。

`NavigableMap` 接口提供了丰富的方法来探索和操作键值对:

1. **定向搜索**: `ceilingEntry()`, `floorEntry()`, `higherEntry()`和 `lowerEntry()` 等方法可以用于定位大于、小于、大于等于、小于等于给定键的最接近的键值对。
2. **子集操作**: `subMap()`, `headMap()`和 `tailMap()` 方法可以高效地创建原集合的子集视图，而无需复制整个集合。
3. **逆序视图**:`descendingMap()` 方法返回一个逆序的 `NavigableMap` 视图，使得可以反向迭代整个 `TreeMap`。
4. **边界操作**: `firstEntry()`, `lastEntry()`, `pollFirstEntry()`和 `pollLastEntry()` 等方法可以方便地访问和移除元素。

实现`SortedMap`接口让 `TreeMap` 有了对集合中的元素根据键排序的能力。默认是按 key 的升序排序，不过我们也可以指定排序的比较器。

**相比于`HashMap`来说， `TreeMap` 主要多了对集合中的元素根据键排序的能力以及对集合内元素的搜索的能力。**



**集合框架底层使⽤了什么数据结构？**

1. List接⼝的实现
   1. ArrayList： 基于动态数组实现。底层使⽤数组作为存储结构。 
   2. LinkedList： 基于双向链表实现。底层使⽤节点（Node）连接形成链表结构。 
   3. Vector： 类似于 ArrayList，但是是线程安全的。底层也是使⽤数组实现。 
2. Set接⼝ 
   1. HashSet： 基于哈希表实现。底层使⽤⼀个数组和链表/红⿊树的结构来存储元素。 
   2. LinkedHashSet： 在 HashSet 的基础上加⼊了链表，使得迭代顺序可预测。 
   3. TreeSet： 基于红⿊树实现。底层使⽤⾃平衡的⼆叉搜索树存储元素，以保持有序性。 
3. Queue接⼝ 
   1. LinkedList： 同时实现了 List、Queue 和 Deque 接⼝。底层是基于双向链表的。 
   2. ArrayDeque： 基于动态数组的双端队列。底层使⽤循环数组实现。 
   3. PriorityQueue： 基于优先级堆实现的队列。底层使⽤数组表示的⼆叉堆。
4. Map接⼝ 
   1. HashMap： 基于哈希表实现。底层使⽤⼀个数组和链表/红⿊树的结构来存储键值对。 
   2. LinkedHashMap： 在 HashMap 的基础上加⼊了链表，使得迭代顺序可预测。 
   3. TreeMap： 基于红⿊树实现。底层使⽤⾃平衡的⼆叉搜索树存储键值对，以保持有序性。 
   4. Hashtable： 类似于 HashMap，但是是线程安全的。底层也是使⽤哈希表。

