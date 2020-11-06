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
通过投机的移动元素 和 纠正后续的错误 来分离比较的支点和物理重组。主要针对的标准cracking的过多的分支预测错误导致大量不必要的执行代码。在标准cracking中，是基于元素与支点的比较的结果来进行指针的移动和元素的交换；而预测cracking是推测性的执行这些重组，然后和 元素与支点的比较结果进行比对。当比较结果出来后，不正确的重组需要被纠正。为了保证这种随机的写不会丢失，需要备份被覆盖的元素。这种概念让算法变得branch free，预测错误的不利也不会长期存在。 但是——这种预测的写比起标准crack也增加了开销。问题是，这种权衡是否能改善运行时间。

在预测crack中，数据的粒度被固定为单个元素。作者提出了一个一般化的概念叫向量化crack，数据被备份和分区在大小可调的更大的块中。这使得数据昂贵的备份和实际的分区脱钩。它是用较大的重组工作量来换分值预测错误的惩罚。
![2020-11-04 21-39-52屏幕截图.png](https://upload-images.jianshu.io/upload_images/11576561-4fa635150f4b5b2c.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

![2020-11-04 22-01-35屏幕截图.png](https://upload-images.jianshu.io/upload_images/11576561-1d2a4578f508bd53.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

图6，从实际效果来看预测crack和向量化crack都没有超过标准crack

![2020-11-05 09-20-45屏幕截图.png](https://upload-images.jianshu.io/upload_images/11576561-946031dd3a89dd13.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)
### Hybrid Cracking

创建未排序的初始分区，不断地像最终分区合并(快速收敛)

混合crack旨在把提升标准crack到完全索引的较差的收敛性。标准crack的问题主要是每次至多创建连个新的分区，因此需要多个查询才能收敛到一个完整的索引。但是自适应合并第一个查询给所有初始分区排序的初始代价非常高。混合crack结合两者的优点，减少初始成本，提升收敛速度。

发现标准crack最适合初始分区，排序最适合最终分区。通过以轻量级的方式创建初始分区引入多个分区边界，混合crack可以更快的收敛。

### 横向cracking
自适应的创建、对其并crack每个选择投影的属性对 为了更快的元组重构。

横向cracking用cracker maps来解决标准cracking中的元组重构效率低的问题。cracker maps包含两个逻辑列，一个cracked列和一个投影列，它用来保持投影属性与选择属性对齐。执行一个查询，横向cracking会创建并crack那些包含所有访问元素的crakcer maps。结果就是每个访问的列总是与cracker map中的cracked列对齐。

### 随机Cracking
用辅助随机支点元素创建更加平衡的分区以获得健壮的查询性能。

随机cracking解决了在database cracking中性能无法预测的问题。标准的cracking的一个关键问题就是分区边界很大程度上取决于传入的查询范围。偏斜的查询范围可能会导致分区大小的不平衡，并且连续的查询仍可能扫描大部分数据。为了解决以上问题，在查询时除了有查询驱动的crack外，随机cracking还引入了其他crack。这些额外引入的crack有助于更均匀的crack。

**查询驱动和数据驱动**
DDC(Data Divern Cracking)：完全的数据驱动，总是把片均分不管查询是什么。具体是这样，在查询边界low和high落入的分区进行递归的划分（均分）直到片足够小，然后按照边界值进行crack。（中值的找法是一个被充分研究的问题，本文使用的是IntroSelect算法）。DDC同时也是一个查询驱动的，因为它只会crack查询边界落入的块。

DDR(Data Driven Random Cracking)：与DDC相比就是放宽了均分这个要求，这里是用的是随机crack，直到边界落入的piece足够小然后按边界值进行crack。

DD1C和DD1R：按照DDC和DDR的方法，在刚开始的查询中需要递归的把边界落入的块划分代价非常大，而DD1C和DD1R只需要进行一次额外的Crack。

MDD1R：前面的DD1C和DD1R虽然已经想尽力减少first query的开销了，但是因为刚开始要处理的是一整列，或者是刚开始的是顺序查询，所以开销还是很大。M是materialization。和DD1R不同的是，它在第一次的随机crack之后不再按边界进行crack，而是把查询结果放入一个新数组中。

随机cracking提出了几种变体来引入额外的crack，包括数据驱动和概率决策。通过改变辅助工作的数量和crack的位置，随机cracking需要在方差和cracking的开销之间取舍。MDD1R是随机cracking中整体表现最好。图7.c中可以看到MDD1R和标准cracking相近，略差，这是因为上面默认的访问模式是均匀随机访问，这就导致标准cracking也能均匀划分，下面还会讨论其他访问模式。

## Cracking分类
cracking本质上都是在逐渐划分数据。不同的算法对数据进行不同的划分。把cracking算法分为三个维度：1.引入的分割线数量 2.划分的策略 3.划分的时间。

![2020-11-05 14-50-53屏幕截图.png](https://upload-images.jianshu.io/upload_images/11576561-5c0033784c81ab05.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

分割线的数量  cracking算法的核心要求所有cracking算法都要做一些索引上的工作(对未来的索引提供帮助)，即在查询到达时至少引入一条分割线。分为四级 0(扫描) few several和all(全排序)。

在划分策略上： 基于查询的(标准cracking)， 基于数据的（全排序），随机（随机cracking）和无（全扫描）。

划分时间点：提前（在回答任何查询之前，全索引），查询前（所有的cracking算法都是先crack再得到查询结果），never（全扫描）

## 实验评估
比较用不同的排序和索引基准的cracking算法。

![2020-11-05 17-09-01屏幕截图.png](https://upload-images.jianshu.io/upload_images/11576561-d4bc35d9f2d9063e.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

### 排序算法
之前cracking使用的basaline是全索引，做法是用快速排序对数据进行全排序，然后用二分搜索处理查询找到最后的结果。完全排序比cracking的first query满了30倍。下面比较不同的排序算法。

把快排和就地的基数排序进行了比较。在长度(需要排序的数量)小于64时，基数排序换成插入排序。图11.a可以看到基数排序的初始化时间减少一般，与标准cracking查询相比仅慢14倍，而快排要慢30倍。并且在600个查询之后，基数排序的累计查询时间(总)就已经超过了标准cracking，而快排需要12000个查询。

### 索引baseline
本节主要是比较不同的索引结构和有序二分查找的对比。目的是看用复杂的索引结构作为cracking的baseline是否有意义。考虑三种结构 1.AVL树 2.B+树 3.ART

首先看1000个查询不同索引的总indexing effort。二分查询就是 radix-introsort，而对其他索引来说，除了排序还需要把数据加载到索引结构中。

![2020-11-06 15-32-11屏幕截图.png](https://upload-images.jianshu.io/upload_images/11576561-e73ef13212489c15.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

标准cracking会把indexing effort分给1000个查询，而剩下的方法都需要在第一个查询前进行排序和索引。对B+树提出了两种变体：批量加载和tuple-wise加载，图11.b。从图中可以看到，就索引工作而言，AVL是最贵的，而标准cracking
是最便宜的。二分查找和B+树(批量价在)接近标准cracking，B+树和ART需要更多的工作，因为它俩都需要按元组进行加载(这里用的是非批量加载的ART，我不确定现在有没有批量加载的)。下面看不同索引的查询性能，11.c显示不同索引方法的pre-query的响应时间几乎不会影响查询性能(为什么？)。

### 选择度的影响
