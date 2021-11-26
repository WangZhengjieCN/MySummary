# 材料物理系mgt集群搭建过程：

系统版本`centos-7.5-x86_64-DVD-1804.iso`

系统设置：关闭`selinux`和`firewall`

`IB`驱动版本：`MLNX_OFED_LINUX-4.9-3.1.5.0-rhel7.5-x86_64`



## `IB`驱动

##### 计算节点

```bash
# 添加yum外网代理：
vim /etc/yum.conf
proxy=http://192.168.1.1:3128

#卸载IB多余依赖
yum remove opa* -y

#下载相关依赖
yum install tcl tk -y

#临时屏蔽内核
mv /boot/initramfs-3.10.0-862.el7.x86_64.img /boot/initramfs-3.10.0-862.el7.x86_64.img.bak

#开始安装IB驱动
./mlnxofedinstall

#取消屏蔽内核
mv /boot/initramfs-3.10.0-862.el7.x86_64.img.bak /boot/initramfs-3.10.0-862.el7.x86_64.img

#删除多余模块
modprobe -r ib_srpt
modprobe -r ib_isert
modprobe -r rpcrdma

#重启IB驱动
/etc/init.d/openibd restart

#IB驱动端口自启
systemctl enable openibd.service

#重启服务器
reboot
```



## NFS:

io01作为服务端共享`/home 4.4T /dev/sdc`,

mgt作为服务端共享`/opt `

##### 服务端

##### 客户端

```bash
[root@node01 ~]# vi /etc/fstab
imgt:/opt	/opt	nfs	 defaults,nfsvers=3 0 0
iio01:/home  /home  nfs  defaults,nfsvers=3 0 0

# 挂载
[root@mgt ~]# mount -a

# 查看挂载情况，mount 和 df -h
[root@node01 ~]# df -h
Filesystem               Size  Used Avail Use% Mounted on
/dev/mapper/centos-root  491G  4.0G  487G   1% /
devtmpfs                  63G     0   63G   0% /dev
tmpfs                     63G     0   63G   0% /dev/shm
tmpfs                     63G   19M   63G   1% /run
tmpfs                     63G     0   63G   0% /sys/fs/cgroup
/dev/sda1                4.0G  149M  3.9G   4% /boot
iio01:/home              4.4T   97M  4.2T   1% /home
tmpfs                     13G     0   13G   0% /run/user/0
imgt:/opt                491G  6.4G  484G   2% /opt
```



## NIS

服务端:

1、`yum install ypserv yp-tools rpcbind -y`

2、在`/etc/sysconfig/network`中添加

```bash
NISDOMAIN=nisdomain.upc.edu
```

3、在`/etc/rc.local/rc.local`中添加

```bash
/bin/nisdomainname nisdomain.upc.edu
```

4、域名生效`nisdomainname nisdomain.upc.edu`

5、修改`/etc/ypserv.conf`，在文件中添加：

```bash
127.0.0.0/255.255.255.0       : * : * : none
192.168.1.0/255.255.255.0  : * : * : none
                             *: * : * : deny
```

6、启动服务项

```bash
systemctl enable ypserv.service
systemctl enable yppasswdd.service
chmod 754 ypserv.service
chmod 754 yppasswdd.service
systemctl start ypserv.service
systemctl start yppasswdd.service

#测试
[root@mgt ~]# rpcinfo -u mgt ypserv
program 100004 version 1 ready and waiting
program 100004 version 2 ready and waiting

#生成NIS数据库
[root@mgt ~]# /usr/lib64/yp/ypinit -m

At this point, we have to construct a list of the hosts which will run NIS
servers.  mgt is in the list of NIS server hosts.  Please continue to add
the names for the other hosts, one per line.  When you are done with the
list, type a <control D>.
        next host to add:  mgt
        next host to add:
The current list of NIS servers looks like this:

mgt

Is this correct?  [y/n: y]  y
We need a few minutes to build the databases...
Building /var/yp/nisdomain.upc.edu/ypservers...
Running /var/yp/Makefile...
gmake[1]: Entering directory `/var/yp/nisdomain.upc.edu'
Updating passwd.byname...
Updating passwd.byuid...
Updating group.byname...
Updating group.bygid...
Updating hosts.byname...
Updating hosts.byaddr...
Updating rpc.byname...
Updating rpc.bynumber...
Updating services.byname...
Updating services.byservicename...
Updating netid.byname...
Updating protocols.bynumber...
Updating protocols.byname...
Updating mail.aliases...
gmake[1]: Leaving directory `/var/yp/nisdomain.upc.edu'

mgt has been set up as a NIS master server.

Now you can run ypinit -s mgt on all slave server.
[root@mgt ~]# systemctl restart ypserv.service
[root@mgt ~]# systemctl restart yppasswdd.service
```

NIS客户端:

![image-20211114165736310](C:\Users\Administrator\AppData\Roaming\Typora\typora-user-images\image-20211114165736310.png)

![image-20211114165807575](C:\Users\Administrator\AppData\Roaming\Typora\typora-user-images\image-20211114165807575.png)

![image-20211114155829856](C:\Users\Administrator\AppData\Roaming\Typora\typora-user-images\image-20211114155829856.png)

```bash
[root@node02 ~]yum -y install ypbind yp-tools rpcbind

[root@node02 ~]setup
# 按按上图所示进行设置

[root@node02 ~]# systemctl start ypbind
[root@node02 ~]# systemctl start rpcbind
[root@node02 ~]# systemctl enable ypbind
[root@node02 ~]# systemctl enable rpcbind

systemctl start ypbind
systemctl start rpcbind
systemctl enable ypbind
systemctl enable rpcbind
```



## NTP网络时间服务

主节点：

```bash
# 安装依赖
[root@mgt ~]# yum -y install ntp ntpdate
# 修改配置
[root@mgt ~]# vi /etc/ntp.conf
## 客户端范围
restrict 192.168.1.0 mask 255.255.255.0 nomodify notrap
## 设置NTP服务器
server ntp1.aliyun.com
server ntp2.aliyun.com
server ntp3.aliyun.com
server 127.0.0.1
Fudge 127.0.0.1 stratum 10
```

其他节点：

```bash
# 安装依赖[root@node01 ~]# yum -y install ntp ntpdate# 添加ntp服务端ip[root@node01 ~]# vi /etc/ntp.confserver 192.168.1.1[root@node01 ~]# systemctl enable ntpd[root@node01 ~]# systemctl restart ntpd[root@node01 ~]# systemctl status ntpd[root@node01 ~]# ntpq -p     remote           refid      st t when poll reach   delay   offset  jitter============================================================================== mgt             203.107.6.88     3 u   18   64    1    0.266  39166.3   0.000  systemctl enable ntpd && systemctl restart ntpd && systemctl status ntpd
```



## munge和slurm的安装：

##### 主节点

```bash
yum -y install python python3 epel-release gtk2 gtk2-devel polkit  rng-tools
yum -y install munge*#mungeecho -n "XXX"|sha1sum |cut -d' ' -f1>/etc/munge/munge.key
chown munge:munge /etc/munge/munge.key
systemctl restart munge.service
systemctl enable munge.service
[root@mgt ~]# id munge
uid=988(munge) gid=981(munge) groups=981(munge)
#slurm
groupadd -g 1001 slurm
useradd -u 1001 -g slurm slurm

# 下载slurm安装包
tar -xvf slurm.tar.gz
cd slurm
./configure && make && make install
cp ./etc/{slurmctld.service,slurmdbd.service,slurmd.service} /usr/lib/systemd/system
mkdir /var/spool/slurm-llnl
chown slurm:slurm /var/spool/slurm-llnl
```

##### 计算节点

```bash
yum -y install python python3 epel-release gtk2 gtk2-devel polkit  rng-tools
yum -y install munge*
# munge
echo -n "XXX"|sha1sum |cut -d' ' -f1>/etc/munge/munge.keychmod 400 /etc/munge/munge.key
# 修改munge用户的UID和GID与主节点一致
usermod -u  988  munge
groupmod -g 981  munge
chown -R munge:munge /etc/munge/
chmod 400 /etc/munge/munge.key
chown -R munge:munge /var/run/munge
chown -R munge:munge /var/lib/munge
chown -R munge:munge /var/log/munge
systemctl restart munge.service
systemctl enable munge.service
systemctl status munge.service
# slurm
scp mgt:~/build/slurm-21.08.0-0rc1.tar.bz2 ./
tar -xvf slurm-21.08.0-0rc1.tar.bz2
cd slurm-21.08.0-0rc1
./configure && make && make install
cp ./etc/slurmd.service /usr/lib/systemd/system
groupadd -g 1001 slurm
useradd -u 1001 -g slurm slurm
scp mgt:/usr/local/etc/slurm.conf /usr/local/etc/
chown slurm:slurm /usr/local/etc/slurm.conf
systemctl daemon-reload && systemctl restart slurmd.service
systemctl enable slurmd.service && systemctl status slurmd.service

# 节点重启后需要手动上线
scontrol update NodeName=node[01-30] State=RESUME
```



##### 用户和用户组设置

```bash
# 创建用户组
[root@mgt ~]# groupadd guowy
[root@mgt ~]# groupadd zhangj
[root@mgt ~]# groupadd husq
[root@mgt ~]# groupadd luxq

# 新建用户
useradd -d /home/guowy/wangzj -g guowy wangzj
useradd -d /home/husq/student_A -g husq student_A
useradd -d /home/luxq/student_B -g luxq student_B
useradd -d /home/zhangj/student_C -g zhangj student_C

# 将用户添加到gaussian用户组
gpasswd -a wangzj gaussian
```



##### `quota`磁盘限额

```bash
#检查内核是否支持磁盘配额
[root@io01 home]# grep CONFIG_QUOTA /boot/config-3.10.0-862.el7.x86_64
CONFIG_QUOTA=y
CONFIG_QUOTA_NETLINK_INTERFACE=y
# CONFIG_QUOTA_DEBUG is not set
CONFIG_QUOTA_TREE=y
CONFIG_QUOTACTL=y
CONFIG_QUOTACTL_COMPAT=y
[root@io01 home]# yum install quota -y
[root@io01 ~]# vi /etc/fstab
#home 4T
UUID=87e316e1-7775-43d2-93f7-e51fb2d9b802 /home ext4 defaults,usrquota,grpquota 0 0
# 重新挂载硬盘，并启动quota服务
[root@io01 ~]# mount -o remount /home
[root@io01 ~]#quotaon -vug /home
/dev/sdc [/home]: group quotas turned on
/dev/sdc [/home]: user quotas turned on
# 磁盘划分
[root@io01 ~]# quota -vgs guowy
Disk quotas for group guowy (gid 1002):
Filesystem   space   quota   limit   grace   files   quota   limit   grace
  /dev/sdc   8108M   1055G   1055G            193k       0       0
[root@io01 ~]# quota -vgs luxq
Disk quotas for group luxq (gid 1005):
Filesystem   space   quota   limit   grace   files   quota   limit   grace  
  /dev/sdc      0K   1127G   1127G               0       0       0
[root@io01 ~]# quota -vgs husq
Disk quotas for group husq (gid 1004):
Filesystem   space   quota   limit   grace   files   quota   limit   grace
  /dev/sdc      0K   1127G   1127G               0       0       0
[root@io01 ~]# quota -vgs zhangj
Disk quotas for group zhangj (gid 1008):
Filesystem   space   quota   limit   grace   files   quota   limit   grace
  /dev/sdc      0K   1127G   1127G
```

