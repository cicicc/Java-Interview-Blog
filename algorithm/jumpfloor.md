![](https://img.hacpai.com/bing/20180511.jpg?imageView2/1/w/960/h/520/interlace/1/q/100) 


### 题目背景简介
这是博主最近准备秋招时看的剑指offer上的一道题目,题目不是很难,但是我觉得比较有意思,所以发在了博客上面.

### 题目描述
> 一只青蛙一次可以跳上1级台阶,也可以跳上2级.....它也可以跳上n级,求该青蛙跳上一个n级台阶总共有多少种方法


如果哪位路过的朋友觉得这道题比较有意思的话可以暂时先不看下面的分析,自己先尝试一下写这道题.

![一张用来挡视线的图](http://qiniuyun.indispensable.cn//file/2018/07/ef124f3764b445f0b8df44848e4dad86_image.png) 



### 题目分析

在看到这个题目后,首先想到的应该是斐波那契数列的变形,那么思考这个问题应该从斐波那契数列的方向上着手.
当n=1时,f(n)=1,当n>2时,可以从以下角度进行考虑,青蛙最后一次上台阶可能是跳了1-n中任意一个数的台阶.
如果青蛙最后一步是跳了1个台阶的话,那么前面的n-1个台阶的跳法就有`f(n-1)`种,如下图:
![最后一步跳一个台阶=f(n-1)](http://qiniuyun.indispensable.cn//file/2018/07/595f6191336846b78755afccd7cb57cd_image.png) 


如果青蛙最后一步跳了2个台阶的话,那么前面的n-2个台阶的跳法就有`f(n-2)`种,如下图:
![最后一步跳一个台阶=f(n-2)](http://qiniuyun.indispensable.cn//file/2018/07/15d28b6e6633463baf18a5c8e03f8d57_image.png) 
依次类推青蛙最后一步跳了n-1个台阶的话,那么前面的1个台阶跳法等于`f(1)`,
除此之外还有一种情况就是青蛙一步从最低端跳到了最高点,也就是一步跳了n个台阶,在之前方法数目的基础上再次加`1`
![假装四级台阶是很大的n](http://qiniuyun.indispensable.cn//file/2018/07/99663babea6e4b239ac61cc41ba67366_image.png) 
所以跳到n级台阶的方法就有
>  f(n)=f(n-1)+f(n-2)....+f(1)+1

跳到n-1级台阶的方法就有
>  f(n-1)=f(n-2)....+f(1)+1

用`f(n)-f(n-1)`,有
>  f(n)-f(n-1)=f(n-1)

即`f(n)=2f(n-1)`,又因为f(1)=1,当n>=2时有
![通式](http://qiniuyun.indispensable.cn//file/2018/07/f67c50fb39184ef6abd8c06aa3ffa723_image.png) 
f(1)=1,为2的0次方,也满足上式,所以`上式为通式`,

###代码(Java)
由此写出的代码如下:

```
  
public Integer jumpFloor(int n) {
  if (n <= 0) {
  //错误的无法计算的台阶数目
  // throw new Exception("Error input,The number of steps must be greater than zero");
  return -1;
  } else if (n == 1) {
  return 1;
  } else {
  return 2*jumpFloor(n - 1);
  }
}

```
### 要注意两个事情
1. 在这个代码中,由于方法的数目是指数型增长的,所以当n>=32时,程序所得的数据就不正常了,因为此时的数据大于int的范围,int的范围[-2^31 , 2^31 -1] 即 [-2147483648,2147483647].
2. 程序的栈大小有限,当递归层数超过一定程度的时候,就会导致栈溢出现象,报错:
>Exception in thread "main" java.lang.StackOverflowError

当然在这一题里面,我指定 -Xss1m,栈溢出的时候所求的n已经大于9000了,数据早就已经超过int的范围了.感兴趣的读者可以使用BigInteger重写一下代码.











