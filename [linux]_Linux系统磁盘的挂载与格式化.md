# Linux系统磁盘的挂载与格式化

这篇文章以自己使用阿里云的云盘经验总结而来，其中采用的Linux发行版本是CentOS7。

Linux上对空间小于2TB和超过2TB的磁盘格式化时，适用的命令是不同的。其中小于2TB的磁盘采用fdisk，超过2TB的磁盘可以采用parted命令。

需要注意，Linux上对磁盘的操作，需要以root权限进行。

### 一、两个基础命令

#### 1、df命令

```shell
# 查看已挂载的磁盘信息
df -lhT
```

#### 2、fdisk命令

```shell
# 查看所有磁盘信息，找到为挂载的磁盘
fdisk -l
```

### 二、挂载小于2TB的磁盘

假设通过fdisk命令找到未挂载的磁盘是*/dev/vdb*。

#### 1、磁盘分区

首先需要对磁盘进行分区，这一步可以理解为Linux系统在一块实体磁盘划定一块空间，并为其创建一个虚拟名称。

```shell
# 操作命令
fdisk /dev/vdb
# 根据提示，依次输入 n p 1
# 最后再输入w保存分区，并结束分区流程
```

结束分区后，查看分区结果。

```shell
# 操作命令
fdisk -l
# 如果分区成功，将出现分区/dev/vdb1
```

#### 2、分区格式化

完成分区后，还需要将分区格式化，即初始化分区的文件系统。Linux系统常用的文件系统有ext3、ext4、xfs等，这里采用ext4文件系统。

```shell
# 操作命令
mkfs.ext4 /dev/vdb1
# 再输入y保存
```

#### 3、分区挂载

磁盘分区后，还需要将其挂载到一个特定的路径（作为访问的起始路径）才能使用。好比windows上为分区命名为C、D、E，这样每个分区才有了一个起始路径。

Linux上，起始路径的命名非常具有自主性，例如可以将其命名为/data。

```shell
# 先新建挂载点
mkdir /data
# 挂载分区
mount /dev/vdb1 /data
```

#### 4、保存分区挂载

经过上面的操作后，分区可以正常使用了。但是分区挂载是即时的，如果系统重启需要重新进行分区挂载。为了避免这种情况，需要将分区挂载信息写入静态信息文件*/etc/fstab*中。

```shell
# 操作命令
echo '/dev/vdb1 /data ext4 defaults 0 0’ >> /etc/fstab
```

这样，即使系统重启，分区也能自动挂载上。

### 三、挂载2TB以上的磁盘

挂载2TB以上的磁盘的流程，处理采用的分区命令不同外，和挂载2TB以下的磁盘基本一致。

同样假设通过fdisk命令找到未挂载的磁盘是*/dev/vdd*。

#### 1、安装parted命令

```shell
# 操作命令
yum install parted
```

#### 2、磁盘分区与格式化

```shell
# 操作命令
parted /dev/vdd
mklabel gpt
yes
unit TB
mkpart primary 0.00TB 4.00TB  # 创建4TB大小的分区
print  # 查看分区情况
```

#### 3、分区挂载

```shell
mkdir /data2
mount /dev/vdd1 /data2
```

#### 4、保存分区挂载

```shell
echo '/dev/vdd1 /data2 ext4 defaults 0 0' >> /etc/fstab
```

以上即完成空间大于2TB磁盘的分区与挂载。