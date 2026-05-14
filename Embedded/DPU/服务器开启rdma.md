sudo <串口工具>

eg: sudo cutecom



用户名:admin

密码：Password123



conf t

en conf t

roce

wr

copy run st

show roce

int m 0

int

interface mgmt 0



安装OFED

root 

./mlnxofedinstall --force

将缺少的依赖包安装一下



/etc/init.d/openibd restart



cd /etc/init.d/

mlxfwmanager

mst start

mst start

flint -d /dev/mst/mt41686_pciconf0 q

ibstat