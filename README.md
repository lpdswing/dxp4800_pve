# dxp4800_pve
pve安装dxp4800虚拟机，解决在线更新系统问题

# 新增boot启动磁盘
1. pve新增2个磁盘，大小都给0.004G
2. 用dd命令把2个存储启动信息的分区传输给新建的2个虚拟磁盘
```shell
dd if=/dev/mmcblk0boot0 of=/dev/mapper/pve-vm--100--disk--2
dd if=/dev/mmcblk0boot1 of=/dev/mapper/pve-vm--100--disk--3
```
# 创建虚拟磁盘的映射
## 查看磁盘总线地址
   ```shell
     ssh进绿联虚拟机执行下面命令
     lsscsi
     输出：
        [1:0:0:0]    cd/dvd  QEMU     QEMU DVD-ROM     2.5+  /dev/sr0
        [6:0:0:0]    disk    ATA      QEMU HARDDISK    2.5+  /dev/sda
        [7:0:0:0]    disk    ATA      QEMU HARDDISK    2.5+  /dev/sdb
        [8:0:0:0]    disk    ATA      QEMU HARDDISK    2.5+  /dev/sdc
     或者执行： udevadm info -q path -n /dev/sdb
     输出：
     /devices/pci0000:00/0000:00:1e.0/0000:05:01.0/0000:06:07.0/ata8/host7/target7:0:0/7:0:0:0/block/sdb
   
   6:0:0:0就是sda磁盘的总线，每次重启sda，sdb可能都会变，要用固定不变的总线做条件映射
  ```
## 创建udev规则，每次开机自动给磁盘一个SYMLINK
  - 创建文件 /etc/udev/rules.d/10-rename-disks.rules
    用vim编辑器编辑这个文件，填入下面代码
  
      ```bash
        KERNELS=="6:0:0:0", SYMLINK+="mmcblk0"
        KERNELS=="7:0:0:0", SYMLINK+="mmcblk0boot0"
        KERNELS=="8:0:0:0", SYMLINK+="mmcblk0boot1"
        KERNELS=="6:0:0:0", KERNEL=="sd?[1-7]", SUBSYSTEMS=="scsi", ATTRS{scsi_level}=="6", SYMLINK+="mmcblk0p%n"
      ```

  - 执行命令： `udevadm control --reload-rules && udevadm trigger` 让规则生效
  执行 `ls /dev/mmc*` 看一下
  如果出现`/dev/mmcblk0  /dev/mmcblk0boot0  /dev/mmcblk0boot1  /dev/mmcblk0p1  /dev/mmcblk0p2  /dev/mmcblk0p3  /dev/mmcblk0p4  /dev/mmcblk0p5  /dev/mmcblk0p6  /dev/mmcblk0p7` 就成功了。

# pve设置BIOS的sn和uuid
- 查询sn和uuid
  ssh进pve执行
  ```shell
  dmidecode -s system-uuid    # uuid
  dmidecode -s system-serial-number    # sn
  ```

- pve修改BIOS信息
  选项-SMBIOS设置，只填uuid和串行（就是sn），**不要泄露sn**

重启，OK。

# pve中控制4800的6个灯

参考这个仓库： https://github.com/miskcoo/ugreen_dx4600_leds_controller
发现ssh会自动把端口给关闭参考矿神博客：  https://imnks.com/10101.html