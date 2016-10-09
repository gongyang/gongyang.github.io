---
title : linux 文件时间atime、mtime、ctime
date: 2016-3-27
---
### 概念
对于linux文件系统中的文件和目录，有三种时间状态atime、mtime和ctime，他们分别代表的意思是：
***atime：access time***
访问时间，读一次这个文件的内容，这个时间就会更新。比如对这个文件运用 more、cat等命令。ls、stat命令都不会修改文件的访问时间。
***mtime：modified time***
修改时间，修改时间是文件内容最后一次被修改时间。比如：vi后保存文件。ls -l列出的时间就是这个时间。
***ctime：change time***
状态改动时间。是该文件的inode节点最后一次被修改的时间，通过chmod、chown命令修改一次文件属性，这个时间就会更新。

**如何查看atime、mtime和ctime**
ls -lu filename         列出文件的 atime
ls -l  filename          列出文件的 mtime 
ls -lc filename         列出文件的 ctime
也可以使用stat filename来同时查看三个时间
```
[ggyy@gy-vm03 ~]$ stat 1.txt 
  File: `1.txt'
  Size: 11        	Blocks: 8          IO Block: 4096   regular file
Device: 802h/2050d	Inode: 394206      Links: 1
Access: (0664/-rw-rw-r--)  Uid: (  500/    ggyy)   Gid: (  500/    ggyy)
Access: 2016-03-27 00:09:04.196004991 +0800
Modify: 2016-03-27 00:08:48.150005058 +0800
Change: 2016-03-27 00:08:48.150005058 +0800
```
在linux中stat函数中，用st_atime表示文件数据最近的存取时间(last accessed time)；用st_mtime表示文件数据最近的修改时间(last modified time)；使用st_ctime表示文件i节点数据最近的修改时间(last i-node's status changed time)。
 
 字段           说明                  例子           ls(-l)
 st_atime  文件数据的最后存取时间       read            -u
 st_mtime  文件数据的最后修改时间       write           缺省
 st_ctime  文件数据的最后更改时间       chown,chmod     -c

#### 实例 1
有时候我们会发现，即是访问了文件，该文件的访问时间也不会变，例如
```
[ggyy@gy-vm03 ~]$ stat 1.txt 
  File: `1.txt'
  Size: 5         	Blocks: 8          IO Block: 4096   regular file
Device: 802h/2050d	Inode: 394206      Links: 1
Access: (0664/-rw-rw-r--)  Uid: (  500/    ggyy)   Gid: (  500/    ggyy)
Access: 2016-03-27 00:01:09.379005522 +0800
Modify: 2016-03-26 23:27:18.032007859 +0800
Change: 2016-03-26 23:27:18.032007859 +0800
[ggyy@gy-vm03 ~]$ cat 1.txt 
test
[ggyy@gy-vm03 ~]$ stat 1.txt 
  File: `1.txt'
  Size: 5         	Blocks: 8          IO Block: 4096   regular file
Device: 802h/2050d	Inode: 394206      Links: 1
Access: (0664/-rw-rw-r--)  Uid: (  500/    ggyy)   Gid: (  500/    ggyy)
Access: 2016-03-27 00:01:09.379005522 +0800
Modify: 2016-03-26 23:27:18.032007859 +0800
Change: 2016-03-26 23:27:18.032007859 +0800
[ggyy@gy-vm03 ~]$
```
其原因是kernel更新文件访问时间是有策略的，其策略经过了三个发展阶段

- 初始阶段 
linux总是实时更新访问时间的，但这会涉及到大量的写磁盘操作，严重影响到linux性能。

- 第二阶段 
考虑到1的缺陷，可以在挂载文件系统时使用noatime属性来停止更新atime。但这会对许多应用程序造成影响，比如邮箱，备份软件，时间同步工具

- 第三阶段 
综合1,2，现在的linux使用了折中方案，挂载文件系统时使用的默认属性为relatime。其更新策略为：当满足以下任一条件时才更新访问时间，
1、访问时间早于修改时间或改变状态时间
2、距离上次更新时间间隔大于24h

综上所叙：当访问时间大于修改时间和改变时间时，且距离上次更新时间小于24h时，访问时间不再更新，这就是造成上述访问时间不变的原因。

#### 实例 2
有时候，我们会发现某一文件的访问时间甚至比修改时间还要早，如下所示， 
```
[ggyy@gy-vm03 ~]$ stat 1.txt 
  File: `1.txt'
  Size: 11        	Blocks: 8          IO Block: 4096   regular file
Device: 802h/2050d	Inode: 394206      Links: 1
Access: (0664/-rw-rw-r--)  Uid: (  500/    ggyy)   Gid: (  500/    ggyy)
Access: 2016-03-27 00:09:04.196004991 +0800
Modify: 2016-03-27 00:08:48.150005058 +0800
Change: 2016-03-27 00:08:48.150005058 +0800
[ggyy@gy-vm03 ~]$ echo "1122" >> 1.txt 
[ggyy@gy-vm03 ~]$ stat 1.txt 
  File: `1.txt'
  Size: 16        	Blocks: 8          IO Block: 4096   regular file
Device: 802h/2050d	Inode: 394206      Links: 1
Access: (0664/-rw-rw-r--)  Uid: (  500/    ggyy)   Gid: (  500/    ggyy)
Access: 2016-03-27 00:09:04.196004991 +0800
Modify: 2016-03-27 00:35:04.421003193 +0800
Change: 2016-03-27 00:35:04.421003193 +0800
[ggyy@gy-vm03 ~]$
```
这是因为在修改文件的操作中，有的会访问到该文件，有的不会。

- 比如，使用命令vim file修改文件,同时也访问了该文件。所以，修改完后首先更新文件的修改时间，接着更新访问时间。由于此时的访问时间已经小于新的修改时间，所以更新访问时间到与修改时间相同。

- 使用命令echo "1122" >> 1.txt，此命令仅仅修改了文件，并没有访问，故仅更新修改时间，访问时间因为没变而晚于修改时间，此即上述现象产生的原因。
