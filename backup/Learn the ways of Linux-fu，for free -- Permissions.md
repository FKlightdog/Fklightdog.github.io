该文章为学习[*Learn the ways of Linux-fu，for free*](https://linuxjourney.com/)的记录，本文为*Permissions*
### 文件权限
对于Linux系统而言，每个文件都有自己的权限
```bash
$ ls -l Desktop/
drwxr-xr-x 2 pete penguins 4096 Dec 1 11:45 .
```
上述中的drwxr-xr-x为权限描述符，开头的d表示这是一个文件夹，而其他文件会以-开头，除了开头字母以为，其他元素以3bits来进行分割，如下
`d|rwx|r-x|r-x`
对于第一个3bits组而言表示的是个人权限，其次为小组权限，最后为其他用户的权限
对于上述的每个字符的含义如下：

- r:代表可读权限
- w：代表可写权限
- x：代表可执行权限

### 修改权限
我们可以使用 **chmod** 语句进行权限的修改，我们可以用 **+** 或者 **-** 号进行权限的增加和删除，下面为例子：
```bash
$ chmod u+x a.txt
$ chmod u-w a.txt
```
其中u表示user
除此之外我们也可以利用数字来进行修改，利用数字我们可以一口气把所有权限都进行修改，包括：用户权限、组权限和其他用户的权限。
在Linux中我们用数字可以代表权限：

- 4：代表可读权限 **r**
- 2：代表可写权限 **w**
- 1：代表可执行权限 **x**

例如
```bash
$ chmod 755 a.txt
-rwxr-xr-x 1 fklightdog fklightdog 0 Sep 15 16:08 a.txt
```
其中7 = 4 + 2 + 1; 5 = 4 + 1;
### 更多的权限
```bash
$ sudo chown bob a.txt
$ sudo chgrp whales a.txt
$ sudo chown bob:whales a.txt
```
上述第一条会将a.txt归属到bob，第二条会将a.txt归属到whales group，第三条可以一口气执行完毕
### 修改创建文件时的文件权限
我们可以使用 **umask** 语句进行修改，其会改变创建文件时候其的权限，但是使用其为一次性的，永久修改要在.profile进行修改，具体如下：
```bash
$ umask 021
$ touch b.txt
-rw-r--rw- 1 fklightdog fklightdog 0 Sep 15 19:48 b.txt
```
这样默认创建出来的文件权限为 **-rw-r--rw-**， 这个结果是由666 - 021 = 743 得到的
uask的原理是将默认的权限 - 掩码来决定最终的权限
```
文件权限 = 默认权限 (666) - umask 掩码
目录权限 = 默认权限 (777) - umask 掩码
```
### 设置uid
在Linux中我们有时需要root权限来做很多事情，但是每次访问被保护的文件时候，不能每次都输入password，因此有特殊的文件权限可以方便我们。设置用户ID (SUID)允许用户以程序文件的所有者而不是自己的身份运行程序。
典型的例子就是 **passwd** 语句进行修改密码
当执行这条语句的时候，发生了什么？主要是在修改一些文件，其中最重要的是 /etc/shadow，让我们看看这个文件：
```bash
$ ls -l /etc/shadow
-rw-r----- 1 root shadow 859 Sep 15 14:12 /etc/shadow
```
我们可以发现这是root文件，那我们是怎么修改用户的密码的？
让我们看看另外一个重要文件：
```bash
$ ls -l /usr/bin/passwd
-rwsr-xr-x 1 root root 59976 Feb  6  2024 /usr/bin/passwd
```
我们可以发现这里面有**s**， 这个s就是SUID，有了这个，我们就可以以这个文件的所有者的身份来运行，在此例中我们为root权限，因此passwd运行时候我们是以root身份的，因此我们可以在期间访问/etc/shadow,如果你删除上面的s的话，我们就无法修改密码

同样，我们可以用**chmod** 来修改权限
```bash
sudo chmod u+s myfile
sudo chmod 4755 myfile // 4为s权限
```
### 设置gid
gid的情况类似于上面的uid，只不过是允许群组成员能临时以程序文件的所有者身份来运行
```bash
$ ls -l /usr/bin/wall
-rwxr-sr-x 1 root tty 19024 Dec 14 11:45 /usr/bin/wall
```
我们可以利用同样的方法进行修改权限
```bash
$ sudo chmod g+s myfile
$ sudo chmod 2555 myfile//其中2代表gid
```
### 进程权限
前面已经讨论过了**passwd**，这是否意味着我们可以借此修改root文件，好消息是，这并不会进行，因为Linux有很多的uid，有三个uid跟每个进程有关系

当你启动一个进程时，它以与运行它的用户或组相同的权限运行，这被称为有效用户ID。该UID用于向进程授予访问权限。因此，很自然地，如果Bob运行touch命令，进程将以他的身份运行，他创建的任何文件都归他所有。

还有另一个UID，称为真实用户ID，这是启动进程的用户的ID。这些用于跟踪启动进程的用户是谁。

最后一个UID是保存的用户ID，这允许进程在有效UID和真实UID之间切换，反之亦然。这很有用，因为我们不希望我们的进程一直以提升的权限运行，在特定时间使用特权是一种良好的实践。

现在我们再看一下passwd命令，把这些内容拼凑起来。

运行passwd命令时，有效的UID是用户ID，现在假设是500。哦，等等，请记住，passwd命令启用了SUID权限。所以当你运行它时，你的有效UID现在是0(0是根节点的UID)。现在这个程序可以以root权限访问文件。

假设你想要修改Sally的密码，Sally的UID是600。你就不走运了，幸运的是这个过程也有你的真实UID 500。它知道你的UID是500，所以你不能修改UID为600的密码。(如果你是一台机器上的超级用户，可以控制和更改一切，这当然会被忽略)。

因为运行了passwd命令，它将使用你的真实UID启动这个进程，并保存文件所有者的UID(有效UID)，因此你可以在两者之间切换。如果不是必需的，不需要修改所有具有root权限的文件。

大多数情况下，真实UID和有效UID是相同的，但在passwd命令这样的情况下，它们会发生变化。

