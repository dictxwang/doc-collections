# GDB简明使用手册

*GDB是GNU开源组织发布的UNIX环境下的调试工具。*

### 一、GDB使用前提
#### 1、安装GDB

Linux（例如CentOS发行版本）基本上都自带了GDB工具，这里主要记录一下MacOS上gdb的安装。

##### A、使用homebrew安装

```shell
brew install gdb
```

##### B、对GDB签名

Mac上需要事先对GDB进行签名，否则系统将限制使用。通常会见到如下提示：

(please check gdb is codesigned - see taskgated(8))

签名的过程分为创建证书和使用证书签名两步，具体操作如下：

*a、创建证书*

```
command+空格   查找钥匙串  并打开
顶部菜单栏钥匙串访问->证书助理->创建证书
输入名称（如gdb_codesgin）
身份类型（自签名根证书）
勾选：让我覆盖这些模式设置
一直next，到证书保存位置时选 系统
选择系统->我的证书->双击刚创建的证书
展开信任节点，选择始终信任
关闭窗口，输入密码保存
```

*b、使用证书签名*

在终端输入 codesign -s gdb_codesign gdb（或者 "$(which gdb)” 指定GDB绝对路径）

#### 2、修改编译参数

要利用GDB进行调试，需要再编译时增加调试信息，具体做法就是增加编译参数 -g

```shell
gcc -g -o main.o main.c
```

否则会提示 *...(no debugging symbols found)...*

### 二、启动GDB的方法与注意事项

#### 1、Mac上启动GDB的注意事项

虽然经过代码签名后GDB可以启动，但是可能会遇到执行run指令后直接卡住不到的情况。因此需要增加如下的配置：

```shell
需要添加~/.gdbinit 文件，并在其中输入set startup-with-shell off，具体实现是：
sudo echo “set startup-with-shell off” ~/.gdbinit
注意这里需要以root权限执行，并且在启动gdb调试时，也需要通过sudo执行，如下：
sudo gdb main.o
```

#### 2、GDB启动的方法

##### A、gdb <program>

​    program也就是你的执行文件，一般在当然目录下。

##### B、gdb <program> core

​    用gdb同时调试一个运行程序和core文件，core是程序非法执行后core dump后产生的文件。

##### C、gdb <program> <PID>

​    如果你的程序是一个服务程序，那么你可以指定这个服务程序运行时的进程ID。gdb会自动attach上去，并调试他。program应该在PATH环境变量中搜索得到。

#### 3、常用的GDB启动开关

GDB启动时，可以加上一些GDB的启动开关，详细的开关可以用gdb -help查看。比较常用的参数：

##### A、-symbols <file>

​      -s <file>

​      从指定文件中读取符号表。

##### B、-se file

​     从指定文件中读取符号表信息，并把他用在可执行文件中。

##### C、-core <file>

​      -c <file>

​     调试时core dump的core文件。

##### D、-directory <directory>

​     -d <directory>

​     加入一个源文件的搜索路径。默认搜索路径是环境变量中PATH所定义的路径。

### 三、GDB的常用操作

(gdb) list 1 (或l 1) 从第一行开始列出源码

(gdb) list main 列出main函数的代码

(gdb) 直接回车（继续执行上一次命令）

(gdb) break 16  在16行设置断点

(gdb) break func  在func函数入口处设置断点

(gdb) info break (或info breakpoints) 查看断点信息

(gdb) run (或r) 运行程序

(gdb) next (或n) 单条执行程序

(gdb) continue (或c) 继续执行程序，相当于跳过断点

(gdb) print i (或p i) 打印i的值

(gdb) bt 查看函数堆栈

(gdb) finish 结束函数

(qdb) quit (或q) 结束gdb

### 四、一些特殊的调试方法

待补充。。。