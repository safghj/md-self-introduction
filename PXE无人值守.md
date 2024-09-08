# PXE无人值守安装系统

其实就是实现自动批量安装系统

部署服务端，服务端和客户端通过交换机连接在一起

+ 工作原理

![image-20220424153756246](28.kickstrat/image-20220424153756246.png)



# 实验部署

主要是部署服务端

server1去做服务端，添加一块仅主机网卡

![image-20220424153947558](28.kickstrat/image-20220424153947558.png)



![](28.kickstrat/image-20220424154002055.png).

因为等会server1要跟另外一台机器模拟在同一个局域网下，NAT是要上网的，所以不用NAT，用仅主机（客户端也用仅主机）

并且关闭vmware的DHCP，因为等会server1上要部署自己的DHCP

配一下server1的网段

![image-20220424154650081](28.kickstrat/image-20220424154650081.png)

![image-20220424154848673](28.kickstrat/image-20220424154848673.png)



+ 准备网卡

添加一个ens37

```bash
nmcli connection add con-name ens37 type ethernet ifname ens37
```

![image-20220424155259036](28.kickstrat/image-20220424155259036.png)

![image-20220424155343071](28.kickstrat/image-20220424155343071.png)



```bash
[root@server1 ~]# nmcli connection modify ens37 ipv4.addresses 192.168.119.200/24 autoconnect yes ipv4.method manual
 
 [root@server1 ~]# nmcli connection down ens37
 [root@server1 ~]# nmcli connection up ens37
```

![image-20220424155813483](28.kickstrat/image-20220424155813483.png)



+ 配置dhcp服务

```bash
yum -y install dhcp
[root@localhost ~]# vim /etc/dhcp/dhcpd.conf

subnet 192.168.119.0 netmask 255.255.255.0 {
range 192.168.119.100 192.168.119.199;
option subnet-mask 255.255.255.0;
default-lease-time 21600;
max-lease-time 21600;
max-lease-time 43200;
next-server 192.168.119.200;
filename "/pxelinux.0";
}
[root@localhost ~]# systemctl restart dhcpd
systemctl enable dhcpd
```

然后去验证一下，开台server2，把网卡改成仅主机模式

![image-20220424160836877](28.kickstrat/image-20220424160836877.png)

![image-20220424161022814](28.kickstrat/image-20220424161022814.png)

到此说明dhcp没问题



+ 安装tftp

```bash
[root@localhost ~]# yum -y install tftp-server.x86_64
[root@localhost ~]# systemctl start tftp
systemctl enable tftp.socket
ss -unap
```

![image-20220424161207966](28.kickstrat/image-20220424161207966.png)

OK



+ pxe引导配置（核心配置是syslinux）

```bash
[root@localhost ~]# yum -y install syslinux
[root@localhost ~]# cd /var/lib/tftpboot/
[root@localhost tftpboot]#  cp /usr/share/syslinux/pxelinux.0 .
# 挂载磁盘
[root@localhost tftpboot]# mkdir -p /media/cdrom
[root@localhost tftpboot]# mount /dev/cdrom /media/cdrom/
# 把磁盘里的东西复制过来
[root@localhost tftpboot]# cp /media/cdrom/images/pxeboot/{vmlinuz,initrd.img} .
# 然后复制跟镜像相关的一些内容
[root@localhost tftpboot]# cp /media/cdrom/isolinux/{vesamenu.c32,boot.msg} .
```

![image-20220424161425139](28.kickstrat/image-20220424161425139.png)



挂载磁盘之前要先确保磁盘已连接

![image-20220424161546526](28.kickstrat/image-20220424161546526.png)



复制之后

![image-20220424161846954](28.kickstrat/image-20220424161846954.png)



+ 配置syslinux服务程序，这个文件是开机时的选项菜单

```bash
[root@localhost tftpboot]# mkdir pxelinux.cfg
[root@localhost tftpboot]# cp /media/cdrom/isolinux/isolinux.cfg pxelinux.cfg/default
[root@localhost tftpboot]# vim pxelinux.cfg/default
```

![image-20220424162530918](28.kickstrat/image-20220424162530918.png)

改第一行和第64行

```bash
append initrd=initrd.img inst.stage2=ftp://192.168.119.200 ks=ftp:
    //192.168.119.200/pub/ks.cfg quiet
```

![image-20220424162655980](28.kickstrat/image-20220424162655980.png)



因为有写ks，所以还要配置vsftpd服务



+ 配置vsftpd服务

```bash
[root@localhost tftpboot]# yum -y install vsftpd
# 把磁盘下的内容全部拷贝到ftp下面
[root@localhost tftpboot]# cp -r /media/cdrom/* /var/ftp/
```



+ 创建kickstart应答文件

kickstart文件是指定了安装时的一些参数

```bash
[root@localhost tftpboot]# cp ~/anaconda-ks.cfg /var/ftp/pub/ks.cfg
[root@localhost tftpboot]# chmod +r /var/ftp/pub/ks.cfg
[root@localhost tftpboot]# vim /var/ftp/pub/ks.cfg
改第六行和第30行
 5 url --url=ftp://192.168.119.200
 30 clear --all --initlabel
```

![image-20220424163428999](28.kickstrat/image-20220424163428999.png)





![image-20220424163720833](28.kickstrat/image-20220424163720833.png)

访问一下没问题



服务端只需要和客户端连接到同一个网段，客户端的各种配置都会自动设置

