# HashSet


## HashMap实现原理
`HashSet` 是一个不允许存储重复元素的集合，它的实现比较简单，底层实际上是一个`HashMap`.使用`HashMap`的键存储使用者想要存入`HashSet`中的值,这样就可以保证`set`不会存储重复的元素了.所以,只要理解了 `HashMap`，`HashSet` 就毫无难度了。

## 成员变量
首先通过源代码了解下 `HashSet` 的成员变量:

```java
    private transient HashMap<E,Object> map;//使用HashMap存储数据,transient关键字保证其不参与自动序列化
    private static final Object PRESENT = new Object();
```

主要就上面两个变量:

- `map` ：用于存放数据的`HashMap`。
- `PRESENT` ：在HashSet中，元素都存到内部`HashMap`键值对的Key中，而Value都有一个`统一的值`,这个值就是`PRESENT`

## 构造函数

```java
    public HashSet() {
        map = new HashMap<>();
    }
    
    public HashSet(int initialCapacity, float loadFactor) {
        map = new HashMap<>(initialCapacity, loadFactor);
    }    
	 public HashSet(Collection<? extends E> c) {
        map = new HashMap<>(Math.max((int) (c.size()/.75f) + 1, 16));
        addAll(c);
    }
	public HashSet(int initialCapacity) {
        map = new HashMap<>(initialCapacity);
    }
```
构造函数很简单，利用了 `HashMap` 初始化了 `map` ,可以指定`内部HashMap`的`初始容量和加载因子`,也可以直接传递一个集合对象过来,将集合对象中所有的数据添加到`内部HashMap`中.

## 方法
内部的方法有以下几个:
![imagepng](http://pcg4drw32.bkt.clouddn.com//file/2018/09/962eb06747684733944d91dd2850a9a2_image.png) 

我们选择其中的一部分进行源码分析.

### add

```java
    public boolean add(E e) {
        return map.put(e, PRESENT)==null;
    }
```

比较关键的就是这个 `add()` 方法。
可以看出它是将存放的对象当做了 `HashMap` 的键，`value` 都是相同的 `PRESENT` 。由于 `HashMap` 的 `key` 是不能重复的，所以每当有重复的值写入到 `HashSet` 时，`value` 会被覆盖(实质上值都是一致的)，但 `key` 不会受到影响，这样就保证了 `HashSet` 中只能存放不重复的元素。

当我们向`HashSet`对象中放入同个元素时,第一个元素放入成功,add方法返回`true`,而第二个元素放入失败,add方法会返回`false`,因为在进行第二次add操作时,`map.put(e, PRESENT)`方法会返回老值(oldValue),这个值为`PRESENT`,也就不等于null了.试验代码如下:
```Java
        Set<String> set = new HashSet<>();
        System.out.println(set.add("chen"));
        System.out.println(set.add("chen"));

```
### remove和clear
```Java
   public boolean remove(Object o) {
        return map.remove(o)==PRESENT;
    }
   public void clear() {
        map.clear();
    }
	
```
`remove`和`clear`方法实际上也是调用了`HashMap`的对应方法.

### writeObject和readObject
这两个方法就是当`HashSet`进行序列化时调用的方法,重写这两个方法的原因就是`HashSet`内部的`HashMap`存在着`空位`,如果使用默认的序列化方法会导致序列化失败.

## 总结

`HashSet` 的原理比较简单，几乎全部借助于 `HashMap` 来实现的。所以 `HashMap` 会出现的线程不安全问题 `HashSet` 依然不能避免。在多线程下使用`HashSet`可能会使其内部的`HashMap对象`中的链表出现环状引用的情况,当查找该链表所在位置的某个不存在的键值对时,会导致`CPU占用达到百分之百的情况`.
顺带提一下,`HashMap`中可放的数据是要注意以下几点的:

- HashMap 的 key 和 value 都可以是 null

- Map 的 key 和 value 都 **不允许** 是 基本数据类型

- HashMap 的 key 可以是 任意对象，但如果 对象的 hashCode 改变了，那么将找不到原来该 key 所对应的 value(此处对HashSet**无影响**)
