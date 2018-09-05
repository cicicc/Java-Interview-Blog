![](https://img.hacpai.com/bing/20180325.jpg?imageView2/1/w/960/h/520/interlace/1/q/100) 

## java面试题之简答题(一)

### 题目一 高并发下的SimpleDateFormat

#### 题目内容
```
在一个有较大并发量的系统中,有人在某个公共类中书写了如下方法:

public static final SimpleDateFormat dateFormat = new SimpleDateFormat("yyyy-MM-dd HH-mm-ss");

public static String formatDate(Date date) {
  return dateFormat.format(date);
}

请问这么写会出现什么问题?请说明原因.

```

#### 题目解析
当高并发情况下,多个线程调用了formatDate方法,可能会出现`NumberFormatException`这个异常,我们使用如下代码进行测试
```
 private static final DateFormat DATE_FORMAT = new SimpleDateFormat("yyyy-MM-dd HH:mm:ss");

        public static void main(String[] args) throws InterruptedException {
            ExecutorService executorService = Executors.newFixedThreadPool(500);
            for (int i = 0; i < 500; i++) {
                executorService.execute(new Runnable() {
                    @Override
                    public void run() {
                        for (int i = 0; i < 1000000; i++) {
                            try {
                                DATE_FORMAT.parse("2018-09-04 00:00:00");
                            } catch (Exception e) {
                                e.printStackTrace();
                            }
                        }
                    }
                });
            }
            Thread.sleep(3000000);
        }

```
报错:
```
java.lang.NumberFormatException: For input string: ""
	at java.lang.NumberFormatException.forInputString(NumberFormatException.java:65)
	at java.lang.Long.parseLong(Long.java:601)
	at java.lang.Long.parseLong(Long.java:631)
	at java.text.DigitList.getLong(DigitList.java:195)
	at java.text.DecimalFormat.parse(DecimalFormat.java:2051)
	at java.text.SimpleDateFormat.subParse(SimpleDateFormat.java:2162)
	at java.text.SimpleDateFormat.parse(SimpleDateFormat.java:1514)
	at java.text.DateFormat.parse(DateFormat.java:364)
	at cn.love.potion.test.DateTest$1.run(DateTest.java:41)
	at java.util.concurrent.ThreadPoolExecutor.runWorker(ThreadPoolExecutor.java:1149)
	at java.util.concurrent.ThreadPoolExecutor$Worker.run(ThreadPoolExecutor.java:624)
	at java.lang.Thread.run(Thread.java:748)

```

这个问题产生一般的原因是`由于传过来的日期格式不对,所以导致解析时出现了问题,`但是我们的源代码中传递过去的日期格式是正确的.我们进入SimpleDateFormat类的源码中可以看到如下注释(基于jdk1.8):
![imagepng](http://pcg4drw32.bkt.clouddn.com//file/2018/09/49ae4544dd444dec9d6a8c79a8afa8da_image.png) 

大致翻译一下的意思就是:
> date formats并未同步。建议为每个线程创建单独的实例。如果多个线程同时访问format方法，则必须额外的同步它。

如果想了解具体错误的产生,可以看一下SimpleDateFormat类的源代码,在这里不做过多的介绍.如果看不懂的话可以在下面给我留言,我会重写一下这个解析的.常规的解决方法就是给方法加锁,使用synchronized就可以了,不需要过于纠结性能问题,`过早的优化是万恶之源`.


### 题目二 LinkedList下的错误操作

#### 题目内容
```
现有一个LinkedList类型的list,其中含有顺序[1....100000]个元素(10w),现在要打印后10000个元素,如下操作方式会有什么方面的问题?
for(int i = 90000; i<list.size(); i++){
    System.out.println(list.get(i));
}

```
#### 题目解析
大家都知道LinkedList的数据结构是基于双向链表的,当我们使用get方法时,会首先判断查找元素的索引是否大于1/2size.如果大于则从最后一个元素处开始向前查询,否则从第一个元素处开始向后查询

```
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

```
#### 解决方法 

`使用get的效率是很低下的`,不建议使用get方法获取大量的连续的LinkedList数据,推荐使用迭代器或者增强for循环(本质就是迭代器),在这一题中,可使用LinkedList自带的如下方法:
```
ListIterator<Integer> iterator = list.listIterator(90000);
   while (iterator.hasNext()) {
       System.out.println(iterator.next());
   }
```

### 题目三 空间复杂度o(1)下的移除ArrayList中的偶数项

#### 题目内容
```
以下代码打印列表中的偶数项,并将其移除,是否有问题?
 public void removeElement(){
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

#### 题目解析
ArrayList是基于类中定义的一个变量`modCount`实现的`fail-fast`机制,每次对ArrayList对象进行增删改时,都会进行`modCount++`这个操作.当我们在遍历ArrayList对象时,每次处理完当前对象以后,会比较对象的modCount是否和开始遍历时的modCount一致,如果不一致就报`ConcurrentModificationException`,这道题目中`在遍历中执行了remove操作`.所以就会报错.

#### 解决方法
在ArrayList类中提供了迭代器,使用其自带的迭代器进行遍历时移除就不会出错了.代码如下:
```

  Iterator<Integer> iterator = list.iterator();
        while (iterator.hasNext()) {
            Integer i = iterator.next();
            if (i % 2 == 0) {
                //说明为偶数,进行删除
                iterator.remove();
            }
        }
```

### 题目四 字符串的"=="和"equals"
#### 题目内容
```
写出如下方法调用的输出结果

String s1 = "hello";
String s2 = "hell";
String s3 = s2 + "o";
if (s1 == s3) {
	System.out.println("A");
} else if (s1.equals(s3)) {
	System.out.println("B");
}
```

#### 题目解析
  本题目的关键重在于理解字符串的创建和销毁过程,当我们使用`String s1 = "hello"`这种方式创建一个字符串时,会首先去常量池中寻找是否存在是否存在此字符串,如果不存在的话就在常量池中创建一个对象,并将栈中的引用指向这个对象.在本题目中,当进行`String s3 = s2 + "o";`时,会首先看常量池中是否存在"hello"对象,不存在则在常量池中创建一个对象,然后在堆中再创建一个对象,并将栈中的引用指向堆中的这个对象.
  所以在本题目中,当使用`==`进行比较时,由于是s1和s3的内存地址不同(一个指向的是常量池中的对象,一个指向的是堆中的对象),所以`s1==s3`这个条件不满足,又因为String类重写了equals方法,其比较的是字符串的内容是否相同而不是字符串的地址是否相同,所以`s1.equals(s3)`成立,打印B.


### 题目五 final修饰变量的不可变性

#### 题目内容
```
下列方法在执行时是否会出现什么问题,为什么?
public int addOne(final int x) {
  return x++;
}
```
#### 题目解析
当我们对一个变量设置为final时,那么就以为着这个变量引用的不可变性,对于基本类型及其包装类型其实也就是相当于数值的不可变,因为对于基础类型的包装类型,每次的++.--操作都是重新创建一个对象,并将引用指向新创建的对象,对于基础类型可以直接理解为内存所指向的对象的不可变性.所以在这一题中,x++操作时无论如何也不能执行的.

#### 题目扩展
当有如下的程序时,请问代码执行是否会出现问题?
```
public class FinalTest {

  public static void main(String[] args) {
  FinalTest finalTest = new FinalTest();
  Other other = new Other();
  finalTest.add(other);
  }

  private void add(final Other other) {
  other.i++;
  }

}
class Other{
  public int i = 0;
}
```
`答`:不会出现问题,因为之前已经说过了,final其实是保证着引用的不可变性,只要other对象地址不变,那么对other对象进行其他操作都是可以的,final未能实现真正的immutable的.
(未完待续.....)


欢迎关注我的博客以及公众号,才刚刚开始写博客,希望以后会更好!
![imagepng](http://pcg4drw32.bkt.clouddn.com//file/2018/09/56d3687bd2084ee5a24d16ce91580492_image.png) 










