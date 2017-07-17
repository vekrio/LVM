# LVM
逻辑卷管理


逻辑卷管理（LVM）指系统将物理卷管理抽象到更高的层次，常常会形成更简单的管理模式。通过使用 LVM，所有物理磁盘和分区，无论它们的大小和分布方式如何，都

被抽象为单一存储（single storage）源。例如，在图 1 所示的物理到逻辑映射布局中，最大的磁盘是 80GB 的，那么用户如何创建更大（比如 150GB）的文件系统
呢？

LVM 可以将分区和磁盘聚合成一个虚拟磁盘（virtual disk），从而用小的存储空间组成一个统一的大空间。这个虚拟磁盘在 LVM 术语中称为卷组（volume group）。

建立比最大的磁盘还大的文件系统并不是这种高级存储管理方法的惟一用途。还可以使用 LVM 完成以下任务：

*在磁盘池中添加磁盘和分区，对现有的文件系统进行在线扩展

*用一个 160GB 磁盘替换两个 80GB 磁盘，而不需要让系统离线，也不需要在磁盘之间手工转移数据

*当存储空间超过所需的空间量时，从池中去除磁盘，从而缩小文件系统

*使用快照（snapshot）执行一致的备份（本文后面会进一步讨论）

LVM2 是一个新的用户空间工具集，它为 Linux 提供逻辑卷管理功能。它完全向后兼容原来的 LVM 工具集。在本文中，将介绍 LVM2 最有用的特性以及几种简化系统

管理任务的方法。（随便说一句，如果您正在寻找关于 LVM 的基本指南，那么可以看看 参考资料 中列出的 LVM HowTo。）

我们来看看 LVM 的结构是什么样子的。

LVM 的结构

# #LVM 被组织为三种元素：

*卷（Volume）：物理 和逻辑卷 和卷组

*区段（Extent）：物理 和逻辑区段

*设备映射器（Device mapper）：Linux 内核模块

# #卷
Linux LVM 组织为物理卷（PV）、卷组（VG）和逻辑卷（LV）。物理卷 是物理磁盘或物理磁盘分区（比如 /dev/hda 或 /dev/hdb1）。卷组 是物理卷的集合。卷

组 可以在逻辑上划分成多个逻辑卷。

物理磁盘 0 上的所有四个分区（/dev/hda[1-4]）以及完整的物理磁盘 1（/dev/hdb）和物理磁盘 2（/dev/hdd）作为物理卷添加到卷组 VG0 中。

卷组是实现 n-to-m 映射的关键（也就是，将 n 个 PV 看作 m 个 LV）。在将 PV 分配给卷组之后， 就可以创建任意大小的逻辑卷（只要不超过 VG 的大小）。在

图 2 的示例中，创建了一个称为 LV0 的卷组，并给其他 LV 留下了一些空间（这些空间也可以用来应付 LV0 以后的增长）。

LVM 中的逻辑卷就相当于物理磁盘分区；在实际使用中，它们就是 物理磁盘分区。

在创建 LV 之后，可以使用任何文件系统对它进行格式化并将它挂载在某个挂载点上，然后就可以开始使用它了。图 3 显示一个经过格式化的逻辑卷 LV0 被挂载在 /var。

# #区段
为了实现 n-to-m 物理到逻辑卷映射，PV 和 VG 的基本块必须具有相同的大小；这些基本块称为物理区段（PE）和逻辑区段（LE）。尽管 n 个物理卷映射到 m 个逻

辑卷，但是 PE 和 LE 总是一对一映射的。

在使用 LVM2 时，对于每个 PV/LV 的最大区段数量并没有限制。默认的区段大小是 4MB，对于大多数配置不需要修改这个设置，因为区段的大小并不影响 I/O 性能。

但是，区段数量太多会降低 LVM 工具的效率，所以可以使用比较大的区段，从而降低区段数量。但是注意，在一个 VG 中不能混用不同的区段大小，而且用 LVM 修改

区段大小是一种不安全的操作，会破坏数据。所以建议在初始设置时选择一个区段大小，以后不再修改。

不同的区段大小意味着不同的 VG 粒度。例如，如果选择的区段大小是 4GB，那么只能以 4GB 的整数倍缩小或扩展 LV。

图 4 用 PE 和 LE 显示与前一个示例相同的布局（VG0 中的空闲空间也由空闲 LE 组成，尽管图中没有显示它们）。

另外，请注意图 4 中的区段分配策略。LVM2 并非总是连续分配 PE；细节参见关于 lvm 的 Linux 手册页（见 参考资料 中的链接）。系统管理员可以设置不同的分

配策略，但是一般不需要这么做，因为默认策略（名为一般分配策略（normal allocation policy））使用符合常规的规则，比如不把并行的条带放在同一物理卷

上。

如果决定创建第二个 LV（LV1），那么最终的 PE 布局可能像图 5 这样。

# #设备映射器

设备映射器（也称为 dm_mod）是一个 Linux 内核模块（也可以是内置的），最早出现在 2.6.9 内核中。它的作用是对设备进行映射 —— LVM2 必须使用这个模块。

在大多数主流发行版中，设备映射器会被默认安装，常常会在引导时或者在安装或启用 LVM2/EVMS 包时自动装载（EVMS 是一种替代 LVM 的工具，更多信息见 参考资

料）。如果没有启用这个模块，那么对 dm_mod 执行 modprobe 命令，在发行版的文档中查找在引导时启用它的方法：modprobe dm_mod。

在创建 VG 和 LV 时， 可以给它们起一个有意义的名称（而不是像前面的示例那样使用 VG0、LV0 和 LV1 等名称）。设备映射器的作用就是将这些名称正确地映射到

物理设备。对于前面的示例，设备映射器会在 /dev 文件系统中创建下面的设备节点：
*/dev/mapper/VG0-LV0

*/dev/VG0/LV0 是以上节点的链接

*/dev/mapper/VG0-LV1

*/dev/VG0/LV1 是以上节点的链接

（注意名称的格式标准：/dev/{vg_name}/{lv_name} -> /dev/mapper/{vg_name}{lv_name}）。

与物理磁盘相反，无法直接访问卷组（这意味着没有 /dev/mapper/VG0 这样的文件，也不能执行 dd if=/dev/VG0 of=dev/VG1）。常常使用 lvm(8) 命令访问卷

组。

# #常见任务

在使用 LVM2 时常常执行的任务包括系统检验（是否安装了 LVM2）以及创建、扩展和管理卷。

回页首

# #系统准备好运行 LVM2 了吗？

检查您的 Linux 发行版是否安装了 LVM2 软件包。如果还没有，就安装它（最好安装发行版附带的软件包）。

设备映射器模块必须在系统启动时装载。用 lsmod | grep dm_mod 命令检查当前是否装载了这个模块。如果没有装载，那么可能需要安装并配置更多的软件包（文档

会说明如何启用 LVM2）。

如果只是想测试一下（或者挽救某个系统），那么可以使用以下命令启动 LVM2：

# #清单 1. 启动 LVM2 的基本命令

#this should load the Device-mapper module

modprobe dm_mod

#this should find all the PVs in your physical disks

pvscan

#this should activete all the Volume Groups

vgchange -ay

如果打算将根文件系统放在一个 LVM LV 中，那么还要注意 initial-ramdisk 映像。同样，发行版常常会负责处理这个问题 —— 在安装 LVM2 包时，它们常常会重

新构建或更新 initrd 映像，在其中添加适当的内核模块和启动脚本。但是，可能需要查看发行版的文档，确保系统支持 LVM2 根文件系统。

注意，通常只有当探测到根文件系统在一个 VG 中时，initial-ramdisk 映像才会启用 LVM。这种探测常常是通过分析 root= 内核参数执行的。不同的发行版以不

同的方式判断根文件系统是否在卷组中。细节参见发行版的文档。如果不确定的话，就需要检查 initrd 或 initramdisk 的配置。


# #创建新的卷

使用您喜欢的分区工具（比如 fdisk、parted 或 gparted），创建一个供 LVM 使用的新分区。尽管 LVM 支持在整个磁盘上使用 LVM，但是不 建议这么做：其他

操作系统可能认为这个磁盘没有初始化，可能会破坏它！更好的方法是创建一个覆盖整个磁盘的分区。

大多数分区工具常常默认使用分区 ID 0x83（或 Linux）来创建新分区。可以使用这个默认 ID，但是为了便于组织，最好将它改为 0x8e（或 Linux LVM）。

在创建分区之后，应该会在分区表中看到一个（或多个）Linux LVM 分区：

root@klausk:/tmp/a# fdisk -l

Disk /dev/hda: 80.0 GB, 80026361856 bytes

255 heads, 63 sectors/track, 9729 cylinders

Units = cylinders of 16065 * 512 = 8225280 bytes

   Device Boot      Start         End      Blocks   Id  System
/dev/hda1   *           1        1623    13036716    7  HPFS/NTFS
/dev/hda2            1624        2103     3855600   8e  Linux LVM
/dev/hda3            2104        2740     5116702+  83  Linux
/dev/hda4            3000        9729    54058725    5  Extended
/dev/hda5            9569        9729     1293232+  82  Linux swap / Solaris
/dev/hda6            3000        4274    10241374+  83  Linux
/dev/hda7            4275        5549    10241406   83  Linux
/dev/hda8            5550        6824    10241406   83  Linux
/dev/hda9            6825        8099    10241406   83  Linux
/dev/hda10           8100        9568    11799711   8e  Linux LVM

Partition table entries are not in disk order

root@klausk:/tmp/a#

现在用 pvcreate 对每个分区进行初始化：

# #清单 2. 分区初始化

root@klausk:/tmp/a# pvcreate /dev/hda2 /dev/hda10

  Physical volume "/dev/hda2" successfully created
  
  Physical volume "/dev/hda10" successfully created
  
root@klausk:/tmp/a#

在一个步骤中同时创建 PV 和 VG：vgcreate：

# #清单 3. 创建 PV 和 VG
root@klausk:~# vgcreate test-volume /dev/hda2 /dev/hda10
  Volume group "test-volume" successfully created
root@klausk:~#
上面的命令创建一个称为 test-volume 的逻辑卷，它使用 /dev/hda2 和 /dev/hda10 作为最初的 PV。
在创建 VG test-volume 之后，使用 vgdisplay 命令查看刚创建的 VG 的基本信息：

# #清单 4. 查看刚创建的 VG 的基本信息
root@klausk:/dev# vgdisplay -v test-volume
    Using volume group(s) on command line
    Finding volume group "test-volume"
  --- Volume group ---
  VG Name               test-volume
  System ID             
  Format                lvm2
  Metadata Areas        2
  Metadata Sequence No  1
  VG Access             read/write
  VG Status             resizable
  MAX LV                0
  Cur LV                0
  Open LV               0
  Max PV                0
  Cur PV                2
  Act PV                2
  VG Size               14.93 GB
  PE Size               4.00 MB
  Total PE              3821
  Alloc PE / Size       0 / 0   
  Free  PE / Size       3821 / 14.93 GB
  VG UUID               lk8oco-ndQA-yIMZ-ZWhu-LtYX-T2D7-7sGKaV
   
  --- Physical volumes ---
  PV Name               /dev/hda2     
  PV UUID               8LTWlw-p1OJ-dF6w-ZfMI-PCuo-8CiU-CT4Oc6
  PV Status             allocatable
  Total PE / Free PE    941 / 941
   
  PV Name               /dev/hda10     
  PV UUID               vC9Lwb-wvgU-UZnF-0YcE-KMBb-rCmU-x1G3hw
  PV Status             allocatable
  Total PE / Free PE    2880 / 2880
   
root@klausk:/dev#
在清单 4 中，可以看到有两个 PV 被分配给这个 VG，总大小为 14.93GB，有 3,821 个 4MB 的 PE，这些 PE 都是空闲的！
既然卷组已经准备好了，就可以像使用磁盘一样用它创建分区（LV）、删除分区和重新设置分区大小 —— 注意，卷组是一个抽象实体，只有 LVM 工具集能够看到它们。使用 lvcreate 创建一个新的逻辑卷：
清单 5. 创建新的逻辑卷（分区）
root@klausk:/# lvcreate -L 5G -n data test-volume
  Logical volume "data" created
root@klausk:/#
# #清单 5 创建一个名为 data 的 5GB LV。创建这个 LV 之后，可以检查它的设备节点：
# #清单 6. 检查 LV 的设备节点
root@klausk:/# ls -l /dev/mapper/test--volume-data 
brw-rw---- 1 root disk 253, 4 2006-11-28 17:48 /dev/mapper/test--volume-data
root@klausk:/# ls -l /dev/test-volume/data 
lrwxrwxrwx 1 root root 29 2006-11-28 17:48 /dev/test-volume/data -> 
/dev/mapper/test--volume-data
root@klausk:/#
还可以用 lvdisplay 命令查看 LV 的属性：
# #清单 7. 查看 LV 的属性
root@klausk:~# lvdisplay /dev/test-volume/data 
  --- Logical volume ---
  LV Name                /dev/test-volume/data
  VG Name                test-volume
  LV UUID                FZK4le-RzHx-VfLz-tLjK-0xXH-mOML-lfucOH
  LV Write Access        read/write
  LV Status              available
  # open                 0
  LV Size                5.00 GB
  Current LE             1280
  Segments               1
  Allocation             inherit
  Read ahead sectors     0
  Block device           253:4
   
root@klausk:~#
在这里可以看到，在实际使用时 LV 的名称/路径是 /dev/{VG_name}/{LV_name}，比如 /dev/test-volume/data。除了用作 /dev/{VG_name}/{LV_name} 链接的目标之外，不应该在其他地方使用 /dev/mapper/{VG_name}-{LV_name} 文件。大多数 LVM 命令要求以 /dev/{vg-name}/{lv-name} 格式指定操作的目标。
建立逻辑卷之后，可以使用任何文件系统对它进行格式化，然后将它挂载在某个挂载点上：
# #清单 8. 挂载 LV
root@klausk:~# mkfs.reiserfs /dev/test-volume/data 
root@klausk:~# mkdir /data
root@klausk:~# mount -t reiserfs /dev/test-volume/data /data/
root@klausk:~# df -h /data
Filesystem            Size  Used Avail Use% Mounted on
/dev/mapper/test--volume-data
                      5.0G   33M  5.0G   1% /data
root@klausk:~#
还可以编辑 fstab(5) 文件，从而在引导时自动挂载这个文件系统：
# #清单 9. 自动挂载 LV
#mount Logical Volume 'data' under /data
/dev/test-volume/data   /data   reiserfs        defaults        0 2
在实际使用中，逻辑卷的表现就像一个块设备，比如可以将它用作数据库的原始分区。实际上，如果希望对数据库执行一致的备份，那么使用 LVM 快照是标准的最佳实践。
回页首
# #扩展卷
扩展卷是非常容易的。如果卷组中有足够的空闲空间，那么只需使用 lvextend 来扩展卷，不需要卸载它。然后，还要扩展逻辑卷中的文件系统（请记住，它们是两回事儿）。根据所用文件系统的不同，也可以进行在线扩展（即在挂载状态下进行扩展）。
如果 VG 中没有足够的空间，那么首先需要添加更多的物理磁盘。步骤如下：
使用一个物理磁盘创建一个分区。建议将分区类型改为 0x8e（Linux LVM），这样便于识别 LVM 分区/磁盘。使用 pvcreate 对物理磁盘进行初始化：pvcreate /dev/hda3。
然后，使用 vgextend 将它添加到现有的 VG 中：vgextend test-volume /dev/hda2。
还可以同时创建或添加多个物理磁盘：
pvcreate /dev/hda2 /dev/hda3 /dev/hda5
vgextend test-volume /dev/hda2 /dev/hda3 /dev/hda5
添加了 PV 之后，就有了足以扩展逻辑卷的空间，就可以使用 lvextend 扩展逻辑卷了：lvextend -L 8G /dev/test-volume/data。这个命令将 /dev/test-volume/data LV 的大小扩展到 8GB。
lvextend 有一些有用的参数：
如果希望让 LV 增加 5GB，那么可以使用 -L +5G。
可以指定扩展部分的位置（也就是，用哪些 PV 提供新的空间）；只需将希望使用的 PV 附加在命令后面。
还可以以 PE 为单位指定绝对/相对扩展大小。
细节参见 lvextend(8)。
在扩展 LV 之后，不要忘记扩展文件系统（这样才能实际使用增加的空间）。根据文件系统类型，这个操作可以在文件系统挂载状态下在线执行。
# #清单 10 是一个用 resize_reiserfs 重新设置 LV 大小的示例（随便说一句，可以在挂载的文件系统上使用这个命令）：resize_reiserfs /dev/test-volume/data。
回页首
# #管理卷
为了管理卷，需要知道如何减小 LV 和删除 PV。
减小逻辑卷
可以按照扩展 LV 的方式使用 lvreduce 命令减小 LV。从 LVM 的角度来说，这个操作可以在卷在线的情况下执行；但是，大多数文件系统不支持缩小在线文件系统。# #清单 10 给出这个过程的示例：
# #清单 10. 减小 LV
#unmount LV
umount /path/to/mounted-volume
#shrink filesystem to 4G
resize_reiserfs -s 4G /dev/test-volume/data
#reduce LV
lvreduce -L 4G /dev/vg00/test
请注意大小和单位：文件系统不应该比 LV 大！
# #删除物理卷
假设出现了以下情况：一个卷组包含两个 80GB 的磁盘，希望将它们替换为 160GB 的磁盘。在使用 LVM 时，可以按照添加 PV 的方式从 VG 中删除 PV（即在在线情况下执行删除）。但是注意，不能删除 LV 中正在使用的 PV。对于这些情况，可以使用 pvmove，它可以释放在线的 PV，这样就可以轻松地替换它们。在热交换环境中，甚至可以交换所有磁盘，而根本不需要停机！
pvmove 的惟一要求是，VG 中连续空闲区段的数量必须等于要从 PV 中删除的区段数量。没有直接判断连续空闲 PE 的最大数量的简便方法，但是可以使用 pvdisplay -m 显示 PV 分配图：
# #清单 11. 显示 PV 分配图
#shows the allocation map
pvdisplay -m
  --- Physical volume ---
  PV Name               /dev/hda6
  VG Name               test-volume
  PV Size               4.91 GB / not usable 1.34 MB
  Allocatable           yes (but full)
  PE Size (KByte)       4096
  Total PE              1200
  Free PE               0
  Allocated PE          1200
  PV UUID               BA99ay-tOcn-Atmd-LTCZ-2KQr-b4Z0-CJ0FjO

  --- Physical Segments ---
  Physical extent 0 to 2367:
    Logical volume      /dev/test-volume/data
    Logical extents     5692 to 8059
  Physical extent 2368 to 2499:
    Logical volume      /dev/test-volume/data
    Logical extents     5560 to 5691

  --- Physical volume ---
  PV Name               /dev/hda7
  VG Name               test-volume
  PV Size               9.77 GB / not usable 1.37 MB
  Allocatable           yes
  PE Size (KByte)       4096
  Total PE              2500
  Free PE               1220
  Allocated PE          1280
  PV UUID               Es9jwb-IjiL-jtd5-TgBx-XSxK-Xshj-Wxnjni

  --- Physical Segments ---
  Physical extent 0 to 1279:
    Logical volume      /dev/test-volume/LV0
    Logical extents     0 to 1279
  Physical extent 1280 to 2499:
    FREE
# #清单 11 显示有 2,499-1,280 = 1,219 个连续空闲区段，这表示最多能够将 1,219 个区段从另一个 PV 转移到 /dev/hda7。
如果希望释放一个 PV 以便进行替换，那么最好禁止它的分配，这样就可以在从卷组中删除它之前确保它一直是空闲的。在转移数据之前，执行以下命令：
# #清单 12. 在释放之前禁止 PV 的分配
#Disable /dev/hda6 allocation
pvchange -xn /dev/hda6
释放之后，PV /dev/hda6 的大小为 1,200 个区段，没有空闲区段了。使用以下命令将数据转移出这个 PV：
# #清单 13. 从释放的 PV 移出数据
#Move allocated extents out of /dev/hda6
pvmove -i 10 /dev/hda6
清单 13 中的 -i 10 参数指示 pvmove 每 10 秒报告一次状态。根据要转移的数据量，这个操作可能要花费几分钟（甚至几小时）。还可以使用 -b 参数将这个操作转到后台执行。在后台执行的情况下，状态报告会发送到系统日志。
如果没有足以进行 pvmove 操作的连续空闲区段，那么可以在 VG 中添加 一个或多个磁盘/分区，从而形成 pvmove 所需的连续空间。
# #其他有用的 LVM 操作
关于下面这些 LVM 操作的细节，请查阅手册页：
pvresize：如果底层分区也已经扩展了，那么可以用这个操作扩展 PV；如果分配图允许的话，它也可以缩小 PV。
pvremove：销毁 PV（清空它的元数据）。只有在用 vgreduce 从 VG 中删除 PV 之后，才能使用这个操作。
vgreduce：从卷组中删除未分配的 PV，这会减小 VG。
vgmerge：将两个 VG 合并成一个。目标 VG 可以是在线的！
vgsplit：分割一个卷组。
vgchange：修改一个 VG 的属性和权限。
lvchange：修改一个 LV 的属性和权限。
lvconvert：在线性卷和镜像或快照之间进行转换。
回页首
# #用快照执行备份
如果在备份过程期间数据没有发生变化，那么就能够获得一致的备份。如果不在备份期间停止系统，就很难保证数据没有变化。
Linux LVM 实现了一种称为快照（Snapshot）的特性，它的作用就像是 “拍摄” 逻辑卷在某一时刻的照片。通过使用快照， 可以获得同一 LV 的两个拷贝 —— 一个可以用于备份，另一个继续用于日常操作。
快照有两大优点：
快照的创建非常快，不需要停止生产环境。
建立两个拷贝，但是它们的大小并不一样。快照使用的空间仅仅是存储两个 LV 之间的差异所需的空间。
快照由一个例外列表（exception list）来实现，每当 LV 之间出现差异时就会更新这个列表（正式的说法是 CoW，Copy-on-Write）。
# #创建新的快照
创建新的快照 LV 也是使用 lvcreate 命令，但是要指定 -s 参数和原来的 LV。在这种情况下，-L size 指定例外列表的大小，这影响快照支持的最大差异量，如果差异超过这个量，就无法保持一致性。
# #清单 14. 建立快照
#create a Snapshot LV called 'snap' from origin LV 'test'
lvcreate -s -L 2G -n snap/dev/test-volume/test
可以使用 lvdisplay 查询特殊信息，比如 CoW 的大小和使用情况：
# #清单 15. CoW 的大小和使用情况
lvdisplay /dev/vg00/snap


  --- Logical volume ---
  LV Name                /dev/vg00/snap
  VG Name                vg00
  LV UUID                QHVJYh-PR3s-A4SG-s4Aa-MyWN-Ra7a-HL47KL
  LV Write Access        read/write
  LV snapshot status     active destination for /dev/vg00/test
  LV Status              available
  # open                 0
  LV Size                4.00 GB
  Current LE             1024
  COW-table size         2.00 GB
  COW-table LE           512
  Allocated to snapshot  54.16%
  Snapshot chunk size    8.00 KB
  Segments               1
  Allocation             inherit
  Read ahead sectors     0
  Block device           254:5
清单 15 表明这个 CoW 的大小为 2GB，其中的 54.16 % 已经使用了。
对于所有日常操作，快照看起来就是 原 LV 的一个拷贝。如果已经建立了文件系统的话，可以用以下命令挂载它：
#mount snapshot volume
mount -o ro /dev/test-volume/test /mnt/snap
在这个命令中，ro 标志表示将它挂载为只读的。可以在 lvcreate 命令后面加上 -p r，这样就在 LVM 级将它设置为只读的。
挂载文件系统之后，就可以用 tar、rsync 或其他备份工具执行备份。如果 LV 不包含文件系统，或者需要原始备份，那么也可以在这个设备节点上直接使用 dd。
复制过程完成之后，就不需要快照了，这时只需用 lvremove 卸载并销毁它：
#remove snapshot
lvremove /dev/test-volume/snap
如果数据库建立在 LV 上，并且需要一个一致的备份，那么一定要刷新表并在获得读取锁（read-lock）的情况下建立快照卷（见下面的伪代码）：
SQL> flush tables read lock
{create Snapshot}
SQL> release read lock
{start copy process from the snapshot LV}
# #备份脚本示例
清单 16 中的脚本直接取自我的笔记本电脑，我在这个脚本中使用 rsync 向一台远程服务器执行每日备份。这个脚本并不适合企业环境；在企业环境中，带历史记录的增量备份更合适，但概念是相同的。
# #清单 16. 简单的备份脚本示例
#!/bin/sh

# we need the dm-snapshot module
modprobe dm-snapshot
if [ -e /dev/vg00/home-snap ]
then
  # remove left-overs, if any
  umount -f /mnt/home-snap && true
  lvremove -f /dev/vg00/home-snap
fi
# create snapshot, 1GB CoW space
# that should be sufficient for accommodating changes during copy
lvcreate -vs -p r -n home-snap -L 1G /dev/vg00/home
mkdir -p /mnt/home-snap
# mount recently-created snapshot as read-only
mount -o ro /dev/vg00/home-snap /mnt/home-snap
# magical rsync command__rsync -avhzPCi --delete -e "ssh -i /home/klausk/.ssh/id_rsa" \
      --filter '- .Trash/' --filter '- *~' \
      --filter '- .local/share/Trash/' \
      --filter '- *.mp3' --filter '- *Cache*' --filter '- *cache*' \
      /mnt/home-snap/klausk klausk2@pokgsa.ibm.comThis e-mail address is being protected 
      from spam bots, you need JavaScript enabled to view it :bkp/
# unmount and scrap snapshot LV
umount /mnt/home-snap
lvremove -f /dev/vg00/home-snap
在某些特殊情况下，无法估计备份周期或者复制过程很长，那么脚本可以用 lvdisplay 查询 Snapshot CoW 的使用情况并根据需要扩展这个 LV。在极端情况下， 可以让快照与原 LV 同样大 —— 这样就不需要执行查询，因为变化量不会比整个卷更大！
回页首
# #其他 LVM2 系统管理技巧
最后， 我要介绍一些可以用 LVM2 执行的系统管理任务，包括按需虚拟化、用镜像提高容错能力以及透明地对块设备执行加密。
# #快照和虚拟化
在使用 LVM2 时，快照可以不是只读的。这意味着，在创建快照之后， 可以像常规块设备一样挂载和读写快照。
因为流行的虚拟化系统（比如 Xen、VMWare、Qemu 和 KVM）可以将块设备用作 guest 映像，所以可以创建这些映像的完整拷贝，并根据需要使用它们，它们就像是内存占用量很低的虚拟机。这样做的好处是部署迅速（创建快照的时间常常不超过几秒）和节省空间（guest 共享原映像的大多数数据）。
设置的步骤如下：
为原映像创建一个逻辑卷。
使用这个 LV 作为磁盘映像安装 guest 虚拟机。
暂停这个虚拟机。内存映像可以是一个常规文件，所有其他快照都放在里面。
为原 LV 创建一个可读写的快照。
使用快照卷作为磁盘映像生成一个新的虚拟机。如果需要的话，要修改网络/控制台设置。
登录已经创建的虚拟机，修改网络设置/主机名。
完成这些步骤之后， 就可以让用户访问刚创建的虚拟机了。如果需要另一个虚拟机，那么只需重复步骤 4 到 6（所以不需要重新安装虚拟机）。还可以用一个脚本自动执行这些步骤。
在使用完虚拟机之后， 可以停止虚拟机并销毁快照。
# #更好的容错能力
最近的 LVM2 开发成果为逻辑卷提供了高可用性。逻辑卷可以有两个或更多的镜像，镜像可以放在不同的物理卷（或不同的设备）上。当在设备上发现 I/O 错误时，可以使用 dmeventd 让一个 PV 离线，而不会影响服务。更多信息请参考 lvcreate(8)、lvconvert(8) 和 lvchange(8) 手册页。
如果硬件能够支持的话，可以用 dm_multipath 通过不同的通道访问同一设备，这样的话在一个通道发生故障时，可以转移到另一个通道。更多细节请参考 dm_multipath 和 multipathd 的文档。
透明的设备加密
可以用 dm_crypt 对块设备或逻辑卷执行透明的加密。更多信息请参考 dm_crypt 的文档和 cryptsetup(8) 手册页。
