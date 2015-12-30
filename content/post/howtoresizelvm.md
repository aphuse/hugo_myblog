+++
Categories = ["linux", "lvm"]
Description = "如何调整LVM相关PV、VG和LV大小"
Tags = ["linux", "lvm"]
date = "2015-12-30T14:59:54+08:00"
title = "如何调整LVM相关PV、VG和LV大小"

+++

之前介绍了使用LVM管理磁盘基本用法，并且我们建议使用多少空间就分配多少空间，空间不够之后我们再扩大空间。那LVM是如何调整空间的呢？下面我们就来介绍如何调整LVM的PV、VG和LV的大小。

### 物理卷(PV)

- 扩增

增大分区/dev/sda2的容量之后，需要执行以下命令扩展物理卷的大小。
```shell
pvresize /dev/sda2
```

命令将自动探测设备的当前大小并将物理卷扩展到其最大容量。该命令可以在卷在线运行。

- 缩小

在减小某个物理卷所在设备大小之前，需要通过指定--setphysicalvolumesize size参数缩小物理卷大小。
```shell
pvresize --setphysicalvolumesize 40G /dev/sda2
```

该命令可能会提示该物理卷已分配物理区域超过了命令指定的新大小界限，pvresize命令会拒绝将物理卷缩小。若磁盘空间足够大，可以通过pvmove命令将物理区域重新分配至别的卷组来解决这个问题。

- 移动物理区域

使用pvdisplay命令查看物理分段的分布情况，具体命令参数如下：
```shell
pvdisplay -v -m
```

减小物理卷大小，一般必须把所有的已用分段移到前部。假如/dev/sda2的可用空间在第0至第153600分段共153601个可用区域，我们可以从最后的分段中移动相同数目的物理区域来填补这段空间。
```shell
pvmove --alloc anywhere /dev/sda2:307201-399668 /dev/sda2:0-92466
```

参数--alloc anywhere可以用于在同一个分区中移动物理区域的。若要在不同分区中移动，命令形式应该是 pvmove /dev/sda2:1000-1999 /dev/sdb1:0-999 。当操作的数据较多时，移动操作将持续很久（一到两个小时）。最好在Tmux或GNU Screen会话中执行此过程。 **任何形式的意外中断都可能导致致命错误** 。当操作完成后，可运行fsck保证文件系统完整性。

当所有空闲分段都移动到最后的物理区域时，运行vgdisplay查看。之后可以再次运行命令调整物理卷的大小。
```shell
pvresize --setphysicalvolumesize size PhysicalVolume
```

- 调整分区大小

最后，你可以用你喜欢的分区工具来缩小该分区了。

### 卷组(VG)

- 扩增

添加物理卷到卷组中，先创建一个新的物理卷，然后再把卷组扩充到该物理卷上
```shell
pvcreate /dev/sdb1
vgextend datavg /dev/sdb1
```

- 缩小

从卷组中移除分区。首先要把分区中的所有数据移到别的分区上，可以使用下面这个简单的命令实现：
```shell
pvmove /dev/sdb1
```

同样你也可以指定所要转移的目标分区，把目标分区作为pvmove的第二个参数使用就可以了：
```shell
pvmove /dev/sdb1 /dev/sda2
```

然后，可以从卷组中移除物理卷：
```shell
vgreduce datavg /dev/sdb1
```

或者把所有空的物理卷全部移除：
```shell
vgreduce --all datavg
```

最后如果想要使用该分区，而且不想让LVM以为它是一个物理卷，那么你可以执行以下命令：
```shell
pvremove /dev/sdb1
```

### 逻辑卷(LV)

使用lvresize增加或缩小容量，注意：**虽然即便是对根分区，增加文件系统大小一般都可以在线完成（即文件系统已经被挂载），缩小其大小往往要求首先卸载文件系统以避免数据丢失。请首先确定你的文件系统支持相关操作。**

- 扩增

向卷组datavg中的逻辑卷datalv增加2G空间，但不修改其文件系统大小，执行：
```shell
lvresize -L +2G datavg/datalv
```

设置datavg/datalv为15G并同时更改其文件系统大小(注意: **仅支持ext2, ext3, ext4, ReiserFS and XFS file systems。如果使用不同文件系统请使用合适的组件。**)
```shell
lvresize -L 15G -r datavg/datalv
```

如果想将卷组所有剩下的可用空间都加入一个逻辑卷中，可以执行：
```shell
lvresize -l +100%FREE datavg/datalv
```

- 缩小

从逻辑卷datavg/datalv中减少500MB空间，但不修改其文件系统大小（需要确保文件系统已经缩小过）
```shell
lvresize -L -500M datavg/datalv
```

- 调整文件系统大小

如果上面执行lv{resize,extend,reduce}时没有使用-r, --resizefs选项， 或文件系统不支持fsadm(8)（如Btrfs, ZFS等），则需要在缩小逻辑卷之前或扩增逻辑卷后手动调整文件系统大小。注意：**并非所有的文件系统都支持无损或/且在线调整大小。**

对于ext2/ext3/ext4文件系统使用下列命令

将文件系统大小扩展到逻辑卷支持的最大容量：
```shell
resize2fs datavg/datalv
```

将文件系统减小到其所需的最小容量，也可以指定具体的尺寸：
```shell
resize2fs -M datavg/datalv
resize2fs datavg/datalv NewSize
```

- 移除逻辑卷

警告: **在移除逻辑卷之前，请先备份好数据以免丢失！**

首先要找到你所要移除的逻辑卷的名称及其挂载点。
```shell
lvs
lsblk
```

如果要移除的逻辑卷有挂载点，那要先卸载挂载点
```shell
umount /<mountpoint>
```

最后可以移除逻辑卷了
```shell
lvremove <volume_group>/<logical_volume>
```

最后请不要忘记更新/etc/fstab。

以上就是LVM调整相关组件大小的操作。
