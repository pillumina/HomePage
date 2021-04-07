---
title: "Docker Fundamentals: AUFS"
date: 2021-04-06T11:22:18+08:00
hero: /images/posts/k8s-docker.jpg
menu:
  sidebar:
    name: Docker Fundamentals (AUFS)
    identifier: docker-aufs
    parent: cloud-computing
    weight: 10
draft: false
---

  AUFS是一种Union File System，所谓的UnionFS实际上就是把不同物理位置的目录合并mount到同一个目录当中。一种典型的UnionFS的应用，就是把一张CD/DVD和一个硬盘目录联合mount在一起，然后你就可以对这个只读的CD/DVD上的文件进行修改。

  AUFS又叫做Another UnionFS，后面改成Alternative UnionFS，然后又变成Advance UnionFS.....当然名字的改变叫啥不重要，本质还是没变的。2006年Junjiro Okajima开发了AUFS，完全重写了早期的UnionFS 1.X，主要目的是为了可靠性和性能，再引入一些新的功能，例如可写分支的负载均衡。不过很有意思的是，AUFS的性能比UnionFS 1.X好很多，后面UnionFS 2.x就抄AUFS的功能，而AUFS本身却没有合入到Linux主线，因为代码量太大质量也不好。虽然后面Junjiro不断提升代码质量，不断提交但是还是被Linus拒绝了。所以哪怕是今天AUFS也没进到Linux里，虽然质量已经可以了。

  不过一些发行版比如：Ubuntu 10.04，Debian6.0都支持AUFS，所以也还好。我在Ubuntu 14.04演示一下例子。

  首先，我们建立两个水果和蔬菜的目录，在这个目录上放一些文件，水果里有苹果和番茄，蔬菜有胡萝卜和番茄:

```shell
$ tree
.
├── fruits
│   ├── apple
│   └── tomato
└── vegetables
    ├── carrots
    └── tomato
```

  然后输入:

```shell
# 创建一个mount目录
$ mkdir mnt

# 把水果目录和蔬菜目录union mount到 ./mnt目录中
$ sudo mount -t aufs -o dirs=./fruits:./vegetables none ./mnt

#  查看./mnt目录
$ tree ./mnt
./mnt
├── apple
├── carrots
└── tomato
```

  可以看到`mnt`目录下有三个文件，水果和蔬菜的目录被合并起来了。如果我们修改一下文件内容:

```shell
$ echo mnt > ./mnt/apple
$ cat ./mnt/apple
mnt
$ cat ./fruits/apple
mnt
```

  可以发现如果修改了`/mnt/apple`下的内容，`/fruits/apple`下的内容也会被修改。

```shell
$ echo mnt_carrots > ./mnt/carrots
$ cat ./vegetables/carrots 

$ cat ./fruits/carrots
mnt_carrots
```

  但是这里又变得奇怪，我们修改了`/mnt/carrots`的内容，按照道理说应该是`/vegetables/carrots`被修改，但发现并不是，反而在`/fruits`下面出现了`carrots`的文件，并且我们的修改出现在这里面。换句话说，**我们在`mount aufs`的时候没有指定vegetable和fruits的目录权限**，默认来说命令行第一个的目录是`rw`的，后面的都是`ro`。如果我们在`mount aufs`的时候指定一下权限，就会有不一样的效果（先把刚才的`/fruits/carrots`删了）：

```shell
$ sudo mount -t aufs -o dirs=./fruits=rw:./vegetables=rw none ./mnt

$ echo "mnt_carrots" > ./mnt/carrots 

$ cat ./vegetables/carrots
mnt_carrots

$ cat ./fruits/carrots
cat: ./fruits/carrots: No such file or directory
```

  现在我们再看看修改:

```shell
$ echo "mnt_tomato" > ./mnt/tomato 

$ cat ./fruits/tomato
mnt_tomato

$ cat ./vegetables/tomato
I am a vegetable
```

  看上去就对味了。

  

  我们可以思考一下一些使用场景，例如我们可以把一个目录，例如我们自己的source code，作为只读的模板，和另一个working directory给union起来，那么我们就可以随便魔改不用害怕把源代码改坏。有点像`ad hoc snapshot`。

  

## Docker分层镜像对UnionFS的使用

  Docker就把UnionFS的技术借鉴到了容器中。在Linux Namespace中我们讨论了mount namespace和chroot去`fake`了一个镜像。而UnionFS的技术可以用来制作分层镜像。

  下面是Docker官方文档[Layer](http://docs.docker.com/terms/layer/)，展示了Docker用UnionFS搭建制作的分层镜像：

![docker layer](https://coolshell.cn/wp-content/uploads/2015/04/docker-filesystems-multilayer.png)

  docker的分层镜像不单单可以使用aufs，还支持例如btrfs, devicemapper和vfs。可以使用`-s`或者`-storage-driver=`选项来指定相关的镜像存储。Ubuntu14.04的环境里docker默认用的是aufs，而Centos7下用的是devicemapper，这个会有另一个post来讨论。

  可以在下面的路径查看每个层的镜像:

`/var/lib/docker/aufs/diff/<id> `

  docker执行起来以后，`docker run -it ubuntu /bin/bash`，可以从`/sys/fs/aufs/si_[id]`目录查看aufs的mount情况，比如:

```shell
#ls /sys/fs/aufs/si_b71b209f85ff8e75/
br0      br2      br4      br6      brid1    brid3    brid5    xi_path
br1      br3      br5      brid0    brid2    brid4    brid6 

# cat /sys/fs/aufs/si_b71b209f85ff8e75/*
/var/lib/docker/aufs/diff/87315f1367e5703f599168d1e17528a0500bd2e2df7d2fe2aaf9595f3697dbd7=rw
/var/lib/docker/aufs/diff/87315f1367e5703f599168d1e17528a0500bd2e2df7d2fe2aaf9595f3697dbd7-init=ro+wh
/var/lib/docker/aufs/diff/d0955f21bf24f5bfffd32d2d0bb669d0564701c271bc3dfc64cfc5adfdec2d07=ro+wh
/var/lib/docker/aufs/diff/9fec74352904baf5ab5237caa39a84b0af5c593dc7cc08839e2ba65193024507=ro+wh
/var/lib/docker/aufs/diff/a1a958a248181c9aa6413848cd67646e5afb9797f1a3da5995c7a636f050f537=ro+wh
/var/lib/docker/aufs/diff/f3c84ac3a0533f691c9fea4cc2ceaaf43baec22bf8d6a479e069f6d814be9b86=ro+wh
/var/lib/docker/aufs/diff/511136ea3c5a64f264b78b5433614aec563103b4d4702f3ba7d4d2698e22c158=ro+wh
64
65
66
67
68
69
70
/run/shm/aufs.xino
```

  可以看到，只有最顶层的是有`rw`权限，其他都是`ro+wh`权限只读。

  docker的aufs配置，可以在`/var/lib/docker/repositories-aufs`这个文件中看到。



## AUFS特性

  AUFS包含了所有UnionFS的特性，把多个目录合并成一个目录，对每个需要合并的目录指定权限，可以实时得去添加、删除或者修改已经被mount的目录。甚至，还可以在多个可写的`branch/dir`之间做负载均衡。

  以上是AUFS的mount特性，我们再来看一下被union的目录的相关权限:

- `rw`表示可读可写
- `ro`表示只读，如果在使用的时候不指定，那么除了第一个以外，其他都是`ro`。也就是时候，`ro`的branch不会接收到写的操作，也不会收到查找`whiteout`的操作。
- `rr`表示`real-read-only`，这个和`ro`还是有区别的。`rr`指的是天生就是只读的分支，能够让AUFS提供性能，例如可以不用设置`inotify`来检查文件变动通知。

- `whiteout`属性一般来说`ro`分支都会有，上面的snippet中也展示了`ro+wh`，而它的意思就是，如果在union中删除某个文件，实际上是处于一个`readonly`的分支目录上。在mount的union这个目录你会看不到这个文件，但是`readonly`这个层上我们无法做任何的修改，因此我们就必须对`readonly`目录里的文件做`whiteout`。AUFS的`whiteout`实现是通过在上层的可写目录下建立对应`whiteout`隐藏文件夹实现的。

  举个🌰：

  还是最开始的例子，我们有以下的结构:

```shell
# tree
.
├── fruits
│   ├── apple
│   └── tomato
├── test
└── vegetables
    ├── carrots
    └── tomato
```

  我们按照下面的指令进行mount和权限分配:

```shell
$ mkdir mnt

$ mount -t aufs -o dirs=./test=rw:./fruits=ro:./vegetables=ro none ./mnt

$ ls ./mnt/
apple  carrots  tomato 
```

  我们在权限为`rw`的test目录下新建一个whiteout的隐藏文件`.wh.apple`，可以发现`./mnt/apple`这个目录之间消失了:

```shell
$ touch ./test/.wh.apple

$ ls ./mnt
carrots  tomato
```

  也就是说这个操作和`rm ./mnt/apple`是一样的。



### 术语

**Branch** – 就是各个要被union起来的目录（就是我在上面使用的dirs的命令行参数）

- Branch根据被union的顺序形成一个stack，一般来说最上面的是可写的，下面的都是只读的。
- Branch的stack可以在被mount后进行修改，比如：修改顺序，加入新的branch，或是删除其中的branch，或是直接修改branch的权限

**Whiteout** 和 **Opaque**

- 如果UnionFS中的某个目录被删除了，那么就应该不可见了，就算是在底层的branch中还有这个目录，那也应该不可见了。

- Whiteout就是某个上层目录覆盖了下层的相同名字的目录。用于隐藏低层分支的文件，也用于阻止readdir进入低层分支。

- Opaque的意思就是不允许任何下层的某个目录显示出来。

- 在隐藏低层档的情况下，whiteout的名字是’.wh.<filename>’。

- 在阻止readdir的情况下，名字是’.wh..wh..opq’或者 ’.wh.__dir_opaque’。

  

### 问题

1. **要有文件在原来的地方被修改了会怎么样，mount的目录会一起改变吗？**

   会也可能不会。因为你可以指定一个叫udba的参数（全称：User’s Direct Branch Access），这个参数有三个取值：

   - **udba=none** – 设置上这个参数后，AUFS会运转的更快，因为那些不在mount目录里发生的修改，aufs不会同步过来了，所以会有数据出错的问题。
   - **udba=reval** – 设置上这个参数后，AUFS会去查文件有没有被更新，如果有的话，就会把修改拉到mount目录内。
   - **udba=notify** – 这个参数会让AUFS为所有的branch注册inotify，这样可以让AUFS在更新文件修改的性能更高一些。

2. **如果有多个rw的branch（目录）被union起来了，那么，当我创建文件的时候，aufs会创建在哪里呢？**

    AUFS提供了一个叫`create`的参数可以供你来配置相当的创建策略，下面有几个例子：

   - **create=rr | round−robin** 轮询。下面的示例可以看到，新创建的文件轮流写到三个目录中：

     ```shell
     derios$ sudo mount -t aufs  -o dirs=./1=rw:./2=rw:./3=rw -o create=rr none ./mnt
     derios$ touch ./mnt/a ./mnt/b ./mnt/c
     derios$ tree
     .
     ├── 1
     │   └── a
     ├── 2
     │   └── c
     └── 3
         └── b
     ```

   - **create=mfs[:second] | most−free−space[:second]** 选一个可用空间最好的分支。可以指定一个检查可用磁盘空间的时间。
   - **create=mfsrr:low[:second]** 选一个空间大于low的branch，如果空间小于low了，那么aufs会使用 round-robin 方式。

   一些AUFS的细节参数，建议还是`man aufs`查看。



## AUFS的性能

  AUFS把所有的分支mount起来，所以在查找文件上是慢一些。因为它要遍历所有的分支，O(N)复杂度的算法。因此分支越多，查找文件的性能也就越慢。但是一旦AUFS找到了这个文件的inode，那之后的读写和操作源文件基本是一样的。

  所以如果程序跑在AUFS下，那么`open`和`stat`操作会有明显的性能下降，分支越多性能就越差。但在`write/read`操作上性能没有什么变化。

  这里有一份IBM做的Docker性能报告《[An Updated Performance Comparison of Virtual Machinesand Linux Containers](http://domino.research.ibm.com/library/cyberdig.nsf/papers/0929052195DD819C85257D2300681E7B/$File/rc25482.pdf)》。

