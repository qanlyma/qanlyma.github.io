---
title: 「学」 Linux 使用教程
category_bar: true
date: 2022-11-10 22:51:47
tags:
categories: 学习笔记
---

Linux，全称 GNU/Linux，是一种免费使用和自由传播的类 UNIX 操作系统。作为研究生，我一开始学习 Linux 的源动力自然也是任职要求上的那一条：熟悉 Linux 环境。那么 Linux 系统到底好在哪里？

<!-- more -->

## Linux 的优势

很多公司包括我们自己的实验室服务器都是用的 Linux 系统，其优点概括而言就是开源、免费、生态好、安全性高。

首先 Linux 系统稳定性极高且不容易染毒（从来没有听说过要装什么杀毒软件或者需要清理内存，而且我们实验室的服务器永远都不会关机），它自带的命令功能十分强大（比如你可以用简单的几行命令就搞定软件的下载安装整个步骤），还拥有开放的源代码和高度的可定制性。其次，Linux 系统的远程管理非常方便。通过 ssh 或者 telnet 的连接，在很小的带宽环境下也能很轻松实现远程操作。就如服务器摆在眼前一样的感觉。这是 windows 系统无法比拟的。

## Linux 发行版本

![发行版说简单点就是将 Linux 内核与应用软件做一个打包。](1.png)

关于不同版本的安装网上教程很多，就不在此赘述。

## Linux 启动过程

Linux系统的启动过程并不是大家想象中的那么复杂，其过程可以分为5个阶段：

* **内核的引导**
	当计算机打开电源后，首先是BIOS开机自检，按照BIOS中设置的启动设备（通常是硬盘）来启动。操作系统接管硬件以后，首先读入 /boot 目录下的内核文件。

* **运行 init**
	init 进程是系统所有进程的起点，你可以把它比拟成系统所有进程的老祖宗，没有这个进程，系统中任何进程都不会启动。init 程序首先是需要读取配置文件 /etc/inittab。

* **系统初始化**
	在init的配置文件中有这么一行： `si::sysinit:/etc/rc.d/rc.sysinit`　它调用执行了 /etc/rc.d/rc.sysinit，而 rc.sysinit 是一个 bash shell 的脚本，它主要是完成一些系统初始化的工作，rc.sysinit 是每一个运行级别都要首先运行的重要脚本。它主要完成的工作有：激活交换分区，检查磁盘，加载硬件模块以及其它一些需要优先执行任务。

* **建立终端**
	rc 执行完毕后，返回 init。这时基本系统环境已经设置好了，各种守护进程也已经启动了。 init 接下来会打开终端，以便用户登录系统。

* **用户登录系统**
	一般来说，用户的登录方式有三种：
	（1）命令行登录
	（2）ssh登录
	（3）图形界面登录

![启动过程](2.png)

## Linux 目录结构

![树状目录结构](3.jpg)

> /

>> /bin：
bin 是 Binaries (二进制文件) 的缩写, 这个目录存放着最经常使用的命令。

>> /boot：
这里存放的是启动 Linux 时使用的一些核心文件，包括一些连接文件以及镜像文件。

>> /dev ：
dev 是 Device(设备) 的缩写, 该目录下存放的是 Linux 的外部设备，在 Linux 中访问设备的方式和访问文件的方式是相同的。

>> /etc：
etc 是 Etcetera(等等) 的缩写,这个目录用来存放所有的系统管理所需要的配置文件和子目录。

>> /home：
用户的主目录，在 Linux 中，每个用户都有一个自己的目录，一般该目录名是以用户的账号命名的，如上图中的 alice、bob 和 eve。

>> /lib：
lib 是 Library(库) 的缩写这个目录里存放着系统最基本的动态连接共享库，其作用类似于 Windows 里的 DLL 文件。几乎所有的应用程序都需要用到这些共享库。

>> /lost+found：
这个目录一般情况下是空的，当系统非法关机后，这里就存放了一些文件。

>> /media：
linux 系统会自动识别一些设备，例如U盘、光驱等等，当识别后，Linux 会把识别的设备挂载到这个目录下。

>> /mnt：
系统提供该目录是为了让用户临时挂载别的文件系统的，我们可以将光驱挂载在 /mnt/ 上，然后进入该目录就可以查看光驱里的内容了。

>> /opt：
opt 是 optional(可选) 的缩写，这是给主机额外安装软件所摆放的目录。比如你安装一个ORACLE数据库则就可以放到这个目录下。默认是空的。

>> /proc：
proc 是 Processes(进程) 的缩写，/proc 是一种伪文件系统（也即虚拟文件系统），存储的是当前内核运行状态的一系列特殊文件，这个目录是一个虚拟的目录，它是系统内存的映射，我们可以通过直接访问这个目录来获取系统信息。

>> /root：
该目录为系统管理员，也称作超级权限者的用户主目录。

>> /sbin：
s 就是 Super User 的意思，是 Superuser Binaries (超级用户的二进制文件) 的缩写，这里存放的是系统管理员使用的系统管理程序。

>> /selinux：
 这个目录是 Redhat/CentOS 所特有的目录，Selinux 是一个安全机制，类似于 windows 的防火墙，但是这套机制比较复杂，这个目录就是存放selinux相关的文件的。

>> /srv：
 该目录存放一些服务启动之后需要提取的数据。

>> /sys：
该文件系统是内核设备树的一个直观反映。
当一个内核对象被创建的时候，对应的文件和目录也在内核对象子系统中被创建。

>> /tmp：
tmp 是 temporary(临时) 的缩写这个目录是用来存放一些临时文件的。

>> /usr：
usr 是 unix shared resources(共享资源) 的缩写，这是一个非常重要的目录，用户的很多应用程序和文件都放在这个目录下，类似于 windows 下的 program files 目录。

>>> /usr/bin：
系统用户使用的应用程序。

>>> /usr/sbin：
超级用户使用的比较高级的管理程序和系统守护程序。

>>> /usr/src：
内核源代码默认的放置目录。

>> /var：
var 是 variable(变量) 的缩写，这个目录中存放着在不断扩充着的东西，我们习惯将那些经常被修改的目录放在这个目录下。包括各种日志文件。

>> /run：
是一个临时文件系统，存储系统启动以来的信息。当系统重启时，这个目录下的文件应该被删掉或清除。如果你的系统上有 /var/run 目录，应该让它指向 run。

## Linux 权限管理

初学 Linux 我遇到过很多次没有权限而产生的报错，之前的解决办法一直是使用 `su` 命令给自己 root 权限，注意此时命令行的 `$` 会变成 `#` 。Linux 系统是一种典型的多用户系统，不同的用户处于不同的地位，拥有不同的权限。为了保护系统的安全性，Linux 系统对不同的用户访问同一文件（包括目录文件）的权限做了不同的规定。

在 Linux 中我们通常使用以下两个命令来修改文件或目录的所属用户与权限：

* chown (change owner) ： 修改所属用户与组。
* chmod (change mode) ： 修改用户的权限。

在 Linux 中我们可以使用 `ll` 或者 `ls –l` 命令来显示一个文件的属性以及文件所属的用户和组

![文件属性及权限](4.jpg)

第一位表示文件的属性： `d` 是目录， `-` 是文件， `l` 表示链接文档等等。

接下来的字符中，以三个为一组，且均为 `rwx` 的三个参数的组合。

* `r` 代表可读(read)
* `w` 代表可写(write)
* `x` 代表可执行(execute)
* `-` 代表没有此权限

而这三组也分别对应属主（该文件的所有者）权限、属组权限、其他用户权限。

我们可以使用 chmod 更改文件上述的 9 个属性，将三组权限看作三个二进制数，开启设 1 关闭设 0
如可以使用我们常见的 `chmod 777 文件名` 来开启所有权限。

## Linux 用户管理

此节是关于 Linux 的用户以及用户组管理的，我目前用的不多，请参考[菜鸟教程](https://www.runoob.com/linux/linux-user-manage.html)。

## Linux 磁盘管理

同上，请参考[菜鸟教程](https://www.runoob.com/linux/linux-filesystem.html)

## Linux 常用命令

* ls （list files）: 列出目录及文件名
* cd （change directory）：切换目录
* pwd （print work directory）：显示目前的目录
* mkdir （make directory）：创建一个新的目录
* rmdir （remove directory）：删除一个空的目录
* cp （copy file）: 复制文件或目录
* rm （remove）: 删除文件或目录
* mv （move file）: 移动文件与目录，或修改文件与目录的名称
* cat （concatenate）：显示文件内容或是将多个文件合并成一个文件
* yum（Yellow dog Updater, Modified）：一个在 Fedora 和 RedHat 以及 SUSE 中的 Shell 前端软件包管理器
* apt（Advanced Packaging Tool）：一个在 Debian 和 Ubuntu 中的 Shell 前端软件包管理器

你可以使用 man [命令] 来查看各个命令的使用文档，如： `man cp` 。此外文本编辑器 [vim](https://www.runoob.com/linux/linux-vim.html) 也是很值得学习一下的。

我们通常会在命令后添加参数来执行更多功能，具体参数和使用方法请参考[这篇笔记](https://blog.csdn.net/weixin_66975803/article/details/123693997)。

## Linux 更多命令

* [进程相关](https://blog.csdn.net/weixin_45004203/article/details/125885958)：`top`, `ps`, `pidof`, `kill`, `killall`, `pkill`

* 端口相关：`lsof -i:port`, `netstat -nltp | grep port`

## Bash Shell

我之前一直知道 Shell 这个东西，但是对于它的认识很模糊，关于 Shell 有如下两条解释：

* Shell 是一个应用程序，它连接了用户和 Linux 内核，让用户能够更加高效、安全、低成本地使用 Linux 内核，不启动 Shell 的话，用户就没办法使用 Linux。

* Shell 是一个命令语言解释器, 在操作系统的最外层, 是用户（用户程序）与操作系统（Linux）内核的接口程序，用户输入的每个命令都由 Shell 先翻译再传给 Linux 内核, 并将处理后的结果输出至屏幕。

常用的 Shell 功能有两种形式外在形式： GUI 和 cmdline。

Shell 的使用方法有两种：1. 直接输入命令； 2. 使用 .sh 脚本。 对于脚本语法感兴趣请学习 [Shell 教程](https://www.runoob.com/linux/linux-shell.html)。

Bash (GUN Bourne-Again Shell）是许多 Linux 发行版本默认的 Shell。

## 写在最后

我在这篇文章里面列出了我自己认为 Linux 中比较重要的内容，学习它的办法唯有多用，逐渐感受它的强大与可靠，你一定会理解为什么它会受到那么多公司的青睐。 Linux 上手可能会比 Windows 要困难一点，尤其是在没有装图形化界面的服务器上，各位同学不必害怕，熟能生巧，习惯以后真的非常好用。相信不久后你也可以在简历里面加上一句 “熟悉 Linux 开发环境” 了。

附两个我常用的在本地与服务器传文件的命令：

```cmd
// 文件从远程系统上用户的主目录复制到本地当前目录
scp username@ip_address:/home/username/filename .

// 将本地文件复制到远程系统上用户名的主目录
scp filename username@ip_address:/home/username

// 也可以复制目录
scp -r source_dir username@ip_address:/home/username/target_dir
```