---
title : tmpwatch
date: 2016-8-28
---
## tmpwatch
什么是tmpwatch？
tmpwatch - removes files which haven't been accessed for a period of time
就是对一段时间内没有访问过的文件进行删除。

### 安装
直接使用yum进行安装
```
 yum install tmpwatch
```
安装完成之后，会在/etc/cron.daily/目录下生成一个tmpwatch的文件。
```
[root@gy-vm02 cron.daily]# cat tmpwatch 
#! /bin/sh
flags=-umc
/usr/sbin/tmpwatch "$flags" -x /tmp/.X11-unix -x /tmp/.XIM-unix \
	-x /tmp/.font-unix -x /tmp/.ICE-unix -x /tmp/.Test-unix \
	-X '/tmp/hsperfdata_*' 10d /tmp
/usr/sbin/tmpwatch "$flags" 30d /var/tmp
for d in /var/{cache/man,catman}/{cat?,X11R6/cat?,local/cat?}; do
    if [ -d "$d" ]; then
	/usr/sbin/tmpwatch "$flags" -f 30d "$d"
    fi
done
```
/usr/sbin/tmpwatch ``` "$flags" -f 30d "$d" ```这一行，就是说清理30天没有被访问过的文件或文件夹

### tmpwatch参数说明：
```
-u, --atime 基于访问时间来删除文件，默认的。
-m, --mtime 基于修改时间来删除文件。
-c, --ctime 基于创建时间来删除文件，对于目录，基于mtime。
-M, --dirmtime 删除目录基于目录的修改时间而不是访问时间。
-a, --all 删除所有的文件类型，不只是普通文件，符号链接和目录。
-d, --nodirs 不尝试删除目录，即使是空目录。
-d, --nosymlinks 不尝试删除符号链接。
-f, --force 强制删除。
-q, --quiet 只报告错误信息。
-s, --fuser 如果文件已经是打开状态在删除前，尝试使用“定影”命令。默认不启用。
-t, --test 仅作测试，并不真的删除文件或目录。
-U, --exclude-user=user 不删除属于谁的文件。
-v, --verbose 打印详细信息。
-x, --exclude=path 排除路径，如果路径是一个目录，它包含的所有文件被排除了。如果路径不存在，它必须是一个绝对路径不包含符号链接。
-X, --exclude-pattern=pattern 排除某规则下的路径。

```

