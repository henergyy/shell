````bash
#!/bin/bash
#

# 安装软件
yum -y install vsftpd dhcp tftp-server xinetd syslinux expect

# 部署ftp服务提供系统安装源
ftp() {
	mkdir /var/ftp/centos76
	mount /dev/sr0 /mnt &> /dev/null
	cp -r /mnt/* /var/ftp/centos76/ &
	systemctl start vsftpd
	systemctl enable --now vsftpd &> /dev/null
	if netstat -antp | grep ftp >> /dev/null ; then
		echo "安装源提供成功"
	fi
}

#部署tftp服务
tftp() {
	cp /mnt/isolinux/* /var/lib/tftpboot
	mkdir /var/lib/tftpboot/centos76
	mv /var/lib/tftpboot/vmlinuz /var/lib/tftpboot/initrd.img /var/lib/tftpboot/centos76
# 共享pxelinux.0文件
	cp /usr/share/syslinux/pxelinux.0 /var/lib/tftpboot
# 编辑菜单文件
	mkdir /var/lib/tftpboot/pxelinux.cfg
	mv /var/lib/tftpboot/isolinux.cfg /var/lib/tftpboot/pxelinux.cfg/default
	sed -i '60,/#/d' /var/lib/tftpboot/pxelinux.cfg/default
	echo "label centos76
  menu label Install CentOS76
  kernel centos76/vmlinuz       //以相对路径【相对于tftp的数据目>录】方式指定对应菜单项的内核
  append initrd=centos76/initrd.img inst.stage2=ftp://192.168.126.11/centos76 inst.repo=ftp://192.168.126.11/centos76
  ///centos7之前写到这儿就行append initrd=centos76/initrd.img
" >> /var/lib/tftpboot/pxelinux.cfg/default	
# 启动tftp服务
	sed -i '/disable/c \disable = no' /etc/xinetd.d/tftp
	systemctl restart xinetd
	systemctl enable xinetd
	netstat -anup | grep 69
}


# 配置DHCP服务
dhcp() {
/usr/bin/expect << eof
set timeout 3
spawn cp -r /usr/share/doc/dhcp-4.2.5/dhcpd.conf.example /etc/dhcp/dhcpd.conf 
expect "cp: overwrite ‘/etc/dhcp/dhcpd.conf’?"
send "yes\n"
expect eof
eof
cat << sb >> /etc/dhcp/dhcpd.conf
subnet 192.168.126.0 netmask 255.255.255.0 {
 range 192.168.126.100 192.168.126.200;
option routers 192.168.126.254;
option domain-name-servers 114.114.114.114,223.5.5.5;
next-server 192.168.126.11;
filename "pxelinux.0";
}
sb
systemctl start dhcpd
systemctl enable dhcpd
netstat -luntp | grep dhcp
}

%post
ftp
tftp
dhcp
````

