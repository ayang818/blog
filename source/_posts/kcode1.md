---
title: 快手kcode程序设计大赛初赛总结
date: 2020-07-02 19:33:57
tags: [性能挑战赛]
---
> 最终分数：13622.19
> 仓库地址：https://github.com/ayang818/KCoder-First

为时14天的kcode初赛算是结束了（6/17 - 7/1），由于个人的时间安排关系，我差不多是从24号开始正式开始写代码的。由于分数不算很高，排名按照封榜前大概是 （30 ~ 35） / 100，估计还有一大堆没写出来的，毕竟群里900多号人，但是前10的大佬们已经基本都是2.5w+了，但是由于爬分的过程还是非常有趣的，所以还是还是打算写一篇博客来梳理一下优化的过程。

我的爬分过程：**600 -> 3600 ->   5000 -> 9400 ->  11000 -> 13600**


<!-- more -->
在分析优化方案之前先做一下赛题解析

## 题目内容

实现进程内对微服务调用信息的分析和查询系统，要求包含的功能如下：

- 接收和分析调用信息的接口
- 根据给定的条件进行查询和分析，描述如下：
  - 输入主调服务名、被调服务名和时间（分钟粒度），返回在这一分钟内主被调按ip聚合的成功率和P99；
  - 输入被调服务名和一个时间范围（分钟粒度, **==闭区间==**），返回在区间内被调的平均成功率

## 数据和输入输出说明：

输入数据格式：

```
每个调用记录存储在一行，数据直接以逗号分割(,), 换行符号(\n)

主调服务名,主调方IP,被调服务名,被调方IP,结果,耗时(ms), 调用发生的时间戳(ms)
commentService,172.17.60.2,userServie,172.17.60.3,true,89,1590975771020
commentService,172.17.60.3,userServie,172.17.60.4,false,70,1590975771025
commentService,172.17.60.2,userServie,172.17.60.3,true,103,1590975771030
commentService,172.17.60.3,userServie,172.17.60.4,true,79,1590975771031
commentService2,172.17.60.3,userServie,172.17.60.4,false,88,1590975771031
commentService2,172.17.60.3,userServie,172.17.60.4,true,91,1590975771032
```

成功率定义：

```
单位时间内被调服务成功率 =  （单位时间内被调用结果为ture的次数）/ 单位时间内总被调用次数  【结果换算成百分比，小数点后保留两位数字,不足两位补0，其余位数直接舍弃】

以上示例数据计算一分钟的成功率
 - （2020-06-01 09:42）userServie在被调成功率: 4/6= 66.66%
 - （2020-06-01 09:42）userServie在172.17.60.4机器上成功率: 2 / 4 = 50.00%
 如无特殊说明，赛题中说的成功率以单位时间为一分钟计算
```

如果计算结果是0，请表示成 **.00%**

查询输出格式说明：

```
查询1（checkPair）：
输入：主调服务名、被调服务名和时间（分钟粒度）
输出：返回在这一分钟内主被调按ip聚合的成功率和P99, 无调用返回空list （不要求顺序）
输入示例：commentService，userServie，2020-06-01 09:42
输出示例：
172.17.60.3,172.17.60.4,50.00%,79
172.17.60.2,172.17.60.3,100.00%,103

查询2（checkResponder）：
输入： 被调服务名、开始时间和结束时间
输出：平均成功率，无调用结果返回 -1.00%
平均成功率 = （被调服务在区间内各分钟成功率总和）/ (存在调用的分钟数）【结果换算成百分比，小数点后保留两位数字,不足两位补0，其他位直接舍弃，不进位】

输入示例：userServie, 2020-06-01 09:42, 2020-06-01 09:44
输出示例：66.66% 
计算过程示例：
（2020-06-01 09:42）userServie被调成功率: 4/6= 66.66%
（2020-06-01 09:43）userServie无调用
（2020-06-01 09:44）userServie无调用
 那么userServie在2020-06-01 09:42到2020-06-01 09:44的平均成功率为（66.66%）/ 1 = 66.66%【示例数据仅仅在2020-06-01 09:42有调用】
```

如果计算结果是0，请表示成 **.00%**

## 评测环境&启动参数

- JDK 版本： 1.8
- jvm内存设置 : -XX:+UseG1GC -XX:MaxGCPauseMillis=500 -Xss256k -Xms6G -Xmx6G -XX:MaxDirectMemorySize=1G
- 评测机器硬件信息（docker）：
  - 操作系统 CentOS 7.3 64位
  - CPU 8核 3.00GHz
  - 硬盘：容量 100GB， 吞吐量 > 100MB/S
- 如果需要输出文件，请使用 /tmp/ 目录，**禁止使用其他目录**


题目内容就这么些，题目是很容易读懂的，简单的来说就是读入解析一个12G的具有一定格式的日志文件，然后建立一定的便于查询的数据结构，进行两种方式的查询。主要的得分点

- 第一阶段快速读入解析文件
- 二三阶段查询的时间复杂度尽可能小


## 爬分过程

### 600分baseline

![](https://pic-serve.oss-cn-beijing.aliyuncs.com/blog/image-20200702203840861.png)

600分的baseline大概花了一天不到就写完了，其实本来是一个晚上左右就写完了的，但是由于各种综合原因踩到了一个比较低级的坑，所以在线上无日志的情况下找了将近一天的BUG。

baseline的做法非常暴力

1.第一阶段。首先我的第一阶段的目标是就是在单线程环境下加速IO速度，根据我以往的优化方案，第一想法就是使用块读写。

```java
RandomAccessFile memoryMappedFile = new RandomAccessFile(path, "r");
FileChannel channel = memoryMappedFile.getChannel();
// try to use 16KB buffer
ByteBuffer byteBuffer = ByteBuffer.allocateDirect(1024 * 64);
int size = 0;
while (channel.read(byteBuffer) != -1) {
byteBuffer.flip();
    int remain = byteBuffer.remaining();
    byte[] bts = new byte[remain];
    byteBuffer.get(bts, 0, remain);
    byteBuffer.clear();
    processBlock(bts);
    size += 64;
}
```

这里也是我踩到的第一个坑，由于平时都没有注意文件的权限管理，我的baseline在线上死活通不过的原因就是因为顺手写了 w 权限，

```java
RandomAccessFile memoryMappedFile = new RandomAccessFile(path, "rw");
```

但是很显然赛题方是不会给我们写权限的，所以这里线上就一直报错，由于线上的日志只会打印Exception中的信息，但是这里在线上报的Security Exception又被catch了，导致没有一点点错误痕迹。只能通过肉眼在全局代码中一行行找可疑代码。这里花了我将近一天的时间。

但是其实对于块读来说，对于需要按行解析的数据是非常不友好的，由于每行的长度并不是固定的，所以不管一次块读的大小是多少，总会有块的末尾的数据不是完整的，需要和下一个块的开头的不完整部分的数据做拼接。并且也是需要遍历所有的字节，找到所有的  `\n` 换行符来构建行，从而解析行，同时解析行同样是需要花费时间的。

实测在单线程环境下，如果处理行数据不是那么快，使用块读的效率和使用 `bufferedReader.readLine()` 的速度基本上也差不了多少， 因为这个时候瓶颈就不是调用IO，而是CPU解析的时间了。



2.第二阶段、查询 1，第二阶段稍微麻烦点的是p99的计算，我建立了如下的数据结构来做为baseline的方案。

```java
Map<caller+responder, Map<timestamp, Map<callerIp+responderIp, Span>>

Span:
static class Span {
    int sucTime;
    int totalTime;
    List<Integer> list;

    public Span() {
        sucTime = 0;
        totalTime = 0;
    }
 }
```

由于需要计算p99，最简单的做法当然是把所有数据都存起来，然后需要p99时，将所有数据排序，然后取 len * 0.99 位置的数值即可。这里的时间复杂度由于需要查询p99时需要排序，并且需要遍历 `Map<callerIp+responderIp, Span>` 来构造List，所以时间复杂度为 `O(n^2logn)`



3.第三阶段、查询 2，第三阶段比较简单，建立的存储数据结构如下

```java
Map<responder, Map<timestamp, Span>>
```

由于第三阶段是一个时间区间，所以查询的时候，只需要遍历需要收集的时间区间然后计算成功率即可。时间复杂度为 `O(n)`，



这样的一个快速方案原型，在我本地跑出的耗时如下，其中第二阶段查询次数为1w，第三界阶段查询次数为100w

![](https://pic-serve.oss-cn-beijing.aliyuncs.com/blog/image-20200702212127887.png)




### 3600分

![](https://pic-serve.oss-cn-beijing.aliyuncs.com/blog/image-20200702221645278.png)

写出baseline之后，首先优化的是第一阶段，由于线上的机器配置非常好，8核CPU，所以肯定需要尽量释放机器性能。当前的想法有两个

1. 多线程读文件，多线程处理数据
2. 单线程读文件，多线程处理数据

但是在使用块读的情况下，要是使用多线程块读 读文件，同步策略会变得异常复杂，编码难度也大大增加，在短暂的尝试后，我选择了单线程读文件，多线程处理数据的方案。

但是当我把每一行作为一个任务提交到 8 线程线程池中进行处理解析是，运行时发现，运行时间却大大增加，统计了每行平均处理时间后，发现每行的处理时间大概在 200-2000ns，然而根据网上搜到的资料，线程切换导致的上下文切换开销也基本是这个量级，所以虽然使用了线程池，但是还是导致了大量的线程切换。对此我的[解决方案](https://github.com/ayang818/KCoder-First/blob/master/src/main/java/com/kuaishou/kcode/KcodeRpcMonitorImpl.java#L75)就是先收集 n 行数据，然后再一起提交到线程池中处理。

```java
index++;
while (readline()) {
    ...
    index++;
    if (index >= threshold) {
        final String[] tmp = list;
        threadPool.execute(() -> handleLines(tmp));
        list = new String[threshold];
        index = 0;
    }
}
```

到这里我的本地prepare耗时降到了（由于速度基本没有什么差距，所以我干脆把块读换成了代码更简洁的 `BufferReader.readLine()`）

![](https://pic-serve.oss-cn-beijing.aliyuncs.com/blog/image-20200702214034791.png)



一阶段的优化到这里基本就结束了，线上的一阶段耗时大致在 20s 左右。



接下来是第二阶段的优化，最需要优化的肯定是 p99 的计算，其实简单思考一下就可以想出 p99 的内存和时间上的最佳方案—— **桶排序**，由于 p99 统计的是调用时间，而调用时间一般都集中在 0 - 350 这个区间中，所以这里的代码就很简单了，只需要让调用时间对应下标的bucket的值+1即可。

```java
// 解析
synchronized (this) {
	bucket[costTime] += 1;
}

// 计算p99
public int getP99() {
	int pos = (int) (totalTime.get() * 0.01) + 1;
	int len = bucket.length;
	for (int i = len - 1; i >= 0; i--) {
		pos -= bucket[i];
		if (pos <= 0) return i;
	}
	return 0;
}
```

所以获取p99的复杂度从 `O(nlogn)`降到了 `O(1)`，阶段二的总体复杂度也降到了 `O(n)`

本地耗时，二阶段10w次，三阶段10w次

![](https://pic-serve.oss-cn-beijing.aliyuncs.com/blog/image-20200702215752247.png)



### 5000分

![](https://pic-serve.oss-cn-beijing.aliyuncs.com/blog/image-20200702221743598.png)

小的优化：

1. 重写了自定义的DateFormatter。

2. 重写了截取两位小数的DecimalFormatter。
3. 优化查询2数据结构 `Map<responder, Map<timestamp, Span>>` 变成  `Map<responder, Span[]>`，虽然这两个数组和map的复杂度都是 `O(1)`，但是map在get时还是会需要进行hash运算，而数组则是直接通过偏移量取值。

（本来到了5000分就不想继续优化代码了，毕竟考试太多了，但是 kcode 的 QQ 群里突然有人发了个6000分的demo，瞬间自闭。但是本着比赛期间不太想看别人的代码的意思，于是还是决定继续对着自己的代码琢磨，（考试，危！



### 9400分

![](https://pic-serve.oss-cn-beijing.aliyuncs.com/blog/image-20200702221759478.png)

这里算是分数进步最大的一次优化了，虽然感觉代码上并不是很难。。。

对于阶段二，其实当前的时间复杂度为`O(n)`，因为需要在查询时遍历一组服务队下所有的 IP 对，但是我想了一下之后，发现其实只要在阶段一结束后，遍历收集好的整个 Map，首先将这些 IP 对中的数据直接收集到一个 List 中即可。这样查询时就可以根据服务名+时间戳直接找到对应的 List 了

```java
checkOneMap.forEach((key, timestampMap) -> {
    // 遍历所有时间戳
    timestampMap.forEach((k, ipPairMap) -> {
        String[] split = key.split(",");
        String resKey = split[0] + split[1] + k;
        List<String> resList = new ArrayList<>(20);
        ipPairMap.forEach((ipPair, span) -> resList.add(span.getRes()));
        checkOneResMap.put(resKey, resList);
    });
});
```



对于阶段三，由于需要遍历区间中所有的分钟，才可以得到区间整体的成功率，所以这里就是一个非常烦的地方，虽然可以预先生成可能的所有区间的值，但是这种做法对于实际工程来说过于离谱。所以几经斟酌后还是没有这样做。

最后感觉可能只能构建缓存来让一部分 `O(n)` 操作变成 `O(1)`

```java
@Override
public String checkResponder(String responder, String start, String end) {
    String hash = responder + start + end;
    String res;
    if ((res = checkTwoResMap.get(hash)) == null) {
        Span[] timestampMap = checkTwoMap.get(responder);
        if (timestampMap == null) {
            checkTwoResMap.put(hash, "-1.00%");
            return "-1.00%";
        }
        long startMil = parseDate(start);
        long endMil = parseDate(end);
        double times = 0;
        double sum = 0;
        Span span;
        for (long i = startMil; i <= endMil; i += 60000) {
            span = timestampMap[minuteHash(i)] ;
            if (span != null && span.sucRate != 0) {
                sum += (span.getSucRate() * 100);
                times++;
            }
        }
        if (sum == 0) {
            if (times == 0) {
                checkTwoResMap.put(hash, "-1.00%");
                return "-1.00%";
            }
            checkTwoResMap.put(hash, ".00%");
            return ".00%";
        }
        String value = formatDouble(sum / times) + "%";
        checkTwoResMap.put(hash, value);
        return value;
    }
    return res;
}
```

但是运气比较好，这一次构建缓存的尝试让我的分数高了不少，到达了接近1w的分数，但是进复赛还是没有很稳，所以顶着两天连续考试的压力，又继续尝试优化。



### 11000分

这里其实没有做什么优化，单纯的把线程池的大小从 8 调到了 16，但是分数提升还挺多。



### 13600分

![](https://pic-serve.oss-cn-beijing.aliyuncs.com/blog/image-20200702222500471.png)

到了11000后其实感觉还不是很稳，但是实在想不到以当前的代码还有什么可以优化的，但是感觉进复赛还不是很稳。由于当前二三阶段的查询都已经优化到了 `O(1)` ，所以感觉可以优化的只有常数。观察原来的代码，可能比较耗时的操作应该就是

```java
String hash = responder + start + end;
```

和

```java
String resKey = caller + responder + parseDate(time);
```

所以最后在分数的诱惑下还是选择了写一个自定义hash函数取代拼接字符串，但是实际工程中是不可能这么做的，只能寄希望于线上不要有hash冲突。

```java
	public Integer hash(Object a, Object b, Object c) {
        Integer res = 1;
        res = 1313 * res + a.hashCode();
        res = 1313 * res + b.hashCode();
        res = 1313 * res + c.hashCode();
        return res;
    }

// 查询1
Integer resKey = hash(caller, responder, parseDate(time));
// 查询2
Integer hash = hash(responder, start, end);
```

最后拿到了13600这个差不多稳进复赛的成绩，心累。



## 技术外的一些想法

2020的上半年，这个大二下学期的尾巴可以说过的非常忙碌了，六月参加了两个比赛，天池的中间件大赛、快手的kcode程序设计大赛、然后还有一大堆赶不完的大作业、期末考的复习...，不过还算都完成的还能接受，运气好一点两个都进复赛，参加这两个比赛对我也有很大的锻炼（至少比写CRUD要开心多了）。

想一想放完暑假就是大三，大三上学期过完就要去找暑期实习了，这么想想感觉大学剩下来的时间也不算很多了。然而感觉大学还是有好多事情没有做过啊，没去过一直想去的日本，没谈过恋爱，连之前一直非常喜欢的羽毛球也没有继续打下去...... ，一大堆没有干过的事情让我有种到底是我太忙了还是我不想去做的错觉。下学期社团和工作室的和辅导员对接好的一堆赛事和活动我也得去组织和安排，郁闷。如果自私一点想的话，这些东西不去做也不会有啥问题吧，无非就是科协还是没有生气的老样子、工作室人也越来越少。但是这些地方都让我收获了很多东西，虽然他们变得越来越差也不会对我以后有任何影响，但总还是有一些不甘心，想尝试去挽救一下，希望人没事~

写了快一个晚上了，还是希望下半年也能一切顺利吧，感觉会有好事发生。