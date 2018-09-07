![](https://img.hacpai.com/bing/20180803.jpg?imageView2/1/w/960/h/520/interlace/1/q/100)

## 引言
对于Java程序员而言,集合类框架是必须要掌握的,笔者前一段时间被问到了一个关于ArrayList如何在遍历list时删除特定元素的问题,当时没有立即回答上来.随后想起,还是看源码看的太少了,所以就有了这一篇关于ArrayList的解读.
那么在进行这篇文章的阅读之前,也希望读者们能够把这个问题思考一下,看看自己是否已经有了问题的答案吧!
```
以下代码打印列表中的偶数项,并将其移除,是否有问题? 
public void errorRemove(){
        List<Integer> list = new ArrayList<>();
        Random random = new Random();
        for (int i = 0; i < 1000; i++) {
            list.add(random.nextInt(10000));
        }
        for (Integer i : list) {
            if (i % 2 == 0) {
                list.remove(i);
            }
        }
    }  
```
## 解析ArrayList

### ArrayList的应用
ArrayList又称为动态数组.在程序开发过程中,我们经常使用ArrayList来存储不定数量的固定类型数据(比如使用`ArrayList<Integer>`存储Integer对象),ArrayList在list的末尾增加元素以及根据索引获取元素上比较快速(时间复杂度`O(1)`),但在元素的删除以及元素的插入上相较于LinkedList来说较为缓慢.(ArrayList这两项的操作时间复杂度为`O(n)`,而LinkedList的时间复杂度为`O(1)`).

### 常用参数
ArrayList中的参数如下
>private static final int DEFAULT_CAPACITY = 10; //默认的数组初始化容量,也就是elementData默认的大小

>private int size; //数组中存放着的元素数目

>private static final Object[] EMPTY_ELEMENTDATA = {};//一个空数组,当用户指定数组默认容量为0时返回该数组

>private static final Object[] DEFAULTCAPACITY_EMPTY_ELEMENTDATA = {};//当用户调用空参构造器时返回该数组

>transient Object[] elementData;//实际上用来存放元素的数组

除了上述几个参数以外,还有一个需要重点强调的参数就是`modCount`,这个参数来源于ArrayList继承的`AbstractList`类,表示ArrayList对象的修改次数,可用来实现`fail-fast`机制.比如说用户在遍历ArrayList对象中的元素时,该ArrayList对象进行了增删改操作,使得modCount++,这时就会抛出`ConcurrentModificationException`异常.
>protected transient int modCount = 0;

### 常用方法

#### add(E element)
`add(E e)`方法为在数组插入对象,源码如下:

```
    public boolean add(E e) {
        ensureCapacityInternal(size + 1);  // 确定数组中是否有多余的空间以供添加元素,并增加modCount
        elementData[size++] = e;
        return true;
    }
```

- 首先进行扩容校验。判断数组中是否有空余空间可供添加元素,如果没有的话就进行扩容
- 将插入的值放到数组`size+1`的位置，并将 size自增.

#### add(int index, E element)
`add(index,e)`方法为在指定位置插入对象,源码如下:
```
    public void add(int index, E element) {
        rangeCheckForAdd(index);

        ensureCapacityInternal(size + 1);  // 确定数组中是否有多余的空间以供添加元素,并增加modCount
        //复制数组，并将插入位置后的元素向后移动
        System.arraycopy(elementData, index, elementData, index + 1,
                         size - index);
        elementData[index] = element;
        size++;
    }
```


- 也是首先扩容校验。判断数组中是否有空余空间可供添加元素,如果没有的话就进行扩容
- 接着对数据进行复制，目的是把 index 位置空出来放本次插入的数据，并将后面的数据向后移动一个位置。

其实扩容最终调用的代码是类中的`grow(int)`方法:
```
    private void grow(int minCapacity) {
        // overflow-conscious code
        int oldCapacity = elementData.length;
        int newCapacity = oldCapacity + (oldCapacity >> 1);
        if (newCapacity - minCapacity < 0)
            newCapacity = minCapacity;
        if (newCapacity - MAX_ARRAY_SIZE > 0)
            newCapacity = hugeCapacity(minCapacity);
        // minCapacity is usually close to size, so this is a win:
        elementData = Arrays.copyOf(elementData, newCapacity);
    }
```

grow方法进行的是一个数组复制的过程:

- 首先计算新数组的长度()newCapacity)为原容量的1.5倍
- 取newCapacity和minCapacity中的最大值作为新数组的长度
- 判断新数组的长度是否大于数组最大长度,如果大于就使用Integer.MAX_VALUE作为数组长度

由此可见 `ArrayList` 对内存的主要消耗是数组扩容以及在指定位置添加数据，在日常使用时最好是指定大小，尽量减少扩容。更要减少在指定位置插入数据的操作。

#### get(int index)
获取指定位置的对象
```
 public E get(int index) {
        rangeCheck(index);//校验index是否在数组的长度范围内

        return elementData(index);
    }

```

#### set(int index, E element)
替换指定位置的对象
```
    public E set(int index, E element) {
        rangeCheck(index);

        E oldValue = elementData(index);
        elementData[index] = element;
        return oldValue;
    }
```

#### remove(Object o)
根据对象内容移除数组中指定对象
```
 public boolean remove(Object o) {
        if (o == null) {
            for (int index = 0; index < size; index++)
                if (elementData[index] == null) {
                    fastRemove(index);
                    return true;
                }
        } else {
            for (int index = 0; index < size; index++)
                if (o.equals(elementData[index])) {
                    fastRemove(index);
                    return true;
                }
        }
        return false;
    }
```
#### 序列化与反序列化

由于 ArrayList 是基于动态数组实现的，所以并不是所有的空间都被使用。如果被自动序列化的话就会抛出异常.因此使用了 `transient` 修饰，可以防止被自动序列化。又`自定义了writeObject和readObject`方法,,JVM会调用类中的这两个方法实现对象序列化与方序列化时.

```
transient Object[] elementData;
```

```
    private void writeObject(java.io.ObjectOutputStream s)
        throws java.io.IOException{
        // Write out element count, and any hidden stuff
        int expectedModCount = modCount;
        s.defaultWriteObject();

        // Write out size as capacity for behavioural compatibility with clone()
        s.writeInt(size);

        // Write out all elements in the proper order.
        //只序列化被使用的位置的数据(i<size)
        for (int i=0; i<size; i++) {
            s.writeObject(elementData[i]);
        }
	//如果在序列化1过程中对ArrayList对象进行了增删改操作,就会抛出异常
        if (modCount != expectedModCount) {
            throw new ConcurrentModificationException();
        }
    }

    private void readObject(java.io.ObjectInputStream s)
        throws java.io.IOException, ClassNotFoundException {
        elementData = EMPTY_ELEMENTDATA;

        // Read in size, and any hidden stuff
        s.defaultReadObject();

        // Read in capacity
        s.readInt(); // ignored

        if (size > 0) {
            // be like clone(), allocate array based upon size not capacity
            int capacity = calculateCapacity(elementData, size);
            SharedSecrets.getJavaOISAccess().checkArray(s, Object[].class, capacity);
            ensureCapacityInternal(size);

            Object[] a = elementData;
            // 只反序列化被使用的位置的数据
            for (int i=0; i<size; i++) {
                a[i] = s.readObject();
            }
        }
    }
```
 ArrayList 只序列化和反序列化数组中被使用位置的数据。

#### iterator()
在文章的最开始时,我们提到了关于ArrayList遍历时不能修改的问题,由于arrayList是基于类中的`modCount`字段实现的`fail-fast`机制.比如说我们对一个size为12的ArrayList进行,每次遍历其中一个元素后,都要比较当前的modCount和开始遍历时的modCount是否一致,如果不一致的话,就会抛出`ConcurrentModificationException`(同步修改异常).这一点我们可以通过类中的`foreach`方法理解一下.
![imagepng](http://pcg4drw32.bkt.clouddn.com//file/2018/09/77242e23b5f742bd85b192d989ae5b96_image.png) 

ArrayList为我们提出了解决方法,这个方法就是使用迭代器对list进行遍历,首先调用`iterator()`方法获取一个迭代器对象(在下面的解释中,使用`iterator`命名这个对象),使用`iterator.hasNext()`方法判断是否还有可迭代对象,最后使用`iterator.next()`方法获取到下一个元素,如果想要对当前iterator进行`remove`操作,可以调用`iterator.remove()`,这个方法的关键在于将`迭代器中的modCount更新为当前对象最新的modCount值`.
![imagepng](http://pcg4drw32.bkt.clouddn.com//file/2018/09/fed4303ef70f4a6cbfc79889da5510ca_image.png) 
开头那个问题的详细解答读者可以看一下[空间复杂度o(1)下的移除ArrayList中的偶数项](http://www.indispensable.cn/articles/2018/09/05/1536080959531.html#b3_solo_h3_8).

### 多线程下的ArrayList
ArrayList实际上是非线程安全的,`在多个线程进行add操作时可能会导致elementData数组越界`,当ArrayList对象,仅能再添加一个元素时便需要扩容时.此时有两个线程都进行add操作.
![imagepng](http://pcg4drw32.bkt.clouddn.com//file/2018/09/5a30f77610744ab8b063fc85b113b4b5_image.png) 
![imagepng](http://pcg4drw32.bkt.clouddn.com//file/2018/09/9d1966a6d0cf406bac8cafceaaa6a51a_image.png) 

线程1首先进行扩容校验(ensureCapacityInternal)以后确定还有空余的一个位置,正要进行插入操作时,此时CPU将控制权交给了线程2,线程2确定了有空余空间以后,并进行了插入操作.此时的ArrayList对象的数组情况如下所示:
![imagepng](http://pcg4drw32.bkt.clouddn.com//file/2018/09/fa3369a8725a4b36aa156aa04b3dad05_image.png) 

`由于线程1之前已经进行了扩容校验`,此时将直接进行元素的插入操作,但实际上数组已经没有空余空间以供插入元素了.所以此时将会导致`数组越界异常`.可以通过打断点进行调试的方式复现此异常.
![imagepng](http://pcg4drw32.bkt.clouddn.com//file/2018/09/a1da966cd83a4506a1992813c1d41de3_image.png) 


## Vector

`Vector` 也是实现于 `List` 接口，底层数据结构和 `ArrayList` 类似,同样是一个动态数组存放数据。不过是在 `add()` 方法的时候使用 `synchronized` 进行同步，但是对资源的开销较大，所以 Vector `是一个同步容器`并不是一个并发容器。

### add(E e)
以下是 Vector的`add(E e)` 方法源码,主要就是在方法上加上了`synchronized`关键字：
```
    public synchronized boolean add(E e) {
        modCount++;
        ensureCapacityHelper(elementCount + 1);
        elementData[elementCount++] = e;
        return true;
    }
```
### add(int index, E element)
以及在指定位置插入数据:
```
    public void add(int index, E element) {
        insertElementAt(element, index);
    }
    public synchronized void insertElementAt(E obj, int index) {
        modCount++;
        if (index > elementCount) {
            throw new ArrayIndexOutOfBoundsException(index
                                                     + " > " + elementCount);
        }
        ensureCapacityHelper(elementCount + 1);
        System.arraycopy(elementData, index, elementData, index + 1, elementCount - index);
        elementData[index] = obj;
        elementCount++;
    }
```

## 后记
`ArrayList`经常会和`LinkedList`进行比较,二者的主要区别有以下几点:

- ArrayList是实现了基于动态数组的数据结构，LinkedList基于链表的数据结构。 
- 对于随机访问get和set，ArrayList觉得优于LinkedList，因为LinkedList要移动指针。 
- 对于新增和删除操作add和remove，LinedList比较占优势，因为ArrayList要移动数据。(在末尾新增元素二者并无差异)

`ArrayList`是非线程安全的,而Vector虽然是线程安全的,但是对系统资源的开销较大,如果想要在多线程情况下使用动态数组存储数据可以使用并发包下的`CopyOnWriteArrayList`类,有机会会在后面的文章中介绍这个类.



