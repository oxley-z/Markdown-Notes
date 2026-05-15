---
github.io: true
csdn: false
---
# nmcli

`nmcli`是 `NetworkManager` 的命令行工具。

`nm` 代表 `NetworkManager`，`cli` 代表 `Command-Line` 命令行。

## 语法

```bash
# nmcli [参数] [网卡名]
$ nmcli [OPTIONS] OBJECT { COMMAND | help }
```

## 常用参数

```bash
-f 设置要显示的字段名
-h 显示帮助信息
-p 美观输出信息
-t 简洁输出信息
-v 显示版本信息
```

## 选项

```bash
OPTIONS
	-t[erse]						#	terse	output	简洁的输出
	-p[retty]						#	pretty	output	漂亮的输出
	-m[ode]	tabular| multiline		#	output	mode	输出模式
	-f[ields]	<field1,field2,...>|all|common		#	specify	fields	to	output 指定要输出的字段
	-e[scape]	yes|no				#	escape	columns	separators	in values	在值中转义列分隔符
	-n[ocheck]						#	不要检查nmcli和NetworkManager版本
	-a[sk]							#	要求缺少参数
	-w[ait]	<seconds>				#	设置超时等待整理操作
	-v[ersion]						#	显示程序版本
	-h[elp]							#	打印此帮助
OBJECT
	g[eneral]       # NetworkManager的一般状态和操作
  	n[etworking]    # 整体组网控制
  	r[adio]         # NetworkManager切换开关
  	c[onnection]    # NetworkManager的连接
  	d[evice]        # 由NetworkManager管理的设备
  	a[gent]         # NetworkManager秘密代理或polkit代理
```

## 例子

### 网络连接配置

```bash
$ nmcli connection reload	# 重启服务
$ nmcli c load ifcfg-ens33	# 重载某一指定连接的配置文件
$ nmcli c on				# 开启网络连接
$ nmcli c off				# 关闭网络连接
$ nmcli con up static       # 启用static连接配置
$ nmcli con up default      # 启用default连接配置 
$ nmcli c down ens33		# 关闭指定连接
$ nmcli c delete ens33		# 删除指定连接
```

#### 网络配置文件

网络配置文件默认保存在 `/etc/NetworkManage/SystemConnection/` 路径下，如果配置文件在其它位置，则需要填写完整路径。

网络配置文件的命名方式就是`ifcfg-连接名`，例如 ens33 的配置文件名为`ifcfg-ens33`。

网络配置文件说明：

```bash
TYPE=Ethernet           # 以太网
PROXY_METHOD=none
BROWSER_ONLY=no
BOOTPROTO=none          # 静态配置，等同于 ipv4.method manual
DEFROUTE=yes
IPV4_FAILURE_FATAL=no
IPV6INIT=no
NAME=ens33              # 连接名
UUID=16862343-b55e-4248-b05d-a3ea662a84a4
DEVICE=ens33            # 设备名
ONBOOT=yes              # 开机自启
IPADDR=192.168.80.10    # IP 地址
IPADDR1=192.168.80.100
PREFIX=24               # 子网掩码
PREFIX1=24
GATEWAY=192.168.80.2    # 网关
DNS1=192.168.80.2       # DNS1
DNS2=114.114.114.114    # DNS2
```

#### 创建连接

创建动态获取ip地址的连接，con-name是创建连接的名字，ifname是物理设备，网络接口

```bash
$ nmcli c add type eth con-name dhcp-ens33 ifname ens33
```

对一个指定的网络会话连接添加DNS地址参数

```bash
$ nmcli connection modify company ipv4.dns 8.8.8.8
```

### 修改连接

```bash
$ nmcli c modify ens33  [ + | - ]选项 选项值
$ nmcli c m ens33  [ + | - ]选项 选项值
```

#### 常用示例

```bash
$ nmcli c m ens33 ipv4.address 192.168.80.10/24  # 修改 IP 地址和子网掩码
$ nmcli c m ens33 +ipv4.addresses 192.168.80.100/24
$ nmcli c m ens33 ipv4.method manual             # 修改为静态配置，默认是 auto
$ nmcli c m ens33 ipv4.gateway 192.168.80.2      # 修改默认网关
$ nmcli c m ens33 ipv4.dns 192.168.80.2          # 修改 DNS
$ nmcli c m ens33 +ipv4.dns 114.114.114.114      # 添加一个 DNS
$ nmcli c m ens33 ipv6.method ignored            # 将 IPv6 禁用，针对CentOS8，三个可选项：disabled、auto、manual
$ nmcli c m ens33 connection.autoconnect yes     # 开机启动
```

注意：必须先修改 ipv4.address，然后才能修改 ipv4.method！

用空引号`""`代替选项的值，可将选项设回默认值（以 ipv4.method 为例）

```bash
$ nmcli c m ens33 ipv4.method ""
```

### 查看网络

```bash
$ nmcli connection show				# 查看当前连接状态
$ nmcli connection show -active		# 显示活动的连接
$ nmcli connection show "ens33"		# 显示指定一个网络连接配置
$ nmcli device status				# 显示设备状态
$ nmcli device show					# 显示全部接口属性
$ nmcli device show ens33			# 显示指定接口属性
$ nmcli n connectivity				# 显示网络状态
$ nmcli general status				# 显示系统网络状态
```

网络连接状态共有五种：`full`、`limited`（连网，但无法上网）、`portal`（连网，但需要登陆认证后才能上网）、`none`（没连网）和 `unknown`。

### 查看帮助

```bash
$ nmcli con add help
```

### 修改IP地址

```bash
$ nmcli c mod static-ens33 ip4 192.168.1.12
```

修改后不会立即生效，需要执行

```bash
$ nmcli c up static-ens33
```

## 其他相关命令

1. 查看状态：`systemctl status NetworkManager`
2. 启动：`systemctl start NetworkManager`
3. 重启：`systemctl restart NetworkManager`
4. 关闭：`systemctl stop NetworkManager`
5. 查看是否开机启动：`systemctl is-enabled NetworkManager`
6. 开机启动：`systemctl enable NetworkManager`
7. 禁止开机启动：`systemctl disable NetworkManager`

**注意：**`NetworkManager` 中开头的 N 和中间的 M 必须大写。

# 参考

[关于Linux中网络连接配置(NetworkManager)的一些笔记-云社区-华为云 (huaweicloud.com)](https://bbs.huaweicloud.com/blogs/345720)

[Manage network connections from the Linux command line with nmcli | Opensource.com](https://opensource.com/article/20/7/nmcli)

[nmcli命令详解 - 自然洒脱 - 博客园 (cnblogs.com)](https://www.cnblogs.com/caibao666/p/12321888.html)

[9.2. 使用 NetworkManager 命令行工具 nmcli Red Hat Enterprise Linux 7 | Red Hat Customer Portal](https://access.redhat.com/documentation/zh-cn/red_hat_enterprise_linux/7/html/networking_guide/sec-network_bridging_using_the_networkmanager_command_line_tool_nmcli)