«Create RAID»

 ### 1.	Ошибка по сетевой карте:**
Создал новую сетевую 192.168.56.101
Потом в Vagrantfile исправляем на свои IP 192.168.56.101
       :ip_addr => '192.168.56.101',

### 2.	Фатальная ошибка комментим последний блок**
'#         box.vm.provision "shell", inline: <<-SHELL
'#             mkdir -p ~root/.ssh
'#              cp ~vagrant/.ssh/auth* ~root/.ssh
'#             yum install -y mdadm smartmontools hdparm gdisk
'#         SHELL

Добавляем диск 
},
                :sata5 => {
                        :dfile => './sata5.vdi',
                        :size => 250, # Megabytes
                        :port => 5

 ### 3.	Установка пакетов 
** sudo yum install -y mdadm smartmontools hdparm gdisk **

 ### 4.	Создаем RAID 0 из 4 дисков
**[vagrant@otuslinux ~]$ sudo mdadm --create --verbose /dev/md0 -l 5 -n 4 /dev/sd{b,c,d,e} **

mdadm: layout defaults to left-symmetric
mdadm: layout defaults to left-symmetric
mdadm: chunk size defaults to 512K
mdadm: size set to 253952K
mdadm: Defaulting to version 1.2 metadata
mdadm: array /dev/md0 started.

** [vagrant@otuslinux ~]$ cat /proc/mdstat 
 [vagrant@otuslinux ~]$ cat /proc/mdstat ** 
Personalities : [raid6] [raid5] [raid4] 
md0 : active raid5 sde[4] sdd[2] sdc[1] sdb[0]
      761856 blocks super 1.2 level 5, 512k chunk, algorithm 2 [4/4] [UUUU]
      
unused devices: <none>
** [vagrant@otuslinux ~]$ sudo mdadm -D /dev/md0 
 [vagrant@otuslinux ~]$ sudo mdadm -D /dev/md0**  
/dev/md0:
           Version : 1.2
     Creation Time : Wed Sep 20 09:01:53 2023
        Raid Level : raid5
        Array Size : 761856 (744.00 MiB 780.14 MB)
     Used Dev Size : 253952 (248.00 MiB 260.05 MB)
      Raid Devices : 4
     Total Devices : 4
       Persistence : Superblock is persistent

       Update Time : Wed Sep 20 09:02:16 2023
             State : clean 
    Active Devices : 4
   Working Devices : 4
    Failed Devices : 0
     Spare Devices : 0

            Layout : left-symmetric
        Chunk Size : 512K

Consistency Policy : resync

              Name : otuslinux:0  (local to host otuslinux)
              UUID : acf617b6:47b7334e:5784d9d3:5c65d31e
            Events : 18

    Number   Major   Minor   RaidDevice State
       0       8       16        0      active sync   /dev/sdb
       1       8       32        1      active sync   /dev/sdc
       2       8       48        2      active sync   /dev/sdd
       4       8       64        3      active sync   /dev/sde

### 5.	Верна ли информация

**[vagrant@otuslinux ~]$ sudo mdadm --detail --scan --verbose** 
ARRAY /dev/md0 level=raid5 num-devices=4 metadata=1.2 name=otuslinux:0 UUID=acf617b6:47b7334e:5784d9d3:5c65d31e
   devices=/dev/sdb,/dev/sdc,/dev/sdd,/dev/sde

### 6.	Создаем конфиг

**[vagrant@otuslinux ~]$ sudo su
[root@otuslinux vagrant]# echo "DEVICE partitions" > /etc/mdadm/mdadm.conf

mdadm --detail --scan --verbose | awk '/ARRAY/ {print}' >> /etc/mdadm/mdadm.conf**

### 7.	Зафейлить диск

**[root@otuslinux vagrant]# mdadm /dev/md0 --fail /dev/sde **
mdadm: set /dev/sde faulty in /dev/md0***

**[root@otuslinux vagrant]# cat /proc/mdstat **
Personalities : [raid6] [raid5] [raid4] 
md0 : active raid5 sde[4](F) sdd[2] sdc[1] sdb[0]
      761856 blocks super 1.2 level 5, 512k chunk, algorithm 2 [4/3] [UUU_]
      
unused devices: <none>

**[root@otuslinux vagrant]# mdadm -D /dev/md0**                                    
/dev/md0:
           Version : 1.2
     Creation Time : Wed Sep 20 09:01:53 2023
        Raid Level : raid5
        Array Size : 761856 (744.00 MiB 780.14 MB)
     Used Dev Size : 253952 (248.00 MiB 260.05 MB)
      Raid Devices : 4
     Total Devices : 4
       Persistence : Superblock is persistent

       Update Time : Wed Sep 20 09:05:53 2023
             State : clean, degraded 
    Active Devices : 3
   Working Devices : 3
    Failed Devices : 1
     Spare Devices : 0

            Layout : left-symmetric
        Chunk Size : 512K

Consistency Policy : resync

              Name : otuslinux:0  (local to host otuslinux)
              UUID : acf617b6:47b7334e:5784d9d3:5c65d31e
            Events : 20

    Number   Major   Minor   RaidDevice State
       0       8       16        0      active sync   /dev/sdb
       1       8       32        1      active sync   /dev/sdc
       2       8       48        2      active sync   /dev/sdd
       -       0        0        3      removed

       4       8       64        -      faulty   /dev/sde

### 8.	Удаляем раздел 

**[root@otuslinux vagrant]# mdadm /dev/md0 --remove /dev/sde ** 
mdadm: hot removed /dev/sde from /dev/md0

**[root@otuslinux vagrant]# cat /proc/mdstat** 
Personalities : [raid6] [raid5] [raid4] 
md0 : active raid5 sdd[2] sdc[1] sdb[0]
      761856 blocks super 1.2 level 5, 512k chunk, algorithm 2 [4/3] [UUU_]
      
unused devices: <none>

### 9.	Добавляем раздел который у нас был в запасе ‘sdf’

**[root@otuslinux vagrant]# mdadm /dev/md0 --add /dev/sdf **
mdadm: added /dev/sdf
** [root@otuslinux vagrant]# cat /proc/mdstat ** 
Personalities : [raid6] [raid5] [raid4] 
md0 : active raid5 sdf[4] sdd[2] sdc[1] sdb[0]
      761856 blocks super 1.2 level 5, 512k chunk, algorithm 2 [4/3] [UUU_]
      [======>..............]  recovery = 34.6% (88320/253952) finish=0.1min speed=17664K/sec
      
unused devices: <none>

Готово все скопировалось
** [root@otuslinux vagrant]# cat /proc/mdstat **
Personalities : [raid6] [raid5] [raid4] 
md0 : active raid5 sdf[4] sdd[2] sdc[1] sdb[0]
      761856 blocks super 1.2 level 5, 512k chunk, algorithm 2 [4/4] [UUUU]
      
unused devices: <none>

** [root@otuslinux vagrant]# mdadm -D /dev/md0 **
/dev/md0:
           Version : 1.2
     Creation Time : Wed Sep 20 09:01:53 2023
        Raid Level : raid5
        Array Size : 761856 (744.00 MiB 780.14 MB)
     Used Dev Size : 253952 (248.00 MiB 260.05 MB)
      Raid Devices : 4
     Total Devices : 4
       Persistence : Superblock is persistent

       Update Time : Wed Sep 20 09:14:31 2023
             State : clean 
    Active Devices : 4
   Working Devices : 4
    Failed Devices : 0
     Spare Devices : 0

            Layout : left-symmetric
        Chunk Size : 512K

Consistency Policy : resync

              Name : otuslinux:0  (local to host otuslinux)
              UUID : acf617b6:47b7334e:5784d9d3:5c65d31e
            Events : 40

    Number   Major   Minor   RaidDevice State
       0       8       16        0      active sync   /dev/sdb
       1       8       32        1      active sync   /dev/sdc
       2       8       48        2      active sync   /dev/sdd
       4       8       80        3      active sync   /dev/sdf



### 10.	Создаем GPT на RAID

** [root@otuslinux vagrant]# parted -s /dev/md0 mklabel gpt**

### 11. Создаем партиции
**[root@otuslinux vagrant]# parted /dev/md0 mkpart primary ext4 0% 20% **     
Information: You may need to update /etc/fstab.
** [root@otuslinux vagrant]# parted /dev/md0 mkpart primary ext4 20% 40%  **
Information: You may need to update /etc/fstab.

** [root@otuslinux vagrant]# parted /dev/md0 mkpart primary ext4 40% 60%    **  
Information: You may need to update /etc/fstab.

** [root@otuslinux vagrant]# parted /dev/md0 mkpart primary ext4 60% 80%      **
Information: You may need to update /etc/fstab.

** [root@otuslinux vagrant]# parted /dev/md0 mkpart primary ext4 80% 100%    **
Information: You may need to update /etc/fstab.



### 11.	Создание на партициях ФС
** [root@otuslinux vagrant]# for i in $(seq 1 5); do sudo mkfs.ext4 /dev/md0p$i;done**
mke2fs 1.42.9 (28-Dec-2013)
Filesystem label=
OS type: Linux
Block size=1024 (log=0)
Fragment size=1024 (log=0)
Stride=512 blocks, Stripe width=1536 blocks
37696 inodes, 150528 blocks
7526 blocks (5.00%) reserved for the super user
First data block=1
Maximum filesystem blocks=33816576
19 block groups
8192 blocks per group, 8192 fragments per group
1984 inodes per group
Superblock backups stored on blocks: 
	8193, 24577, 40961, 57345, 73729

Allocating group tables: done                            
Writing inode tables: done                            
Creating journal (4096 blocks): done
Writing superblocks and filesystem accounting information: done 

mke2fs 1.42.9 (28-Dec-2013)
Filesystem label=
OS type: Linux
Block size=1024 (log=0)
Fragment size=1024 (log=0)
Stride=512 blocks, Stripe width=1536 blocks
38152 inodes, 152064 blocks
7603 blocks (5.00%) reserved for the super user
First data block=1
Maximum filesystem blocks=33816576
19 block groups
8192 blocks per group, 8192 fragments per group
2008 inodes per group
Superblock backups stored on blocks: 
	8193, 24577, 40961, 57345, 73729

Allocating group tables: done                            
Writing inode tables: done                            
Creating journal (4096 blocks): done
Writing superblocks and filesystem accounting information: done 

mke2fs 1.42.9 (28-Dec-2013)
Filesystem label=
OS type: Linux
Block size=1024 (log=0)
Fragment size=1024 (log=0)
Stride=512 blocks, Stripe width=1536 blocks
38456 inodes, 153600 blocks
7680 blocks (5.00%) reserved for the super user
First data block=1
Maximum filesystem blocks=33816576
19 block groups
8192 blocks per group, 8192 fragments per group
2024 inodes per group
Superblock backups stored on blocks: 
	8193, 24577, 40961, 57345, 73729

Allocating group tables: done                            
Writing inode tables: done                            
Creating journal (4096 blocks): done
Writing superblocks and filesystem accounting information: done 

mke2fs 1.42.9 (28-Dec-2013)
Filesystem label=
OS type: Linux
Block size=1024 (log=0)
Fragment size=1024 (log=0)
Stride=512 blocks, Stripe width=1536 blocks
38152 inodes, 152064 blocks
7603 blocks (5.00%) reserved for the super user
First data block=1
Maximum filesystem blocks=33816576
19 block groups
8192 blocks per group, 8192 fragments per group
2008 inodes per group
Superblock backups stored on blocks: 
	8193, 24577, 40961, 57345, 73729

Allocating group tables: done                            
Writing inode tables: done                            
Creating journal (4096 blocks): done
Writing superblocks and filesystem accounting information: done 

mke2fs 1.42.9 (28-Dec-2013)
Filesystem label=
OS type: Linux
Block size=1024 (log=0)
Fragment size=1024 (log=0)
Stride=512 blocks, Stripe width=1536 blocks
37696 inodes, 150528 blocks
7526 blocks (5.00%) reserved for the super user
First data block=1
Maximum filesystem blocks=33816576
19 block groups
8192 blocks per group, 8192 fragments per group
1984 inodes per group
Superblock backups stored on blocks: 
	8193, 24577, 40961, 57345, 73729

Allocating group tables: done                            
Writing inode tables: done                            
Creating journal (4096 blocks): done
Writing superblocks and filesystem accounting information: done


### 13.	Монтируем их по каталогам 
**
[root@otuslinux vagrant]# mkdir -p /raid/part{1,2,3,4,5}
[root@otuslinux vagrant]# for i in $(seq 1 5); do mount /dev/md0p$i /raid/part$i; done**

[root@otuslinux vagrant]# /raid/part
part1/ part2/ part3/ part4/ part5/ 
[root@otuslinux vagrant]# /raid/part

