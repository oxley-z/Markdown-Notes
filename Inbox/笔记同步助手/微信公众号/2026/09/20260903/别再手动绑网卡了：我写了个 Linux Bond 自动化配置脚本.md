---
author: 释然
source: 微信公众号
url: https://mp.weixin.qq.com/s?__biz=Mzg5OTY2MzEyMA==&mid=2247490180&idx=1&sn=b3120c6ea821d953b0a2f4d722bacc3e&chksm=c19829538baadab75f582e5816b461888bb4160844197ce0e4daa43f08d767215581ea56de74&mpshare=1&scene=1&srcid=0903KQYMYDL3NR75O98uDVeD&sharer_shareinfo=ba86dab18ee08e5bf90d43362ba28861&sharer_shareinfo_first=ba86dab18ee08e5bf90d43362ba28861#rd
saved: 2026-09-03 15:19:38
tags:
  - 笔记同步助手
id: 08c4d9b3-1843-4315-9ec7-3bcc8644e7d0
---

公众号名称：Linux运维进阶之路

作者名称：释然

发布时间：2026-09-03 08:08

服务器有多块网卡时，通常会通过 Bond 实现主备切换或负载均衡。

但 CentOS 常用 `nmcli`，Ubuntu 常用 Netplan，加上网卡名称、IP 和 Bond 模式各不相同，手动配置容易出错。

因此整理了一个交互式 Linux Bond 配置脚本，无需联网，也不用额外安装工具。

## 主要功能

-   支持 NetworkManager 和 Netplan
    

-   支持 CentOS、RHEL、Rocky Linux、AlmaLinux、Ubuntu
    

-   自动识别可用网卡
    

-   支持选择两块或更多成员网卡
    

-   支持静态 IP和 DHCP
    

-   自动设置网关和 DNS
    

-   自动清理成员网卡原有 IP、路由和旧连接
    

-   配置前自动备份
    

-   默认只预览，不直接修改网络
    

支持七种 Bond 模式：

-   ```
    active-backup
    ```
    

-   ```
    balance-rr
    ```
    

-   ```
    balance-xor
    ```
    

-   ```
    broadcast
    ```
    
-   802.3ad
    

-   ```
    balance-tlb
    ```
    

-   ```
    balance-alb
    ```
    

一般服务器高可用场景，推荐使用 `active-backup`。该模式通常不需要交换机配置。

```
#!/usr/bin/env bash
# Configure a Linux bond using tools already present on the host.
# Supported backends: NetworkManager (nmcli) and Netplan.

set -Eeuo pipefail

PROGRAM=${0##*/}
BACKEND=""
APPLY=0

die() { printf '错误：%s\n' "$*" >&2; exit 1; }
info() { printf '[信息] %s\n' "$*"; }

usage() {
  cat <<EOF
用法：sudo bash $PROGRAM [--apply] [--backend nmcli|netplan]

默认仅收集参数并预览配置；带 --apply 时，确认后备份并应用。
脚本不会联网，也不会安装任何软件。
EOF
}

while (($#)); do
  case "$1" in
    --apply) APPLY=1 ;;
    --backend) shift; (($#)) || die "--backend 缺少参数"; BACKEND=$1 ;;
    -h|--help) usage; exit 0 ;;
    *) die "未知参数：$1" ;;
  esac
  shift
done

[[ ${EUID:-$(id -u)} -eq 0 ]] || die "请使用 root 权限运行"
command -v ip >/dev/null 2>&1 || die "系统缺少 ip 命令"

if [[ -z $BACKEND ]]; then
  AVAILABLE_BACKENDS=()
  command -v nmcli >/dev/null 2>&1 && nmcli -t general status >/dev/null 2>&1 && AVAILABLE_BACKENDS+=(nmcli)
  command -v netplan >/dev/null 2>&1 && AVAILABLE_BACKENDS+=(netplan)
  ((${#AVAILABLE_BACKENDS[@]})) || die "未发现可用的 nmcli 或 netplan；为避免改坏网络，脚本不会自动安装工具"
  printf '选择网络配置后端：\n'
  for i in "${!AVAILABLE_BACKENDS[@]}"; do printf '  %d) %s\n' "$((i + 1))" "${AVAILABLE_BACKENDS[$i]}"; done
  while :; do
    read -r -p '后端 [1]：' choice
    choice=${choice:-1}
    [[ $choice =～ ^[0-9]+$ ]] && ((choice >= 1 && choice <= ${#AVAILABLE_BACKENDS[@]})) && { BACKEND=${AVAILABLE_BACKENDS[$((choice - 1))]}; break; }
    printf '选择无效。\n' >&2
  done
fi
[[ $BACKEND == nmcli || $BACKEND == netplan ]] || die "不支持的后端：$BACKEND"
command -v "$BACKEND" >/dev/null 2>&1 || die "系统中没有 $BACKEND"

mapfile -t IFACES < <(
  for path in /sys/class/net/*; do
    [[ -d $path ]] || continue
    name=${path##*/}
    [[ $name == lo || -d "$path/bonding" ]] && continue
    # Bond、bridge、veth 等软件接口没有 device 链接，不作为成员网卡列出。
    [[ -e $path/device ]] || continue
    printf '%s\n' "$name"
  done | sort
)
((${#IFACES[@]} >= 2)) || die "至少需要两块可用网卡"

printf '\n检测到以下网卡：\n'
for i in "${!IFACES[@]}"; do
  state=$(cat "/sys/class/net/${IFACES[$i]}/operstate" 2>/dev/null || printf unknown)
  mac=$(cat "/sys/class/net/${IFACES[$i]}/address" 2>/dev/null || printf unknown)
  printf '  %2d) %-16s 状态=%-8s MAC=%s\n' "$((i + 1))" "${IFACES[$i]}" "$state" "$mac"
done

while :; do
  read -r -p '选择加入 Bond 的网卡（至少两块，序号用逗号分隔，如 1,2）：' iface_choices
  IFS=, read -r -a selected <<<"$iface_choices"
  SLAVES=()
  valid_selection=1
  for choice in "${selected[@]}"; do
    choice=${choice//[[:space:]]/}
    if [[ $choice =～ ^[0-9]+$ ]] && ((choice >= 1 && choice <= ${#IFACES[@]})); then
      iface=${IFACES[$((choice - 1))]}
      duplicate=0
      if ((${#SLAVES[@]})); then
        for existing_iface in "${SLAVES[@]}"; do
          [[ $existing_iface == "$iface" ]] && { duplicate=1; break; }
        done
      fi
      ((duplicate == 1)) || SLAVES+=("$iface")
    else
      valid_selection=0
    fi
  done
  ((valid_selection == 1 && ${#SLAVES[@]} >= 2)) && break
  printf '选择无效，必须选择至少两块不同网卡。\n' >&2
done
PRIMARY=${SLAVES[0]}

read -r -p 'Bond 名称 [bond0]：' BOND
BOND=${BOND:-bond0}
[[ $BOND =～ ^[a-zA-Z0-9_.-]+$ && $BOND != lo ]] || die "Bond 名称不合法"
[[ ! -e /sys/class/net/$BOND || -d /sys/class/net/$BOND/bonding ]] || die "$BOND 已存在且不是 Bond 接口"

printf '\n选择 Bond 模式：\n'
printf '  1) active-backup  主备（无需交换机配置，推荐）\n'
printf '  2) balance-rr     轮询\n'
printf '  3) balance-xor    XOR 负载均衡\n'
printf '  4) broadcast      广播\n'
printf '  5) 802.3ad        LACP（需要交换机配置）\n'
printf '  6) balance-tlb    自适应发送负载均衡\n'
printf '  7) balance-alb    自适应负载均衡\n'
while :; do
  read -r -p '模式 [1]：' MODE_CHOICE
  case ${MODE_CHOICE:-1} in
    1) BOND_MODE=active-backup; break ;;
    2) BOND_MODE=balance-rr; break ;;
    3) BOND_MODE=balance-xor; break ;;
    4) BOND_MODE=broadcast; break ;;
    5) BOND_MODE=802.3ad; break ;;
    6) BOND_MODE=balance-tlb; break ;;
    7) BOND_MODE=balance-alb; break ;;
    *) printf '请输入 1～7。\n' >&2 ;;
  esac
done

if [[ $BOND_MODE == 802.3ad ]]; then
  printf '注意：802.3ad 必须在交换机侧配置相匹配的 LACP 聚合。\n'
fi

XMIT_HASH_POLICY=""
if [[ $BOND_MODE == balance-xor || $BOND_MODE == 802.3ad ]]; then
  printf '哈希策略：1) layer2  2) layer2+3  3) layer3+4\n'
  while :; do
    read -r -p '策略 [1]：' HASH_CHOICE
    case ${HASH_CHOICE:-1} in
      1) XMIT_HASH_POLICY=layer2; break ;;
      2) XMIT_HASH_POLICY=layer2+3; break ;;
      3) XMIT_HASH_POLICY=layer3+4; break ;;
      *) printf '请输入 1～3。\n' >&2 ;;
    esac
  done
fi

valid_ipv4() {
  local ip=$1 part
  [[ $ip =～ ^([0-9]{1,3}\.){3}[0-9]{1,3}$ ]] || return 1
  IFS=. read -r -a parts <<<"$ip"
  for part in "${parts[@]}"; do ((10#$part <= 255)) || return 1; done
}
valid_cidr() {
  local value=$1 ip prefix
  [[ $value == */* ]] || return 1
  ip=${value%/*}; prefix=${value##*/}
  valid_ipv4 "$ip" && [[ $prefix =～ ^[0-9]+$ ]] && ((prefix <= 32))
}

printf '\nIPv4 配置：1) 静态地址  2) DHCP\n'
while :; do
  read -r -p '方式 [1]：' IP_CHOICE
  case ${IP_CHOICE:-1} in 1) IP_METHOD=static; break ;; 2) IP_METHOD=dhcp; break ;; *) printf '请输入 1 或 2。\n' >&2 ;; esac
done
ADDRESS=""; GATEWAY=""; DNS=""; DNS_LIST=()
if [[ $IP_METHOD == static ]]; then
  while :; do read -r -p '静态 IPv4/CIDR（例 192.168.116.200/24）：' ADDRESS; valid_cidr "$ADDRESS" && break; echo '格式错误。'; done
  while :; do read -r -p 'IPv4 网关：' GATEWAY; valid_ipv4 "$GATEWAY" && break; echo '格式错误。'; done
  read -r -p 'DNS，逗号分隔 [8.8.8.8,114.114.114.114]：' DNS
  DNS=${DNS:-8.8.8.8,114.114.114.114}
  IFS=, read -r -a DNS_LIST <<<"$DNS"
  for dns in "${DNS_LIST[@]}"; do valid_ipv4 "${dns//[[:space:]]/}" || die "DNS 地址不合法：$dns"; done
fi

FAIL_OVER_MAC=""
if [[ $BOND_MODE == active-backup ]]; then
  read -r -p '是否启用 fail_over_mac=active（虚拟机常用）？[y/N]：' answer
  [[ $answer =～ ^[Yy]$ ]] && FAIL_OVER_MAC=active
fi

printf '\n===== 配置预览 =====\n'
printf '后端：%s\nBond：%s（%s）\n成员网卡：%s\nIPv4：%s\n' "$BACKEND" "$BOND" "$BOND_MODE" "${SLAVES[*]}" "$IP_METHOD"
if [[ $IP_METHOD == static ]]; then printf '地址：%s\n网关：%s\nDNS：%s\n' "$ADDRESS" "$GATEWAY" "$DNS"; fi
[[ -n $XMIT_HASH_POLICY ]] && printf '哈希策略：%s\n' "$XMIT_HASH_POLICY"
printf 'fail_over_mac：%s\n' "${FAIL_OVER_MAC:-默认值}"
printf '将清理的成员接口：%s（原 IP、路由及绑定连接）\n' "${SLAVES[*]}"

if ((APPLY == 0)); then
  info "当前为预览模式，未修改系统。确认参数后使用：sudo bash $PROGRAM --apply"
  exit 0
fi

printf '\n警告：应用网络配置可能导致当前 SSH 会话中断。\n'
read -r -p '确认已有控制台或带外管理，并继续应用？请输入 APPLY：' CONFIRM
[[ $CONFIRM == APPLY ]] || die "用户取消"

STAMP=$(date +%Y%m%d-%H%M%S)
BACKUP_DIR="/var/backups/bond-config-$STAMP"
mkdir -p "$BACKUP_DIR"

backup_runtime_network() {
  ip -details address show >"$BACKUP_DIR/ip-address.txt"
  ip route show table all >"$BACKUP_DIR/ip-route.txt"
  ip rule show >"$BACKUP_DIR/ip-rule.txt"
}

clear_slave_runtime() {
  local iface
  for iface in "${SLAVES[@]}"; do
    info "清除成员网卡 $iface 的现有 IP 和路由"
    ip address flush dev "$iface" || true
    ip route flush dev "$iface" || true
  done
}

apply_nmcli() {
  local iface conn uuid bound_iface opts="mode=$BOND_MODE,miimon=100"
  [[ $BOND_MODE == active-backup ]] && opts+=",primary=$PRIMARY,primary_reselect=always"
  [[ -n $XMIT_HASH_POLICY ]] && opts+=",xmit_hash_policy=$XMIT_HASH_POLICY"
  [[ -n $FAIL_OVER_MAC ]] && opts+=",fail_over_mac=$FAIL_OVER_MAC"

  if [[ -d /etc/NetworkManager/system-connections ]]; then
    cp -a /etc/NetworkManager/system-connections "$BACKUP_DIR/"
  fi
  nmcli -f all connection show >"$BACKUP_DIR/nmcli-connections.txt"
  backup_runtime_network

  if nmcli -g NAME connection show | grep -Fxq "$BOND"; then
    nmcli connection delete "$BOND"
  fi
  # 删除所有明确绑定到成员网卡的连接，包括当前未激活但可能自启动的旧配置。
  for iface in "${SLAVES[@]}"; do
    while IFS= read -r uuid; do
      [[ -n $uuid ]] || continue
      bound_iface=$(nmcli -g connection.interface-name connection show "$uuid" 2>/dev/null || true)
      if [[ $bound_iface == "$iface" ]]; then
        conn=$(nmcli -g connection.id connection show "$uuid" 2>/dev/null || printf '%s' "$uuid")
        info "删除网卡 $iface 的旧连接：$conn"
        nmcli connection delete uuid "$uuid"
      fi
    done < <(nmcli -g UUID connection show)
  done

  nmcli connection add type bond con-name "$BOND" ifname "$BOND" bond.options "$opts"
  for iface in "${SLAVES[@]}"; do
    nmcli connection add type ethernet con-name "$BOND-$iface" ifname "$iface" master "$BOND" slave-type bond
    nmcli connection modify "$BOND-$iface" connection.autoconnect yes
  done
  if [[ $IP_METHOD == static ]]; then
    nmcli connection modify "$BOND" ipv4.method manual ipv4.addresses "$ADDRESS" ipv4.gateway "$GATEWAY" ipv4.dns "$DNS"
  else
    nmcli connection modify "$BOND" ipv4.method auto
  fi
  # CentOS 7 / NetworkManager 1.x does not support ipv6.method=disabled.
  # "ignore" is accepted by both old and current NetworkManager releases and
  # leaves IPv6 management alone without requiring an additional package.
  nmcli connection modify "$BOND" ipv6.method ignore connection.autoconnect yes
  # 旧版 NetworkManager 可能不会随 master 自动拉起 slave。
  # 属性存在时启用自动拉起；不支持该属性的版本由下面的显式 up 兜底。
  nmcli connection modify "$BOND" connection.autoconnect-slaves 1 2>/dev/null || true
  clear_slave_runtime
  nmcli connection up "$BOND"
  for iface in "${SLAVES[@]}"; do
    nmcli connection up "$BOND-$iface"
  done
  nmcli connection up "$BOND"
}

apply_netplan() {
  local file="/etc/netplan/99-$BOND.yaml" dns_yaml="" dns iface interfaces_yaml=""
  cp -a /etc/netplan "$BACKUP_DIR/"
  backup_runtime_network
  if [[ $IP_METHOD == static ]]; then
    for dns in "${DNS_LIST[@]}"; do
      dns=${dns//[[:space:]]/}
      dns_yaml+="${dns_yaml:+, }$dns"
    done
  fi
  for iface in "${SLAVES[@]}"; do interfaces_yaml+="${interfaces_yaml:+, }$iface"; done
  cat >"$file" <<EOF
network:
  version: 2
  renderer: networkd
  ethernets:
EOF
  for iface in "${SLAVES[@]}"; do
    cat >>"$file" <<EOF
    $iface:
      dhcp4: false
      dhcp6: false
      addresses: []
      routes: []
      nameservers:
        addresses: []
EOF
  done
  cat >>"$file" <<EOF
  bonds:
    $BOND:
      interfaces: [$interfaces_yaml]
EOF
  if [[ $IP_METHOD == static ]]; then
    cat >>"$file" <<EOF
      addresses: [$ADDRESS]
      routes:
        - to: default
          via: $GATEWAY
      nameservers:
        addresses: [$dns_yaml]
EOF
  else
    printf '      dhcp4: true\n' >>"$file"
  fi
  cat >>"$file" <<EOF
      parameters:
        mode: $BOND_MODE
        mii-monitor-interval: 100
EOF
  if [[ $BOND_MODE == active-backup ]]; then
    printf '        primary: %s\n' "$PRIMARY" >>"$file"
    printf '        primary-reselect-policy: always\n' >>"$file"
  fi
  if [[ -n $XMIT_HASH_POLICY ]]; then
    printf '        transmit-hash-policy: %s\n' "$XMIT_HASH_POLICY" >>"$file"
  fi
  if [[ -n $FAIL_OVER_MAC ]]; then
    printf '        fail-over-mac-policy: active\n' >>"$file"
  fi
  chmod 600 "$file"
  netplan generate || { rm -f "$file"; die "Netplan 校验失败；原配置未应用，备份位于 $BACKUP_DIR"; }
  clear_slave_runtime
  netplan apply
}

"apply_$BACKEND"

sleep 2
[[ -r /proc/net/bonding/$BOND ]] || die "未发现 /proc/net/bonding/$BOND，请通过控制台检查；备份位于 $BACKUP_DIR"
ACTUAL_SLAVES=$(cat "/sys/class/net/$BOND/bonding/slaves" 2>/dev/null || true)
for iface in "${SLAVES[@]}"; do
  [[ " $ACTUAL_SLAVES " == *" $iface "* ]] || die "网卡 $iface 未成功加入 $BOND；请通过控制台检查，备份位于 $BACKUP_DIR"
done
printf '\n===== Bond 状态 =====\n'
cat "/proc/net/bonding/$BOND"
ip -br address show "$BOND"
ip -br link show "$BOND"
ip route show default
info "配置已应用；原配置备份：$BACKUP_DIR"
```

**「bond自动化配置脚本」**

链接：<u>https://pan.quark.cn/s/1af5e0bc7b4d</u>

## 使用方法

先查看网卡情况

```
ip -br add
```

![[Inbox/笔记同步助手/微信公众号/2026/09/images/8b77d9b75a34908edf7984d1d3d825da_MD5.jpg]]

![[Inbox/笔记同步助手/微信公众号/2026/09/images/089010ad331daefb96e89e89d5968a81_MD5.jpg]]

预览配置：

```
sudo bash configure-bond.sh
```

![[Inbox/笔记同步助手/微信公众号/2026/09/images/3ffb3186081d62cc822bc8879c43c103_MD5.jpg]]

确认无误后应用：

```
sudo bash configure-bond.sh --apply
```

按照提示选择网卡：

```
选择加入 Bond 的网卡：1,2
```

然后选择 Bond 模式、IP获取方式、网关和 DNS。应用前还需输入 `APPLY` 二次确认。![[Inbox/笔记同步助手/微信公众号/2026/09/images/5a616476ef744c22d7eb439dfb08541f_MD5.jpg]]

## 配置验证

查看 Bond 状态：

```
cat /proc/net/bonding/bond100
```

![[Inbox/笔记同步助手/微信公众号/2026/09/images/e0320a465a69514d4ef795d791acea6e_MD5.jpg]]

查看 IP 和路由：

```
ip -br addr show bond100
ip route
```

![[Inbox/笔记同步助手/微信公众号/2026/09/images/297593e170cf6f53a8e9113241153061_MD5.jpg]]

查看当前活动网卡：

```
cat /sys/class/net/bond100/bonding/active_slave
```

![[Inbox/笔记同步助手/微信公众号/2026/09/images/6afe9ac2827ec46c549dd6b4e4627925_MD5.jpg]]

正常情况下会看到：

```
Bonding Mode: fault-tolerance (active-backup)
Currently Active Slave: ens33
MII Status: up
```

## 注意事项

如果当前 SSH 使用的是所选网卡，应用配置时可能断开。生产环境操作前，请准备 Console、iDRAC、iLO 或 IPMI。

网络配置可以自动化，但配置前备份、应用前确认和配置后验证，一个都不能少。

  

---

内容效果不满意？[点此反馈](https://feedback.notebooksyncer.com/feedback/a575b265_1788419975785?u=https%3A%2F%2Fmp.weixin.qq.com%2Fs%3F__biz%3DMzg5OTY2MzEyMA%3D%3D%26mid%3D2247490180%26idx%3D1%26sn%3Db3120c6ea821d953b0a2f4d722bacc3e%26chksm%3Dc19829538baadab75f582e5816b461888bb4160844197ce0e4daa43f08d767215581ea56de74%26mpshare%3D1%26scene%3D1%26srcid%3D0903KQYMYDL3NR75O98uDVeD%26sharer_shareinfo%3Dba86dab18ee08e5bf90d43362ba28861%26sharer_shareinfo_first%3Dba86dab18ee08e5bf90d43362ba28861%23rd&s=obsidian)