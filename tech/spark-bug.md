## 关于Spark将数据放在jvm堆外所产生的bug(2016-09-28)

### 思路实现与问题产生
   我们内存计算小组博士的文章Deca经过一年的努力终于中了vldb,作为其中参与了一些微小的
工作的男人,虽然对其中一部分细节不是太清楚,因为这个Deca系统的开发涉及到将近5个人,
我的工作囊括一下就是: 1.完成了其中UDF的转换,将方法的操作转为对字节数组的操作;2.手写
了Deca的手动版的代码,就是利用Deca的思想对Spark应用的代码进行改造;3.进行了大量的测试
并统计GC时间,stage时间等相关数据.其实Deca系统的核心思想就是将原有的java大对象转化为
字节数组有序地放置在jvm中,这样一可以减少对内存的使用,也可以基本避免所有的GC.
附上论文[Deca论文地址](https://arxiv.org/pdf/1602.01959v3.pdf)

  老板的top中了之后,当然会将它扩展扩展然后投期刊,这是基本套路.要求扩展30%,其中就包括
将之前的手动版的数据放置在堆外,还是以字节数组的形式来和堆内版本进行比较,理论上来说
堆外版本肯定性能是比堆内好的,毕竟放置在堆外可以完全逃避GC的控制,也更加符合Deca的思想.

  代码实现并不难,基本由Unsafe这个类操作完成.大概思路就是将Spark应用中需要缓存的RDD
其中的partition的对象用字节数组的形式写在堆外,读的时候再直接按照偏移量读取,贴上
部分代码:
```java
  import UnsafePR._
  private val baseAddress = UNSAFE.allocateMemory(size)
  private var curAddress = baseAddress
  def address = baseAddress
  def free:Unit={
    UNSAFE.freeMemory(baseAddress)
  }
  def writeInt(num:Int):Unit={
    UNSAFE.putInt(curAddress,num)
    curAddress += 4
  }
```
  

