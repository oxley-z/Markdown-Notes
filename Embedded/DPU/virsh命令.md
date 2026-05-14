# virsh命令

---

## 1. 查看当前松油虚拟机和虚拟机状态

```bash
virsh list --all
```

## 2.查看虚拟机状态

```bash
virsh dominfo DomainName
```

## 3. 查看虚拟机当前状态

```bash
virsh domstate DomainName
```

## 4. 查看虚拟机当前配置文件

```bash
virsh dumpxml DomainName
```

## 5. 启动虚拟机

```bash
virsh start DomainName
```

## 6. 重启虚拟机

```bash
virsh reboot DomainName
```

## 7. 暂停虚拟机

```bash
virsh suspend DomainName
```

## 8. 唤醒虚拟机

```bash
virsh resume DomainName
```

## 9. 强制关机

```bash
virsh destroy DomainName
```

## 10. 移除虚拟机

```bash
virsh undefine DomainName
sudo rm -rm /var/lib/libvirt/images/DomainName
```







