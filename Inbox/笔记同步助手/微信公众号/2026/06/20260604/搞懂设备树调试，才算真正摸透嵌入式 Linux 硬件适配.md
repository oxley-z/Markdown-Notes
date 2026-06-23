---
author: Debug 蟹老板
source: 微信公众号
url: https://mp.weixin.qq.com/s?__biz=MzkzNDk2NTUwOQ==&mid=2247486244&idx=1&sn=a04226a1bf63425b51d33934cd97bd5f&chksm=c39dbac11e43937d04c5a42655b841639fc9a0b363aca96549fd4e0f6c9cf474130be84fc1ca&mpshare=1&scene=1&srcid=0604oD45wQVURTJ8nvdWyYXj&sharer_shareinfo=56f511fb50501066d9fded686f6f5c93&sharer_shareinfo_first=56f511fb50501066d9fded686f6f5c93#rd
saved: 2026-06-04 08:34:55
tags:
  - 笔记同步助手
id: 72b4b2ff-affc-4197-a8ac-c93776ff76df
---

公众号名称：Linux教程

作者名称：Debug 蟹老板

发布时间：2026-05-12 19:55

![[../../../../images/d5673b61529048fc2392d6b1f5e4601a_MD5.gif]]

大家好，我是蟹老板～

有些东西吧，第一次接触的时候，你会觉得它设计得特别优雅。

比如 Linux 设备树。

再比如 Kubernetes。

还有前几年很火的微服务。

后来你会发现，优雅这俩字，往往和“折磨”是绑定销售的。

设备树（Device Tree）看似简单，但调试起来就像在迷宫里找出口：节点配置错一点系统就跪，属性值写错驱动就挂，日志满天飞却找不到关键线索……

今天这篇文章，我会把我所知道的**8个调试技巧**写出来！

## 一、为什么设备树调试让人头大？

先讲一个真实场景，你在一个 i.MX6 的板子上写 LCD 驱动。设备树里加了一堆节点：

```
&lcdif {
    pinctrl-names = "default";
    pinctrl-0 = <&pinctrl_lcd>;
    display = <&display0>;
    status = "okay";
};
```

编译，烧录，启动。

内核 log 里啥也没有，连个错误都没有，就像你根本没写过这段代码。

纳尼～ 你开始怀疑人生了？是不是编译没生效？是不是烧错了镜像？是不是内核配置有问题？

折腾了半天，最后发现 —— **节点里的 `compatible` 属性拼错了**。驱动在找 `"fsl,imx6q-lcdif"`，你写成了 `"fsl,imx6q-lcd"`。

少了一个 `if`。

就这么一个字母，烧了你两个小时。

**设备树调试最大的痛点就是它不报错。** 或者说，报错的方式特别“优雅” —— 它只是默默地不匹配，然后把你的驱动当作不存在。

所以我们需要一套**主动的、系统性的调试方法**。

下面这些技巧，就是用来解决这个问题的。

---

## 二、基础：设备树长什么样？

在讲调试技巧之前，花 30 秒回顾一下设备树的基本结构。

![[../../../../images/87a962240265c734d635f50173c22830_MD5.jpg]]

懂的老铁可以跳过这节。

设备树是一个**描述硬件**的树形结构。根节点下面挂着 CPU、内存、外设等。

一个典型的节点长这样：

```
uart1: serial@02020000 {
    compatible = "fsl,imx6q-uart", "fsl,imx21-uart";
    reg = <0x02020000 0x1000>;
    interrupts = <0 26 4>;
    clocks = <&clks IMX6QDL_CLK_UART_IPG>;
    status = "disabled";
};
```

-   • `compatible`：驱动匹配用的“身份证”
    
-   • `reg`：寄存器地址和大小
    
-   • `interrupts`：中断号
    
-   • `status`：是否启用
    

注意那个`&`符号 —— 它引用的是另一个节点。这在 overlay 调试里特别重要。

---

## 三、技巧一：反编译 dtb

### 3.1 这个动作应该被做成肌肉记忆

很多人的误区是：**只敢看设备树源码**。

但实际跑在系统里的设备树，是 `.dtb` 二进制文件。这个文件可能跟你源码里的 `.dts`**不一致**。

有几种情况：

1.  1\. 你忘了重新编译 —— 烧进去的还是旧的。
    
2.  2\. 编译过程有警告被忽略了，dtc 帮你“修正”了一些写法。
    
3.  3\. 其他 overlay 动态加载了片段。
    

**所以第一件事：从正在运行的系统里，把 dtb 拽出来反编译。**

命令非常非常简单：

```
# 方法一：如果 /sys/firmware/fdt 存在
dtc -I dtb -O dts /sys/firmware/fdt > current.dts

# 方法二：没有上面那个文件？找找 /proc/device-tree
# 先看看系统用的 dtb 在哪儿
ls -l /proc/device-tree
```

`/sys/firmware/fdt` 这个文件，**是被内核直接暴露出来的**原始 dtb。只要你的内核编译时打开了 `CONFIG_PROC_DEVICETREE`（大多数发行版都开了），它就一直在那儿等你。

你可能会问：`/proc/device-tree` 不是也能看吗？为什么要反编译？

因为 `/proc/device-tree` 是**被内核解析后的视图**。它已经丢失了一些原始信息，比如 phandle 的数值、某些属性的原始顺序等。

反编译出来的 `.dts` 才是最原汁原味的。

### 3.2 反编译后怎么查？

反编译出来的文件很长。但你只需要看重点。

**第一招：grep 你的节点名**

```
dtc -I dtb -O dts /sys/firmware/fdt | grep -A 20 "uart1"
```

**第二招：看属性值是否被篡改**

有时候 dtc 编译器会自动帮你“优化”。比如：

```
// 你写的
interrupts = <1 2 3>;

// 编译后可能变成
interrupts = <0x01 0x02 0x03>;
```

数值上没区别，但如果你用字符串比较工具，就会出问题。

**第三招：检查 phandle 引用**

看这段：

```
clocks = <&clks 42>;
```

反编译后，`&clks` 会被转成一个数字，比如 `&clks` → `phandle = <0x88>`。

如果你看到某个引用变成了 `<0xffffffff>`，那就是 phandle 解析失败了 —— 这是非常隐蔽的错误。

**一个真实案例**：

之前调试一个 I2C 触摸屏，驱动死活读不到触摸数据。反编译后发现了这个：

```
touch@38 {
    compatible = "goodix,gt911";
    reg = <0x38>;
    interrupt-parent = <0xffffffff>;  // 注意这里！
    interrupts = <13 2>;
};
```

`interrupt-parent` 变成了 `0xffffffff` —— 无效的 phandle。

原因是设备树里引用的中断控制器节点被 `status = "disabled"` 了，但引用时没有做条件判断。编译阶段没报错，链接阶段也没报错。**它就这样静默地失败了**。

**所以我的建议是**，每次遇到设备树相关的问题，**先反编译现场 dtb 看一眼**。这个动作应该被做成肌肉记忆。

---

## 四、技巧二：procfs

### 4.1 /proc/device-tree 里有什么？

如果你不想反编译，直接进 `/proc/device-tree` 也能看。

这是内核帮你“展开”的设备树视图。表现形式是**文件系统**：

```
cd /proc/device-tree
ls -l
```

你会看到一堆像目录一样的节点。每个节点对应一个目录。每个属性对应一个文件。

举个例子，查看根节点的 `compatible` 属性：

```
cat /proc/device-tree/compatible
```

输出可能是：

```
fsl,imx6q-sabresd
fsl,imx6q
```

这实际上是一个字符串数组，用 `\0` 分隔。cat 出来时会把第一个 `\0` 当作结束符，所以你只能看到第一段。想看全的用 `od`：

```
od -c /proc/device-tree/compatible
```

### 4.2 怎么利用这个“上帝视角”？

**场景一：确认你的节点是否真的被加载了**

```
ls /proc/device-tree/   # 看看根下有哪些节点
ls /proc/device-tree/soc@0/   # 看看某个子节点
```

如果你的节点路径是 `/soc@0/uart@02020000`，但你在 `/proc/device-tree` 里找不到 `uart@02020000` 目录 —— 说明节点没有被内核解析。

可能的原因：

-   • `status = "disabled"`（最常见）
    
-   • 编译时被 dtc 剔除了（有条件编译指令 `#ifdef` 之类的）
    
-   • 父节点被 disabled 了
    

**场景二：检查属性值类型**

属性有三种类型：字符串、数字数组、字节数组。

怎么判断？看文件是普通文件还是目录。

-   • 普通文件 = 属性
    
-   • 目录 = 子节点
    

一个非常有用的命令：

```
find /proc/device-tree -name "compatible" -exec cat {} \; -print
```

这会递归所有 `compatible` 属性，并打印出路径。可以快速看到哪些驱动被匹配上了。

**场景三：检查 phandle 是否有效**

每个有 `phandle` 属性的节点，在 `/proc/device-tree/` 下会有一个同名文件。例如：

```
ls /proc/device-tree/__symbols__/
```

这个 `__symbols__` 目录（如果存在）记录了所有的标签（label）到 phandle 的映射。

你可以对比一下你的引用是否在这个表里。

**吐槽时间**：我知道有老铁会说“我直接用 `dtc` 反编译不好吗？”好，当然好。但是 `/proc/device-tree` 有一个反编译做不到的优势 —— **它是活的**。如果你动态加载了设备树 overlay，`/proc/device-tree` 会实时更新。而 `/sys/firmware/fdt` 不会。

所以这两个方法，一个静态（dtb），一个动态（procfs），**建议配合使用**。

---

## 五、技巧三：debugfs

### 5.1 打开 debugfs 开关

说实话，这个技巧知道的人不多。真的有点东西。

debugfs 里有一个非常宝藏的接口：`/sys/kernel/debug/devicetree/`。

但很多内核编译时默认**不打开** debugfs。你得先检查一下：

```
mount | grep debugfs
```

如果没有挂载，手动挂：

```
mount -t debugfs none /sys/kernel/debug
```

或者确认内核配置：

```
CONFIG_DEBUG_FS=y
```

### 5.2 有了 debugfs，你可以做这些事

先看一眼：

```
ls /sys/kernel/debug/devicetree/
```

我敢打赌你会看到一个叫 `unflatten` 或 `dtb` 的文件。不同内核版本略有差异。

**最常用的是 `unflatten` 这个接口**。

你能想象吗 —— 你可以把一段设备树文本字符串直接写进去，内核会**实时解析并合并到现有的设备树中**。不需要重新编译 dtb，不需要重启！

这个在调试期间简直神了。

用法：

```
# 创建一个测试片段
echo "/ {
    test-node {
        compatible = \"test,device\";
        status = \"okay\";
    };
};" > /sys/kernel/debug/devicetree/unflatten
```

然后去 `/proc/device-tree/` 下看，`test-node` 已经在了！

**但是注意**：这个操作不会写入持久化存储，重启就没了。**而且有可能把系统搞挂**，不建议在产品设备上用。但在开发板上，这是调试神器。

**另一个宝藏文件：`/sys/kernel/debug/devicetree/dt_regions`**

这个文件显示了所有动态加载的 overlay 的内存区域（`reserved-memory` 相关）。

如果你发现某个 overlay 加载失败，去这个文件看一眼，能发现是不是内存冲突了。

**真实案例**：

之前有一个项目，加载第二个 overlay 时内核报错：“Failed to allocate memory for overlay”。

所有人都以为是内存不够。看 `dt_regions` 才发现 —— 第一个 overlay 申请的内存区域跟第二个 overlay 的基址重叠了。但是设备树里明明写的 `reg = <0x40000000 0x1000>`，为什么会重叠？

因为第一个 overlay 在释放时没有正确清理 `reserved-memory` 节点。**这是内核的一个已知 bug**（某些 4.x 版本）。解决方案是手动在 overlay 的 remove 函数里释放。

这个 bug 是 debugfs 帮我发现的。你说神不神？

---

## 六、技巧四：设备树 overlay 调试

overlay （以下简称 OVL，我也不知道这么说对不对）是现代嵌入式 Linux 的一个热门话题。它允许你在**运行时**动态增加、修改设备树节点。

听起来很美好对不对？

但调试 overlay 的难度，比静态设备树**高出一个数量级**。

### 6.1 先确认 overlay 是否真的被加载了

overlay 加载通常用 `configfs` 接口：

```
# 挂载 configfs
mount -t configfs configfs /config

# 创建 overlay
mkdir /config/device-tree/overlays/panel

# 把 dtbo 文件写进去
cat panel.dtbo > /config/device-tree/overlays/panel/dtbo
```

加载成功后，你会在 `/proc/device-tree/` 下看到新节点。如果没有出现，说明加载失败。

**查看加载状态**：

```
cat /config/device-tree/overlays/panel/status
```

可能的输出：`loaded`, `unloaded`, `error`。

如果是 `error`，看内核日志：

```
dmesg | tail -20
```

**最坑的错误**：overlay 里用到的符号（phandle）在基础设备树里不存在。

举个例子，你在 overlay 里写：

```
&i2c1 {
    touch@38 {
        compatible = "goodix,gt911";
        reg = <0x38>;
    };
};
```

基础设备树里必须有 `i2c1` 这个标签，并且它已经被 `status = "okay"` 了。

如果基础设备树里没有 `i2c1`，或者它被 `disabled` 了，overlay 加载时会报错：

```
OF: overlay: failed to resolve symbol i2c1
```

**这个错误很容易被忽略**，因为 dmesg 里可能只有一行，而且不会导致内核崩溃。

### 6.2 overlay 碎片化调试

overlay 加载后，它的节点跟基础设备树是**物理上分在两个内存区域**的。遍历设备树时，`of_find_node_by_path` 这类函数可以跨区域找到节点。但有些 API（比如老旧版本的 `of_get_child_by_name`）可能只查基础区域。

这就导致了“明明节点存在，但驱动 find 不到”的玄学问题。

**调试方法**：

在驱动里加打印，遍历所有子节点：

```
struct device_node *np = NULL;
for_each_child_of_node(root, np) {
    pr_info("node: %s, full_name: %s\n", np->name, np->full_name);
}
```

如果发现 overlay 节点的 `full_name` 带了一个奇怪的后缀（如 `@0` 变成了 `@0/something`），说明是重叠区域的路径解析出了问题。

**解决方案**：使用 `of_find_node_by_phandle` 替代路径查找，或者在内核配置中打开 `CONFIG_OF_OVERLAY` 和 `CONFIG_OF_RESOLVE`（很多 BSP 忘了开后者）。

### 6.3 一个让老手都翻车的例子

有一个 overlay 用来动态加载一个 FPGA 位流。FPGA 的寄存器地址是 `0x10000000` 到 `0x10000fff`。

overlay 里这么写的：

```
&amba {
    fpga@10000000 {
        compatible = "foo,my-fpga";
        reg = <0x10000000 0x1000>;
        interrupts = <0 42 4>;
    };
};
```

加载后，内核报了一个匪夷所思的错误：

```
OF: overlay: apply error -22
```

`-22` 是 `-EINVAL`。完全没头绪。

后来用 debugfs 的 `unflatten` 功能，一段一段注释掉，发现是 **interrupts 属性里的中断号超出了范围**。这个 SoC 的中断控制器只支持 0～31 号 SPI 中断，42 是不合法的。但是编译 dtbo 时不会报错，因为 dtc 不知道你的中断控制器硬件限制。

那么问题来了：为什么不在加载时报一个更清晰的错误呢？我也不知道这么说对不对 —— 这可能是一个内核设计上的“历史包袱”。

总之，overlay 调试时，**尽量简化属性**，先加载一个空节点，再加属性，逐步逼近问题点。

---

## 七、技巧五：dtc 编译器

### 7.1 打开所有警告

很多人编译设备树用的是：

```
dtc -I dts -O dtb -o foo.dtb foo.dts
```

这就相当于用 `gcc` 编译 C 代码不打开任何警告。**太浪费了**。

正确的打开方式：

```
dtc -I dts -O dtb -o foo.dtb foo.dts -W all
```

如果还要检查过时语法：

```
dtc -I dts -O dtb -o foo.dtb foo.dts -W all -W obsolete
```

**常见的警告及其含义**：

-   • `Warning (reg_format)`: `reg` 属性格式不对，比如长度不是 addr-cells 和 size-cells 的倍数。
    
-   • `Warning (avoid_default_addr_size)`: 缺少 `#address-cells` 或 `#size-cells`，dtc 会使用默认值 2，但可能不是你想要的。
    
-   • `Warning (interrupt_provider)`: 中断控制器节点缺少 `#interrupt-cells`。
    
-   • `Warning (gpios_property)`: `gpios` 属性应该用 `-gpios` 后缀（如 `reset-gpios` 而不是 `reset`）。
    

这些警告**必须全部消除**。不要觉得“只是警告，没事的”。设备树里的警告几乎都意味着运行时行为异常。

### 7.2 利用 dtc 的“语法检查”找错别字

你有一个节点：

```
touch@38 {
    coupled-gpios = <&gpio1 13 GPIO_ACTIVE_LOW>;
};
```

但是内核驱动里读的是 `coupled-gpio`（少了 s）。这种错别字用普通文本编辑器很难发现。

**dtc 的 `-W duplicate_property_names` 警告**只能帮你发现重复属性，不能检查拼写。

但你可以这样做：

```
dtc -I dts -O dts foo.dts -s > expanded.dts
```

`-s` 参数会把所有宏、包含文件展开成一个单一 dts 文件。然后用 `grep` 结合你的驱动源码里的属性名做交叉检查。

更高级的玩法：写一个脚本，提取驱动中用到的所有 `of_property_read_*` 的属性名，跟 dts 中的属性名做 diff。

我一般用这个命令：

```
grep -r "of_property_read" drivers/your_driver/ | sed 's/.*"\([^"]*\)".*/\1/' | sort -u > props_in_driver.txt
grep "=" expanded.dts | grep -o "[a-z-]* =" | sed 's/ *=//' | sort -u > props_in_dts.txt
diff props_in_driver.txt props_in_dts.txt
```

虽然不能百分百准确（因为驱动里可能读的是字符串解析出来的名字），但起码能帮你发现 80% 的拼写错误。

### 7.3 反编译时的“反优化”

有时候你用 `dtc -I dtb -O dts` 反编译出来的文件，跟你原始 dts 长得不一样。

这是 dtc 在做“优化”。比如它会合并连续的 `<...>` 数组，或者把字符串连接起来。

为了得到更接近原始写法的结果，可以加 `-s`（sort）和 `-H epapr`（使用标准格式）：

```
dtc -I dtb -O dts -s -H epapr /sys/firmware/fdt > current.dts
```

**但是**，不管你用什么参数，**丢失的注释永远回不来了**。所以强烈建议把注释写在属性名或节点名旁边，别单独一行。

不好的写法：

```
/* 这个是触摸屏的中断引脚 */
int-gpios = <&gpio1 12 0>;
```

好的写法：

```
int-gpios = <&gpio1 12 0>; /* 触摸屏中断引脚，GPIO1_12，低电平有效 */
```

这样即使反编译后注释没了，属性本身仍然有意义。

---

## 八、技巧六：内核里的 of API

### 8.1 驱动探测时打印设备树节点信息

写驱动时，很多人只在 probe 函数里放一个 `printk`：

```
static int my_probe(struct platform_device *pdev)
{
    printk("probe called\n");
    // ...
}
```

这能告诉你 probe 被调用了，但**为什么 probe 被调用**？因为 compatible 匹配上了？还是因为设备树里匹配到了？

更有用的做法是：打印出这个设备的完整节点路径和 compatible。

```
static int my_probe(struct platform_device *pdev)
{
    struct device_node *np = pdev->dev.of_node;
    
    pr_info("Probing device: %pOF\n", np);  // %pOF 是内核专门为设备节点设计的格式符
    pr_info("compatible: %s\n", of_get_property(np, "compatible", NULL));
    
    // 也可以遍历所有属性
    struct property *prop;
    for_each_property_of_node(np, prop) {
        pr_info("  property: %s\n", prop->name);
    }
    // ...
}
```

**`%pOF` 这个格式符是真的绝了**。它会把节点的 `full_name` 打印出来，而且如果这个节点来自 overlay，它还会标注 `(fragment)`。

### 8.2 检查某个属性是否存在

有时候属性是存在的，但值不对。你可以用 `of_property_read_u32` 这类函数，如果返回错误，打出来：

```
u32 val;
int ret = of_property_read_u32(np, "my-val", &val);
if (ret) {
    pr_err("Failed to read my-val: %d\n", ret);
    // 进一步打印这个属性在设备树里的原始值（如果有）
    const __be32 *prop = of_get_property(np, "my-val", NULL);
    if (prop) {
        pr_err("  Raw prop value: 0x%08x\n", be32_to_cpup(prop));
    }
}
```

`of_get_property` 返回的是原始字节序的指针，记得用 `be32_to_cpup` 转成 CPU 字节序。

### 8.3 跟踪 of 核心代码的调试信息

如果你打开了内核的 `CONFIG_OF_DYNAMIC` 和 `CONFIG_DEBUG_DRIVER`，可以在挂载 debugfs 后看到 of 核心的详细日志：

```
echo 1 > /sys/kernel/debug/dynamic_debug/control
```

但这会打印海量信息。更好的方法是只打开 of 相关文件的调试：

```
echo "file drivers/of/* +p" > /sys/kernel/debug/dynamic_debug/control
```

然后重新加载你的驱动模块（或者触发 probe）。你会看到 `of_match_device`、`of_find_node_by_phandle` 等函数的内部调用过程。

有一次我用这个方法发现，驱动里的 `of_match_table` 指向了一个已经被编译进内核但被 `__initdata` 标记的表。probe 调用时这个表已经被释放了，内容全是垃圾。内核不崩溃已经算仁慈了，它只是匹配不上。

**你说这谁能想到？**

---

## 九、技巧七：devicetree schema

### 9.1 schema 是什么？

近年来，Linux 内核引入了设备树 schema 验证机制（基于 JSON Schema）。你可以理解成 **“设备树的编译器 + 静态检查器”**。

它不仅能检查语法错误，还能检查**语义错误**。比如：

-   • 某个节点的 `compatible` 是否在官方列表中？
    
-   • `reg` 的长度是否匹配 `#address-cells` 和 `#size-cells`？
    
-   • 必需的属性是否缺失？
    

### 9.2 怎么用？

首先安装工具：

```
# Ubuntu/Debian
apt-get install device-tree-compiler python3-dt-schema
```

然后对 dts 文件运行 schema 验证：

```
make dt_binding_check   # 在内核源码目录下运行
```

或者单独验证一个 dts：

```
dt-validate -s dtschema/schema.json your.dts
```

**输出示例**（故意写错）：

```
your.dts:34.21-37.5: ERROR: 'interrupts' is a required property for node '/soc/uart@20000000'
```

这种错误，如果没有 schema，只能在运行到驱动 probe 时才发现 —— 而且大概率只在某个特定的调用路径上触发（比如当你真正去读中断号时）。

有了 schema，**编译阶段就能发现**。

### 9.3 schema 的一个“槽点”

“我也不知道这么说对不对”——schema 验证非常**慢**。

一个中等规模的 dts 文件，验证一次可能需要十几秒。

而且它依赖的 `dt-schema` 包更新频繁，内核版本和 schema 版本不匹配时会报一堆奇怪的错误。

**解决方案**：在 CI/CD 里跑 schema，本地开发时只跑 dtc 警告。或者给 `make` 加一个环境变量跳过 schema：

```
make SKIP_DT_SCHEMA=1 dtbs
```

但千万不要完全放弃 schema。它曾经帮我发现过一个极隐蔽的 bug —— 我把 `#interrupt-cells` 错误地写在了父节点而不是中断控制器节点上。dtc 没报错，因为语法正确。但所有引用这个中断控制器的子节点，内核都解析不到中断号。

这个 bug 藏了两个星期。最后是 schema 揪出来的。

---

## 十、技巧八：常见 bug 速查手册

这节我整理了自己踩过的 **7 个经典坑**。每个坑都配上了“症状”和“解药”。

### 坑 1：节点明明存在，驱动 probe 不调用

**症状**：`/proc/device-tree/` 下能看到节点，但驱动的 `probe` 函数从未执行。

**原因**：`compatible` 字符串写错了（大小写、逗号、空格）。或者驱动中的 `of_match_table` 没有被正确引用。

**解药**：

```
// 在驱动里加上这段，确认 match table 被用了
static const struct of_device_id my_of_match[] = {
    { .compatible = "foo,bar" },
    { }
};
MODULE_DEVICE_TABLE(of, my_of_match);

// 并且在 probe 函数里打印 match 的 compatible
static int my_probe(struct platform_device *pdev)
{
    const struct of_device_id *match;
    match = of_match_device(my_of_match, &pdev->dev);
    if (match)
        pr_info("Matched compatible: %s\n", match->compatible);
}
```

### 坑 2：`reg` 属性读出来全是 0

**症状**：`of_iomap` 或 `platform_get_resource` 返回的地址是 0。

**原因**：父节点的 `#address-cells` 和 `#size-cells` 错了。比如 64 位地址的 SOC，父节点应该是 `#address-cells = <2>`，你写成了 1。

**解药**：检查设备树里的父节点。并且用 `of_address_to_resource` 替代 `platform_get_resource`，它会更严格地检查 cells。

```
struct resource res;
if (of_address_to_resource(np, 0, &res))
    pr_err("Failed to get resource\n");
else
    pr_info("Resource: %pap\n", &res.start);
```

### 坑 3：中断号不生效

**症状**：`request_irq` 返回 `-EINVAL`，或者中断处理函数永远不触发。

**原因**：中断号被映射错了。或者 `interrupt-parent` 指向了一个无效的节点。

**解药**：打印出解析后的中断号：

```
int irq = of_irq_get(np, 0);
pr_info("IRQ number: %d\n", irq);
```

如果 `irq` 是负数，说明根本没解析到。用 `irq_of_parse_and_map` 看能不能映射到虚拟中断号。

另一个常见原因：`interrupts` 里的 cell 个数跟中断控制器的 `#interrupt-cells` 不匹配。比如中断控制器是 3 个 cell（），你只写了两个。

### 坑 4：`gpiod_get` 返回 `-ENOENT`

**症状**：用 GPIO 子系统获取 gpio 描述符失败。

**原因**：设备树里 gpio 属性的后缀写错了。内核期望的是 `foo-gpios`（复数）或 `foo-gpio`（单数，已废弃）。

**解药**：检查 gpio 属性的名字。标准的写法是 `reset-gpios = <&gpio1 2 GPIO_ACTIVE_LOW>;`。然后驱动里用 `gpiod_get(dev, "reset", ...)`，注意这里是去掉 `-gpios` 后的名字。

如果你不想看文档，直接打印设备树里的所有属性名，看看实际的名字是什么。

### 坑 5：pinctrl 配置不生效

**症状**：引脚电平不对，或者功能错误。

**原因**：pinctrl 节点里的 `function` 或 `pins` 写错了。或者 pinctrl 驱动没有加载。

**解药**：

```
// 在驱动 probe 里强制应用 pinctrl
struct pinctrl *p = devm_pinctrl_get(dev);
if (IS_ERR(p))
    pr_err("Failed to get pinctrl\n");
else
    devm_pinctrl_put(p);
```

更直接的方法：进 debugfs 看 pinctrl 状态：

```
cat /sys/kernel/debug/pinctrl//pinmux-pins
```

看看你的引脚当前被 mux 成了什么功能。不是预期的话，就是设备树或 pinctrl 驱动的问题。

### 坑 6：`status` 属性被忽略

**症状**：节点明明写的是 `status = "okay"`，但内核却当成 `disabled`。

**原因**：你写在了错误的位置。`status` 属性可以出现在**任何**节点上，但如果父节点被 `disabled`，子节点即使 `okay` 也不会被初始化。

**解药**：从根节点向下，检查每一级的 `status`。

```
find /proc/device-tree -name "status" -exec cat {} \; -print
```

如果某一级输出是 `disabled`，那么这一级以下的所有节点都不可用。

### 坑 7：内存地址重叠导致的诡异崩溃

**症状**：系统跑几个小时突然崩溃，或者在加载某个驱动时 SIGSEGV。

**原因**：两个设备树节点映射到了同一段物理地址。一个驱动写数据，另一个驱动读到脏数据。

**解药**：检查所有的 `reg` 地址区域是否重叠。可以用这个脚本（粗糙但有用）：

```
dtc -I dtb -O dts /sys/firmware/fdt | grep -E "reg = <" | sed 's/.*<\(.*\)>.*/\1/' | awk '{print "0x"$1, "0x"$2}' | while read start size; do echo $start $size; done | sort -n
```

然后手动看有没有重叠区间。

---

## 十一、实战案例：从报错到修复全过程

光说理论有点干。咱们来一个**真人真事**。

### 背景

一块 RK3568 的板子，外接了一个 MIPI CSI 摄像头。驱动是供应商给的。设备树片段：

```
&i2c2 {
    camera@36 {
        compatible = "ovti,ov13850";
        reg = <0x36>;
        clocks = <&cru CLK_CAM0>;
        clock-names = "xvclk";
        reset-gpios = <&gpio1 21 GPIO_ACTIVE_LOW>;
        pwdn-gpios = <&gpio1 20 GPIO_ACTIVE_LOW>;
        port {
            csi_out: endpoint {
                remote-endpoint = <&csi_in>;
                data-lanes = <1 2>;
            };
        };
    };
};

&csi_dphy {
    status = "okay";
    ports {
        port@1 {
            reg = <1>;
            csi_in: endpoint {
                remote-endpoint = <&csi_out>;
                data-lanes = <1 2>;
            };
        };
    };
};
```

### 症状

启动后 dmesg 看到：

```
ov13850 2-0036: failed to get clock
ov13850: probe of 2-0036 failed with error -2
```

`-2` 是 `-ENOENT`，找不到时钟。

### 调试过程

**第一步：反编译现场 dtb**

```
dtc -I dtb -O dts /sys/firmware/fdt > current.dts
grep -A 10 "camera@36" current.dts
```

发现 clocks 属性变成了：

```
clocks = <0x22 0x1a>;
```

`0x22` 是 cru 节点的 phandle，`0x1a` 是时钟 ID。看起来没问题。

**第二步：检查时钟驱动是否加载**

```
ls /proc/device-tree/clocks/
```

只有 `clock@0` ... 但 cru 节点应该在 `/cru` 路径下。果然，`/proc/device-tree/cru` 不存在！

这意味着 cru 节点被内核跳过了。为什么会跳过？看 `current.dts` 里 cru 的 `status` 是 `disabled`。

但是源码里写的是 `status = "okay"` 啊？被谁改了？

**第三步：检查 include 路径**

发现上游的 `rk3568.dtsi` 里 cru 节点是 `disabled`，然后板级的 dts 里用 `&cru` 重新 `status = "okay"`。但板级 dts 里**同时**存在另一行：

```
&cru {
    status = "disabled";  // 这行是谁加的？
};
```

原来是一个同事在合并代码时误操作，加了一行 `disabled`，覆盖了 `okay`。

**第四步：修复后，时钟有了，但又出现了新错误**

```
ov13850 2-0036: failed to get reset gpio
```

**第五步：用 gpio 测试**

进入 `gpio` 子系统查看：

```
cat /sys/kernel/debug/gpio
```

发现 `gpio1-21` 被另一个驱动占用了。那个驱动是 SPI 的 CS 脚，但在设备树里没有显式声明。原来是因为 pinctrl 配置把同一个引脚分配给了两个功能。

**第六步：修改 pinctrl**

在设备树里禁用 SPI 的 CS 引脚 pinctrl，或者重新分配摄像头的 reset 到另一个 GPIO（刚好有一个备用）。

**最终**：摄像头正常工作。

**这个案例花了 4 个小时**。如果一开始就反编译 + 检查 `/proc/device-tree` + debugfs gpio，可能 40 分钟就搞定了。

---

## 十二、一些“不传之秘”

### 12.1 利用 ftrace 跟踪 of 函数调用

```
# 开启 ftrace
echo function > /sys/kernel/debug/tracing/current_tracer
echo of_* > /sys/kernel/debug/tracing/set_ftrace_filter
echo 1 > /sys/kernel/debug/tracing/tracing_on
# 重新加载驱动或操作设备树
cat /sys/kernel/debug/tracing/trace
```

你会看到 `of_find_node_by_name`、`of_match_device` 等函数的调用栈。

有一次我用它发现，`of_find_node_by_path` 被调用了 200 多次，每次都在遍历整个设备树 —— 性能问题就是这么暴露的。

### 12.2 设备树的内存布局分析

编译 dtb 时加上 `-S` 参数，可以输出一个“符号表”：

```
dtc -S -O dtb -o foo.dtb foo.dts > foo.sym
```

这个符号表里记录了每个节点在 dtb 中的偏移量。配合 `hexdump`，你可以直接看二进制布局。

### 12.3 使用 `device-tree-compiler` 的最新版本

有些发行版自带的 dtc 版本很老（比如 1.4.7）。老版本的 dtc 对 overlay 的支持有 bug。

从内核源码里编译最新的 dtc：

```
make scripts/dtc/  # 在内核源码目录
```

然后把 `scripts/dtc/dtc` 加到 PATH 里。

新版本的 dtc 对警告信息的输出更友好，而且支持 `-@`（生成 symbols，overlay 必需）。

---

## 十三、总结

写了这么多，键盘都冒烟了。

设备树调试，说到底就是**一件事**——**确认“写进去”的和“跑起来”的是否一致**。

因为设备树不像 C 代码，编译错误会告诉你哪行错了。它更像一个“沉默的配置”。只有你主动去检查，它才会告诉你真相。

这 8 个技巧，我建议你收藏一下：

1.  1\. **反编译 dtb** —— 看现场真相
    
2.  2\. **/proc/device-tree** —— 动态上帝视角
    
3.  3\. **debugfs** —— 内核后门，动态加载测试片段
    
4.  4\. **overlay 专用调试** —— 注意符号解析和碎片化
    
5.  5\. **dtc 警告全开** —— 第一道防线
    
6.  6\. **of API 打印** —— 驱动里埋“传感器”
    
7.  7\. **devicetree schema** —— 语义检查
    
8.  8\. **常见 bug 清单** —— 踩坑索引
    

很多人在设备树上每次都要捣鼓半天，不是因为他们水平不够。而是因为**没有建立起“验证”的意识**。

他们总觉得自己写的 dts 一定是对的。编译通过了，就烧进去跑。出了问题就改驱动。

下次再遇到设备树相关的怪问题，请先反问自己三句话：

-   • 反编译现场 dtb 看了吗？
    
-   • 确认过节点真的被内核解析了吗？
    
-   • 在 probe 里打印出设备树属性了吗？
    

这三件事做完了，90% 的问题都已经水落石出。

剩下的 10%，可能需要你打开 debugfs 或者 ftrace。但相信我，那是值得的。

---

内容效果不满意？[点此反馈](https://feedback.notebooksyncer.com/feedback/687118ef_1780533293573?u=https%3A%2F%2Fmp.weixin.qq.com%2Fs%3F__biz%3DMzkzNDk2NTUwOQ%3D%3D%26mid%3D2247486244%26idx%3D1%26sn%3Da04226a1bf63425b51d33934cd97bd5f%26chksm%3Dc39dbac11e43937d04c5a42655b841639fc9a0b363aca96549fd4e0f6c9cf474130be84fc1ca%26mpshare%3D1%26scene%3D1%26srcid%3D0604oD45wQVURTJ8nvdWyYXj%26sharer_shareinfo%3D56f511fb50501066d9fded686f6f5c93%26sharer_shareinfo_first%3D56f511fb50501066d9fded686f6f5c93%23rd&s=obsidian)