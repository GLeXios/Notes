



# Linux

## Linux环境配置文件.bashrc .bash_profile和.profile

### 1、 .bashrc：交互式非登录Shell的配置文件

**1.1、什么是.bashrc？**

bashrc 文件用于配置函数或别名。bashrc 文件有两种级别：

- 系统级：位于`**/etc/bashrc**`，对所有用户生效。
- 用户级：位于`**~/.bashrc**`，仅对当前用户生效

bashrc 文件只会对指定的 shell 类型起作用，bashrc 只会被 bash shell 调用，用于交互式非登录shell会话。**这意味着每次你打开一个新的终端窗口或标签页时，.bashrc中的配置就会被加载**。

**1.2、使用场景**

.bashrc适用于那些需要频繁执行的配置，如：

- 设置shell别名和函数
- 定义环境变量，这些变量仅在当前用户的shell会话中有效
- 修改命令提示符
- 设置shell的查找路径（$PATH）

示例：在.bashrc文件中，你可能会有这样的内容：

```
alias ll='ls -la'
export PATH="$HOME/bin:$PATH"
```

### 2、.bash_profile和.profile：登录Shell的配置文件

**2.1、什么是.bash_profile和.profile？**

profile，路径：**`\*/etc/profile\*`**，用于设置系统级的环境变量和启动程序，在这个文件下配置会对**所有用户**生效。

当用户登录（login）时，文件会被执行，并从`*/etc/profile.d*`目录的[配置文件](https://zhida.zhihu.com/search?content_id=178168828&content_type=Article&match_order=1&q=配置文件&zhida_source=entity)中查找shell设置。

一般**不建议**在**`/etc/profile`**文件中添加环境变量，因为在这个文件中添加的设置会对所有用户起作用。

.bash_profile只对单一用户有效，文件存储位于`~/.bash_profile`，该文件是一个用户级的设置，可以理解为某一个用户的 profile 目录下。

这个文件同样也可以用于配置环境变量和启动程序，但只针对单个用户有效。

和 profile 文件类似，bash_profile 也会在用户登录（login）时生效，也可以用于设置环境变理。

**但与 profile 不同，bash_profile 只会对当前用户生效。**

.bash_profile（对于Bash shell）和.profile（对于其他sh兼容shell）是在登录shell会话开始时加载的配置文件。**当你通过图形界面登录、通过SSH远程连接到系统或通过终端登录时，这些文件中的设置就会生效**。

**2.2、使用场景**

这些文件通常包含：

- 环境变量的设置，这些变量在整个登录会话中都有效
- 启动必要的应用程序
- 读取其他配置文件，如.bashrc

示例：

在.bash_profile或.profile文件中，你可以添加代码来加载.bashrc，**这确保了即使在登录shell中，.bashrc的配置也能被应用**。：

```
if [ -f ~/.bashrc ]; then
    source ~/.bashrc
fi
```

### 3、区别和选择

这三种文件类型的差异用一句话表述就是：

`/etc/profile`，`/etc/bashrc` 是系统全局环境变量设定；

`~/.profile`，`~/.bashrc`用户家目录下的私有环境变量设定

**3.1、交互式非登录vs登录Shell**

.bashrc适用于每次打开新终端时的交互式非登录shell。 .bash_profile和.profile适用于开始一个新的登录shell会话。

**3.2、加载频率**

- .bashrc在每次打开新的终端窗口或标签时加载。
- .bash_profile和.profile仅在登录时加载。

**3.3、通用性**

- .profile可以被多种兼容sh的shell读取，而**.bash_profile特定于Bash**。

**3.4、选择使用哪个文件？**

1. **对于Bash用户：在.bash_profile中设置环境变量，并确保它加载.bashrc**。
2. 对于非Bash sh兼容shell用户：使用.profile来设置环境变量。
3. 通用配置：**可以将通用配置放在.profile中，特定于Bash的配置放在.bashrc中。**

## ${}和$()

**（1）${}**

`${}` 主要用于变量替换。它提供了比简单的 `$variable` 更强大的功能，包括但不限于：

- **参数扩展**：允许你更灵活地处理变量内容。
- **默认值设定**：如果变量未设置或为空时提供一个默认值。
- **字符串操作**：如获取字符串长度、提取子串等。

**示例**

1. **基本变量替换**

   ```
   myvar="Hello"
   echo "${myvar}"
   ```

2. **使用默认值**

   如果 `myvar` 未设置或为空，则输出 "World"。

   ```
   echo "${myvar:-World}"
   ```

3. **去除后缀**

   假设 `filename="test.tar.gz"`，以下命令将移除 `.gz` 后缀。

   ```
   echo "${filename%.gz}"
   ```

4. **字符串长度**

   获取变量值的长度。

   ```
   str="hello"
   echo "${#str}"  # 输出 5
   ```

**（2）$()**

​	`$()` 用于命令替换（Command Substitution），它会执行括号内的命令，并用该命令的输出结果替换掉 `$()` 本身。这是一种非常方便的方式，可以将一个命令的输出作为另一个命令的输入。

**示例**

1. **基本命令替换**

   获取当前目录下的文件列表并打印出来。

   ```
   files=$(ls)
   echo "$files"
   ```

2. **嵌套命令替换**

   可以嵌套使用命令替换来构建复杂的命令序列。例如，查找某个目录下最近修改的文件。

   ```
   newest_file=$(ls -t | head -n 1)
   echo "The newest file is: $newest_file"
   ```

**（3）区别与联系**

- **用途不同**：`${}` 主要用于变量操作，包括变量值的获取、修改等；而 `$()` 则用于执行命令并将结果返回给父命令。
- **灵活性**：`${}` 提供了更多的灵活性来进行字符串操作和条件判断；`$()` 简化了命令执行的结果捕获过程，使得脚本编写更加直观。
- **结合使用**：两者可以在同一个脚本中结合使用。例如，你可以先通过 `$()` 获取某个命令的结果，然后使用 `${}` 对这个结果进行进一步处理。

## sh -c：启动一个子 shell 并执行后续的命令字符串

**通过 `sh -c` 执行一个命令字符串**：**需要用引号将整个命令包裹起来**，确保 `pwd` 被解析为命令：

```
$ sh -c "echo $(pwd)"  # 先执行 `pwd`，再输出结果
/home/user

# 或者直接执行 pwd
$ sh -c "pwd"
/home/user

```

**加引号后**：`sh -c "echo $(pwd)"` 会：

- 先由当前 shell 解析 `$(pwd)`，替换为当前目录路径。
- 再执行 `sh -c "echo /home/user"`，最终输出路径

## 硬链接和软链接

文件链接（File Link）是一种特殊的文件类型，可以在文件系统中指向另一个文件。常见的文件链接类型有两种

**1、硬链接（Hard Link）**

- 在 Linux/类 Unix 文件系统中，**每个文件和目录都有一个唯一的索引节点（inode）号，用来标识该文件或目录**。硬链接通过 inode 节点号建立连接，硬链接和源文件的 inode 节点号相同，两者对文件系统来说是完全平等的（可以看作是互为硬链接，源头是同一份文件），删除其中任何一个对另外一个没有影响，可以通过给文件设置硬链接文件来防止重要文件被误删。
- 只有删除了源文件和所有对应的硬链接文件，该文件才会被真正删除。
- 硬链接具有一些限制，**不能对目录以及不存在的文件创建硬链接**，并且，硬链接也不能跨越文件系统。
- **`ln` 命令用于创建硬链接。**

**2、软链接（Symbolic Link 或 Symlink）**

**`ln -s  [目标文件] [软链接名]`**

- 软链接和源文件的 inode 节点号不同，而是指向一个文件路径。
- 源文件删除后，软链接依然存在，但是指向的是一个无效的文件路径。
- 软连接**类似于 Windows 系统中的快捷方式**。
- 不同于硬链接，可以对目录或者不存在的文件创建软链接，并且，软链接可以跨越文件系统。
- `ln -s` 命令用于创建软链接。

**（1）建立软链接eg:**

```
root@hcss-ecs-7a66 apps]# ln -s d.txt e.txt
[root@hcss-ecs-7a66 apps]# ll
total 8
-rw-r--r-- 1 root root 0 Feb 20 20:56 a.txt
-rw-r--r-- 2 root root 7 Feb 20 20:58 b.txt
-rw-r--r-- 2 root root 7 Feb 20 20:58 c.config
-rw-r--r-- 1 root root 0 Feb 20 20:58 d.txt
lrwxrwxrwx 1 root root 5 Feb 20 21:05 e.txt -> d.txt
```

其中，5字节即代表d.txt的名字有5个字节

**（2）删除软链接：**

- 使用 `unlink` 命令

```
root@hcss-ecs-7a66 apps]# ll
total 8
-rw-r--r-- 1 root root 0 Feb 20 20:56 a.txt
-rw-r--r-- 2 root root 7 Feb 20 20:58 b.txt
-rw-r--r-- 2 root root 7 Feb 20 20:58 c.config
-rw-r--r-- 1 root root 0 Feb 20 20:58 d.txt
lrwxrwxrwx 1 root root 5 Feb 20 20:59 test -> d.txt
[root@hcss-ecs-7a66 apps]# unlink test
[root@hcss-ecs-7a66 apps]# ll
total 8
-rw-r--r-- 1 root root 0 Feb 20 20:56 a.txt
-rw-r--r-- 2 root root 7 Feb 20 20:58 b.txt
-rw-r--r-- 2 root root 7 Feb 20 20:58 c.config
-rw-r--r-- 1 root root 0 Feb 20 20:58 d.txt
```

- 使用rm，注意**不要在删除命令后面加上斜杠**：如果你尝试删除一个指向目录的软链接，请勿在软链接名称后添加斜杠（`/`），因为这将导致命令尝试删除实际指向的目录内容而非软链接本身。 

```
rm path/to/symlink

root@hcss-ecs-7a66 apps]# ll
total 8
-rw-r--r-- 1 root root 0 Feb 20 20:56 a.txt
-rw-r--r-- 2 root root 7 Feb 20 20:58 b.txt
-rw-r--r-- 2 root root 7 Feb 20 20:58 c.config
-rw-r--r-- 1 root root 0 Feb 20 20:58 d.txt
lrwxrwxrwx 1 root root 5 Feb 20 21:05 e.txt -> d.txt
[root@hcss-ecs-7a66 apps]# rm e.txt
rm: remove symbolic link 'e.txt'? y 
[root@hcss-ecs-7a66 apps]# ll
total 8
-rw-r--r-- 1 root root 0 Feb 20 20:56 a.txt
-rw-r--r-- 2 root root 7 Feb 20 20:58 b.txt
-rw-r--r-- 2 root root 7 Feb 20 20:58 c.config
-rw-r--r-- 1 root root 0 Feb 20 20:58 d.txt
```

## mount命令

`mount` 命令是 Linux 系统中用于挂载文件系统的重要工具。通过挂载，可以将存储设备（如硬盘、U 盘、ISO 镜像等）或远程文件系统（如 NFS、SMB）连接到文件系统目录树中，使数据可访问

**（1）基本语法**

```
mount [选项] [设备] [挂载点]
```

- **设备**：要挂载的存储设备（如 `/dev/sdb1`）或文件（如 `.iso` 文件）。
- **挂载点**：挂载的目标目录（必须已存在）。

**（2）常用选项**

| **选项**            | **说明**                                                     |
| :------------------ | :----------------------------------------------------------- |
| `-t <文件系统类型>` | 指定文件系统类型（如 `ext4`、`ntfs`、`iso9660`、`nfs`、`cifs` 等）。 |
| `-o <挂载选项>`     | 指定挂载选项（多个选项用逗号分隔）。常见选项：               |
|                     | - `ro`/`rw`：只读/读写（默认 `rw`）。                        |
|                     | - `remount`：重新挂载（用于修改挂载选项）。                  |
|                     | - `noexec`：禁止执行挂载点中的程序。                         |
|                     | - `loop`：挂载镜像文件（如 `.iso`）。                        |
| `-a`                | 挂载 `/etc/fstab` 中定义的所有文件系统。                     |
| `-l`                | 列出已挂载的文件系统（类似 `mount` 无参数）。                |
| `-v`                | 显示详细输出（调试用）。                                     |

**（3）常见操作示例**

- **挂载物理设备**

```
# 挂载 U 盘（假设设备为 /dev/sdc1，挂载到 /mnt/usb）
sudo mount -t vfat /dev/sdc1 /mnt/usb

# 挂载 NTFS 格式的硬盘（需安装 ntfs-3g）
sudo mount -t ntfs-3g /dev/sdb1 /mnt/data
```

- 挂载ISO镜像

```
# 挂载 ISO 文件到 /mnt/iso 目录
sudo mount -o loop ubuntu-22.04.iso /mnt/iso
```

- **挂载网络文件系统**

```
# 挂载 NFS 共享
sudo mount -t nfs 192.168.1.100:/shared /mnt/nfs

# 挂载 SMB/CIFS 共享（需安装 cifs-utils）
sudo mount -t cifs -o username=user,password=pass //192.168.1.100/share /mnt/smb
```

- **卸载文件系统**

```
# 通过设备名卸载
sudo umount /dev/sdc1

# 通过挂载点卸载
sudo umount /mnt/usb
```

## Linux 文件目录、文件类型与权限

### Linux 文件目录树

Linux 使用一种称为目录树的层次结构来组织文件和目录。目录树由根目录（/）作为起始点，向下延伸，形成一系列的目录和子目录。每个目录可以包含文件和其他子目录。结构层次鲜明，就像一棵倒立的树。

![Linux的目录结构](https://raw.githubusercontent.com/GLeXios/Notes/main/pics/Linux目录树-DzjzZEII.png)

**常见目录说明：**

- **/bin：** 存放二进制可执行文件(bash、ls、cat、mkdir 等)，常用命令一般都在这里；
- **/etc：** 存放系统管理和配置文件；
- **/home：** 存放所有用户文件的根目录，是用户主目录的基点，比如用户 user 的主目录就是/home/user，可以用~user 表示；
- **/usr： 用于存放系统应用程序，usr 是 Unix Software Resource 的缩写， 也就是『Unix操作系统软件资源』所放置的目录，而不是用户的数据**
- **/opt：** 额外安装的可选应用程序包所放置的位置。一般情况下，我们可以把 tomcat 等都安装到这里；
- **/proc：** 虚拟文件系统目录，是系统内存的映射。可直接访问这个目录来获取系统信息；
- **/root：** 超级用户（系统管理员）的主目录（特权阶级^o^）；
- **/sbin:** 存放二进制可执行文件，只有 root 才能访问。这里存放的是系统管理员使用的系统级别的管理命令和程序。如 ifconfig 等；
- **/dev：** 用于存放设备文件；
- **/mnt：** 系统管理员安装临时文件系统的安装点，系统提供这个目录是让用户临时挂载其他的文件系统；
- **/boot：** 存放用于系统引导时使用的各种文件；
- **/lib 和/lib64：** 存放着和系统运行相关的库文件 ；
- **/tmp：** 用于存放各种临时文件，是公用的临时文件存储点；
- **/var：** 用于存放运行时需要改变数据的文件，也是某些大文件的溢出区，比方说各种服务的日志文件（系统启动日志等。）等；
- **/lost+found：** 这个目录平时是空的，系统非正常关机而留下“无家可归”的文件（windows 下叫什么.chk）就在这里

### 文件类型

Linux 支持很多文件类型，其中非常重要的文件类型有: **普通文件**，**目录文件**，**链接文件**，**设备文件**，**管道文件**，**Socket 套接字文件** 等。

- **普通文件（-）**：用于存储信息和数据， Linux 用户可以根据访问权限对普通文件进行查看、更改和删除。比如：图片、声音、PDF、text、视频、源代码等等。
- **目录文件（d，directory file）**：目录也是文件的一种，用于表示和管理系统中的文件，目录文件中包含一些文件名和子目录名。打开目录事实上就是打开目录文件。
- **符号链接文件（l，symbolic link）**：保留了指向文件的地址而不是文件本身。
- **字符设备（c，char）**：用来访问字符设备比如键盘。
- **设备文件（b，block）**：用来访问块设备比如硬盘、软盘。
- **管道文件(p，pipe)** : 一种特殊类型的文件，用于进程之间的通信。
- **套接字文件(s，socket)**：用于进程间的网络通信，也可以用于本机之间的非网络通信。

eg：

```
# 普通文件（-）
-rw-r--r--  1 user  group  1024 Apr 14 10:00 file.txt
# 目录文件（d，directory file）*
drwxr-xr-x  2 user  group  4096 Apr 14 10:00 directory/
# 套接字文件(s，socket)
srwxrwxrwx  1 user  group    0 Apr 14 10:00 socket
```

### 文件权限

​	操作系统中每个文件都拥有特定的权限、所属用户和所属组。权限是操作系统用来限制资源访问的机制，**在 Linux 中权限一般分为读(readable)、写(writable)和执行(executable)，分为三组**。**分别对应文件的属主(owner)，属组(group)和其他用户(other)**，通过这样的机制来限制哪些用户、哪些组可以对特定的文件进行什么样的操作。

eg：

![img](https://raw.githubusercontent.com/GLeXios/Notes/main/pics/157CE32D-EDE5-4CA0-BF8E-789C5F0973E8.png)

第一列的内容的信息解释如下：

![img](https://raw.githubusercontent.com/GLeXios/Notes/main/pics/Linux权限解读-BGnXOuG0.png)

**Linux 中权限分为以下几种：**

- r：代表权限是可读，r 也可以用数字 4 表示
- w：代表权限是可写，w 也可以用数字 2 表示
- x：代表权限是可执行，x 也可以用数字 1 表示

对文件和目录而言，读写执行表示不同的意义。

- 对于文件：

| 权限名称 |                  可执行操作 |
| -------: | --------------------------: |
|        r | 可以使用 cat 查看文件的内容 |
|        w |          可以修改文件的内容 |
|        x |    可以将其运行为二进制文件 |

- 对于目录：

| 权限名称 |               可执行操作 |
| -------: | -----------------------: |
|        r |       可以查看目录下列表 |
|        w | 可以创建和删除目录下文件 |
|        x |     可以使用 cd 进入目录 |

- **所有者(u)**：一般为文件的创建者，谁创建了该文件，就天然的成为该文件的所有者，用 `ls ‐ahl` 命令可以看到文件的所有者 也可以使用 chown 用户名 文件名来修改文件的所有者 。
- **文件所在组(g)**：当某个用户创建了一个文件后，这个文件的所在组就是该用户所在的组用 `ls ‐ahl`命令可以看到文件的所有组也可以使用 chgrp 组名 文件名来修改文件所在的组。
- **其它组(o)**：除开文件的所有者和所在组的用户外，系统的其它用户都是文件的其它组

## 文件权限命令

**（1）chmod**

修改/test 下的 aaa.txt 的权限为文件所有者有全部权限，文件所有者所在的组有读写权限，其他用户只有读的权限。

**`chmod u=rwx,g=rw,o=r aaa.txt`** 或者 **`chmod 764 aaa.txt

![img](https://raw.githubusercontent.com/GLeXios/Notes/main/pics/1EF9FC71-EBFC-48F3-99DE-42D18E7A4C13.png)

**补充一个比较常用的东西:**

假如我们装了一个 zookeeper，我们每次开机到要求其自动启动该怎么办？

- 新建一个脚本 zookeeper
- **为新建的脚本 zookeeper 添加可执行权限，命令是:`chmod +x zookeeper`**
- 把 zookeeper 这个脚本添加到开机启动项里面，命令是：`chkconfig --add zookeeper`
- 如果想看看是否添加成功，命令是：`chkconfig --list`

**（2）chgrp ：改变档案所属群组**

![img](https://raw.githubusercontent.com/GLeXios/Notes/main/pics/2EFHQHY3ACQDW.png)

**（3）改变档案拥有者, chown**

![img](https://raw.githubusercontent.com/GLeXios/Notes/main/pics/QSIXQHY3ABQEO.png)

## 用户管理

Linux 系统是一个多用户多任务的分时操作系统，任何一个要使用系统资源的用户，都必须首先向系统管理员申请一个账号，然后以这个账号的身份进入系统。

用户的账号一方面可以帮助系统管理员对使用系统的用户进行跟踪，并控制他们对系统资源的访问；另一方面也可以帮助用户组织文件，并为用户提供安全性保护。

**Linux 用户管理相关命令:**

- `useradd [选项] 用户名`:创建用户账号。使用`useradd`指令所建立的帐号，实际上是保存在 `/etc/passwd`文本文件中。

  - `-g groupname`: 指定用户的主要组。如果没有指定，默认会创建一个与用户名相同的组。

    ```
    sudo useradd -g developers john
    ```

  - **密码设置**：使用 `useradd` 创建用户后，该用户还没有密码，因此无法直接登录。你需要使用 `passwd` 命令来设置密码。

    ```
    sudo passwd john
    ```

- `userdel [选项] 用户名`:删除用户帐号。

- `usermod [选项] 用户名`:修改用户账号的属性和配置比如用户名、用户 ID、家目录。

- `passwd [选项] 用户名`: 设置用户的认证信息，包括用户密码、密码过期时间等。。例如：`passwd -S 用户名` ，显示用户账号密码信息。`passwd -d 用户名`: 清除用户密码，会导致用户无法登录。`passwd 用户名`，修改用户密码，随后系统会提示输入新密码并确认密码。

- `su - [user_name]`（su 即 Switch User，切换用户）：在当前登录的用户和其他用户之间切换身份。

  - **su**: 表示 switch user。
  - **-**: 这个连字符（或称为破折号）告诉系统不仅切换用户，还要模拟一个完全登录 shell。**这意味着它会加载目标用户的环境变量和配置文件（如 `.bashrc`, `.profile` 等），就像该用户直接登录一样**。
  - **user**: 你想切换到的目标用户名。

## 用户组管理

每个用户都有一个用户组，系统可以对一个用户组中的所有用户进行集中管理。不同 Linux 系统对用户组的规定有所不同，如 Linux 下的用户属于与它同名的用户组，这个用户组在创建用户时同时创建。

用户组的管理涉及用户组的添加、删除和修改。组的增加、删除和修改实际上就是对`/etc/group`文件的更新。

**Linux 系统用户组的管理相关命令:**

- `groupadd [选项] 用户组` :增加一个新的用户组。
- `groupdel 用户组`:要删除一个已有的用户组。
- `groupmod [选项] 用户组` : 修改用户组的属性。

## 文件搜索(grep,awk,sed,正则)

**（1）正则表达式**

**1）基础元字符**

| **符号** | **说明**                               | **示例**                  |
| :------- | :------------------------------------- | :------------------------ |
| `.`      | 匹配 **任意单个字符**（换行符除外）    | `a.c` → "abc"、"a1c"      |
| `^`      | 匹配字符串的 **开头**                  | `^Start` → 行首的 "Start" |
| `$`      | 匹配字符串的 **结尾**                  | `end$` → 行尾的 "end"     |
| `\`      | **转义字符**（将特殊符号转为普通字符） | `\.` → 匹配 "."           |

**2）量词（Quantifiers）**

| **符号** | **说明**                       | **示例**                         |
| :------- | :----------------------------- | :------------------------------- |
| `*`      | 匹配前一个元素 **0 次或多次**  | `ab*c` → "ac"、"abbc"            |
| `+`      | 匹配前一个元素 **1 次或多次**  | `ab+c` → "abc"、"abbc"（非"ac"） |
| `?`      | 匹配前一个元素 **0 次或 1 次** | `colou?r` → "color"、"colour"    |
| `{n}`    | 匹配前一个元素 **恰好 n 次**   | `a{3}` → "aaa"                   |
| `{n,}`   | 匹配前一个元素 **至少 n 次**   | `a{2,}` → "aa"、"aaaaa"          |
| `{n,m}`  | 匹配前一个元素 **n 到 m 次**   | `a{2,4}` → "aa"、"aaa"、"aaaa"   |

**3）字符类（Character Classes）**

| **符号**      | **说明**                            | **示例**                   |
| :------------ | :---------------------------------- | :------------------------- |
| `[abc]`       | 匹配 **a、b 或 c** 中的任意一个字符 | `gr[ae]y` → "gray"、"grey" |
| `[^abc]`      | 匹配 **非 a、b、c** 的任意字符      | `[^0-9]` → 匹配非数字字符  |
| `[a-z]`       | 匹配 **小写字母 a 到 z**            | `[a-f]` → a、b、c、d、e、f |
| `[A-Za-z0-9]` | 匹配 **字母或数字**                 |                            |

**（2）`grep [选项] "搜索模式" 文件名`**

**1）常用选项**

| **选项**       | **说明**                                       |
| :------------- | :--------------------------------------------- |
| `-i`           | 忽略大小写（Case-insensitive）                 |
| `-v`           | 反向匹配（显示 **不包含** 模式的行）           |
| `-n`           | 显示匹配行的行号                               |
| `-r` 或 `-R`   | 递归搜索目录下的所有文件                       |
| `-l`           | 仅显示包含匹配结果的文件名（不显示具体行）     |
| `-c`           | 统计匹配的行数（而非显示具体内容）             |
| `-A <数字>`    | 显示匹配行及其后 N 行（After）                 |
| `-B <数字>`    | 显示匹配行及其前 N 行（Before）                |
| `-C <数字>`    | 显示匹配行及其前后各 N 行（Context）           |
| `-E`           | 启用扩展正则表达式（相当于 `egrep`）           |
| `-w`           | 全词匹配（匹配完整单词）                       |
| `--color=auto` | 高亮显示匹配内容（默认已开启，部分系统需配置） |
| `-o`           | 只输出文件中匹配到的部分                       |
| `-q`           | --quiet或--silent     # 不显示任何信息。       |

**2）例子**

- **搜索文件中包含关键字的行**

```
grep "error" /var/log/syslog          # 查找 "error" 关键字
grep -i "warning" app.log             # 忽略大小写查找 "warning"
```

- **递归搜索目录下的所有文件**

```
grep -r "TODO" /path/to/project/      # 在所有文件中搜索 "TODO"
grep -rl "password" /etc/             # 仅列出包含 "password" 的文件名
```

- **反向匹配（排除特定内容）**

```
grep -v "#" config.conf               # 显示不包含注释符 "#" 的行
```

- **统计匹配结果数量**

```
grep -c "GET" access.log             # 统计 "GET" 请求出现的次数
```

- **显示上下文内容**

```
grep -C 3 "crash" app.log            # 显示匹配行及其前后各 3 行
```

- **结合管道符过滤命令输出**

```
ps aux | grep "nginx"                 # 查找 Nginx 进程
history | grep "ssh"                  # 搜索历史命令中的 "ssh"
```

- 只输出文件中匹配到的部分 **-o** 选项：

```
echo this is a test line. | grep -o -E "[a-z]+\."
line.
```

**（3）awk**





## 目录操作(find cp mv等)

（1）`ll -ah`：`ll` 是 `ls -l` 的别名，ll 命令可以看到该目录下的所有目录和文件的详细信息，**-a：全部信息，-h：按人类信息**

（2）`mkdir [选项] 目录名`：创建新目录（增）。

- 例如：**`mkdir -m 755 my_directory`**，创建一个名为 `my_directory` 的新目录，并将其权限设置为 755，即所有用户对该目录有读、写和执行的权限。**-m**：此选项允许你在创建目录的同时指定其权限模式。

（3）**`find [路径] [表达式]`**：在指定目录及其子目录中搜索文件或目录（查），非常强大灵活。

- 列出当前目录及子目录下所有文件和文件夹: `find .`

- 在`/home`目录下查找以 `.txt` 结尾的文件名:**`find /home -name "\*.txt"`**，忽略大小写: `find /home -i name "*.txt"`

- 当前目录及子目录下查找所有以 `.txt` 或者 `.pdf` 结尾的文件:`find . -name "*.txt" -o -name "*.pdf"`。

- **查找并删除所有 `.tmp` 文件**（**这里的 `{}` 是一个占位符，代表找到的每一个文件名；`\;` 表示 `-exec` 命令的结束。**）：

  ```
  find . -name "*.tmp" -exec rm {} \;
  ```

- **限制搜索深度**：使用 `-maxdepth` 来限制搜索的最大层级

  ```
  find . -maxdepth 2 -name "*.txt" # 只在当前目录及下一级子目录中查找 .txt 文件
  ```

（4）`rm [选项] 文件或目录名`：删除文件/目录（删）。例如：`rm -r my_directory`，删除名为 `my_directory` 的目录，`-r`(recursive,递归) 表示会递归删除指定目录及其所有子目录和文件。

**（5）`cp [选项] 源文件/目录 目标文件/目录`：复制文件或目录（移）。**

- `cp file.txt /home/file.txt`，将 `file.txt` 文件复制到 `/home` 目录下，并重命名为 `file.txt`。
- `cp -r source destination`，将 `source` 目录及其下的所有子目录和文件复制到 `destination` 目录下，并保留源文件的属性和目录结构。

**（6）`mv [选项] 源文件/目录 目标文件/目录`：移动文件或目录（移），也可以用于重命名文件或目录。**

- `mv file.txt /home/file.txt`，将 `file.txt` 文件移动到 `/home` 目录下，并重命名为 `file.txt`。
- `mv file.txt test.txt`，将file.txt重命名为test.txt。
- `mv` 与 `cp` 的结果不同，`mv` 好像文件“搬家”，文件个数并未增加。而 `cp` 对文件进行复制，文件个数增加了。

## 文件操作

（1）`touch [选项] 文件名..`：创建新文件或更新已存在文件（增）

- `touch file1.txt file2.txt file3.txt` ，创建 3 个文件。

（2）`ln [选项] <源文件> <硬链接/软链接文件>`：创建硬链接/软链接。

- `ln -s file.txt file_link`，创建名为 `file_link` 的软链接，指向 `file.txt` 文件。`-s` 选项代表的就是创建软链接，s 即 symbolic（软链接又名符号链接）

（3）`vim 文件名`：修改文件的内容（改）。vim 编辑器是 Linux 中的强大组件，是 vi 编辑器的加强版。

- 在实际开发中，使用 vim 编辑器主要作用就是修改配置文件，下面是一般步骤：`vim 文件------>进入文件----->命令模式------>按i进入编辑模式----->编辑文件 ------->按Esc进入底行模式----->输入：wq/q!` （输入 wq 代表写入内容并退出，即保存；输入 q!代表强制退出不保存）。
- **在底行模式下shift+4跳转到行尾，shift+6跳转到行首**

(4)`cat/more/less/tail 文件名`：文件的查看（查） 。

- 命令 `tail -f 文件` 可以对某个文件进行动态监控 :`tail -f catalina-2016-11-11.log` 监控文件的变化 

## 文件压缩

**（1）打包并压缩文件：**

Linux 中的打包文件一般是以 `.tar` 结尾的，压缩的命令一般是以 `.gz` 结尾的。而一般情况下打包和压缩是一起进行的，打包并压缩后的文件的后缀名一般 `.tar.gz`。

**1）命令**：**`tar -zcvf [打包压缩后的文件名] [要打包压缩的文件或路径]`** ，其中：

- z：调用 gzip 压缩命令进行压缩
- c：打包文件
- v：显示运行过程
- f：指定文件名

eg：假如 test 目录下有三个文件分别是：`aaa.txt`、 `bbb.txt`、`ccc.txt`，如果我们要打包 `test` 目录并指定压缩后的压缩包名称为 `test.tar.gz` 可以使用命令：**`tar -zcvf test.tar.gz aaa.txt bbb.txt ccc.txt` 或 `tar -zcvf test.tar.gz /test/`** 。

**2）`zip [options] zipfile file...`**

- 压缩整个目录（包括其子目录）：

  ```
  zip -r archive.zip directory_name
  ```



**（2）解压压缩包：**

1）**命令：`tar [-xvf] 压缩文件`**

其中 x 代表解压

- 将 `/test` 下的 `test.tar.gz` 解压到当前目录下可以使用命令：`tar -xvf test.tar.gz`
- 将 /test 下的 test.tar.gz 解压到根目录/usr 下:`tar -xvf test.tar.gz -C /usr`（`-C` 代表指定解压的位置）

2）`unzip 压缩文件`

虽然 `unzip` 是专门用于解压 `.zip` 文件的工具，但有时你可能会看到使用 `zip` 的相关讨论。实际上，解压通常通过以下命令完成：

```
unzip archive.zip
```

如果只想查看 `.zip` 文件的内容而不解压，可以使用：

```
unzip -l archive.zip
```





## 文件传输

**（1）`scp [option] source target` （scp 即 secure copy，安全复制）**

- **source**: 指定要复制的源文件或目录。它可以是本地路径，也可以是远程路径。
- **target**: 指定文件或目录的目标位置。同样，这可以是本地路径或远程路径。
- **[option]**: 提供额外的选项以控制传输行为，例如指定端口、限制带宽等。

用于通过 SSH 协议进行安全的文件传输，可以实现从本地到远程主机的上传和从远程主机到本地的下载。

- `scp -r my_directory user@remote_host:/home/user` ，将本地目录`my_directory`上传到远程服务器 `/home/user` 目录下。

- `scp -r user@remote_host:/home/user/ my_directory` ，将远程服务器的 `/home/user` 目录下的`my_directory`目录下载到本地。

- 需要注意的是，`scp` 命令需要在本地和远程系统之间建立 SSH 连接进行文件传输，因此需要确保远程服务器已经配置了 SSH 服务，并且具有正确的权限和认证方式。

- `-P port`: 指定**远程主机的 SSH 端口**（注意大写的 P）。默认情况下，`scp` 使用的是 22 端口。**user指在目标主机（即 `remote_host`）上的用户名**

  ```
  scp -P 2222 file.txt user@remote_host:/home/user/
  ```

**（2）`rsync [选项] 源文件 远程文件` :** 可以在本地和远程系统之间高效地进行文件复制，并且能够智能地处理增量复制，节省带宽和时间。

- `rsync -r my_directory user@remote:/home/user`，将本地目录`my_directory`上传到远程服务器 `/home/user` 目录下。

## Linux 重定向

重定向命令用于控制 Linux 中的输入和输出源，让你可以向文件发送和追加输出流、从文件获取输入、连接多个命令以及将输出分割到多个目的地。

**（1）`>` 重定向标准输出**

重定向操作符 `>` 将命令的标准输出流重定向到文件，而不是打印到终端。**文件中的任何现有内容都将被覆盖**

```
ls -l /home > homelist.txt
```

- 这将执行 `ls -l` ，列出 /home 目录的内容。
- 然后，” `>` “符号将捕获标准输出并写入 homelist.txt，覆盖现有文件内容，而不是将输出打印到终端。

**（2）** **`>>`** **追加标准输出**

**`>>` 操作符将命令的标准输出追加到文件中，而不覆盖现有内容。**

```
tail /var/log/syslog >> logfile.txt
```

这将把 syslog 日志文件的最后 10 行追加到 logfile.txt 的末尾。与 `>` 不同， `>>` 添加输出时不会擦除当前 logfile.txt 的内容。

**（3）** **`<`** **重定向标准输入**

`<` 重定向操作符将文件内容作为标准输入送入命令，而不是从键盘输入。

```
wc -l < myfile.txt
```

该命令将 myfile.txt 的内容作为输入发送给 wc 命令，wc 命令将计算该文件的行数，而不是等待键盘输入。

**（4）** **`|`** **管道输出到另一条命令**

管道 `|` 操作符**将一条命令的输出作为输入发送到另一条命令**，将它们串联起来。

```
ls -l | less
```

该命令将 `ls -l` 的输出导入 less 命令，从而可以滚动浏览文件列表。

管道通常用于将命令串联起来，其中一个命令的输出为另一个命令的输入提供信息。这样就能从较小的单用途程序中构建出复杂的操作。





## 磁盘和内存管理

**（1）`df -h`** ：显示系统中文件系统的磁盘空间使用情况，并且以人类可读的格式（如 MB、GB、TB 等）展示。

- -h：以人类可读的方式显示磁盘空间单位（比如 MB、GB、TB 等）。
- -T：显示文件系统的类型。
- -i：显示 inode 的使用情况，而不是块的使用情况。
- -a：包括所有的文件系统，包括特殊的、伪文件系统（如 proc 和 sysfs）等。
- -t <filesystem_type>：仅显示指定类型的文件系统。

```
Filesystem      Size  Used Avail Use% Mounted on
/dev/sda1        50G   20G   28G  42% /
tmpfs           1.9G     0  1.9G   0% /dev/shm
/dev/sda2       100G   60G   40G  60% /data
tmpfs           396M   12M  384M   3% /run
```

1. ****Filesystem\****：文件系统的名称，通常是设备名称或网络挂载的路径。
2. ****Size\****：文件系统的总大小（以人类可读的单位显示，如 GB、TB）。
3. ****Used\****：已经使用的空间。
4. ****Avail\****：剩余可用的空间。
5. ****Use%\****：已使用的磁盘空间百分比。
6. ****Mounted on\****：**文件系统挂载的路径，即该文件系统关联的目录。**

**解释：**

- /dev/sda1:
  - Filesystem: /dev/sda1 表示第一个磁盘（sda）的第一个分区。
  - Mounted on: /，即这个分区被挂载为根文件系统。
- tmpfs:
  - Filesystem: tmpfs 是一个基于内存的虚拟文件系统。
  - Mounted on: /dev/shm，该挂载点是用于共享内存的临时文件存储区域。
- /dev/sda2:
  - Filesystem: /dev/sda2 表示第一个磁盘的第二个分区。
  - Mounted on: /data，这意味着 /dev/sda2 上的数据被挂载在 /data 目录中，可以通过该目录访问数据。

注意：

- **mount on的位置可以理解为你在linux上面命令操作时候需要将文件存储到哪里**
- **FileSystem理解为实际的存储位置,这个位置是在linux上面cd无法进去的,这个是你的挂载的磁盘的位置**

**（2）`du [选项] [文件]`：**用于查看指定目录或文件的磁盘空间使用情况，可以指定不同的选项来控制输出格式和单位。

- 查看当前/home目录下各个文件及目录占用空间大小
  - 使用 `--max-depth=N` 参数可以限制显示的目录层级深度
  - `--max-depth=1`：这会在当前目录下显示所有直接子目录的大小，但不会递归进入更深层次的子目录。

```
du -h --max-depth=1 /home
```

- 只对某个目录或文件的总磁盘使用量感兴趣，可以加上 `-s` 选项（summarize）

  ```
  du -sh /var/log
  ```

只会输出 `/var/log` 目录的总大小，而不是详细列出每个子目录和文件的大小。

**（3）`free [选项]`：**用于查看系统的内存使用情况，包括已用内存、可用内存、缓冲区和缓存等。

- free -m/g：**free -m是以m形式显示,free -g以g形式显示**

![img](https://raw.githubusercontent.com/GLeXios/Notes/main/pics/JRI4IYQ3AAAAG.png)

- 这个一般只需要看三项,total,used,和free,这里给个建议,free最好占总内存的15%最好,也就是你别把人家内存都给人家用光了.
- **Swap这个参数意思是因为内存不够,可以使用部分磁盘空间来充当内存使用,**虽然可以解决内存紧张的问题,但是慢,毕竟磁盘读写肯定没有内存块,而且在大数据环境下,所以这个是否配置要看公司架构师决定了.

## 系统状态与进程

**（1）`top [选项]`**：用于实时查看系统的 CPU 使用率、内存使用率、进程信息等。

- `top` 命令显示实时 Linux 进程信息，包括 PID、用户、CPU %、内存使用率、运行时间等。与 `ps` 不同的是，它会动态更新显示内容，以反映当前的使用情况。

- ```
  top -u mysql
  ```

  上述命令只监控 “[mysql](https://link.zhihu.com/?target=https%3A//www.wbolt.com/what-is-mysql.html)” 用户的进程。它对识别资源密集型程序很有帮助。

**（2）`ps [选项]`：**用于查看系统中的进程信息，包括进程的 ID、状态、资源使用情况等。

- `ps -ef`/`ps aux`：这两个命令都是查看当前系统正在运行进程，两者的区别是展示格式不同。如果想要查看特定的进程可以使用这样的格式：`ps aux|grep redis` （查看包括 redis 字符串的进程

对于ps -ef命令输出的解释：

![image-20250220232410357](https://raw.githubusercontent.com/GLeXios/Notes/main/pics/image-20250220232410357.png)

- **UID**: 启动该进程的用户 ID 或用户名。这有助于识别哪个用户启动了特定的进程。
- **PID**: 进程 ID。每个进程在系统中都有一个唯一的进程 ID，这对于管理和操作进程非常重要。
- **PPID**: 父进程 ID。表示启动此进程的父进程的 ID。如果一个进程是由另一个进程创建的，则这里的值会指向其父进程的 PID。
- **C**: CPU 使用率或处理器利用率。这个数值可以是最近重新计算的平均使用率，也可以是进程当前使用的 CPU 百分比。
- **STIME**: 启动时间。表示进程开始运行的时间点。对于长时间运行的系统进程来说，可能是系统的启动时间；而对于较新的进程，则可能是具体的时间（如 `HH:MM` 或者更详细的日期和时间）。
- **TTY**: 控制终端。表示与进程相关的终端设备名称。如果是一个守护进程或没有控制终端的进程，这里通常显示为 `?`。
- **TIME**: 累计的 CPU 时间。表示进程自启动以来所占用的总 CPU 时间，包括在用户空间和内核空间消耗的时间。
- **CMD**: 启动该进程的命令及其参数。这是实际执行的命令行，可以帮助你识别进程的功能或目的。

**eg：streamis进程**

```
[hadoop@gz ~]$ ps -ef | grep streamis
hadoop   57995     1  0  2024 ?        06:58:24 java -Xms3G -Xmx3G -XX:+UseG1GC -XX:MaxPermSize=500m -Xloggc:/data/bdp/logs/streamis/streamis-server-2024112712-gc.log -XX:+PrintGCDateStamps -XX:+HeapDumpOnOutOfMemoryError -XX:HeapDumpPath=/data/bdp/logs/streamis/streamis-server-2024112712-dump.log -XX:ErrorFile=/data/bdp/logs/streamis/streamis-server-2024112712-hs_err.log -javaagent:/appcom/Install/IastInstall/vulhunter.jar=app.full.name=BDP-STREAMIS,flag=iast,vul.log.level=INFO -cp /appcom/Install/streamis-server/conf:/appcom/Install/streamis-server/lib/* com.webank.wedatasphere.streamis.server.boot.StreamisServerApplication
```

**基础信息**

- **启动用户**：`hadoop`（以普通用户身份运行，非 root）
- **进程 PID**：`57995`
- **启动时间**：`2024年`（具体日期未显示，可能为长期运行的服务）
- **主类**：`com.webank.wedatasphere.streamis.server.boot.StreamisServerApplication`

**JVM 内存与 GC 配置**

| **参数**               | **说明**                                                     |
| :--------------------- | :----------------------------------------------------------- |
| `-Xms3G -Xmx3G`        | 初始堆内存和最大堆内存均为 3GB，避免堆动态调整带来的性能波动。 |
| `-XX:+UseG1GC`         | 使用 G1 垃圾回收器，适合大内存、低延迟场景。                 |
| `-XX:MaxPermSize=500m` | 设置永久代（Permanent Generation）最大为 500MB（**仅对 Java 7 及以下有效**，Java 8+ 已移除永久代，改用元空间 `Metaspace`）。 |

**日志与故障排查**

| **参数**                                                     | **说明**                                                     |
| :----------------------------------------------------------- | :----------------------------------------------------------- |
| `-Xloggc:/data/bdp/logs/streamis/streamis-server-2024112712-gc.log` | 将 GC 日志输出到指定文件，文件名包含时间戳 `2024112712`（可能是启动时间）。 |
| `-XX:+PrintGCDateStamps`                                     | 在 GC 日志中打印日期时间戳，便于分析。                       |
| `-XX:+HeapDumpOnOutOfMemoryError`                            | 在内存溢出（OOM）时生成堆转储文件。                          |
| `-XX:HeapDumpPath=.../streamis-server-...-dump.log`          | 指定堆转储文件的路径。                                       |
| `-XX:ErrorFile=.../streamis-server-...-hs_err.log`           | 指定 JVM 致命错误日志（如 Crash）的路径。                    |

**关键路径与配置**

| **路径**                                    | **用途**                               | **注意事项**                       |
| :------------------------------------------ | :------------------------------------- | :--------------------------------- |
| `/data/bdp/logs/streamis/`                  | 存放 Streamis 服务的日志文件。         | 需确保目录存在且有写入权限。       |
| `/appcom/Install/streamis-server/conf/`     | 配置文件目录（如 `application.yml`）。 | 配置数据库连接、端口号等关键参数。 |
| `/appcom/Install/streamis-server/lib/*`     | 依赖的第三方 JAR 包。                  | 版本需与 Streamis 兼容，避免冲突。 |
| `/appcom/Install/IastInstall/vulhunter.jar` |                                        |                                    |

**（3）`jps -l | grep StreamisServerApplication`**

- `jps` 是 JDK 自带的一个工具，用来显示当前用户的 Java 进程信息（如果需要查看所有用户的Java进程，需要使用 `sudo` 来运行 `jps`）。
- `-l` 选项表示输出应用程序的主类的完整包名或者应用程序的jar文件的完整路径。
- **`-v` (JVM flags)**:显示传递给 JVM 的标志（即启动时指定的JVM参数）。这对于调试JVM配置非常有用。

## Linux 网络命令

这些命令用于监控连接、排除网络故障、路由选择、DNS 查询和接口配置。

（1）**`ping`** **– 向网络主机发送 ICMP ECHO_REQUEST**

- `-c [count]` – 限制发送的数据包。
- `-i [interval]` – ping 之间的等待间隔秒数。

使用上述命令，你可以 ping [http://google.com](https://link.zhihu.com/?target=http%3A//google.com)，并输出显示连接性和延迟的往返统计信息。一般来说， `ping` 命令用于检查你试图连接的系统是否存在并已连接到网络。

**（2）`telnet [选项] [主机名或IP地址] [端口]`**：测试端口连通性

```bash
telnet example.com 80
```

- **作用**：检查目标主机的指定端口是否开放。
  - 连接成功：显示空白或服务欢迎信息（如 HTTP 服务的响应）。
  - 连接失败：显示 `Connection refused` 或超时。

**（3）`curl命令`**

**1）发起 GET 请求：**

```
curl https://api.example.com/data
```

2）**指定 HTTP 方法**

```bash
# -X：指定请求方法（如 POST、PUT、DELETE）
curl -X POST https://api.example.com/items
```

**3）自定义请求头**

```bash
# -H：添加请求头
curl -H "User-Agent: MyApp/1.0" -H "Accept-Language: en-US" https://example.com
```

4）**上传文件（multipart/form-data）**

```bash
# -F：上传文件（键值对）
curl -F "file=@/path/to/file.txt" https://example.com/upload
```

**5）发送表单数据（POST）**

```bash
# -d：发送表单数据（默认 Content-Type: application/x-www-form-urlencoded）
curl -d "username=admin&password=123456" https://api.example.com/login
```

**6）发送 JSON 数据**

```bash
curl -H "Content-Type: application/json" \
     -d '{"username":"admin","password":"123456"}' \
     https://api.example.com/login
```

**eg：**

```bash
curl -X POST \
     -H "Content-Type: application/json" \
     -d '{"key":"value"}' \
     https://api.example.com/endpoint
```

**==（4）`netstat`（Network Statistics）==**

 Linux/Unix 系统中用于监控网络连接、路由表、接口统计等信息的命令行工具。虽然部分新系统推荐使用 `ss` 或 `ip` 替代，但 `netstat` 仍然广泛用于快速诊断网络问题。

```bash
netstat [选项]
```

**1）常用选项**：**主要tlnp**

| **选项** | **作用**                                              |
| -------- | ----------------------------------------------------- |
| `-a`     | 显示所有连接（包括监听和非监听）。                    |
| `-t`     | 仅显示 TCP 协议相关连接。                             |
| `-u`     | 仅显示 UDP 协议相关连接。                             |
| `-l`     | 仅显示监听（LISTEN）状态的端口。                      |
| `-n`     | 以数字形式显示地址和端口（禁用域名和服务名解析）。    |
| `-p`     | 显示进程的 PID 和程序名（需 root 权限查看所有进程）。 |
| `-r`     | 显示路由表信息。                                      |
| `-s`     | 按协议统计网络数据（如 TCP、UDP 包数量）。            |
| `-c`     | 持续输出（动态刷新）。                                |

**2）输出字段解析**

以 `netstat -tulnp` 的典型输出为例：

```bash
Proto Recv-Q Send-Q Local Address  Foreign Address  State    PID/Program name
tcp   0      0      0.0.0.0:80    0.0.0.0:*        LISTEN   1234/nginx
tcp6  0      0      :::22          :::*             LISTEN   5678/sshd
```

- **Proto**：协议类型（如 `tcp`、`udp`）。
- **Recv-Q/Send-Q**：接收/发送队列中的数据量。
- **Local Address**：本地地址和端口（`0.0.0.0:80` 表示监听所有网卡的 80 端口）。
- **Foreign Address**：远程地址和端口（`*:*` 表示未建立连接）。
- **State**：连接状态（如 `LISTEN`、`ESTABLISHED`、`TIME_WAIT`）。
- **PID/Program name**：占用端口的进程 ID 和程序名。

**3）常见问题排查**

- 查找占用 80 端口的进程。

```bash
sudo netstat -tulnp | grep :80
```

注意：

- 有进程不一定有端口号,即有pid号不一定有对应的端口号
- 服务之间的通讯就是ip+端口号
- 检测一个服务通不通,通过t**elnet ip 端口号**

==**4）LISTEN和ESTABLISHED**==

**1️⃣LISTEN**

- **含义**：
  -  表示服务器端的某个端口处于 **开放并等待连接** 的状态。
  -  该状态下的端口会持续监听客户端的连接请求（`SYN` 包）。

- **场景**：

  - Web 服务器（如 80 端口）等待 HTTP 请求。

  - 数据库服务（如 3306 端口）等待客户端连接。

```bash
# 查看所有处于 LISTEN 状态的端口
$ netstat -tuln | grep LISTEN
tcp6   0   0 :::80      :::*        LISTEN
```

**2️⃣ESTABLISHED**

- **含义**：
  -  表示客户端和服务器之间的 TCP 连接 **已经成功建立**，双方可以开始传输数据。

- **场景**：

  - 用户浏览器与 Web 服务器正在通信。

  - 客户端与数据库正在交互数据。

```bash
# 查看所有已建立的连接
$ netstat -tn | grep ESTABLISHED
tcp   0   0 192.168.1.100:54321   203.0.113.5:80      ESTABLISHED
```

**3️⃣核心区别**

| **特性**         | `LISTEN`                             | `ESTABLISHED`                       |
| ---------------- | ------------------------------------ | ----------------------------------- |
| **角色**         | 服务端                               | 客户端和服务端                      |
| **通信阶段**     | 等待连接请求                         | 已建立连接，正在通信                |
| ==**典型端口**== | ==服务端固定端口（如 80、443、22）== | ==客户端随机高位端口 + 服务端端口== |
| **数据流动**     | 无数据交换                           | 双向数据传输                        |

**4️⃣图解 TCP 状态流转**

```bash
客户端                             服务端
  |---- SYN -------------------->|   LISTEN
  |<---- SYN-ACK ---------------|   SYN_RECEIVED
  |---- ACK -------------------->|   ESTABLISHED
  |<===== 数据传输 =============>|   ESTABLISHED
```

**（5）ss 基本跟netstat一样**

以 `ss -tnp` 的典型输出为例：

```bash
State   Recv-Q  Send-Q  Local Address:Port  Peer Address:Port
ESTAB   0       0       192.168.1.100:22    192.168.1.200:54321  users:(("sshd",pid=1234,fd=3))
```

- **State**：连接状态（如 `ESTAB`、`LISTEN`、`TIME-WAIT`）。
- **Recv-Q/Send-Q**：接收/发送队列中的数据量。
- **Local Address:Port**：本地地址和端口。
- **Peer Address:Port**：远程地址和端口。
- **users**：关联的进程信息（程序名、PID、文件描述符）。

# Shell编程

## shell脚本构成

(1)新建一个文件 [helloworld.sh](http://helloworld.sh) :`touch helloworld.sh`，扩展名为 sh（sh 代表 Shell）（扩展名并不影响脚本执行，见名知意就好，如果你用 php 写 shell 脚本，扩展名就用 php 好了）

(2) 使脚本具有执行权限：`chmod +x helloworld.sh`

(3) 使用 vim 命令修改 [helloworld.sh](http://helloworld.sh) 文件：`vim helloworld.sh`(vim 文件------>进入文件----->命令模式------>按 i 进入编辑模式----->编辑文件 ------->按 Esc 进入底行模式----->输入:wq/q! （输入 wq 代表写入内容并退出，即保存；输入 q!代表强制退出不保存。））

[helloworld.sh](http://helloworld.sh) 内容如下：shell 中 # 符号表示注释。

```bash
#!/bin/bash
#第一个shell小程序,echo 是linux中的输出命令。
echo  "helloworld!"
```

- ==**shell 的第一行比较特殊，一般都会以#!开始来指定使用的 shell 类型。**==
- **在 linux 中，除了 bash shell 以外，还有很多版本的 shell， 例如 zsh、dash 等等...不过 bash shell 还是我们使用最多的。**

(4) 运行脚本:`./helloworld.sh` 。

（注意，一定要写成 `./helloworld.sh` ，而不是 `helloworld.sh` ，运行其它二进制的程序也一样，直接写 `helloworld.sh` ，linux 系统会去 PATH 里寻找有没有叫 [helloworld.sh](http://helloworld.sh) 的，而只有 /bin, /sbin, /usr/bin，/usr/sbin 等在 PATH 里，你的当前目录通常不在 PATH 里，所以写成 `helloworld.sh` 是会找不到命令的，要用`./helloworld.sh` 告诉系统说，就在当前目录找。）

## linux里source、sh、bash、./执行脚本的区别

**（1）`source a.sh`与`. a,sh`等价** ：在**当前shell内**去读取、执行a.sh，而a.sh不需要有"**执行权限**"

```bash
source a.sh
```

**==source命令可以简写为"."==**

```bash
. a.sh
```

- source(或点)命令通常用于重新执行刚修改的初始化文档。
- source命令(从 C Shell 而来)是bash shell的内置命令。
- 点命令，就是个点符号，(从Bourne Shell而来)

**（2）`sh/bash a.sh`**：都是**打开一个subshell**去读取、执行a.sh，而a.sh不需要有"**执行权限**"

```bash
sh a.sh
bash a.sh
```

- **==通常在subshell里运行的脚本里设置变量，不会影响到父shell的。==**
- ==注意：`sh` 和 `bash` 的语法不完全兼容（例如 `sh` 不支持 `[[ ]]` 和数组）。若脚本写的是 Bash 语法，用 `sh` 执行会报错。==
- 每个shell脚本有效地运行在父shell(parent shell)的一个子进程里；这个父shell是指在一个控制终端或在一个xterm窗口中给你命令指示符的进程；shell脚本也可以启动他自已的子进程；这些子shell(即子进程)使脚本并行地，有效率地地同时运行脚本内的多个子任务.

**（3）`./a.sh`**：打开一个subshell**去读取、执行a.sh，但a.sh需要有"**执行权限"

```bash
./a.sh
#bash: ./a.sh: 权限不够
chmod +x a.sh
./a.sh
```

- **同sh/bash，通常在subshell里运行的脚本里设置变量，不会影响到父shell的。**

**（4）fork，source和exec**

- **`fork ( /directory/script.sh)`**
  - fork是最普通的, 就是直接在脚本里面用/directory/script.sh来调用script.sh这个脚本.运行的时候开一个sub-shell执行调用的脚本，sub-shell执行的时候, parent-shell还在。sub-shell执行完毕后返回parent-shell. sub-shell从parent-shell继承环境变量.但是sub-shell中的环境变量不会带回parent-shell、
- **`source (source /directory/script.sh)`**  
  - 与fork的区别是不新开一个sub-shell来执行被调用的脚本，而是在同一个shell中执行. 所以被调用的脚本中声明的变量和环境变量, 都可以在主脚本中得到和使用.
- **`exec (exec /directory/script.sh)`**
  - exec与fork不同，不需要新开一个sub-shell来执行被调用的脚本. 被调用的脚本与父脚本在同一个shell内执行。但是使用exec调用一个新脚本以后, 父脚本中exec行之后的内容就不会再执行了。这是exec和source的区别。

## Shell 变量

**Shell 编程中一般分为三种变量：**

1. **我们自己定义的变量（自定义变量）:** 仅在当前 Shell 实例中有效，其他 Shell 启动的程序不能访问局部变量。
2. **Linux 已定义的环境变量**（环境变量， 例如：`PATH`, `HOME` 等..., 这类变量我们可以直接使用），==使用 **`env`** 命令可以查看所有的环境变量==，而 set 命令既可以查看环境变量也可以查看自定义变量。
3. **Shell 变量**：Shell 变量是由 Shell 程序设置的特殊变量。Shell 变量中有一部分是环境变量，有一部分是局部变量，这些变量保证了 Shell 的正常运行

**常用的环境变量:**

> PATH 决定了 shell 将到哪些目录中寻找命令或程序
> HOME 当前用户主目录
> HISTSIZE 　历史记录数
> LOGNAME 当前用户的登录名
> HOSTNAME 　指主机的名称
> SHELL 当前用户 Shell 类型
> LANGUAGE 　语言相关的环境变量，多语言可以修改此环境变量
> MAIL 　当前用户的邮件存放目录
> PS1 　基本提示符，对于 root 用户是#，对于普通用户是$

## Shell 字符串

**（1）单引号与双引号**

==**字符串可以用单引号，也可以用双引号**==

- 单引号中所有的特殊符号，如$和反引号都没有特殊含义。

- 在双引号中，除了"$"、"\\"、反引号\`\`和感叹号（需开启 `history expansion`），其他的字符没有特殊含义。
- **==反引号与$()差不多，但是需要\\转义，推荐使用$()==**

eg：

**单引号字符串：**

```bash
#!/bin/bash
name='SnailClimb'
hello='Hello, I am $name!'
echo $hello
```

输出内容：

```
Hello, I am $name!
```

**双引号字符串：**

```bash
#!/bin/bash
name='SnailClimb'
hello="Hello, I am $name!"
echo $hello
```

输出内容：

```
Hello, I am SnailClimb!
```

**（2）拼接字符串**

- 使用双引号，内部使用{}


```bash
name="Shell"
url="http://c.biancheng.net/shell/"
str5="${name}Script: ${url}index.html"  # str5 的值为 "ShellScript: http://c.biancheng.net/shell/index.html"
```

**（3）获取字符串长度**

```bash
#!/bin/bash
#获取字符串长度
name="SnailClimb"
# 第一种方式
echo ${#name} #输出 10
# 第二种方式
expr length "$name";
```

```
10
10
```

使用 expr 命令时，表达式中的运算符左右必须包含空格，如果不包含空格，将会输出表达式本身:

```
expr 5+6    // 直接输出 5+6
expr 5 + 6       // 输出 11
```

**（4）截取子字符串:**

简单的字符串截取：

```bash
#从字符串第 1 个字符开始往后截取 10 个字符
str="SnailClimb is a great man"
echo ${str:0:10} #输出:SnailClimb
```

根据表达式截取：

```bash
#!bin/bash
#author:amau

var="https://www.runoob.com/linux/linux-shell-variable.html"
# %表示删除从后匹配, 最短结果
# %%表示删除从后匹配, 最长匹配结果
# #表示删除从头匹配, 最短结果
# ##表示删除从头匹配, 最长匹配结果
# 注: *为通配符, 意为匹配任意数量的任意字符
s1=${var%%t*} #h
s2=${var%t*}  #https://www.runoob.com/linux/linux-shell-variable.h
s3=${var%%.*} #https://www
s4=${var#*/}  #/www.runoob.com/linux/linux-shell-variable.html
s5=${var##*/} #linux-shell-variable.html
```

## Shell 数组

**Bash支持索引数组和关联数组，而其他Shell如sh可能只支持索引数组，同时关联数组在Bash 4.0以上才支持**

**1. 数组类型，定义与初始化**

- **索引数组（Indexed Array）**
  使用整数作为下标（默认从 `0` 开始），元素按顺序存储，分隔符为空格

```bash
  arr=(1 "two" 3)
  
  arr[0]="a"
  arr[1]="b"
  
  arr=("item 1" "item 2")  # 正确
  arr=(item 1 item 2)      # 错误！会被拆分为 4 个元素
```

- **关联数组（Associative Array）**
  使用字符串作为键（Key），类似其他语言的字典（仅 Bash 4.0+ 支持）。

```bash
  #初始化同时赋值
  declare -A map=(["name"]="Alice" ["age"]=30)
  
  declare -A user
  user["name"]="Bob"
  user["id"]="1001"
```

**3. 访问数组元素**

**（1）获取元素**

- 索引数组：

```bash
  echo ${fruits[0]}    # 输出 apple（下标从 0 开始）
  echo ${fruits[-1]}   # 输出最后一个元素（cherry）
```

- 关联数组：

```bash
  echo ${user["name"]} # 输出 Bob
```

**（2）获取所有元素**

- 展开为独立单词：

```bash
  echo ${fruits[@]}    # apple banana cherry
```

- 展开为单个字符串（用空格分隔）：

```bash
  echo ${fruits[*]}    # apple banana cherry
```

**（3）获取数组长度**

- 元素总数：

```bash
  echo ${#fruits[@]}   # 输出 3
```

- 单个元素的长度：

```bash
  echo ${#fruits[1]}   # 输出 6（banana 的长度）
```

**（4） 修改数组**

- **添加元素**

```bash
fruits+=("orange")     # 末尾追加元素
fruits[3]="grape"      # 直接指定下标
```

- **删除元素**

```bash
unset fruits[1]        # 删除下标为 1 的元素（banana）
unset fruits           # 删除整个数组
```

**（5）合并数组**

```bash
arr1=(1 2)
arr2=(3 4)
combined=("${arr1[@]}" "${arr2[@]}")
```

**（6）遍历数组**

- **索引数组**

```bash
for fruit in "${fruits[@]}"; do
    echo "$fruit"
done
```

- **关联数组**

```bash
for key in "${!user[@]}"; do
    echo "$key: ${user[$key]}"
done
```

## Shell运算符

**（1）算数运算符**

![算数运算符](https://raw.githubusercontent.com/GLeXios/Notes/main/pics/4937342.jpg)

**（2）关系运算符**

**关系运算符只支持数字，不支持字符串，除非字符串的值是数字。**

![shell关系运算符](https://raw.githubusercontent.com/GLeXios/Notes/main/pics/64391380.jpg)

**（3）逻辑运算符**

![逻辑运算符](https://raw.githubusercontent.com/GLeXios/Notes/main/pics/60545848.jpg)

**（4）布尔运算符**

![布尔运算符](https://raw.githubusercontent.com/GLeXios/Notes/main/pics/93961425-20250221204154507.jpg)

**（5）字符串运算符**

![ 字符串运算符](https://raw.githubusercontent.com/GLeXios/Notes/main/pics/309094.jpg)

**（6）文件相关运算符**

| 操作符 | 说明           | 示例                  |
| ------ | -------------- | --------------------- |
| `-e`   | 文件/目录存在  | `[ -e "/path/file" ]` |
| `-f`   | 是普通文件     | `[ -f "/path/file" ]` |
| `-d`   | 是目录         | `[ -d "/path/dir" ]`  |
| `-s`   | 文件存在且非空 | `[ -s "/path/file" ]` |
| `-r`   | 文件可读       | `[ -r "/path/file" ]` |
| `-w`   | 文件可写       | `[ -w "/path/file" ]` |
| `-x`   | 文件可执行     | `[ -x "/path/file" ]` |
| `-L`   | 是符号链接     | `[ -L "/path/link" ]` |

## **Shell 条件判断 **if（单括号和双括号）

（1） **逻辑运算符：**

| 操作符 | 说明 | 单括号 `[ ]` 写法 | 双括号 `[[ ]]` 写法 |
| ------ | ---- | ----------------- | ------------------- |
| 逻辑与 | AND  | `-a`              | `&&`                |
| 逻辑或 | OR   | `-o`              | `                   |
| 逻辑非 | NOT  | `!`               | `!`                 |

**示例**：

```bash
# 单括号写法
if [ $a -gt 10 -a $a -lt 20 ]; then
    echo "10 < a < 20"
fi

# 双括号写法（推荐）
if [[ $a -gt 10 && $a -lt 20 ]]; then
    echo "10 < a < 20"
fi
```

（2）**`[[ ]]` 与 `[ ]` 的区别**

| 特性           | `[ ]`（test 命令） | `[[ ]]`（Bash 扩展语法） |
| -------------- | ------------------ | ------------------------ |
| 支持模式匹配   | 不支持             | 支持（如 `==` 通配符）   |
| 支持正则表达式 | 不支持             | 支持（`=\~` 操作符）     |
| 逻辑运算符     | 必须用 `-a`、`-o`  | 可用 `&&`、`             |
| 变量空值处理   | 需手动加引号       | 自动处理空值，更安全     |
| 字符串比较     | 必须用 `=`、`!=`   | 支持 `==`、`!=`          |

```bash
# 模式匹配（双括号）
if [[ "$file" == *.txt ]]; then
    echo "Text file"
fi

# 正则表达式匹配（双括号）
if [[ "$phone" =\~ ^[0-9]{3}-[0-9]{4}$ ]]; then
    echo "Valid phone number"
fi
```

（3）**使用命令返回值作为条件**

- 若命令执行成功（退出码为 0），条件为真：

```bash
  if grep -q "error" log.txt; then
      echo "Error found"
  fi
```

- **==显式检查退出码：==**

```bash
  command
  if [ $? -eq 0 ]; then
      echo "Success"
  else
      echo "Failed"
  fi
```

（4）**算术比较（使用 `(( ))`）**

- 专用于数值运算和比较，更简洁：

```bash
  if (( a > 10 && b < 20 )); then
      echo "Valid range"
  fi
```

**（5）复杂条件清晰化**

- 复杂的逻辑条件用括号分组：

```bash
   if [[ ($a -gt 10 || $b -lt 5) && $c != "skip" ]]; then
       echo "Condition met"
   fi
```

**==注意：==**

 **在 `if` 语句中直接使用 `&&` 或 `||`**

- **可以连接多个命令**，但**本质是检查命令的退出状态码**（而非条件表达式）。
- 逻辑运算符作用于命令的成败，而非测试条件的结果。

```bash
# 检查文件存在且文件内容包含 "success"
if [ -f "log.txt" ] && grep -q "success" log.txt; then
    echo "Condition met"
fi
```

## Shell 流程控制 for、while

### for循环

**（1）基本用法**

```bash
for 变量 in 列表; do
    # 循环体
done
```

**（2）常见用法**

**1) 遍历静态列表**

```bash
for fruit in "apple" "banana" "cherry"; do
    echo "Fruit: $fruit"
done
# 输出：
# Fruit: apple
# Fruit: banana
# Fruit: cherry
```

**2) 遍历数组**

```bash
files=("file1.txt" "file2.txt")
for file in "${files[@]}"; do
    echo "Processing $file"
done
```

**3) 遍历命令输出结果**

```bash
# 遍历当前目录下的所有 .log 文件
for logfile in *.log; do
    echo "Found log: $logfile"
done

# 遍历 `ls` 命令的输出（慎用，文件名含空格会出错）
for file in $(ls); do
    echo "File: $file"
done
```

**4) 遍历数字范围**

```bash
# 遍历 1 到 5（包含5）
for i in {1..5}; do
    echo "Number: $i"
done

# 指定步长（Bash 4+）
for i in {0..10..2}; do  # 0, 2, 4, ..., 10
    echo "Step: $i"
done

# 传统方式（兼容性更好）,变量不需要加$
for ((i=1; i<=5; i++)); do
    echo "i=$i"
done
```

**5) 遍历文件内容**

```bash
# 按行读取文件（避免空格问题）
while IFS= read -r line; do
    echo "Line: $line"
done < input.txt
```

### while循环

**（1）基本语法**

```bash
while 条件; do
    # 循环体
done
```

**（2）常见用法**

**1) 条件判断循环**

```bash
count=0
while [ $count -lt 5 ]; do
    echo "Count: $count"
    ((count++))
done
# 输出：
# Count: 0
# Count: 1
# ...
# Count: 4
```

**2) 逐行读取文件**

```bash
while IFS= read -r line; do
    echo "Line: $line"
done < input.txt
```

**3) 无限循环**

```bash
while true; do
    echo "Running..."
    sleep 1
done

# 或更简洁的写法
while :; do
    echo "Infinite loop"
done
```

**4）注意：输入重定向：在 `while` 循环内使用管道可能导致变量无法保留（子 Shell 问题）：**

```bash
  # 错误：count 在子 Shell 中修改，父 Shell 不感知
  cat input.txt | while read line; do
      ((count++))
  done
  echo "Total lines: $count"  # 输出 0

  # 正确：使用重定向避免管道
  while read line; do
      ((count++))
  done < input.txt
  echo "Total lines: $count"  # 正确计数
```

### 嵌套循环（注意break）

**break默认行为：跳出当前层循环**

- 在嵌套循环中，**默认 `break` 只跳出当前所在层的循环**，继续执行外层循环的下一次迭代。

**示例**：

```bash
for i in {1..3}; do
    echo "外层循环: $i"
    for j in A B C; do
        if [ "$j" = "B" ]; then
            break  # 跳出内层循环，继续外层循环
        fi
        echo "  内层循环: $j"
    done
done
```

**指定跳出层数：`break n`**

- **==使用 `break n` 可以指定跳出 n 层循环（Bash 支持，但部分 Shell 如 `sh` 可能不支持）。==**

**示例**：跳出两层循环

```bash
# 使用break n跳出循环
for i in {1..3}; do
    echo "外层循环: $i"
    for j in A B C; do
        if [ "$j" = "B" ]; then
            break 2  # 直接跳出两层循环，完全终止
        fi
        echo "  内层循环: $j"
    done
done

# 使用标志变量控制
   should_break=false
   for i in {1..3}; do
       echo "外层循环: $i"
       for j in A B C; do
           if [ "$j" = "B" ]; then
               should_break=true
               break  # 跳出内层循环
           fi
           echo "  内层循环: $j"
       done
       if $should_break; then
           break  # 跳出外层循环
       fi
   done
```

## Shell 函数

**（1）定义函数**

- **标准语法**（兼容 POSIX）：

  ```bash
  function_name() {
      # 函数体
  }
  ```

- **Bash 扩展语法**（使用 `function` 关键字）：

  ```shell
  function function_name {
      # 函数体
  }
  ```

**（2）参数传递**

- **位置参数**：函数内部通过 `$1`, `$2`, ..., `$n` 获取参数。
- 特殊变量
  - `$#`：参数个数。
  - `$@`：所有参数（独立单词形式）。
  - `$*`：所有参数（合并为单个字符串）。

```bash
sum() {
    local total=0
    for num in "$@"; do
        total=$((total + num))
    done
    echo "总和：$total"
}
sum 1 2 3  # 输出：总和：6
```

**（3） 返回值**

- **退出状态码**：通过 `return N` 返回，`N` 为 0（成功）或 1-255（错误）。

```bash
  is_file() {
      if [ -f "$1" ]; then
          return 0
      else
          return 1
      fi
  }
  is_file "test.txt" && echo "文件存在" || echo "文件不存在"
```

- **输出返回值**：通过 `echo` 输出数据，使用命令替换捕获：

```bash
  get_date() {
      echo $(date +%F)
  }
  today=$(get_date)
```

**（4）变量作用域**

- **全局变量**：默认情况下，函数内修改的变量会影响全局。
- **局部变量**：使用 `local` 声明局部变量（仅限 Bash）：

```bash
  myfunc() {
      local var="局部变量"
      global_var="全局变量"
  }
  myfunc
  echo $var        # 输出空（局部变量不可见）
  echo $global_var # 输出 "全局变量"
```

**（5）示例**

**文件备份函数**

```bash
# 备份指定目录到目标路径
# 用法：backup <源目录> <目标目录>
backup() {
    local src=$1
    local dest=$2
    if [ ! -d "$src" ]; then
        echo "错误：源目录不存在"
        return 1
    fi
    tar -czf "${dest}/backup_$(date +%Y%m%d).tar.gz" "$src"
    echo "备份完成：${dest}/backup_$(date +%Y%m%d).tar.gz"
}

backup "/data" "/backups"
```

