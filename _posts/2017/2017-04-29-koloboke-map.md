---
title: 使用Koloboke 生成高性能的 JAVA 基本类型 map/set
date: '2017-04-29 18:25'
categories: 开发
tags: java koloboke ConcurrentHashMap 基本类型 map set
---

Java 标准类库自带了很多集合类型, 比如各种 List, Set, Queue, Map. 每种抽象集合类型往往有多种实现, 比如 Map 的实现有 TreeMap, HashMap. TreeMap 采用二叉平衡树实现, 键的存储是有序的, 支持按序遍历和范围搜索. HashMap用哈希表实现, 只支持精确搜索, 但是效率更高一些. 每种集合类型都有相应的支持并发操作的实现版本,比如最常用的ConcurrentHashMap. 而且所有集合类型都支持泛型, 这让我们用 JAVA 开发时, 几乎不用重复造轮子, 不用自己实现这些基本类型. 但是, 永远会有一个但是:) JAVA 的泛型不支持基本类型, 如果我们想要一个键值都为 int 类型的 map, 不能用 `Map<int, int>`, 只能用 `Map<Integer, Integer>`. `Map<int, int>` 和 `Map<Integer, Integer>`之间的差别有多大? 首先后者的put 等操作, 如果传进的是一个 int 类型, 则java 编译器会产生一个 box 操作, 把 int 转会为 Integer. 其次这两者的最主要差别在于存储空间利用. 如果这两个 map 都采用哈希表实现的话, `Map<int, int>`可以用采用两个 int 数组(int[])的 哈希表实现, 如果哈希表的 capacity 为1M, 那么该数据结构占用的空间大概是8M. 而`Map<Integer, Integer>`的实现则不大一样. 下面我们看看向一个ConcurrentHashMap<Integer, Integer>对象插入1M 个键值对后的内存占用情况.(由工具 [jol](http://openjdk.java.net/projects/code-tools/jol/) 产生)

```
java.util.concurrent.ConcurrentHashMap@67f89fa3d footprint:
     COUNT       AVG       SUM   DESCRIPTION
         1   8388624   8388624   [Ljava.util.concurrent.ConcurrentHashMap$Node;
   1999872        16  31997952   java.lang.Integer
         1        64        64   java.util.concurrent.ConcurrentHashMap
   1000000        32  32000000   java.util.concurrent.ConcurrentHashMap$Node
   2999874            72386640   (total)
```
最后一行是一个统计数据, 表明该ConcurrentHashMap对象总共包含了2999874个对象, 占用72386640字节内存, 即约72M. 内存主要被以下三项占用:
- 一个 Node数组, 占用8M. 因为一个 Node 指针的大小为4字节, 所以数组长度为2M.
- 1M 个 Node 对象, 占用32M. 刚好对应我们插入的1M 个对象
- 1999872个 Integer 对象, 占用大概32M.

你可能会疑惑, 为什么是1999872个 Integer 对象, 而不是2000000个? 这是因为 JAVA 对[-128,127]之间的256个 Integer 对象做了缓存, 而ConcurrentHashMap中插入的1M 个键值对为从(0,0)到(999999,999999). 这样有[0,127]共128个数字落在这个缓存中, 他们会直接引用缓存中的对象, 而不会额外创建两个(键和值分别一个). 这也是为什么只有1999872个(少了128个) Integer 的原因.

从上面的例子我们可以看出, 由于 java 的泛型不支持基本类型, 导致程序会占用大量额外的内存和对象. 这也是大量对性能要求比较高的 JAVA 程序, 都会自己实现相应的针对基本类型的集合类的原因. 但是实现一套高质量的集合类型并不是一件容易的事情, 有没有办法快速生成呢? [Koloboke](https://github.com/leventov/Koloboke)就是用来做这个的. 下面我们看看使用 Koloboke 编写的支持并发读写的 Map 类.

```java
@KolobokeMap
public abstract class SynchronizedMap implements Map<Integer, Integer> {
    public static SynchronizedMap withExpectedSize(int expectedSize) {
        return new KolobokeSynchronizedMap(expectedSize);
    }

    public final synchronized int get(int key) {
        return subGet(key);
    }

    public final synchronized int put(int key, int value) {
        return subPut(key, value);
    }

    public final synchronized int size() {
        return subSize();
    }

    @MethodForm("get")
    abstract int subGet(int key);

    @MethodForm("put")
    abstract int subPut(int key, int value);

    @MethodForm("size")
    abstract int subSize();
}
```

在 pom.xml 中添加
```xml
<dependency>
    <groupId>com.koloboke</groupId>
    <artifactId>koloboke-compile</artifactId>
    <version>0.5.1</version>
    <scope>provided</scope>
</dependency>
<dependency>
    <groupId>com.koloboke</groupId>
    <!-- `jdk6-7` instead of `jdk8` if you use Java 6 or 7 -->
    <artifactId>koloboke-impl-common-jdk8</artifactId>
    <version>1.0.0</version>
</dependency>
```

然后运行 mvn package, koloboke就会帮我们生成`KolobokeSynchronizedMap`这个类, 是不是很棒?! 我们再用 jol 看看koloboke生成的 map 的内存占用情况. 同样是插入1M 个键值对, KolobokeSynchronizedMap只有8个对象(ConcurrentHashMap有2999874个), 占用16M(ConcurrentHashMap占用72M), 差距非常大!!

```
qye.KolobokeSynchronizedMap@17d265fdd footprint:
     COUNT       AVG       SUM   DESCRIPTION
         1  16777232  16777232   [J
         1        48        48   com.koloboke.collect.hash.AutoValue_HashConfig
         1        24        24   com.koloboke.collect.impl.Scaler$10
         1        24        24   com.koloboke.collect.impl.Scaler$4
         1        24        24   com.koloboke.collect.impl.Scaler$8
         1        24        24   com.koloboke.collect.impl.Scaler$9
         1        40        40   com.koloboke.collect.impl.hash.HashConfigWrapper
         1        40        40   qye.KolobokeSynchronizedMap
         8            16777456   (total)
```

接下来, 我们再看看用 jmh做的性能比较. 测试代码在https://github.com/yeqinyong/java-samples/tree/master/jmh2 上, 运行在我的 Macbook Pro i5上. 从表格可以看出, ConcurrentHashMap在读上的并发性更好, 而且随着线程数的增长, 性能提高的较明显. 而KolobokeSynchronizedMap由于采用了简单的 synchronized同步原语(注意, Koloboke目前并不支持生成高并发的集合类型, 这里我只是简单的给相关操作添加了 synchronized封装), 所以性能随线程数的增长不大明显. 这里比较令我惊讶的是, KolobokeSynchronizedMap的写并发能力比 ConcurrentHashMap要高很多. 总体上说两者差距不是很大, 每秒都能达到1亿次以上的并发操作. 鉴于实际单机程序中基本不可能出现这么高的并发, 所以采用简单的 synchronized的KolobokeSynchronizedMap表现会更好一些, 再综合考虑内存方面的优势, KolobokeSynchronizedMap优势明显.

|    Benchmark    | ConcurrentHashMap | KolobokeSynchronizedMap | Units |
| --------------- | ----------------- | ----------------------- | ----- |
| 1. 两个线程读   | 146567762         | 122998850               | ops/s |
| 2. 四个线程读   | 193395876         | 137766515               | ops/s |
| 3. 两个线程写   | 38986997          | 91224159                | ops/s |
| 4. 四个线程写   | 39626626          | 124515715               | ops/s |
| 5. 三读一写(总) | 157196396         | 132967698               | ops/s |
| 5. 三读一写(读) | 142679014         | 94062720                | ops/s |
| 5. 三读一写(写) | 14517381          | 38904978                | ops/s |
| 6. 两读两写(总) | 124233730         | 128455954               | ops/s |
| 6. 两读两写(读) | 97179608          | 57874991                | ops/s |
| 7. 两读两写(写) | 27054121          | 70580962                | ops/s |

除了 Map 类型, koloboke还支持生成 set 类型, 是一个很好的用来生产/编写高性能的基本类型map/set 的工具. 更多具体用法请参考[koloboke](https://github.com/leventov/Koloboke)
