# LRU算法

LRU是Least Recently Used的缩写，即最近最少使用，常用于页面置换算法，是为虚拟页式存储管理服务的。

即当一个数据最近一段时间没有被访问，未来被访问的概率也很小。当空间被占满后，最先淘汰最近最少使用的数据。

## Java中的LRU实现思路

根据LRU算法，在Java中实现需要这些条件：

- 底层数据使用双向链表，方便在链表的任意位置进行删除，在链表尾进行添加

这一点用单链表比较费劲，当然用数组等结构也都很费劲
当然双向链表在查找时也麻烦，但下述可以结合HashMap使用
需要将链表按照访问（使用）顺序排序

- 数据量超过一定阈值后，需要删除Least Recently Used数据
Java中一个简单的LRUCache实现

对于上述的实现思路，java.util.LinkedHashMap已经实现了其中的99%，因此直接基于LinkedHashMap实现LRUCache非常简单。

## LinkedHashMap实现LRUCache

构造方法提供了accessOrder选项，开启后会get方法会有额外操作保证链表顺序按访问顺序逆序排列
底层结构使用双向链表，查找可以使用HashMap的特点
覆盖了父类HashMap的newNode方法和newTreeNode方法，这两个方法在HashMap中只是创建Node用的，而在LinkedHashMap中不但创建Node，还将Node放在链表末尾
父类HashMap提供了3个void的Hook方法，方法没做任何事：
- afterNodeRemoval 父类在remove一个集合中存在的元素后调用
- afterNodeInsertion 父类在put、compute、merge后调用
- afterNodeAccess 父类在replace、compute、merge等替换值后会调用，LinkedHashMap在get中开启accessOrder时调用，究其根本是在对数据有操作时会调用

LinkedHashMap本质上还是复用HashMap的绝大部分功能，包括底层的Node<K, V>[]，因此能支持原本HashMap的功能

但是LinkedHashMap实现了父类HashMap的3个Hook方法：
- afterNodeRemoval 实现链表的删除操作
- afterNodeInsertion 并没有实现链表的插入操作，但新添加了一个Hook方法boolean removeEldestEntry，当这个Hook方法返回true时，删除链表头的节点
- afterNodeAccess 如前所述，开启accessOrder后会将被操作的节点放在链表末尾，保证链表顺序按访问顺序逆序排列

上一条3个方法是用来构建双向链表的，LinkedHashMap还覆盖了父类的3个方法：

- newNode 在创建一个Node的同时，将Node添加到链表末尾
- newTreeNode 创建TreeNode的同时，将Node添加到链表末尾
- get 完成get功能的同时，如果accessOrder开启，会调用afterNodeAccess将Node移动到链表末尾

覆盖newNode和newTreeNode方法后，在put方法中调用的newNode和newTreeNode方法也就连带实现了链表的插入操作

综上，我们可以了解到LinkedHashMap为什么能够轻松实现LRUCache

继承父类HashMap，拥有HashMap的功能，因此在查找一个节点时时间复杂度为O(1)，再加上链表是双向，做链表任意节点的删除工作就非常简单

通过HashMap提供的3个Hook方法并覆盖了2个创建Node的方法，实现了自身链表的添加、删除工作，保证在不影响原本Array功能的前提下，正确完成自身的链表构建；这个过程实际上均是通过Hook方式增强原有功能的，因为原本的HashMap中创建节点其实也是使用的Hook方法

提供属性accessOrder并实现了afterNodeAccess方法，因此能够根据访问或操作顺序将最近使用或最近插入的数据放在链表尾，越久没被使用的数据就越靠近链表头，实现了整个链表按照LRU的要求排序

提供了一个Hook方法boolean removeEldestEntry，这个方法返回true时将会删除表头节点，即LRU中应当淘汰的节点，但是这个方法在LinkedHashMap中的实现永远返回false

到这为止，实现一个LRUCache就很简单了：实现这个removeEldestEntryHook方法，给LinkedHashMap设置一个阈值，那么超过这个阈值时就会进行LRU淘汰。



```java


public class LRU<K, V> {
 
	/**
	 * LinkedHashMap负载因子默认0.75
	 */
	private static final float hashLoadFactory = 0.75f;
	private LinkedHashMap<K, V> map;
	private int cacheSize;
 
	/**
	 * @param cacheSize 容量
	 */
	public LRU(int cacheSize) {
		this.cacheSize = cacheSize;
		int capacity = (int) Math.ceil(cacheSize / hashLoadFactory) + 1;
		map = new LinkedHashMap<K, V>(capacity, hashLoadFactory, true) {
			private static final long serialVersionUID = -5291175172111746517L;
 
			/*
			 * 当map容量大于LRU的容量时，删除最近最少使用的数据
			 */
			@Override
			protected boolean removeEldestEntry(Entry<K, V> eldest) {
				return size() > LRU.this.cacheSize;
			}
		};
	}
 
	public synchronized V get(K key) {
		return map.get(key);
	}
 
	public synchronized void put(K key, V value) {
		map.put(key, value);
	}
 
	public synchronized void clear() {
		map.clear();
	}
 
	public synchronized int usedSize() {
		return map.size();
	}
 
	public void print() {
		for (Map.Entry<K, V> entry : map.entrySet()) {
			System.out.print(entry.getValue() + "--");
		}
		System.out.println();
	}
}

```