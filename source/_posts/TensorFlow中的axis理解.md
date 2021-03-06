---
title: TensorFlow中的axis理解
date: 2018-04-11 20:05:01
tags: [TensorFlow,tf.reduce_max,tf.concat，axis]
categories:
- TensorFlow
---

在TensorFlow的很多函数中都有到了axis参数，对于这个参数怎么理解容易让人犯晕，二维情况下还好，如果升到高维更加难以理解，所以本文用三维情况下的两个常见函数为例，让我们彻底理解这个参数的意义。

在开始之前，先开宗明义，axis参数的意义就是在哪个维度上做操作

## tf.reduce_max()

在介绍这个函数之前，我们先简单说一下reduce的含义，这个操作无论在python或者是更高级的mapreduce中都会碰到，我对它的解释就是对列表中的元素做某一连续操作。例如reduce_max就是做连续的max操作，例如对列表`[1,2,3,4]`进行reduce_max就是先对元素1和2进行max操作，然后其结果和3再做max，再然后结果和4做max。理解了reduce操作后我们自然就明白了axis的必要性，如果把一个多维数组简单的当做一个一维列表的话进行reduce操作毫无疑问是没问题的，事实上如果不加axis参数的话，tf.reduce_max就是这么干的。

例如如下代码

```
import tensorflow as tf
a=[[[1,2,3,4],[5,6,7,8],[8,9,10,11]],[[11,12,13,14],[15,16,17,18],[19,20,21,22]]]
sess = tf.InteractiveSession()
print tf.reduce_max(a).eval()
```

运行结果为

```
22
```

但是有时候我们不希望这么简单粗暴，毕竟如果这么计算的话我们定义这么复杂的结构就毫无意义了，例如我们有一个二维数组

```
s=[[14,175],
   [13,182]]
```

代表了我们班里有两个同学，小明14岁身高175，小刚13岁身高182，我们想通过一个运算获取班里最大的同学多少岁，以及最高的同学多高，这个时候axis就排上了用场，用上面的例子来说，axis为0表示同学维度，axis为1表示同学自身属性维度。

```
   ___属性__
 | 
 学[[14,175],
 生[13,182]]
 |
```

在维度0运行reduce操作表示在学生维度上进行迭代，最终学生维度小时，两个属性维度保留，我们得到了一个‘超级学生’，岁数最大，身高最高

```
print tf.reduce_max(s,0).eval()  #output: [ 14 182]
```

在维度1上运行reduce操作表示在属性维度上进行迭代，我们依然保留了两个‘学生’，只不过他的属性只剩下了最突出（max）的一个

```
print tf.reduce_max(s,0).eval()  #output: [ 175 182]
```

虽然两个运算后都只留下了1维，但是一维中的两个元素的含义是不同的。

---

有了上面的简单示例，我们再用一个稍复杂的例子加深印象

```
import tensorflow as tf
a=[[[1,2,3,4],[5,6,7,8],[8,9,10,11]],[[11,12,13,14],[15,16,17,18],[19,20,21,22]]]
sess = tf.InteractiveSession()
print tf.reduce_max(a,0).eval()
print tf.reduce_max(a,1).eval()
print tf.reduce_max(a,2).eval()
```

输出结果为

```
[[11 12 13 14]
 [15 16 17 18]
 [19 20 21 22]]
[[ 8  9 10 11]
 [19 20 21 22]]
[[ 4  8 11]
 [14 18 22]]
```

这是一个[2,3,4]维的数组，怎么找每一维呢，很简单，一个中括号代表一维，最外侧的中括号是第0维，我们可以看成

```
a=[A1，B1]  #第0维
A=[[1,2,3,4],[5,6,7,8],[8,9,10,11]]
B=[[11,12,13,14],[15,16,17,18],[19,20,21,22]]
A=[AA,AB,AC]
AA=[1,2,3,4]
AB=[5,6,7,8]
AC=[19,20,21,22]
```



对第0维操作，就是求A和B较大值（按对应元素求），所以得到一个[3,4]维度的数组

第1维也很明显了，就是A1和B1，那么对第1维操作时，第0维不动，仍然是A，B，但是对A、B本身要进行操作(这里就跟前面的二维数组类似了)，以A为例，要求AA，AB，AC的最大值（按对应元素求），所以A变为了[ 8  9 10 11]，B变为了[19 20 21 22]

第2维留给读者练习

## tf.concat()

与reduce不同，reduce可以看成是对某一维的缩减，但是concat作为一个连接操作，则是对某一维的扩充。

请看如下示例

```
import tensorflow as tf
a=[[14,175],
   [13,182]]
b=[[14,173],
   [15,180]]
sess = tf.InteractiveSession()
print tf.concat([a,b],0).eval() #可以看成两班学生合并，因为是在学生维度合并
print tf.concat([a,b],1).eval()
```

输出结果

```
[[ 14 175]
 [ 13 182]
 [ 14 173]
 [ 15 180]]
[[ 14 175  14 173]
 [ 13 182  15 180]]
```

按照第0维连接，则[2,2]变成了[4,2]，也即可以看成

```
a=[A,B]
b=[C,D]
#连接后是[A,B,C,D]
```

按照第1维连接，要保证0维还是两个元素，然后要对A进行操作，和C合并。

## 总结

axis的意义就是你选择的操作在某一维度上执行，执行后的结果与你选择的操作有关，你选择reduce操作后，所选择维度小时，你选择concat操作后所选择维度进行连接，元素扩展，多tensor操作时按照对应位置进行操作，如concat示例中A对应C，选择维度直观上说就是看第几层括号，以最后一个示例为例，维度0就是最外层括号内元素变化，维度1就是里层括号`A=[14,175]`合并了`C=[13,173]`。

