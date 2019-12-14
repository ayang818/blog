---
title: 使用0xff来保证二进制补码的一致性
date: 2019-12-14 22:58:36
tags: [老代码]
---
## 写这篇文章的起因
本来今天在写[浅析分布式唯一ID生成算法——snowflake的实现](https://ayang818.gitee.io/blog/2019/12/14/%E5%86%99%E5%86%99%E5%88%86%E5%B8%83%E5%BC%8F%E5%94%AF%E4%B8%80ID-snowflake%E7%AE%97%E6%B3%95%E7%9A%84%E5%AE%9E%E7%8E%B0/)这篇博客的时候，写的好好的，但是写到一种生成方式
```java
UUID.randomUUID().toString();
```
发现自己并不是很知道这个怎么实现的，然后点开了源码打算读一下。结果就发现了一段代码。
```java
/*
* Private constructor which uses a byte array to construct the new UUID.
*/
private UUID(byte[] data) {
    long msb = 0;
    long lsb = 0;
    assert data.length == 16 : "data must be 16 bytes in length";
    for (int i=0; i<8; i++)
        msb = (msb << 8) | (data[i] & 0xff);
    for (int i=8; i<16; i++)
        lsb = (lsb << 8) | (data[i] & 0xff);
    this.mostSigBits = msb;
    this.leastSigBits = lsb;
}
```
其中有一小段(下方代码段)我不是很理解有什么作用，因为data[i]的类型实际上是byte类型的，byte的十进制范围是-128-127，而0xff的十进制是15 * 16 + 16 = 256，转化为八位二进制就是11111111，我想着data[i] & 0xff的话是使用0xff做截取操作，但是byte转化为二进制的长度肯定在8位以内，所以这个截取操作第一眼看起来并没有什么用哈？
```java
data[i] & 0xff
```

<!-- more -->

但是看着这个写法感觉好像有些熟悉，因为之前写md5加解密好像看到过类似的代码，但那个时候是因为赶作业，所以就没有深究。我找回了当时的代码。代码地址如下——[代码地址](https://github.com/ayang818/GoopigeonTranslate/blob/master/app/src/main/java/com/example/finalwork/MainActivity.java#L206)

![](1.png)

同样也有类似的写法，但是这个代码里的写法又有些不一样。因为这里写的是0x000000ff，那就是32位。

我这才意识到原来0xff是一个int类型的值。那么完整的写就是0x000000ff，任何数与0x000000ff做截取的结果就是令高24位都为0，保留低八位。

而data[i]作为一个byte类型的值。比如data[i] = -127，并且byte类型是有符号类型。那么-127的8位二进制补码表示就是0x10000001，但是事实上byte和int做&运算的时候byte若为负数，那么data[i]会转型为int，转化后的高24位都会为1，这个时候转换为int后的data[i]和原来的data[i]他们的源码十进制数虽然相同，但是事实上他们的二进制补码已经发生了改变。

用下面的代码举个例子(Integer.toHexString()的作用就是查看他们的16进制补码)
```java
byte a = -128;
byte b = 127;
int a1 = a & 0xff;
int b1 = b & 0xff;
int res1 = a * 200;
int res2 = b * 200;
int res3 = a1 * 200;
int res4 = b1 * 200;
System.out.println(res1);
System.out.println(res2);
System.out.println(res3);
System.out.println(res4);
System.out.println(Integer.toHexString(res1));
System.out.println(Integer.toHexString(res2));
System.out.println(Integer.toHexString(res3));
System.out.println(Integer.toHexString(res4));
```
结果为
```
-25600
25400
25600
25400
ffff9c00
6338
6400
6338
```
可以看到对于正数的byte类型，和int互操作不会有什么影响；但是对于负数的byte，若是不对其进行& 0xff的话，转化后的值的补码前24为均为1(结果中的ffff9c00)。

---
所以说到这里已经很明白了，使用0xff做截取转换的作用就是防止byte为负时与int进行计算，从而令byte先转化为高24位均为1(补码)的int。虽然这样使用0xff截取后，byte转化为int后的十进制并不等于原byte的十进制，但是却保证了他们的二进制补码的一致性。