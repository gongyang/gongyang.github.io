---
title : rsync实现机制原理
date: 2016-3-13
---

### 为什么要使用rsync
在linux文件系统下面，我们通常使用scp来进行两台机器之间传输文件，大部分情况下，这是一个简单好用的方式。然而，对于大数据文件的拷贝，或者是文件的同步，使用scp就会出现问题，因为scp不支持断点续传。什么是断点续传，就是在拷贝文件的过程中，因为某些原因，同步失败了，如果使用scp，那么你得重新开始进行同步。这个时候，使用rsync，就方便多了。
### 什么是rsync
rsync（remote synchronize）是Liunx/Unix下的一个远程数据同步工具。它可通过LAN/WAN快速同步多台主机间的文件和目录，并适当利用rsync算法（差分编码）以减少数据的传输。rsync算法并不是每一次都整份传输，而是只传输两个文件的不同部分，因此其传输速度相当快。
### 实现原理
首先， 我们先来想一下rsync要解决的问题，如果我们要同步的文件只想传不同的部分，我们就需要对两边的文件做diff，但是这两个问题在两台不同的机器上， 无法做diff。如果我们做diff，就要把一个文件传到另一台机器上做diff，但这样一来，我们就传了整个文件，这与我们只想传输不同部的初衷相背。于是我们就要想一个办法，让这两边的文件见不到面，但还能知道它们间有什么不同。这就出现了rsync的算法。
**分块Checksum算法**
假设我们同步源文件名为fileSrc，同步目的文件叫fileDst
首先，我们会把fileDst的文件平均切分成若干个小块，比如每块512个字节（最后一块会小于这个数），然后对每块计算两个checksum

- 一个叫rolling checksum，是弱checksum，32位的checksum，其使用的是Mark Adler发明的adler-32算法，
- 另一个是强checksum，128位的，以前用md4，现在用md5 hash算法。

为什么要这样？因为若干年前的硬件上跑md4的算法太慢了，所以，我们需要一个快算法来鉴别文件块的不同，但是弱的adler32算法碰撞概率太高了，所以我们还要引入强的checksum算法以保证两文件块是相同的。也就是说，弱的checksum是用来区别不同，而强的是用来确认相同。

**传输文件**
同步目标端会把fileDst的一个checksum列表传给同步源，这个列表里包括了三个东西，rolling checksum(32bits)，md5 checksume(128bits)，文件块编号。同步源机器拿到了这个列表后，会对fileSrc做同样的checksum，然后和fileDst的checksum做对比，这样就知道哪些文件块改变了。
然而，事情并没有那么简单

- 如果我fileSrc这边在文件中间加了一个字符，这样后面的文件块都会位移一个字符，这样就完全和fileDst这边的不一样了，但理论上来说，我应该只需要传一个字符就好了。这个怎么解决？
- 如果这个checksum列表特别长，而我的两边的相同的文件块可能并不是一样的顺序，那就需要查找，线性的查找起来应该特别慢吧。这个怎么解决？

**checksum查找算法**
同步源端拿到fileDst的checksum数组后，会把这个数据存到一个 hash table中，用rolling checksum做hash，以便获得O(1)时间复杂度的查找性能。这个hash table是16bits的，所以，hash table的尺寸是2的16次方，对rolling checksum的hash会被散列到0 到 2^16 – 1中的某个整数值。

**比对算法**
这是最关键的部分
1、取fileSrc的第一个文件块（我们假设的是512个长度），也就是从fileSrc的第1个字节到第512个字节，取出来后做rolling checksum计算。计算好的值到hash表中查。
2、如果查到了，说明发现在fileDst中有潜在相同的文件块，于是就再比较 md5的checksum，因为rolling checksume太弱了，可能发生碰撞。于是还要算md5的128bits的checksum。如果rolling checksum和md5 checksum都相同，这说明在fileDst中有相同的块，我们需要记下这一块在fileDst下的文件编号。
3、如果fileSrc的rolling checksum 没有在hash table中找到，那就不用算md5 checksum了。表示这一块中有不同的信息。总之，只要rolling checksum 或 md5 checksum 其中有一个在fileDst的checksum hash表中找不到匹配项，那么就会触发算法对fileSrc的rolling动作。于是，算法会住后step 1个字节，取fileSrc中字节2-513的文件块要做checksum，重复第一步，所以我们叫做rolling checksum。
这样，我们就可以找出fileSrc相邻两次匹配中的那些文本字符，这些就是我们要往同步目标端传的文件内容了。

上面的文字有些同学可能已经看晕了，下面这张是我在网上找的一张非常经典的图，很好的描述了上面的步骤。
![1](http://coolshell.cn/wp-content/uploads/2012/05/rsync-algorithm.jpg)
