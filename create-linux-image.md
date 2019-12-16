## mount linux image:
### 用kpartx建立分区映射后，再mount映射后的设备即可，操作实例如下

	[root@jay-linux image]# kpartx -av sles11sp2-i386.img
	add map loop3p1 (253:2): 0 4206592 linear /dev/loop3 2048
	add map loop3p2 (253:3): 0 37734400 linear /dev/loop3 4208640
	
	[root@jay-linux image]# mount /dev/mapper/loop3p2 /media/
	
	[root@jay-linux image]# ls /media/
	bin   dev  home  lost+found  mnt  proc  sbin     srv      sys  usr
	boot  etc  lib   media       opt  root  selinux  success  tmp  var
	[root@jay-linux image]# umount /media/
	[root@jay-linux image]# mount /dev/mapper/loop3p1 /media/
	/dev/mapper/loop3p1 looks like swapspace - not mounted
	mount: you must specify the filesystem type
	（其中的交换分区，我也还不知道是否可以mount；其实mount交换分区也没意义）
	
	（使用完成后，卸载挂载点、删除映射关系即可）
	[root@jay-linux image]# umount /media/
	[root@jay-linux image]# kpartx -d sles11sp2-i386.img
	loop deleted : /dev/loop3

#### shell
	zhe@zhe-vm:~/am335x/image$ sudo kpartx -av sd-new.img 
	[sudo] password for zhe: 
	add map loop0p1 (252:0): 0 1638400 linear /dev/loop0 2048
	add map loop0p2 (252:1): 0 852425 linear /dev/loop0 1640448
	
	zhe@zhe-vm:~/am335x/image$ sudo kpartx -d sd-new.img 
	[sudo] password for zhe: 
	loop deleted : /dev/loop0

### 扩展分区脚本：
	zhe@zhe-vm:/media/zhe/rootfs/etc$ cat rc.local
	#!/bin/bash
	do_expand_rootfs() {
	  ROOT_PART=$(mount | sed -n 's|^/dev/\(.*\) on / .*|\1|p')
	
	  PART_NUM=${ROOT_PART#mmcblk0p}
	  if [ "$PART_NUM" = "$ROOT_PART" ]; then
	    echo "$ROOT_PART is not an SD card. Don't know how to expand"
	    return 0
	  fi
	
	  # Get the starting offset of the root partition
	  PART_START=$(parted /dev/mmcblk0 -ms unit s p | grep "^${PART_NUM}" | cut -f 2 -d: | sed 's/[^0-9]//g')
	  [ "$PART_START" ] || return 1
	  # Return value will likely be error for fdisk as it fails to reload the
	  # partition table because the root fs is mounted
	  fdisk /dev/mmcblk0 <<EOF
	p
	d
	$PART_NUM
	n
	p
	$PART_NUM
	$PART_START
	
	p
	w
	EOF
	
	cat <<EOF > /etc/rc.local &&
	#!/bin/sh
	echo "Expanding /dev/$ROOT_PART"
	resize2fs /dev/$ROOT_PART
	rm -f /etc/rc.local; cp -f /etc/rc.local.bak /etc/rc.local; /etc/rc.local
	
	EOF
	reboot
	exit
	}
	raspi_config_expand() {
	/usr/bin/env raspi-config --expand-rootfs
	if [[ $? != 0 ]]; then
	  return -1
	else
	  rm -f /etc/rc.local; cp -f /etc/rc.local.bak /etc/rc.local; /etc/rc.local
	  reboot
	  exit
	fi
	}
	raspi_config_expand
	echo "WARNING: Using backup expand..."
	sleep 5
	do_expand_rootfs
	echo "ERROR: Expanding failed..."
	sleep 5
	rm -f /etc/rc.local; cp -f /etc/rc.local.bak /etc/rc.local; /etc/rc.local
	exit 0

系统会在下次启动的时候将rootfs分区扩展到整个卡

### Create image file:
	zhe@zhe-vm:~/am335x/image$ dd if=/dev/zero of=test.img bs=1M count=30
	30+0 records in
	30+0 records out
	31457280 bytes (31 MB) copied, 0.670787 s, 46.9 MB/s

### Create partition:
	zhe@zhe-vm:~/am335x/image$ sudo fdisk test.img 
	Device contains neither a valid DOS partition table, nor Sun, SGI or OSF disklabel
	Building a new DOS disklabel with disk identifier 0x4ced772d.
	Changes will remain in memory only, until you decide to write them.
	After that, of course, the previous content won't be recoverable.
	
	Warning: invalid flag 0x0000 of partition table 4 will be corrected by w(rite)
	
	Command (m for help): n
	Partition type:
	   p   primary (0 primary, 0 extended, 4 free)
	   e   extended
	Select (default p): 
	Using default response p
	Partition number (1-4, default 1): 
	Using default value 1
	First sector (2048-61439, default 2048): 
	Using default value 2048
	Last sector, +sectors or +size{K,M,G} (2048-61439, default 61439): +1M
	
	Command (m for help): n
	Partition type:
	   p   primary (1 primary, 0 extended, 3 free)
	   e   extended
	Select (default p): 
	Using default response p
	Partition number (1-4, default 2): 
	Using default value 2
	First sector (4096-61439, default 4096): 
	Using default value 4096
	Last sector, +sectors or +size{K,M,G} (4096-61439, default 61439): 
	Using default value 61439
	
	Command (m for help): w
	The partition table has been altered!
	
	Syncing disks.
	
	Make file system:
	zhe@zhe-vm:~/am335x/image$ sudo kpartx -av test.img 
	add map loop0p1 (252:0): 0 2048 linear /dev/loop0 2048
	add map loop0p2 (252:1): 0 57344 linear /dev/loop0 4096
	
	zhe@zhe-vm:~/am335x/image$ sudo mkfs.ext4 -L boot /dev/mapper/loop0p1 
	mke2fs 1.42.9 (4-Feb-2014)
	
	Filesystem too small for a journal
	Discarding device blocks: done                            
	Filesystem label=boot
	OS type: Linux
	Block size=1024 (log=0)
	Fragment size=1024 (log=0)
	Stride=0 blocks, Stripe width=0 blocks
	128 inodes, 1024 blocks
	51 blocks (4.98%) reserved for the super user
	First data block=1
	Maximum filesystem blocks=1048576
	1 block group
	8192 blocks per group, 8192 fragments per group
	128 inodes per group
	
	Allocating group tables: done                            
	Writing inode tables: done                            
	Writing superblocks and filesystem accounting information: done
	
	zhe@zhe-vm:~/am335x/image$ sudo mkfs.ext4 -L rootfs /dev/mapper/loop0p2
	mke2fs 1.42.9 (4-Feb-2014)
	Discarding device blocks: done                            
	Filesystem label=rootfs
	OS type: Linux
	Block size=1024 (log=0)
	Fragment size=1024 (log=0)
	Stride=0 blocks, Stripe width=0 blocks
	7168 inodes, 28672 blocks
	1433 blocks (5.00%) reserved for the super user
	First data block=1
	Maximum filesystem blocks=29360128
	4 block groups
	8192 blocks per group, 8192 fragments per group
	1792 inodes per group
	Superblock backups stored on blocks: 
		8193, 24577
	
	Allocating group tables: done                            
	Writing inode tables: done                            
	Creating journal (1024 blocks): done
	Writing superblocks and filesystem accounting information: done
	
	zhe@zhe-vm:~/am335x/image$ sudo kpartx -d test.img 
	loop deleted : /dev/loop0

