# 高并发下的HashMap

## 正式开始前的小准备

### 提出问题
  在前面的[HashMap源码分析-基于jdk7/8](http://www.indispensable.cn/solo/articles/2018/07/26/1532583898001.html)文章中,我已经详细的介绍了在jdk7和jdk8下HashMap的实现,在那篇文章的底部,留下了几个问题,其中一个问题就是今天的主题,为什么说HashMap是线程不安全的,在多线程的作用下,HashMap究竟发生了什么.

### 回顾一下HashMap
  不知道你还记不记得,HashMap中put的实现,如果不记得也没有关系,在这里,我来简单的回顾一下.
当我们的HashMap对象进行put操作时,首先计算key的hash值,并根据hash值确定其在数组中对应的位置来,其次用新插入的键值对的key,和该位置的所有键值对的key进行比较,判断是否有相同的key存在于此位置上,如果有,就进行值覆盖,没有的话就将新的键值对插入到该位置上.
## 基于jdk7的源码分析

### 问题产生的地方
  那么问题产生在哪里呢?
  我们首先根据jdk7分析一下问题的由来,在put的最后,万事俱备,我们调用了addEntry方法.
> addEntry(hash, key, value, i);
 
 addEntry的实现如下:
```java
   void addEntry(int hash, K key, V value, int bucketIndex) {
	//判断当前map中键值对的数目是否大于阈值
	//并且插入的键值对所对应数组位置已经存在了键值对
	//size >= threshold等价于size >= capacity*loadFactory
        if ((size >= threshold) && (null != table[bucketIndex])) {
	   //扩容HashMap
            resize(2 * table.length);
            hash = (null != key) ? hash(key) : 0;
            bucketIndex = indexFor(hash, table.length);
        }
        createEntry(hash, key, value, bucketIndex);
    }
 ```
 由源码可知,当map中键值对的数目大于阈值,并且新插入的键值对,其所对应的数组位置已经存在了键值对,那么就会进行resize操作.resize的操作分为以下几个步骤:
1.创建一个新的数组,新数组的容量是旧数组容量的2倍(判断数组的容量是否大于MAXIMUM_CAPACITY此处不做过多介绍)
2.调用transfer方法将旧数组的数据运输到新的数组中去
3. 给table和threshold(阈值)赋新值

源代码如下:
 ```java
    void resize(int newCapacity) {
        Entry[] oldTable = table;
        int oldCapacity = oldTable.length;
	//判断是否可以继续扩容
        if (oldCapacity == MAXIMUM_CAPACITY) {
            threshold = Integer.MAX_VALUE;
            return;
        }
	//创建新的数组,容量为之前数组的两倍
        Entry[] newTable = new Entry[newCapacity];
	//将旧数组中的键值对运输到新的数组中去
        transfer(newTable, initHashSeedAsNeeded(newCapacity));
        table = newTable;
        threshold = (int)Math.min(newCapacity * loadFactor, MAXIMUM_CAPACITY + 1);
    }
 
 ```
 
 在这个过程中,我们将关注的重心放在transfer方法上,在transfer方法中
 
 
 
```java
    /**
     * Transfers all entries from current table to newTable.
     */
    void transfer(Entry[] newTable, boolean rehash) {
        int newCapacity = newTable.length;
        for (Entry<K,V> e : table) {
            while(null != e) {
                Entry<K,V> next = e.next;
                if (rehash) {
                    e.hash = null == e.key ? 0 : hash(e.key);
                }
                int i = indexFor(e.hash, newCapacity);
                e.next = newTable[i];
                newTable[i] = e;
                e = next;
            }
        }
    }
 
  ```  
 
 
 
 
  
