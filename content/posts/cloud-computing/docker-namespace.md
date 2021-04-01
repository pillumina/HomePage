---
title: "Docker Fundamentals: Namespace"
date: 2021-04-01T11:22:18+08:00
hero: /images/posts/k8s-docker.jpg
menu:
  sidebar:
    name: Docker Fundamentals (Namespace)
    identifier: docker-namespace
    parent: cloud-computing
    weight: 10
draft: false
---

容器技术出现已经很久，只不过Docker容器平台的出现它变火了。Docker是第一个让容器能在不同机器之间移植的系统，它简化了打包应用的流程，也简化了打包应用的库和各种依赖。思考下整个OS的file system能直接被打包成一个简单的可移植的包，一开始的时候概念上还是很有趣的。



有时候我认为自己的阅读比较碎片化(~~short-term memory越来越少~~)，所以我想把之前学习容器知识的一些基础技术再整理出来，也算是给自己学习的反馈。这个基础系列从Linux Namespace开始，后续会陆续介绍比如cgroup、aufs、devicemapper等技术。



## 什么是Namespace

简单来说，linux namespace是Linux提供的一种内核级别环境隔离的方法。在早期的Unix中，提供了一种叫做chroot的系统调用：通过修改root目录把用户关到一个特定的目录下面。这种就是简单的隔离方式，也就是chroot内部的file system无法访问外部的内容。Linux Namespace在此基础之上，提供了对UTS、IPC、mount、network、PID、User等隔离机制。

这里可以简单举例，比如Linux的超级父进程的PID为1，如果我们可以把用户的进程空间关到某个进程分支之下，并且像chroot那样能够让下面的进程看到那个超级父进程的PID为1，而不同PID Namespace中的进程无法看到彼此，这样就能达到进程隔离。

Linux Namespace有以下的种类，供给后续参考（刚看有个印象就行）：

| 分类                   | 系统调用参数  | 相关内核版本                                                 |
| :--------------------- | :------------ | :----------------------------------------------------------- |
| **Mount namespaces**   | CLONE_NEWNS   | [Linux 2.4.19](http://lwn.net/2001/0301/a/namespaces.php3)   |
| **UTS namespaces**     | CLONE_NEWUTS  | [Linux 2.6.19](http://lwn.net/Articles/179345/)              |
| **IPC namespaces**     | CLONE_NEWIPC  | [Linux 2.6.19](http://lwn.net/Articles/187274/)              |
| **PID namespaces**     | CLONE_NEWPID  | [Linux 2.6.24](http://lwn.net/Articles/259217/)              |
| **Network namespaces** | CLONE_NEWNET  | [始于Linux 2.6.24 完成于 Linux 2.6.29](http://lwn.net/Articles/219794/) |
| **User namespaces**    | CLONE_NEWUSER | [始于 Linux 2.6.23 完成于 Linux 3.8)](http://lwn.net/Articles/528078/) |

其主要涉及到三个系统调用：

1. `clone()`： 实现线程的系统调用，用来创建新的线程，并可通过涉及上述参数做到隔离
2. `unshare()`： 让某一个线程脱离某namespace
3. `setns()`: 把某一个线程加到某namespace

如果读者你想看具体的实例，请自己man一下(关注一下自己的linux虚拟机内核)，或者google一下，我这里贴一个`clone()`的source code：

```c++
#define _GNU_SOURCE
#include <sys/types.h>
#include <sys/wait.h>
#include <stdio.h>
#include <sched.h>
#include <signal.h>
#include <unistd.h>
/* 定义一个给 clone 用的栈，栈大小1M */
#define STACK_SIZE (1024 * 1024)
static char container_stack[STACK_SIZE];
char* const container_args[] = {
    "/bin/bash",
    NULL
};
int container_main(void* arg)
{
    printf("Container - inside the container!\n");
    /* 直接执行一个shell，以便我们观察这个进程空间里的资源是否被隔离了 */
    execv(container_args[0], container_args); 
    printf("Something's wrong!\n");
    return 1;
}
int main()
{
    printf("Parent - start a container!\n");
    /* 调用clone函数，其中传出一个函数，还有一个栈空间的（为什么传尾指针，因为栈是反着的） */
    int container_pid = clone(container_main, container_stack+STACK_SIZE, SIGCHLD, NULL);
    /* 等待子进程结束 */
    waitpid(container_pid, NULL, 0);
    printf("Parent - container stopped!\n");
    return 0;
}
```

上面的程序注释写的比较明白，和pthreads差不多。不过这个程序里，父子进程的进程空间没有什么区别，父进程能访问到的明显子进程也能访问。

我们用几个例子来看看linux的namespace到底是啥样的，运行的虚拟机为`ubuntu14.4`



## UTS Namespace

这里略去一些头文件和数据结构的定义：

```c++
int container_main(void* arg)
{
    printf("Container - inside the container!\n");
    sethostname("container",10); /* 设置hostname */
    execv(container_args[0], container_args);
    printf("Something's wrong!\n");
    return 1;
}
int main()
{
    printf("Parent - start a container!\n");
    int container_pid = clone(container_main, container_stack+STACK_SIZE, 
            CLONE_NEWUTS | SIGCHLD, NULL); /*启用CLONE_NEWUTS Namespace隔离 */
    waitpid(container_pid, NULL, 0);
    printf("Parent - container stopped!\n");
    return 0;
}
```

`执行上述的c++程序，会发现子进程的hostname变为了container`

```shell
derios@ubuntu:~$ sudo ./uts
Parent - start a container!
Container - inside the container!
root@container:~# hostname
container
root@container:~# uname -n
container
```



## IPC Namespace

IPC(Inter-Process Communication)，是Unix/Linux下的一种通信方式。IPC有共享内存、信号量、消息队列等方法。所以如果要隔离我们也要把IPC进行隔离。换句话说，这样可以保证只有在同一个namespace下的进程之间才能互相通信。目前我对IPC的原理没什么研究，查了查资料，IPC需要有个全局的ID，那么如果我们要做隔离，namespace肯定需要对这个全局ID进行隔离，不能和其他namespace中的进程共享。

要启动IPC隔离，我们需要在`clone`时加上`CLONE_NEWPIC`的参数:

```c++
int container_pid = clone(container_main, container_stack+STACK_SIZE, 
            CLONE_NEWUTS | CLONE_NEWIPC | SIGCHLD, NULL);
```

我们先创建一个IPC的queue，下面的全局ID为0：

```shell
derios@ubuntu:~$ ipcmk -Q 
Message queue id: 0
derios@ubuntu:~$ ipcs -q
------ Message Queues --------
key        msqid      owner      perms      used-bytes   messages    
0xd0d56eb2 0          hchen      644        0            0
```

如果我们不加`CLONE_NEWIPC`参数运行程序，我们可以看到在子进程中还是能看到全局的IPC queue：

```shell
derios@ubuntu:~$ sudo ./uts 
Parent - start a container!
Container - inside the container!
root@container:~# ipcs -q
------ Message Queues --------
key        msqid      owner      perms      used-bytes   messages    
0xd0d56eb2 0          hchen      644        0            0
```

如果我们运行加上了`CLONE_NEWIPC`的程序，可以有如下的结果:

```shell
root@ubuntu:~$ sudo./ipc
Parent - start a container!
Container - inside the container!
root@container:~/linux_namespace# ipcs -q
------ Message Queues --------
key        msqid      owner      perms      used-bytes   messages
```

可见IPC已经被隔离。



## PID Namespace

我们继续修改上述的程序：

```c++
int container_main(void* arg)
{
    /* 查看子进程的PID，我们可以看到其输出子进程的 pid 为 1 */
    printf("Container [%5d] - inside the container!\n", getpid());
    sethostname("container",10);
    execv(container_args[0], container_args);
    printf("Something's wrong!\n");
    return 1;
}
int main()
{
    printf("Parent [%5d] - start a container!\n", getpid());
    /*启用PID namespace - CLONE_NEWPID*/
    int container_pid = clone(container_main, container_stack+STACK_SIZE, 
            CLONE_NEWUTS | CLONE_NEWPID | SIGCHLD, NULL); 
    waitpid(container_pid, NULL, 0);
    printf("Parent - container stopped!\n");
    return 0;
}
```

运行看一下，发现子进程的PID为1：

```shell
derios@ubuntu:~$ sudo ./pid
Parent [ 3474] - start a container!
Container [ 1] - inside the container!
root@container:~# echo $$
1
```

这里的1有啥意义，你可能会问。其实在传统UNIX系统中，PID为1的进程地位比较特殊，指代`init`

，作为所有进程的父进程，有非常多的特权（信号屏蔽etc.），此外它还会检查所有进程的状态，而且如果子进程脱离了父进程（父进程没有wait它），那么`init`会负责回收资源并结束这个子进程。所以，要做到进程空间的隔离，首先要创建pid为1的进程，比如可以像chroot一样，把子进程的pid在容器内变为1。

不过，很奇怪的是，**我们在子进程的shell里执行top, ps等命令，还是可以看到所有的进程。**这意味着隔离并没有完全。因为像ps, top这些命令会读取`/proc`文件系统，而因为`/proc`文件系统在父子进程里都是一样的，所以命令的回显也都是一样的。

因此，我们还要做到**对文件系统的隔离**。



## Mount Namespace

下面的程序，我们在启用mount namespace并在子进程中重新mount了`/proc`文件系统。

```c++
int container_main(void* arg)
{
    printf("Container [%5d] - inside the container!\n", getpid());
    sethostname("container",10);
    /* 重新mount proc文件系统到 /proc下 */
    system("mount -t proc proc /proc");
    execv(container_args[0], container_args);
    printf("Something's wrong!\n");
    return 1;
}
int main()
{
    printf("Parent [%5d] - start a container!\n", getpid());
    /* 启用Mount Namespace - 增加CLONE_NEWNS参数 */
    int container_pid = clone(container_main, container_stack+STACK_SIZE, 
            CLONE_NEWUTS | CLONE_NEWPID | CLONE_NEWNS | SIGCHLD, NULL);
    waitpid(container_pid, NULL, 0);
    printf("Parent - container stopped!\n");
    return 0;
}
```

执行结果如下:

```shell
derioshen@ubuntu:~$ sudo ./pid.mnt
Parent [ 3502] - start a container!
Container [    1] - inside the container!
root@container:~# ps -elf 
F S UID        PID  PPID  C PRI  NI ADDR SZ WCHAN  STIME TTY          TIME CMD
4 S root         1     0  0  80   0 -  6917 wait   19:55 pts/2    00:00:00 /bin/bash
0 R root        14     1  0  80   0 -  5671 -      19:56 pts/2    00:00:00 ps -elf
```

我们看到只有2个进程了，`pid=1`的是我们的`/bin/bash`，同时再看看`/proc`目录，也变得比较干净:

```shell
root@container:~# ls /proc
1          dma          key-users   net            sysvipc
16         driver       kmsg        pagetypeinfo   timer_list
acpi       execdomains  kpagecount  partitions     timer_stats
asound     fb           kpageflags  sched_debug    tty
buddyinfo  filesystems  loadavg     schedstat      uptime
bus        fs           locks       scsi           version
cgroups    interrupts   mdstat      self           version_signature
cmdline    iomem        meminfo     slabinfo       vmallocinfo
consoles   ioports      misc        softirqs       vmstat
cpuinfo    irq          modules     stat           zoneinfo
crypto     kallsyms     mounts      swaps
devices    kcore        mpt         sys
diskstats  keys         mtrr        sysrq-trigger
```

通过`CLONE_NEWNS`创建mount namespace以后，父进程会把自己的文件结构复制给子进程。而子进程中新的namespace中所有mount操作都只会影响自身的文件系统，不会对外界产生任何影响，这就做到了严格的隔离。

那么我们是不是还有别的一些文件系统也要mount？答案是肯定的。



## Docker的Mount Namespace

我们可以简单搞个小的镜像，这种玩法是我google参考来的，模仿docker的mount namespace。

首先，我们需要一个rootfs， 也就是我们需要把我们要做的镜像中的那些命令什么的copy到一个rootfs的目录下，我们模仿Linux构建如下的目录：

```shell
derios@ubuntu:~/rootfs$ ls
bin  dev  etc  home  lib  lib64  mnt  opt  proc  root  run  sbin  sys  tmp  usr  var
```

然后，我们把一些我们需要的命令copy到 rootfs/bin目录中（sh命令必需要copy进去，不然我们无法 chroot ）：

```shell
derios@ubuntu:~/rootfs$ ls ./bin ./usr/bin
 
./bin:
bash   chown  gzip      less  mount       netstat  rm     tabs  tee      top       tty
cat    cp     hostname  ln    mountpoint  ping     sed    tac   test     touch     umount
chgrp  echo   ip        ls    mv          ps       sh     tail  timeout  tr        uname
chmod  grep   kill      more  nc          pwd      sleep  tar   toe      truncate  which

./usr/bin:
awk  env  groups  head  id  mesg  sort  strace  tail  top  uniq  vi  wc  xargs
```

注：你可以使用ldd命令把这些命令相关的那些so文件copy到对应的目录：

```shell
derios@ubuntu:~/rootfs/bin$ ldd bash
  linux-vdso.so.1 =>  (0x00007fffd33fc000)
  libtinfo.so.5 => /lib/x86_64-linux-gnu/libtinfo.so.5 (0x00007f4bd42c2000)
  libdl.so.2 => /lib/x86_64-linux-gnu/libdl.so.2 (0x00007f4bd40be000)
  libc.so.6 => /lib/x86_64-linux-gnu/libc.so.6 (0x00007f4bd3cf8000)
  /lib64/ld-linux-x86-64.so.2 (0x00007f4bd4504000)
```

下面是我的rootfs中的一些so文件：

```shell
derios@ubuntu:~/rootfs$ ls ./lib64 ./lib/x86_64-linux-gnu/

./lib64:
ld-linux-x86-64.so.2

./lib/x86_64-linux-gnu/:
libacl.so.1      libmemusage.so         libnss_files-2.19.so    libpython3.4m.so.1
libacl.so.1.1.0  libmount.so.1          libnss_files.so.2       libpython3.4m.so.1.0
libattr.so.1     libmount.so.1.1.0      libnss_hesiod-2.19.so   libresolv-2.19.so
libblkid.so.1    libm.so.6              libnss_hesiod.so.2      libresolv.so.2
libc-2.19.so     libncurses.so.5        libnss_nis-2.19.so      libselinux.so.1
libcap.a         libncurses.so.5.9      libnss_nisplus-2.19.so  libtinfo.so.5
libcap.so        libncursesw.so.5       libnss_nisplus.so.2     libtinfo.so.5.9
libcap.so.2      libncursesw.so.5.9     libnss_nis.so.2         libutil-2.19.so
libcap.so.2.24   libnsl-2.19.so         libpcre.so.3            libutil.so.1
libc.so.6        libnsl.so.1            libprocps.so.3          libuuid.so.1
libdl-2.19.so    libnss_compat-2.19.so  libpthread-2.19.so      libz.so.1
libdl.so.2       libnss_compat.so.2     libpthread.so.0
libgpm.so.2      libnss_dns-2.19.so     libpython2.7.so.1
libm-2.19.so     libnss_dns.so.2        libpython2.7.so.1.0
```

包括这些命令依赖的一些配置文件：

```shell
derios@ubuntu:~/rootfs$ ls ./etc
bash.bashrc  group  hostname  hosts  ld.so.cache  nsswitch.conf  passwd  profile  
resolv.conf  shadow
```

看到现在你可能比较懵逼，有的比较熟悉os的同学也可能会问：有的配置希望是在容器起动时给他设置的，而不是hard code在镜像中的。比如：/etc/hosts，/etc/hostname，还有DNS的/etc/resolv.conf文件。OK, 那我们在rootfs外面，我们再创建一个conf目录，把这些文件放到这个目录中:

```shell
derios@ubuntu:~$ ls ./conf
hostname     hosts     resolv.conf
```

这样，我们的父进程就可以动态地设置容器需要的这些文件的配置， 然后再把他们mount进容器，这样，容器的镜像中的配置就比较灵活了。

接下来是程序(~~Google真好~~)

```c++
#define _GNU_SOURCE
#include <sys types.h="">
#include <sys wait.h="">
#include <sys mount.h="">
#include <stdio.h>
#include <sched.h>
#include <signal.h>
#include <unistd.h>

#define STACK_SIZE (1024 * 1024)

static char container_stack[STACK_SIZE];
char* const container_args[] = {
    "/bin/bash",
    "-l",
    NULL
};

int container_main(void* arg)
{
    printf("Container [%5d] - inside the container!\n", getpid());

    //set hostname
    sethostname("container",10);

    //remount "/proc" to make sure the "top" and "ps" show container's information
    if (mount("proc", "rootfs/proc", "proc", 0, NULL) !=0 ) {
        perror("proc");
    }
    if (mount("sysfs", "rootfs/sys", "sysfs", 0, NULL)!=0) {
        perror("sys");
    }
    if (mount("none", "rootfs/tmp", "tmpfs", 0, NULL)!=0) {
        perror("tmp");
    }
    if (mount("udev", "rootfs/dev", "devtmpfs", 0, NULL)!=0) {
        perror("dev");
    }
    if (mount("devpts", "rootfs/dev/pts", "devpts", 0, NULL)!=0) {
        perror("dev/pts");
    }
    if (mount("shm", "rootfs/dev/shm", "tmpfs", 0, NULL)!=0) {
        perror("dev/shm");
    }
    if (mount("tmpfs", "rootfs/run", "tmpfs", 0, NULL)!=0) {
        perror("run");
    }
    /* 
     * 模仿Docker的从外向容器里mount相关的配置文件 
     * 你可以查看：/var/lib/docker/containers/<container_id>/目录，
     * 你会看到docker的这些文件的。
     */
    if (mount("conf/hosts", "rootfs/etc/hosts", "none", MS_BIND, NULL)!=0 ||
          mount("conf/hostname", "rootfs/etc/hostname", "none", MS_BIND, NULL)!=0 ||
          mount("conf/resolv.conf", "rootfs/etc/resolv.conf", "none", MS_BIND, NULL)!=0 ) {
        perror("conf");
    }
    /* 模仿docker run命令中的 -v, --volume=[] 参数干的事 */
    if (mount("/tmp/t1", "rootfs/mnt", "none", MS_BIND, NULL)!=0) {
        perror("mnt");
    }

    /* chroot 隔离目录 */
    if ( chdir("./rootfs") != 0 || chroot("./") != 0 ){
        perror("chdir/chroot");
    }

    execv(container_args[0], container_args);
    perror("exec");
    printf("Something's wrong!\n");
    return 1;
}

int main()
{
    printf("Parent [%5d] - start a container!\n", getpid());
    int container_pid = clone(container_main, container_stack+STACK_SIZE, 
            CLONE_NEWUTS | CLONE_NEWIPC | CLONE_NEWPID | CLONE_NEWNS | SIGCHLD, NULL);
    waitpid(container_pid, NULL, 0);
    printf("Parent - container stopped!\n");
    return 0;
}
</container_id></unistd.h></signal.h></sched.h></stdio.h></sys></sys></sys>
```

sudo运行上面的程序，你会看到下面的挂载信息以及一个所谓的“镜像”：

```shell
derios@ubuntu:~$ sudo ./mount 
Parent [ 4517] - start a container!
Container [    1] - inside the container!
root@container:/# mount
proc on /proc type proc (rw,relatime)
sysfs on /sys type sysfs (rw,relatime)
none on /tmp type tmpfs (rw,relatime)
udev on /dev type devtmpfs (rw,relatime,size=493976k,nr_inodes=123494,mode=755)
devpts on /dev/pts type devpts (rw,relatime,mode=600,ptmxmode=000)
tmpfs on /run type tmpfs (rw,relatime)
/dev/disk/by-uuid/18086e3b-d805-4515-9e91-7efb2fe5c0e2 on /etc/hosts type ext4 (rw,relatime,errors=remount-ro,data=ordered)
/dev/disk/by-uuid/18086e3b-d805-4515-9e91-7efb2fe5c0e2 on /etc/hostname type ext4 (rw,relatime,errors=remount-ro,data=ordered)
/dev/disk/by-uuid/18086e3b-d805-4515-9e91-7efb2fe5c0e2 on /etc/resolv.conf type ext4 (rw,relatime,errors=remount-ro,data=ordered)

root@container:/# ls /bin /usr/bin
/bin:
bash   chmod  echo  hostname  less  more  mv   ping  rm   sleep  tail  test    top   truncate  uname
cat    chown  grep  ip        ln    mount  nc   ps    sed  tabs   tar   timeout  touch  tty     which
chgrp  cp     gzip  kill      ls    mountpoint  netstat  pwd   sh   tac    tee   toe    tr   umount

/usr/bin:
awk  env  groups  head  id  mesg  sort  strace  tail  top  uniq  vi  wc  xargs
```

