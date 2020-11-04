# An experimental evaluation and analysis of database cracking
比较了18种cracking方法，6种排序算法和三种完全索引。
结果表明：1.之前的算法可重复；2.还有足够的空间改善之前的算法；3.有效并行化crack算法很困难；4.crack很大程度上取决于选择度；5.crack需要赶上现代索引趋势；6.不同的索引算法具有不同的特征

**传统索引两个假设:**
1. 查询工作负载是可知的
2. 有足够的空闲时间创建索引

这两个假设在当下并不总是适用了，工作负载未知或不断变化，在数据到达时立刻查询。

Database Cracking的几个研究方面包含更新，元组重构，收敛，并发控制和健壮性。这里对Cracking算法之间进行了比较，并与最新的内存优化索引技术包含ART进行了比较。

评估了先进的Crack算法，分别是预测crack（fast scan not poor sort），混合crack(HCC, HCR, HRC, HRR)，横向Crack和随机Crack。

## 回顾Cracking

### Crack-in-two vs. Crack-in-three

Crack-in-three可以用两个Crack-in-two来表示。每次分裂肯定能把数据分为p%和(100-p)%，当分裂点在中间时crack的成本是最高的，因为这样会有最多的数据移动。
![Crack-in-two和Crack-in-three的比较.png](https://upload-images.jianshu.io/upload_images/11576561-f0d7fd408aed0619.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

可以看到Crack-in-two是对称的，但是Crack-in-three并不是，因为Crack-in-three总是从上部开始考虑(也就是说对于每条数据都是先判断属不属于上半部，然后中部，然后下半部，所以下半部数据占比较大时需要更多的比较)

### 标准Cracking算法
用AVL树作为cracker index，图2.b为其性能，可以看到它从第一次查询的接近扫描 逐渐 到最后接近全索引。**注意：这个和后面的渐进索引的实验结果是相悖的，本文说cracking 0.3s而扫描0.24s相差非常小，全索引的建立10.2s；然而在渐进索引中，首次查询cracking5.26s，扫描0.75秒，相差就比较多了。我倾向于相信渐进索引，毕竟首次crack需要一次全拷贝并且要交换大量的数据。**

### 代价分析
cracking查询的大部分时间花在了哪里，并如何变化，这里将成本分为四部分：索引(cracker index)查找成本以表示要crack的分区；交换数据的重组成本(crack)；索引(cracker index)更新的成本；实际访问满足查询需求的数据访问成本。

从图2.c可见，数据移动是主要成本，但数据移动随着查询成本越来越低，在1000次查询后不占据主要地位。还可以看到索引的索引的查找与更新比数据移动的成本小几个量级，几乎没有索引维护开销。但随着查询的进行，数据移动成本降低，而索引的查找与维护的成本增加。

### 标准Cracking的主要方面
![标准Crack测试.png](https://upload-images.jianshu.io/upload_images/11576561-ab6832476f886009.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)
CPU效率，收敛到全索引，投影属性，查询性能方差

CPU效率：主要讨论的分支预测错误。在最坏的情况下，分割点是将一个分区均分，预测错误将超过50%。

收敛：图3.b显示在1000查询以后，查询时间仍然比查询索引高40%。

元组重构：默认情况下，Cracking会造成一个未聚簇索引，也就是需要额外的查找来得到投影的属性。（但是，列存储数据库同一元组不同列不连续存储这是它的本性啊，不是cracking带来的问题）

选择度的影响：cracking是根据谓词的查询范围对索引列进行分区的，偏斜的查询范围会导致偏斜的分区，从而导致不可预测的查询性能。查询时间的方法不像全索引一样稳定，但是从图中可以看到，从开始到1000次查询，查询时间方差减少了5个数量级。

## 当前的Cracking算法(2015年)

### 预测和向量化cracking (Database Cracking: fancy scan, not poor man's sort!)
通过投机的移动元素 和 纠正后续的错误 来分离比较的支点和物理重组。