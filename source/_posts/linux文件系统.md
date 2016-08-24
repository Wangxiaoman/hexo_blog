title: linux文件系统
date: 2016-8-24 9:50:49
tags: [文件系统,linux]
category: linux
---

### 概述
一块磁盘在投入使用之后，需要分区、格式化，磁盘分区可以看做是物理操作，而格式化是将磁盘定义为合适的文件系统，以便操作系统来使用（一般情况来讲，一块分区是一个单独的文件系统）。
每个文件都有数据信息，也有文件本身的属性，比如创建者、权限、创建时间等等，一般文件的信息和本身属性是分开存放的。

<!--more-->

### 基本定义以及组成

* 一般文件系统会包含如下的组成部分，block（存放文件的数据，有编号一般从零开始，块的大小为1、2、4k），inode（存放文件属性），superblock（文件系统的基本信息），block group（每个group下都有自己的inode，block，superblock）。

* Data Block：用于存储文件的数据，Linux支持Block可以是1、2、4k，格式化的时候选择。每一个块都有自己的编号。

* Inode Table：用于存放文件的属性，包括权限、时间、所有者、所属组、大小、文件对应的block编号。每个inode的大小只有128byte，每个文件对应一个inode。每次系统读取文件的时候，先从inode中获取权限属性，权限合适，才会去block中读取数据。
因为一个inode只有128byte，而记录一个block的编号就需要4byte，这样存储的文件大小最大只有4k*128/4 = 128k（因为inode还需要记录属性信息，实际存储block编号的空间小于128byte）。这样对于大文件，linux采用将block块作为inode的子节点来记录block块，这样就可以记录大文件了。

* SuperBlock：存放文件系统的整体信息，比如有多少block、inode使用量，剩余量，文件格式等等（SuperBlock可能并不是每个block group都有，可能就第一个block group才有）。

* FileSystem Description：整个文件的描述，blockgroup中的block开始块到结束块的号码。

* Block Bitmap：记录具体block块号码使用、未使用的情况（看名称就知道使用位图来存储的了，节省空间）。

* Inode Bitmap：记录具体的inode号码使用、未使用的情况。

### 文件的读取过程

#### 文件读取（/etc/password）
1. 读取根目录 / 的inode，获取权限，有权限读取block中内容
2. 读取根目录 / 的block，获取 /etc 目录的inode号码
3. 读取目录 /etc 的inode，获取权限，有权限则读取block中内容
4. 读取目录 /etc 的block，获取 /etc/password 的inode号码
5. 读取目录 /etc/password的inode，获取权限，有权限则读取 /etc/password中block的内容

#### 文件写入
1. 确定用户权限
2. 查询inode bitmap中未使用的inode，将文件的属性、权限、时间等信息写入inode中
3. 查询block bitmap中未使用的block，将文件的内容写入，对应的block块码写入inode中
4. 将inode bitmap和block bitmap中的号码使用情况更新 










