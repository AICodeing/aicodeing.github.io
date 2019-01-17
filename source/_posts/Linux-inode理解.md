---
title: Linux inode理解
date: 2019-01-15 16:50:05
tags:
	- Linux
category: Linux
comments: ture
---
`inode`在`Linux` 文件系统中具有灵魂级作用，我们就谈一下对`inode`的理解。  
首先了解一下`Linux`的文件系统。 
 
### EXT4 文件系统

Linux 基于ext4 文件系统，简单来说 它将存储划分成一组组大小相同的逻辑块，这样做的好处是减少了文件管理的开销，按照 __整数个__ `基础逻辑块`来分配存储大小方便高效，而且利于提高大文件的传输效率，一个文件所占用的块可以是连续的，也可以是非连续的。而为了标记这些逻辑块的引用，文件系统中还引入了 `inode`。  
`inode`是linux 文件系统中的数据索引标记符。 如下图:  
 ![](/img/inode/inode_1.jpg "")
 
 `Linux`的文件系统被分为了两部分，一部分是 `inode`区，一部分是 `block`区，一个`inode` 的默认大小为 128Byte，当然，不同的Linux 发行版本可能大小定义不同，inode的大小在文件系统被格式化之后就无法更改了，格式化前可以指定inode大小。`inode`用来记录:
   
```
文件的权限(r,w,x)
文件的所有者和属组
文件的大小
文件的状态改变时间（ctime）
文件的最近一次读取时间（atime）
文件的最近一次修改时间（mtime
文件的数据真正保存的 block 编号。
```

每个文件需要占用一个 `inode`。而`inode` 中是不记录文件的名称的，文件名称记录在文件所在目录的`block`块中。  
 `block`块的大小可以是1KB、2KB、4KB，一般默认往往是4KB，包含了连续8个扇区，每个扇区存储512个字节。
 
`inode`就告诉了文件位于哪个“块”，于是系统就会从这个“block块”开始读取内容。如果一个 block 放不下数据，则可以占用多个 block。例如，有一个 10KB 的文件需要存储，则会占用 3 个 block，虽然最后一个 block 不能占满，但也不能再放入其他文件的数据。这 3 个 block 有可能是连续的，也有可能是分散的。
 
### 查看文件的inode

我们可以使用 `stat` 命令来查看文件的 `inode` 信息

在 test 目录下创建一个 `src.txt` 文件  

```shell
➜  test touch src.txt
➜  test ll -l
total 0
-rw-r--r--  1 wangwei  staff     0B  1 15 17:51 src.txt
➜  test stat -x src.txt
  File: "src.txt"
  Size: 4            FileType: Regular File
  Mode: (0644/-rw-r--r--)         Uid: (  502/ wangwei)  Gid: (   20/   staff)
Device: 1,4   Inode: 8625061903    Links: 2
Access: Tue Jan 15 18:36:36 2019
Modify: Tue Jan 15 18:34:09 2019
Change: Tue Jan 15 18:37:29 2019
➜  test
```

通过 `stat -x file`命令(Mac Os 下) 可以得到 文件的 详细信息，`Inode: 8625061903` 指明 该文件的 `inode`号是 `8625061903`  

通过`stat -s file`命令:  
  
```shell
➜  test stat -s src.txt
st_dev=16777220 st_ino=8625061903 st_mode=0100644 st_nlink=2 st_uid=502 st_gid=20 st_rdev=0 st_size=4 st_atime=1547548596 st_mtime=1547548449 st_ctime=1547548649 st_birthtime=1547545866 st_blksize=4096 st_blocks=8 st_flags=0
➜  test
```
   
可以得到更多信息，如果了解更多请访问 [linux-stat]('https://linux.die.net/man/2/stat')   

```shell
st_dev: 文件所在的 device ID
st_ino: 文件的 inode 号
st_mode: 文件的类型，如 常规文件，文件夹，字符设备文件，块文件 等等
st_nlink：文件的硬链接个数，如果为0 就可以被删除了
st_rdev: 设备号的意思，只有字符特殊设备和块特殊设备才会有st_rdev值。此值包含实际设备的设备号
st_size: 文件占用的字节大小 ,单位字节
st_blksize: 每个 block 的大小，一般情况下是4KB
st_blocks: number of 512B blocks allocated, 以 512B 为单位分配给文件的个数,用 st_size/512B 就得到 st_blocks，
当 st_size 小于一个 block的大小，即文件大小小于默认的4KB时 实际还是分配一个 block，st_blocks = 1个blocksize/512B,
在该例子中，src.txt 大小是4B ，但是需要占用一个 block 也就是 4KB的实际存储空间，计算得到的st_blocks = 8

```

还可以使用命令`ll -i` 查看文件的 inode  
 
 ```shell
 ➜  test ll -i
total 0
8625061903 -rw-r--r--  1 wangwei  staff     0B  1 15 17:51 src.txt
➜  test
 ```
 `8625061903 ` 就是 文件 `src.txt`的 inode编码,而权限后面的 `1`是该文件的链接数，也即是 `硬链接数`。  
  
### inode的使用  

一个文件在创建的时候就会分配一个 inode节点，以一个简单的例子来说明，如何访问一个文件?  
要打开一个文件，需要要找到这个文件占用的 block 块。首先会找到文件所在的目录，在linux文件系统中，目录也是一个文件，__目录文件的内容就是该目录下的文件名与 inode的映射表，即一个个的目录项__。当访问一个文件的时候首先查询到上一级目录，在 Linux 文件系统中 每一个目录中都有 `.`和`../`两个目录   

```shell
➜  test ll -ai
total 16
8625052613 drwxr-xr-x   4 wangwei  staff   128B  1 15 17:51 .
   8475120 drwxr-xr-x+ 85 wangwei  staff   2.7K  1 15 18:13 ..
8625053037 -rw-r--r--@  1 wangwei  staff   6.0K  1 15 15:39 .DS_Store
8625061903 -rw-r--r--   1 wangwei  staff     0B  1 15 17:51 src.txt
➜  test
```
 __`.`__ 目录代表当前目录的 __硬连接__ ；__`..`__ 代表着上级目录的 __硬链接__
 示例中的 8625052613 就是当前test 目录的inode编号，8475120 是上级目录的编号。因此也说明了，任何一个目录的硬链接总数，总是等于 2 加上它的子目录和子文件总数,即使是一个空文件夹，它的硬链接数也是2。  
 
文件系统根据 父目录的目录项找到 需要打开文件的inode 节点，通过读 inode 节点找到需要打开的 block，载入 block块中的数据，即完成了打开文件的操作，当然通过 inode 做了权限判断，如果无权访问该文件，不需要访问 block块，通过目录项找到 inode 节点就能读出权限信息。  

### inode的链接数
通过 `ll -i`命令可以查看文件的 inode 信息，inode 信息中有一个 __链接数__

 ```shell
 ➜  test ll -i
total 0
8625061903 -rw-r--r--  1 wangwei  staff     0B  1 15 17:51 src.txt
 ```
 `-rw-r--r--` 后面的 __1__ 表示 src.txt 文件当前有一个引用，当 链接数变成0 则释放 inode，释放block 块的数据内容。 
 
 对 inode 链接数 的影响 来自 前面多次提到的 __硬链接__ 
 
### 硬链接
我们常见的软连接 通过 `ln`命令生成。同样，硬链接也是通过`ln`命令生成。

```shell
➜  test ll -i
total 0
8625061903 -rw-r--r--  1 wangwei  staff     0B  1 15 17:51 src.txt
➜  test ln src.txt hard.txt
➜  test ll -i
total 0
8625061903 -rw-r--r--  2 wangwei  staff     0B  1 15 17:51 hard.txt
8625061903 -rw-r--r--  2 wangwei  staff     0B  1 15 17:51 src.txt
➜  test
```
 示例中 创建了一个硬链接 `hard.txt`， 这个文件 的 `inode` 和 `src.txt` 是一样的，说明在目录项中 这两个文件名都指向 同一个`inode`，`inode` 指向一个 `block`，这就说明 这两个文件共用一个 `block 块`，不管修改哪个文件，另一个文件也会被修改。
 
 ```shell
 ➜  test echo 111 > src.txt
➜  test cat src.txt
111
➜  test cat hard.txt
111
➜  test
 ```
 示例中我们向 __`src.txt`__ 文件写入了 __`111`__ 字符，查看发现 `hard.txt` 也做了同样的修改。
 
那么两个文件指向同一个`block`，那占用的空间肯定也就是一份了，有意思的事情来了  

```shell
➜  test ll -h
total 8 // 8个 以512B为单位的存储占用 即 8*512B = 4KB
-rw-r--r--  1 wangwei  staff     4B  1 15 18:34 src.txt
➜  test ln src.txt hard.txt
➜  test ll -h
total 16 // 16个 以512B为单位的存储占用 即 16*512B = 8KB
-rw-r--r--  2 wangwei  staff     4B  1 15 18:34 hard.txt
-rw-r--r--  2 wangwei  staff     4B  1 15 18:34 src.txt
➜  test
```
**查看存储占用，发现 硬链接 导致了 存储的翻倍了。这个不符合原理啊!!!**  
其实这是这个**`ll –h`** 或者 **`ls –h`** 命令进行统计文件总大小的时候并不是从磁盘进行统计的，而是根据文件属性中的大小叠加得来的。而硬链接的文件属性中的大小就是就是inode号对应的数据块的大小，所以total中进行统计就把各个文件属性中的大小加起来作为总和，这种统计是不标准，也不具有代表性的，正真的查看某个文件夹占用磁盘空间大小命令是：du –h   这个命令是从磁盘上进行统计，不会被文件的属性中大小影响，所以更准确。

所以得出结论: __硬链接并不占用磁盘空间！__  

那么我们常见的软连接呢?

### 软连接
软连接的创建方式和 硬链接的创建方式类似，都是使用 `ln` 命令，只是需要加上 `-s` 标记要创建软连接。软连接就相当于`windows系统` 上的快捷方式，只是指向一个文件。

```shell
➜  test ll -i
total 16
8625061903 -rw-r--r--  2 wangwei  staff     4B  1 15 18:34 hard.txt
8625061903 -rw-r--r--  2 wangwei  staff     4B  1 15 18:34 src.txt
➜  test ln -s src.txt soft.txt
➜  test ll -i
total 16
8625061903 -rw-r--r--  2 wangwei  staff     4B  1 15 18:34 hard.txt
8625064180 lrwxr-xr-x  1 wangwei  staff     7B  1 15 18:48 soft.txt -> src.txt
8625061903 -rw-r--r--  2 wangwei  staff     4B  1 15 18:34 src.txt
➜  test
```
创建一个 软连接， `soft.txt` 发现它的`inode`是全新的。

__软链接__ 和 __硬链接__ 在原理上最主要的不同在于: __硬链接不会建立自己的 inode 索引和 block（数据块），而是直接指向源文件的 inode 信息和 block，所以硬链接和源文件的 inode 号是一致的；而软链接会真正建立自己的 inode 索引和 block，所以软链接和源文件的 inode 号是不一致的，而且在软链接的 block 中，写的不是真正的数据，而仅仅是源文件的文件名及 inode 号。__

既然软连接分配了新的`inode`，也就是占用了新的`block`，那肯定会占用存储空间，由于软连接的`block` 存储的是 真正文件的路径和`inode号`，所以占用的空间是极小的。

__此外，硬链接是共享 `inode`的，而 `inode` 是同一个文件系统上的唯一标记，所以 硬链接是不能跨文件系统的，而软连接是新的`inode`，所以是可以跨文件系统的。__

### 文件操作对 `inode` 的影响

#### `cp` 命令
`cp` 命令是拷贝一个新文件，分配新的`inode`，必然导致分配新的`block`。所以会占用一份新的存储空间。`cp`命令执行的内部操作如下:  

```
1.分配一个未被使用的 inode 号，在 inode 表中新添一个项目
2.在目录中新建一个目录项，并指向步骤 1 中的 inode
3.把数据拷贝到新block中
```

#### `rm` 命令
`rm` 会删除指定的文件，所做的操作:

```
1.减少待删除文件名所对应的 inode 的链接数量，如果链接数变为0，则释放 inode。同时block块标记为可用状态。
2.删除文件目录中的对应目录项。
```

#### `mv` 命令
如果 源文件和目标文件在 __`同一个文件系统`__ 中，会复用`inode` 内部操作如下:

```
1.在目标文件的目录中新建目录项
2.删除源文件的目录中的目录项
3.目标文件名会指向源文件名的 inode。因此该操作对 inode 没有影响（除了时间戳），对数据的位置也没有影响，不移动任何数据。
```
所以在同一文件系统下，移动命令不会涉及到数据真正移动。

如果在 __`不同的文件系统`__ 中，其实就是相当于 拷贝一份新数据到 目标目标，然后删除 源文件。
这个过程会设计到 源文件 `inode` 的释放和 新 `inode` 的生成，还有数据拷贝，执行了`IO` 操作。

### inode 的场景利用
`inode` 中携带者文件的很多信息，我们可以通过充分利用 `inode` 的特性实现高效的文件访问操作，如果 文件权限的修改，文件的极速扫描，快速查找大文件，删除顽固文件，通过`inode`号来识别文件。