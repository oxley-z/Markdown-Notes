# 申请Tailscale的auth key

登录 Tailscale 管理后台 → Settings → Keys → Generate auth key（注意，这个key只显示一次，记得保存），选可重复使用的（Reusable）。这个 key 是凭证，用于访问Tailscale。

# 配置FLClash

进入 配置->长按配置项->扩展->脚本->前往配置脚本->加号->从零开始->键入以下内容：

```yml
// FlClash 覆写脚本：给订阅加一个 Mihomo 内置 Tailscale 出站节点
// 用法：FlClash → 工具 → 进阶配置 → 覆写脚本 → 新建脚本，粘贴这段，
// 把 tskey-auth-xxxx 换成你自己在 Tailscale 后台 Keys 页生成的 auth key，
// 然后在订阅的「扩展」页把模式切到「脚本」并选中这个脚本。
// 要求 FlClash ≥ 0.8.93（内核 mihomo ≥ v1.19.25 才有 type: tailscale）。

注意以下为需要修改的地方
// name-mihomo为在Tailscale网页端显示的名称，可更改为需要的名称
// tskey-auth-xxxx为创建的auth key


function main(config) {
  config["proxies"] = config["proxies"] || [];
  config["proxy-groups"] = config["proxy-groups"] || [];
  config["rules"] = config["rules"] || [];

  // mihomo 内置的 tailscale 出站节点
  config["proxies"].push({
    name: "tailscale",
    type: "tailscale",
    hostname: "name-mihomo",
    "auth-key": "tskey-auth-xxxx",
    "control-url": "https://controlplane.tailscale.com",
    "state-dir": "./tailscale",
    ephemeral: false,
    udp: true,
    "accept-routes": true,
    "ip-version": "ipv4-prefer",
  });

  config["proxy-groups"].unshift({
    name: "TAILNET",
    type: "select",
    proxies: ["tailscale", "DIRECT"],
  });

  // 插到所有订阅规则前面，避免原配置里 100.64.0.0/10 走 DIRECT 那条先把流量抢走
  config["rules"] = [
    "IP-CIDR,100.64.0.0/10,TAILNET,no-resolve",
    "IP-CIDR,100.100.100.100/32,TAILNET,no-resolve",
    "DOMAIN-SUFFIX,ts.net,TAILNET",
    ...config["rules"],
  ];

  return config;
}

```

![](image/Pasted%20image%2020261126190722.png)

![](image/Pasted%20image%2020261126190743.png)

![](image/Pasted%20image%2020261126190758.png)
