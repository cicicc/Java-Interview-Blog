## 引言

在前面的文章中, 我们对 ArrayList 做了较为详细的源码解读, 今天将在这篇文章中继续对 LinkedList 的源码作出解读, 本文中的 LinkedList 是基于 jdk1.8. 

## LinkedList 底层分析

### LinkedList 数据结构

![](http://qiniuyun.indispensable.cn/file/2018/09/ed3f89bd227b4e6d83156f4a555ed044_image.png)

![](http://qiniuyun.indispensable.cn/file/2018/09/e8a1a2a4b08c499b95b22a9f64cefc94_image.png)

如上图所示 `LinkedList` 底层是基于`双向链表`实现的, 其中 LinkedList 的头结点不存放数据, 仅存放指针.

### 常用字段

> transient int size = 0;// 当前 LinkedList 对象中存储的数据数目
> transient Node first;// 起始节点
> transient Node last;// 结束节点
> 以上的三个字段全都使用了`transient`关键字进行修饰, 因为该三个字段可能为空, 以免在自动序列化中引起错误使得序列化失败

## 常用方法

### add(E e)

将元素插入`链表末尾`
 ```Java
 public boolean add(E e) {
        linkLast(e);// 将元素插入 ` 链表末尾 `
        return true;
    }
 void linkLast(E e) {
	final Node<E> l = last;// 获取结束节点
	final Node<E> newNode = new Node<>(l, e, null);
	last = newNode;
	if (l == null)
		first = newNode;// 若链表为空链表, 将新节点赋值给给 first 节点
	else
		l.next = newNode;
	size++;
	modCount++;
}  
```	

由源码可见每次插入数据都是移动指针，相比 `ArrayList`的拷贝数组来说效率要高上不少。时间复杂度为`o(1)`

### add(int index, E element)

将元素插入`链表指定位置`
```Java
  public void add(int index, E element) {
        checkPositionIndex(index);

        if (index == size)
            linkLast(element);
        else
            linkBefore(element, node(index));
    }

  Node<E> node(int index) {
	  // assert isElementIndex(index);
	 //判断index是否大于1/2的size,若小于1/size则从first结点开始向后遍历,否则从last节点开始向前遍历
	  if (index < (size >> 1)) {
		  Node<E> x = first;
		  for (int i = 0; i < index; i++)
			  x = x.next;
		  return x;
	  } else {
		  Node<E> x = last;
		  for (int i = size - 1; i > index; i--)
			  x = x.prev;
		  return x;
	  }
  } 
```	

上述代码，利用了双向链表的结构特点 (可双向遍历)，如果`index`离链表头比较近，就从节点头部遍历。否则就从节点尾部开始遍历。使用空间（双向链表的每个节点的前后指针所占用空间）来换取时间。

- `node()`会以`O(n/2)`的时间复杂度去获取一个结点
- 如果索引值大于链表大小的一半，那么将从尾结点开始遍历

这样的效率是非常低的，特别是当 index 越接近 size 的中间值时。

### get(int index)

获取指定索引的元素

```Java
public E get(int index) {
        checkElementIndex(index);
        return node(index).item;
    }
 Node<E> node(int index) {
        // assert isElementIndex(index);

        if (index < (size >> 1)) {
            Node<E> x = first;
            for (int i = 0; i < index; i++)
                x = x.next;
            return x;
        } else {
            Node<E> x = last;
            for (int i = size - 1; i > index; i--)
                x = x.prev;
            return x;
        }
    }
```	

`get()`方法和前面的`add(int index, E element)`一样, 需要调用到`node(int index)`方法进行查找.

### peek()和 poll()

`peek()`方法和`poll()`是如果 list 中的`开始节点`, 但`peek()`方法不会将节点从链表中移除, 而`poll()`方法会.

```Java
public E peek() {
   final Node<E> f = first;
   return (f == null) ? null : f.item;
}

public E poll() {
   final Node<E> f = first;
   return (f == null) ? null : unlinkFirst(f);
} 
```

其余方法如`writeObject(),``readObject()`方法与`ArrayList`中的方法类似, 此处不做过多介绍.

## 最后的思考题
```Java
现有一个LinkedList类型的list,其中含有顺序[1....100000]个元素(10w),现在要打印后10000个元素,如下操作方式会有什么方面的问题? 
for(int i = 90000; i<list.size(); i++){
    System.out.println(list.get(i)); 
} 
```

具体答案可以参考[LinkedList 下的错误操作](http://www.indispensable.cn%2Farticles%2F2018%2F09%2F05%2F1536080959531.html%23b3_solo_h3_4), 这篇文章就这么结束了, 似乎写的有点水啊! 读者觉得有什么需要补充的欢迎在下面留言.
ps:由于之前不小心部署了两个博客系统,在这里出现了问题,所以我重新上传了一下这篇文章
