## cinder volume
## ceph flatten
### 相关配置
```
# Flatten volumes created from snapshots to remove dependency from volume to snapshot (boolean value)
rbd_flatten_volume_from_snapshot = true
# 在创建基于snapshot的卷时候是否flatten(扶平）新的volume独立出来,就是让新的volume和snapshot没有关系。
```

### 相关原理(我的理解)
新的volume通过snapshot创建，那么如果不flatten，会导致这个新的volume(称之为parent_volume吧)与某个卷有联系，当我们多次用这个parent volume的snapshot创建多个新的volume时，我们访问这些新的volume的时候会对parent_volume造成多次读取，如果parent_volume或者其中一个volume损坏的话也很麻烦...(所以这里也解释了为什么不能直接将这个参数设置为false来解决问题)
反之，我们flatten的话，会切断新的volume与snapshot的联系。上面的问题会解决。但是也会引起新的问题。就是这个issue遇到的问题。

flatten的步骤是 写->读->写
1. 读取parent_volume
2. 合并data_new与data_old
3. 写入新的volume

我的理解就是大概就是要去将与parent_volume重合的那些东西写入到新的volume中，如果要写入的东西很多的话(例如snapshot很大的话)，要很久，会导致后续的volume创建请求执行(volume假死, 这个不太明白, 后台认为它创建失败？超时机制么)和与rabbitmq的心跳包连接

## Cinder
Cinder是OpenStack中的一个项目，主要用于块存储。比如，Glance需存储的镜像、Nova所启动的虚拟机的Guest disk、以及虚拟机所attach的volume一般都是来源于Cinder的提供。

## ceph driver
Cinder项目中添加一种专门的driver来支持它的块存储硬件

  - Ceph RBD driver: RBD image. (RBD即RADOS Block Device)
  - Lenovo iSCSI driver
  - Ceph iSCSI driver


## 目前的解决方案
创建volume的任务丢到了线程池，主进程继续向下走。不存在等待耗时太久的问题。
 - 如果子线程执行失败的话，会怎样？

## 备盘池
在 Cube 中增加备盘池的功能，每次创建vm从备盘池中选择系统盘挂载到虚拟机上

我的理解：
- 提前创建好几个新的volume，比如说issue中提到的windows镜像使用的snapshot来创建，当用户需要使用的时候不用去等待创建volume的时间，直接从备盘池取出volume，就可以使用。大大提高效率。

Backup相关
Backup与Snapshot区别
1. snapshot依赖源volume，不能独立存在；而backup不依赖volume，即便源volume不存在了，仍可以restore。
2. snapshot与源volume通常存放在一起，由同一个volume provider管理；backup存放在独立的备份设备中，有自己的备份方案和实现。
3. backup具有容灾功能；而snapshot则提供volume provider内便捷的回溯功能。

所以如果后续要做的话要继续了解cinder-backup？不知道有没有理解错
  - 以NFS为backend
  - 以LVM为backend
  - 以ceph为backend(使用非rbd或者使用rbd)
  在这种情况下，即cinder-volume和cinder-backup都是用rbd作为backend，是支持增量备份的。增量备份的实现完全依赖于ceph处理差量文件的特性，所谓ceph处理差量文件的能力，即ceph可以将某个rbd image不同时刻的状态进行比较，并且将其差量导出成文件。另外，ceph也可以将这个差量文件导入到某个image中。

问题：
1. 预期的实现效果是怎样？需求 功能设计 实际实现 输入输出是个啥
2. 备盘池里面需要有哪些volume？(如何确定哪些volume需要备盘呢)
3. 当备盘池中的volume被使用之后呢？

## 升级cinder
如何评估升级后的cinder？指标？
升级可能会带来其他问题
