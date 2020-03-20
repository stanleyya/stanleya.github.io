# 一.修改IP,hostname

```
127.0.0.1        localhost

192.168.6.103    rh1

192.168.6.104    rh1-vip

10.10.10.3       rh1-priv

 

192.168.6.105    rh2

192.168.6.106    rh2-vip

10.10.10.5       rh2-priv

 

[root@rh1 u01]# vi /etc/sysconfig/network

NETWORKING=yes

NETWORKING_IPV6=no

HOSTNAME=rh1

 

[root@rh1 u01]# vi /etc/sysconfig/network-scripts/ifcfg-eth0

DEVICE=eth0

BOOTPROTO=static

ONBOOT=yes

IPADDR=192.168.6.103

GATEWAY=192.168.6.1

NETMASK=255.255.255.0

 

[root@rh1 u01]# vi /etc/sysconfig/network-scripts/ifcfg-eth1

DEVICE=eth1

BOOTPROTO=static

ONBOOT=yes

IPADDR=10.10.10.3

NETMASK=255.255.255.0
```
# 二.增加交换分区
```
[root@rh1 u01]# dd if=/dev/zero of=/u01/swap1 bs=1024k count=2048

2048+0 records in

2048+0 records out

2147483648 bytes (2.1 GB) copied, 29.9432 seconds, 71.7 MB/s

 

[root@rh1 u01]# mkswap -c /u01/swap1

Setting up swapspace version 1, size = 2147479 kB

 

[root@rh1 u01]# swapon /u01/swap1

 

[root@rh1 u01]# vi /etc/fstab


LABEL=/                               /                           ext3          defaults        1 1
LABEL=/u01                        /u01                    ext3          defaults        1 2
tmpfs                                    /dev/shm            tmpfs      defaults,size=2g        0 0
devpts                                  /dev/pts             devpts      gid=5,mode=620  0 0
sysfs                                      /sys                      sysfs          defaults        0 0
proc                                      /proc                   proc         defaults        0 0
LABEL=SWAP-sda2           swap                    swap       defaults        0 0
/dev/sda3                           /soft                     ext3         defaults        0 0

/u01/swap1                        swap                   swap        defaults        0 0



 

[root@rh2 /]# free -m

             total       used       free     shared    buffers     cached

Mem:          1341       1312         28          0          4       1062

-/+ buffers/cache:        244       1096

Swap:         4095          0       4095
```
# 三.创建用户，组和目录并赋予权限   设置oracle用户环境变量
```
groupadd -g 200 oinstall

groupadd -g 201 dba

useradd -u 200 -g oinstall -G dba oracle

passwd oracle
```


**节点1**
```
su - oracle

vi .bash_profile

PATH=$PATH:$HOME/bin

export ORACLE_BASE=/u01/app/oracle

export ORACLE_HOME=$ORACLE_BASE/product/10.2.0/db_1

export ORA_CRS_HOME=/u01/crs_1

export ORACLE_SID=prod1

export PATH=.:${PATH}:$HOME/bin:$ORACLE_HOME/bin

export PATH=${PATH}:/usr/bin:/bin:/usr/bin/X11:/usr/local/bin

export PATH=${PATH}:$ORACLE_BASE/common/oracle/bin

export LD_LIBRARY_PATH=$ORACLE_HOME/lib

export LD_LIBRARY_PATH=${LD_LIBRARY_PATH}:$ORACLE_HOME/oracm/lib

export LD_LIBRARY_PATH=${LD_LIBRARY_PATH}:/lib:/usr/lib:/usr/local/lib

export EDITOR=vi
```
**节点2 SID=prod2其他不变**

 

**创建目录**
```
mkdir -p /u01/app/oracle/product/10.2.0/db_1

mkdir -p /u01/crs_1

mkdir -p /u01/app/oraInventory

chmod -R 775 /u01/app/oracle/product/10.2.0/db_1

chmod -R 775 /u01/app/oraInventory/

chmod -R 775 /u01/crs_1/
```

# 四.建立用户等效性
```
[oracle@rh1 ~]$ ssh-keygen -t rsa

Generating public/private rsa key pair.

Enter file in which to save the key (/home/oracle/.ssh/id_rsa):

Enter passphrase (empty for no passphrase):

Enter same passphrase again:

Your identification has been saved in /home/oracle/.ssh/id_rsa.

Your public key has been saved in /home/oracle/.ssh/id_rsa.pub.

The key fingerprint is:

32:8a:e1:8c:0d:70:86:8e:88:97:5a:3a:15:fd:a3:dd oracle@rh1

[oracle@rh1 ~]$ ssh-keygen -t dsa

Generating public/private dsa key pair.

Enter file in which to save the key (/home/oracle/.ssh/id_dsa):

Enter passphrase (empty for no passphrase):

Enter same passphrase again:

Your identification has been saved in /home/oracle/.ssh/id_dsa.

Your public key has been saved in /home/oracle/.ssh/id_dsa.pub.

The key fingerprint is:

d8:88:23:9b:6f:cd:74:81:8f:f9:13:16:98:5f:8a:8f oracle@rh1
```
**节点2执行相同操作**
```
[oracle@rh2 ~]$ ssh-keygen -t rsa
[oracle@rh2 ~]$ ssh-keygen -t dsa
```
**节点1**
```shell
[oracle@rh1 ~]$ cat .ssh/id_rsa.pub >> .ssh/authorized_keys

[oracle@rh1 ~]$ cat .ssh/id_dsa.pub >> .ssh/authorized_keys

[oracle@rh1 ~]$ ssh rh2 cat .ssh/id_rsa.pub >> .ssh/authorized_keys  这一步是将节点2的id_rsa.pub追加写入到节点1的authorized_keys文件里

oracle@rh2's password:

[oracle@rh1 ~]$ ssh rh2 cat .ssh/id_dsa.pub >> .ssh/authorized_keys

oracle@rh2's password:

[oracle@rh1 ~]$ scp .ssh/authorized_keys rh2:~/.ssh  将写有节点1和节点2密码文件的authorized_keys拷贝到节点2，两边就都拥有密钥了

oracle@rh2's password:

authorized_keys 100% 1988 1.9KB/s 00:00
```
**验证信任关系**

```
ssh rh1 date

ssh rh1-priv date

ssh rh2 date

ssh rh2-priv date
```
**两个节点都要验证**

# 五.修改系统参数

```
[root@rh1:/root]# vi /etc/sysctl.conf   在最后添加下面的参数

fs.aio-max-nr = 1048576

fs.file-max = 6815744

kernel.shmall = 2097152

kernel.shmmax = 536870912

kernel.shmmni = 4096

kernel.sem = 250 32000 100 128

net.ipv4.ip_local_port_range = 9000 65500

net.core.rmem_default = 262144

net.core.rmem_max = 4194304

net.core.wmem_default = 262144

net.core.wmem_max = 1048586

[root@rh1:/root]# sysctl -p  立即生效

 

[root@rh1:/root]# vi /etc/security/limits.conf               限制用户访问的CPU和内存资源

oracle  soft    nproc   2047

oracle  hard    nproc   16384

oracle  soft    nofile  1024

oracle  hard    nofile  65536

oracle  soft    stack   10240

 

 

[root@rh1:/root]# vi /etc/pam.d/login         linux操作系统的登陆配置文件 

添加

session required /lib/security/pam_limits.so           当用户登陆以后自动启用上面的限制

 

[root@rh1:/root]# vi /etc/profile

添加

 

if [ $USER = "oracle" ]; then

        if [ $SHELL = "/bin/ksh" ]; then

                ulimit -p 16384

                ulimit -n 65536

        else

                ulimit -u 16384 -n 65536

        fi

fi
```

# 六.时间同步

```
用ntp

vi /etc/ntp.conf

节点1做服务端

broadcastclient

driftfile /var/lib/ntp/drift

server 127.127.1.0

节点2为客户端

driftfile /var/lib/ntp/drift

server 192.168.6.103      节点1的IP

 

使NTP服务可以在系统引导的时候自动启动，执行：    

# chkconfig ntpd on   

启动/关闭/重启NTP的命令：   

# /etc/init.d/ntpd start   

# /etc/init.d/ntpd stop   

# /etc/init.d/ntpd restart   

#service ntpd restart

将同步好的时间写到CMOS里   

vi /etc/sysconfig/ntpd   

SYNC_HWCLOCK=yes   

 

每次修改了配置文件后都需要重新启动服务来使配置生效。

可以使用下面的命令来检查NTP服务是否启动，你应该可以得到一个进程ID号：   

# pgrep ntpd   

使用下面的命令检查时间服务器同步的状态：   

# ntpq -p   

用ntpstat 也可以查看一些同步状态，用netstat -ntlup查看端口使用情况！ 

 

安装完毕客户端需过5-10分钟才能从服务器端更新时间！  
```

**一些补充和拾遗（挺重要）**

**1. 配置文件中的driftfile是什么?**

> 我们每一个system clock的频率都有小小的误差,这个就是为什么机器运行一段时间后会不精确. NTP会自动来监测我们时钟的误差值并予以调整.但问题是这是一个冗长的过程,所以它会把记录下来的误差先写入driftfile.这样即使你重新开机以后之前的计算结果也就不会丢失了

**2. 如何同步硬件时钟?**

>NTP一般只会同步system clock. 但是如果我们也要同步RTC(hwclock)的话那么只需要把下面的选项打开就可以了

> 代码:

> vi /etc/sysconfig/ntpd

> SYNC_HWCLOCK=yes

# 七.格式化分区

**ocr 我们要划分成2个100M，分别来存放ocr配置文件，votingdisk我们要分成3个100M的。 Asm4data 要分成两个，10G 存放数据文件，5G 用在flashback上。 Backup 我们分成一个区。**

**在一个结点执行格式化就可以了，因为他们是共享的。**

```
[root@rh1 soft]# fdisk /dev/sdc

Device contains neither a valid DOS partition table, nor Sun, SGI or OSF disklabel

Building a new DOS disklabel. Changes will remain in memory only,

until you decide to write them. After that, of course, the previous

content won't be recoverable.

 

 

The number of cylinders for this disk is set to 2610.

There is nothing wrong with that, but this is larger than 1024,

and could in certain setups cause problems with:

1) software that runs at boot time (e.g., old versions of LILO)

2) booting and partitioning software from other OSs

   (e.g., DOS FDISK, OS/2 FDISK)

Warning: invalid flag 0x0000 of partition table 4 will be corrected by w(rite)

 

Command (m for help): n

Command action

   e   extended

   p   primary partition (1-4)

p

Partition number (1-4): 1

First cylinder (1-2610, default 1):

Using default value 1

Last cylinder or +size or +sizeM or +sizeK (1-2610, default 2610): +100m

 

Command (m for help): n

Command action

   e   extended

   p   primary partition (1-4)

p

Partition number (1-4): 2

First cylinder (14-2610, default 14):

Using default value 14

Last cylinder or +size or +sizeM or +sizeK (14-2610, default 2610): +100m

 

Command (m for help): n

Command action

   e   extended

   p   primary partition (1-4)

p

Partition number (1-4): 3

First cylinder (27-2610, default 27):

Using default value 27

Last cylinder or +size or +sizeM or +sizeK (27-2610, default 2610): +100m

 

Command (m for help): n

Command action

   e   extended

   p   primary partition (1-4)

e

Selected partition 4

First cylinder (40-2610, default 40):

Using default value 40

Last cylinder or +size or +sizeM or +sizeK (40-2610, default 2610):

Using default value 2610

 

Command (m for help): n

First cylinder (40-2610, default 40):

Using default value 40

Last cylinder or +size or +sizeM or +sizeK (40-2610, default 2610): +100m

 

Command (m for help): n

First cylinder (53-2610, default 53):

Using default value 53

Last cylinder or +size or +sizeM or +sizeK (53-2610, default 2610): +100m

 

Command (m for help): n

First cylinder (66-2610, default 66):

Using default value 66

Last cylinder or +size or +sizeM or +sizeK (66-2610, default 2610): +10g

 

Command (m for help): n

First cylinder (1283-2610, default 1283):

Using default value 1283

Last cylinder or +size or +sizeM or +sizeK (1283-2610, default 2610): +5g

 

Command (m for help): n

First cylinder (1892-2610, default 1892):

Using default value 1892

Last cylinder or +size or +sizeM or +sizeK (1892-2610, default 2610):

Using default value 2610

 

Command (m for help): p

 

Disk /dev/sdc: 21.4 GB, 21474836480 bytes

255 heads, 63 sectors/track, 2610 cylinders

Units = cylinders of 16065 * 512 = 8225280 bytes

 

   Device Boot      Start        End        Blocks            Id    System

/dev/sdc1               1             13        104391             83    Linux

/dev/sdc2              14           26        104422+           83    Linux

/dev/sdc3              27           39        104422+           83    Linux

/dev/sdc4              40         2610        20651557+    5    Extended

/dev/sdc5              40           52        104391            83    Linux

/dev/sdc6              53          65        104391             83    Linux

/dev/sdc7              66        1282        9775521        83    Linux

/dev/sdc8            1283     1891        4891761        83    Linux

/dev/sdc9            1892     2610        5775336        83    Linux
```

# 八.配置raw 设备

所谓raw 设备，就是通过字符方式访问的设备，也就是读写设备不需要缓冲区。 在Linux 下，对磁盘值提供了块方式的访问。要想通过字符方式访问，必须配置raw 设备服务，并且Oracle 用户对这些raw 设备必须有访问的权限。

 

在2个节点上做如下操作：

```
1.修改/etc/udev/rules.d/60-raw.rules 文件

 

root@rh1 soft]# vi /etc/udev/rules.d/60-raw.rules

ACTION=="add", KERNEL=="sdc1",RUN+="/bin/raw /dev/raw/raw1 %N"

ACTION=="add", KERNEL=="sdc2",RUN+="/bin/raw /dev/raw/raw2 %N"

ACTION=="add", KERNEL=="sdc3",RUN+="/bin/raw /dev/raw/raw3 %N"

ACTION=="add", KERNEL=="sdc5",RUN+="/bin/raw /dev/raw/raw4 %N"

ACTION=="add", KERNEL=="sdc6",RUN+="/bin/raw /dev/raw/raw5 %N"

ACTION=="add",KERNEL=="raw[1-5]", OWNER="oracle", GROUP="oinstall", MODE="660"

 

2. 重启服务：

[root@rac1 ~]# start_udev

Starting udev:         [  OK  ]

 

3. 查看raw设备：

 

[root@rh1 soft]# ls -lrt /dev/raw

total 0

crw-rw---- 1 oracle oinstall 162, 5 May 17 15:02 raw5

crw-rw---- 1 oracle oinstall 162, 3 May 17 15:02 raw3

crw-rw---- 1 oracle oinstall 162, 2 May 17 15:02 raw2

crw-rw---- 1 oracle oinstall 162, 1 May 17 15:02 raw1

crw-rw---- 1 oracle oinstall 162, 4 May 17 15:02 raw4

```
# 九.安装ASM软件
这里附一下ASM软件，我下了很多次，安装后总是configure失败，最后找到一个可以用的，应该是之前的版本不对
http://pan.baidu.com/s/1bnthLYB

两个节点都要安
```
[root@rh2 asm]# rpm -ivh *

warning: oracleasm-2.6.18-194.el5-2.0.5-1.el5.x86_64.rpm: Header V3 DSA signature: NOKEY, key ID 1e5e0159

Preparing...                ########################################### [100%]

   1:oracleasm-support      ########################################### [ 20%]

   2:oracleasm-2.6.18-194.el########################################### [ 40%]

   3:oracleasm-2.6.18-194.el########################################### [ 60%]

   4:oracleasm-2.6.18-194.el########################################### [ 80%]

   5:oracleasmlib           ########################################### [100%]

安装后在各个节点都要configure    oracle dba y y

[root@rh2 asm]# service oracleasm configure

Configuring the Oracle ASM library driver.

 

This will configure the on-boot properties of the Oracle ASM library

driver.  The following questions will determine whether the driver is

loaded on boot and what permissions it will have.  The current values

will be shown in brackets ('[]').  Hitting without typing an

answer will keep that current value.  Ctrl-C will abort.

 

Default user to own the driver interface [oracle]:

Default group to own the driver interface [dba]:

Start Oracle ASM library driver on boot (y/n) [y]:

Scan for Oracle ASM disks on boot (y/n) [y]:

Writing Oracle ASM library driver configuration: done

Initializing the Oracle ASMLib driver:                     [  OK  ]

Scanning the system for Oracle ASMLib disks:      [  OK  ]

                                                          

创建ASM磁盘

节点1创建

[root@rh1 asm]# service oracleasm createdisk ASM_DATA1 /dev/sdc7

Marking disk "ASM_DATA1" as an ASM disk:                   [  OK  ]

[root@rh1 asm]# service oracleasm createdisk ASM_DATA2 /dev/sdc8

Marking disk "ASM_DATA2" as an ASM disk:                   [  OK  ]

[root@rh1 asm]# service oracleasm listdisks

ASM_DATA1

ASM_DATA2

 

节点2要scan一下

[root@rh2 asm]# service oracleasm listdisks

[root@rh2 asm]# service oracleasm scandisks

Scanning the system for Oracle ASMLib disks:               [  OK  ]

[root@rh2 asm]# service oracleasm listdisks

ASM_DATA1

ASM_DATA2
```

# 十.安装clusterware

> http://download.oracle.com/otn/linux/oracle10g/10201/10201_clusterware_linux_x86_64.cpio.gz
>
> 因为Oracle10发行的时候还不支持rh5，修改安装目录里面oraparam.ini文件redhat-4直接改成5(可以用find -name oraparam.ini )，或者/etc/redhat-release 改成4
>
>  
>
> Has 'rootpre.sh' been run by root? [y/n] (n)

```
[root@rh1 rootpre]# sh rootpre.sh

No OraCM running

这里我们跑rootpre提示这个

网上说可以忽略

 

进行安装前的校验时又出现

[oracle@rh1 cluvfy]$ ./runcluvfy.sh stage -pre crsinst -n rh1,rh2 -verbose

ERROR:

Could not find a suitable set of interfaces for VIPs.

网上说可以忽略，之后跑vipca即可
```
![](https://raw.githubusercontent.com/stanleyya/pic/master/1.png)


