# 飞腾uboot编译

## uboot编译

```bash
make D2000_config
make ARCH=arm CROSS_COMPILE=/opt/gcc-linaro-6.1.1-2016.08-x86_64_aarch64-linux-gnu/bin/aarch64-linux-gnu-
```

交叉编译工具链下载 [Linaro Releases](http://releases.linaro.org/components/toolchain/binaries/6.1-2016.08/aarch64-linux-gnu/)

## uboot打包

```bash
cd image_fix
cp xxx/u-boot.bin ./bl33_new.bin
./my_scripts/fix_parameter.sh
./my_scripts/image-fix.sh
mv ./bl33_new.bin ./6d02_0806uboot.bin
```

# 常见问题

## 无法访问64位内存空间超过2G内存部分

### 解决方法

修改uboot源码d2000.h中PHY_SDRAM_2_SIZE宏的大小至超过2G，具体大小根据需要修改。

## 执行./my_scripts/image-fix.sh出现错误如下

```bash
my_scripts/UpdateBIOS.c:1:10: fatal error: openssl/md5.h: 没有那个文件或目录
    1 | #include <openssl/md5.h>
      |          ^~~~~~~~~~~~~~~
compilation terminated.
```

### 解决方法

安装libssl-dev包即可



# 参考

《D2000 固件编译打包说明V1.0.6.pdf》
