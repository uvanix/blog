过滤ffmpeg进程并杀掉

ps axo pid,command | grep "ffmpeg" | grep -v "grep" | sed "s/^[ \t]*//g" | cut -d \  -f 1 | xargs -r kill -9



1.查找大文件：

% find . -type f -size +100M #查找100M以上的文件
1
对查找结果按照文件大小做一个排序

% find . -type f -size +100M -print0 | xargs -0 du -h | sort -nr

后缀查找：find . -name '*.tmp' -type f -size +100M -print0 | xargs -0 du -h | sort -nr | nl

删除：find . -name '*.log.*' -type f -size +100M | xargs rm


1
2.查找当前目录下前20的大目录

sudo du -hm --max-depth=2 | sort -nr | head -20



LINUX的磁盘管理du命令详解
du(disk usage)命令可以计算文件或目录所占的磁盘空间。没有指定任何选项时，它会测量当前工作目录与其所有子目录，分别显示各个目录所占的快数，最后才显示工作目录所占总快数。

命令用途
du(disk usage)命令可以计算文件或目录所占的磁盘空间。没有指定任何选项时，它会测量当前工作目录与其所有子目录，分别显示各个目录所占的快数，最后才显示工作目录所占总快数。

命令格式

du [OPTION]… [FILE]…
-a, –all 包括了所有的文件，而不只是目录
–apparent-size print apparent sizes, rather than disk usage; although the apparent size is usually smaller, it may be larger due to holes in (’sparse’) files, internal fragmentation, indirect blocks, and the like
-B, –block-size=SIZE use SIZE-byte blocks
-b, –bytes 以字节为计算单位
-k 以千字节（KB）为计算单位
-m 以兆字节（M）为计算单位
-c, –total 最后加上一个总计（系统缺省）
-D, –dereference-args dereference FILEs that are symbolic links
-H 跟 --si效果一样。
-h, –human-readable 以比较阅读的方式输出文件大小信息 (例如，1K 234M 2G)。注：该选项在很多其他命令（df, ls）中也有效。
–si 跟-h 效果一样，只是以1000为换算单位
-l, –count-links 计算所有的文件大小，对硬链接文件，则计算多次。
-L, –dereference 显示选项中所指定符号连接的源文件大小。
-P, –no-dereference 不跟随任何的符号连接（缺省）
-S, –separate-dirs 计算目录所占空间时不包括子目录的大小。
-s, –summarize 只显示工作目录所占总空间
-x, –one-file-system 以一开始处理时的文件系统为准，若遇上其它不同的文件系统目录则略过。
-X FILE, –exclude-from=FILE 排除掉指定的FILE
–exclude=PATTERN 排除掉符合样式的文件,Pattern就是普通的Shell样式，？表示任何一个字符，*表示任意多个字符。
–max-depth=N 只列出深度小于max-depth的目录和文件的信息 –max-depth=0 的时候效果跟–s是 一样
1
2
3
4
5
6
7
8
9
10
11
12
13
14
15
16
17
18
19
20
21
使用案例

root@ubuntu:/# cd /home/web/
root@ubuntu:/home/web# du -s
793832 .–不指定FILE名字计算出当前目录所占用的空间大小。
root@ubuntu:/#$ du -sh
776M .–不指定FILE名字计算出当前目录所占用的空间大小。-h选项使得输出结果跟容易阅读（跟上例比较）
root@ubuntu:/#$ du –max-depth=1 -h
–输出当前目录下各个子目录所使用的空间
83M ./java
87M ./build
197M ./jboss
128M ./lib
1.1M ./bin
52K ./synclogs
4.8M ./sql
920K ./conf
52K ./logs
20K ./mail_group
56K ./.svn
144M ./htdocs
56K ./jboss-conf
2.7M ./auto-conf
8.0K ./.mule
23M ./classes
43M ./templates
144K ./project
776M .
1
2
3
4
5
6
7
8
9
10
11
12
13
14
15
16
17
18
19
20
21
22
23
24
25
26
找出大文件
磁盘空间被耗尽的时候，免不了要清理一下，比如说/home目录太大，就可以使用下面命令看看到底是谁：

du -s /home/* | sort -nr
1
linux磁盘空间不足怎么办，磁盘清理方法
由于当初安装系统设计不合理，有些分区的过小，以及网络通讯故障等造成日志文件速度增长等其他原因都可以表现为磁盘空间满，造成无法读写磁盘，应用程序无法执行等。下面就给你支几招（以/home空间满为例）：

定期对重要文件系统扫描，并作对比，分析那些文件经常读写
#IS-IR/home>;files.txt
#diff filesold.txt files.txt
1
2
通过分析预测空间的增长情况，同时可以考虑对不经常读写文件进行压缩，以减少占用空间。

查看空间文件系统的inodes消耗
#df-i/home
1
如果还有大量的inpde可用，说明大文件占用空间，否贼可能大量小文件占用空间。

找出占用空间较大的目录
查看/home占用的空间
#du-hs/home
1
查看/home下占用空间超过1000m

#du/awk'$1>;2000'
1
找出占用空间较大的文件
#find/home-size +2000K
1
找出最近修改或创建的文件
先TOUCH一个你想要的时间的文件如下
#TOUCH-t 08190800 test
#find/home-newer test-print
1
2
删除日志

#rm-rf/var/log/*
1
对分区做连接
在有空间的分区，对没有空进分区做连接
#in-s/home/use/home
1
找出耗费大量的空间的进程
根据不同的应用，找出对应的进程，分析原因。

检查并修复文件系统

#fsck-y/home
1
重启机器
有了以上的十招，应该可以解决大部分问题，但是关键还是安装时要规划好分区。另外发现磁盘蛮时，不能急，小心操作，认真分析原因，然后小心应对。需要注 意，以上十招不需要顺序执行，有的可能一招封喉，有的可能需要数招并用，删除操作一定要小心。如果还不行，只有采取增加硬盘，重新安装系统等“硬”办法了
　　还可以：

cd/
du-h--max-depth=q/grep M/sort-n
1
2
　　找到最大的那个目录后进入该目录
　　再运行du-h-max-depth=1/grep M /sort-n
　　找出来以后看是否有用的文件
　　没用就删掉
