+++
Categories = ["linux", "lvm"]
Description = "介绍LVM的基本用法"
Tags = ["linux", "lvm"]
date = "2015-12-17T14:59:54+08:00"
title = "LVM管理磁盘的基本用法"

+++

&emsp;&emsp;LVM 是一种可用在Linux内核的逻辑分卷管理器；可用于管理磁盘驱动器或其他类似的大容量存储设备。

### LVM基本组成

LVM利用Linux内核的device-mapper来实现存储系统的虚拟化。通过LVM可以实现存储空间的抽象化，并在上面建立虚拟分区，可以很简便的扩大和缩小分区大小，可以在增加或者删除分区操作时无需担心某个硬盘上没有足够的连续空间。LVM是用来方便管理的，不会提供额外的安全保证。

LVM的基本组成：
- 物理卷Physical volume (PV)：可以在上面建立卷组的媒介，可以是硬盘分区，也可以是硬盘本身或者回环文件。
- 卷组Volume group (VG)：将一组物理卷收集为一个管理单元。
- 逻辑卷Logical volume (LV)：虚拟分区，由物理区域(physical extents)组成。
- 物理区域Physical extent (PE)：硬盘可提供指派给逻辑卷的最小单位（通常为4MB）。

#### 优点

比起正常的磁盘分区管理，LVM更富有弹性：
- 使用卷组(VG)，使众多磁盘空间看起来像一个大磁盘。
- 使用逻辑卷(LV)，可以创建跨越众多磁盘空间的分区。
- 可以创建小的逻辑卷(LV)，在空间不足时再动态调整它的大小。
- 在调整逻辑卷(LV)大小时可以不用考虑逻辑卷在磁盘上的位置，不用担心没有可用的连续空间。
- 可以在线对逻辑卷(LV)和卷组(VG)进行创建、删除、调整大小等操作。
- 无需重新启动服务，就可以将服务中用到的逻辑卷(LV)在线或动态迁移到别的磁盘上。
- 允许创建快照，可以保存文件系统的备份，提高数据恢复的速度。

#### 缺点

- 在系统设置时需要更复杂的额外步骤。

### 创建物理卷(PV)

注意：**请确认您对正确的分区进行操作！**

您可以通过以下方式找到类型“Linux LVM”的分区（这是您在分区的时候就设定好的，“Linux LVM”分区码对应的十六进制码为8e(MBR)或8e00(GPT)） ：
- MBR格式：fdisk -l
- GPT格式：先用命令lsblk，再用命令gdisk -l disk-device

在该分区（假设是/dev/sda2）下创建一个物理卷(PV)：
```shell
pvcreate /dev/sda2
```

查看创建好的物理卷(PV)
```shell
pvdisplay
pvs
```

### 创建卷组(VG)

创建完物理卷(PV)之后，我们就可以开始创建卷组(VG)了。如果您有两个以上的物理卷(PV)（例如：/dev/sda2和/dev/sdb1），并且您希望只使用一个卷组(VG)来管理这些物理卷(PV)，那么您首先要在其中一个物理卷(PV)上创建一个卷组(VG)，然后再让该卷组(VG)扩大到其他所有的物理卷(PV)：
```shell
vgcreate datavg /dev/sda2
vgextend datavg /dev/sdb1
```

上面的命令中，datavg是您取得卷组名称，您可以换成您喜欢的名称。

查看卷组信息命令：
```shell
vgdisplay
vgs
```

### 创建逻辑卷(LV)

创建完卷组(VG)之后，我们就可以开始创建逻辑卷(LV)了。下面是在datavg卷组(VG)上创建一个名为datalv大小为10G的逻辑卷(LV)：
```shell
lvcreate -L 10G datavg -n datalv
```

如果要创建的是swap分区，那么需要加上-C y参数，该参数用来指定逻辑卷的空间分配是连续的，保证您所创建的swap空间不会被分散在不连续的物理空间甚至不同的硬盘中。如在datavg卷组(VG)上创建10G大小名为swaplv的逻辑卷(LV)：
```shell
lvcreate -C y -L 10G datavg -n swaplv
```

如果您想要让创建的逻辑卷(LV)拥有卷组(VG)的所有未使用的空间，可以使用下面命令：
```shell
lvcreate -l +100%FREE datavg -n datalv
```

查看逻辑卷(LV)信息
```shell
lvdisplay
lvs
```

现在您新建的逻辑卷应该已经在/dev/mapper和/dev/rootvg中了。如果您无法在以上位置找到它，请使用一下命令来加载模块，并扫描与激活卷组：
```shell
modprobe dm-mod
vgscan
vgchange -ay
```

### 创建文件系统(File system)和挂载逻辑卷(LV)

现在您可以在逻辑卷(LV)上创建文件系统并像普通分区一样挂载它了。
```shell
mkfs.ext4 /dev/mapper/rootvg-datalv
mount /dev/mapper/rootvg-datalv /data
```

以上就是LVM的基本使用介绍。后面还会介绍如何调整逻辑卷(LV)的大小。
