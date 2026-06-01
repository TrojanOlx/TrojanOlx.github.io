---
layout:     post
title:      开了 Clash Verge TUN 后，Docker 服务公网访问全挂？
subtitle:   用 connmark + 策略路由精准修复入站回程
date:       2026-06-01
author:     Trojan
header-img: img/post-bg-57.jpg
catalog: true
tags:
    - Clash
    - Clash Verge
    - Mihomo
    - TUN
    - Docker
    - iptables
    - 策略路由
    - Linux 网络
---

## 一、背景：一个看似自相矛盾的需求

我在一台 Linux 服务器（Kali）上跑了 Clash Verge，开了 **TUN 模式**做全局代理——因为机器上有好几个程序都得靠它出墙。

同时，这台机器用 Docker 跑了两个容器：

- `cli-proxy-api`（[CLIProxyAPI](https://github.com/router-for-me/CLIProxyAPI)，把各家 LLM CLI 的凭据转成 OpenAI 兼容 API），监听 `8317`
- `cpa-manager-plus`（它的管理面板），监听 `18317`

网络拓扑是：**公网 `203.0.113.10` → 经 H3C 防火墙做端口映射 → 内网 `10.110.10.110`**。

我的诉求很简单，但三个条件叠在一起就别扭了：

1. `cli-proxy-api` 要能经 Clash 出墙（去访问被墙的 LLM API）；
2. 公网要能访问到 `cli-proxy-api` 和 `cpa-manager-plus`；
3. **TUN 不能关**，因为别的程序也指着它出网。

结果一开 TUN，公网访问这两个容器就**全部超时**；一关 TUN，公网立刻恢复，但其它程序又出不了墙。卡在这里查了很久，最后定位到——**就是 TUN 的问题**。

这篇就把来龙去脉、根因和最终的解决方案完整记录下来，自己以后备查，也给遇到同样坑的人参考。

---

## 二、现象

| 操作 | 公网访问 `:8317`/`:18317` | 容器出墙 |
|---|---|---|
| 关闭 TUN | ✅ 正常 | ❌ 出不去 |
| 开启 TUN | ❌ 超时/握手失败 | ✅ 正常 |

很典型的"按下葫芦浮起瓢"。容器**出站**没问题，偏偏**入站**回不来。这个不对称是解题的最大线索。

---

## 三、根因：TUN 劫持了「入站连接的回程包」

Clash Verge 用的是 Mihomo（Clash.Meta）内核。它的 TUN 模式 `auto-route` 会往系统里塞一套**策略路由规则**，把几乎所有流量导向 TUN 接口，交给内核代理处理。

用 `ip rule show` 一看就清楚了：

```text
0:      from all lookup local
9000:   from all to 198.18.0.0/30 lookup 2022
9001:   not from all dport 53 lookup main suppress_prefixlength 0
9001:   from all iif Mihomo goto 9010
9002:   not from all iif lo lookup 2022      # ← 元凶：凡不是来自 lo 的流量，统统查 table 2022（走 TUN）
9002:   from 0.0.0.0 iif lo lookup 2022
9002:   from 198.18.0.0/30 iif lo lookup 2022
9010:   from all nop
32766:  from all lookup main
32767:  from all lookup default
```

而 `table 2022` 就是一条「全走 TUN」：

```text
default via 198.18.0.2 dev Mihomo table 2022
```

关键在 `9002: not from all iif lo lookup 2022`：**只要进来的接口不是 `lo`，就查 2022 表，走 Mihomo 这块 TUN 网卡。**

于是当外部经 H3C 进来访问 `10.110.10.110:18317` 时，发生了这样的"非对称路由"：

```text
入站请求（正常）
  客户端 ──> H3C ──> eth0 ──> [DNAT] ──> 容器 172.19.0.x:18317   ✅ 能到

回程响应（被劫持）
  容器 172.19.0.x ──> 网桥 ──> 宿主机要回包
                                 │ 查路由，命中 9002 ──> table 2022 ──> dev Mihomo
                                 ▼
                              丢进 TUN ──> 走代理出去了  ❌ 客户端永远收不到
```

- **入站请求**包能顺利到达容器；
- 但容器的**回程响应**包在宿主机做路由决策时，命中了 Mihomo 那条 catch-all 规则，被丢进了 TUN，从代理节点发出去了；
- 客户端在原路苦等一个永远不会回来的握手响应 → 超时。

这就解释了为什么**出站正常、入站全挂**：出站本来就该走 TUN，没毛病；而入站的回程包也被无差别地塞进了 TUN，链路就断了。

---

## 四、为什么那些"更简单"的方案都不行

在动手之前，先排除几个看起来更省事的思路：

- **关掉 TUN**：直接违背"其它程序还要用 TUN"这条硬约束。❌
- **只给容器配 `HTTP_PROXY` 走 Clash 混合端口**：这只解决容器**出站**，可 TUN 还开着，**入站回程**照样被 9002 劫持，公网还是进不来。❌
- **让整个 Docker 网桥的流量绕过 TUN**：那容器的**出站**（`cli-proxy-api` 访问 LLM）也会一起绕过 TUN，它就出不了墙了。❌

问题的本质是：**我要区别对待"外部发起的入站连接"和"容器发起的出站连接"**——前者的回程要绕过 TUN，后者要继续走 TUN。

---

## 五、核心思路：按「连接发起方向」分流

Linux 的 `conntrack` + `CONNMARK` 正好能干这事：给**外部发起的入站连接**打一个标记，靠这个标记把它的回程包单独拎出来走 `main` 表。

| 流量类型 | 是否打标记 | 路由结果 |
|---|---|---|
| **入站连接**（公网 → `8317`/`18317`） | ✅ 打 `0x1` | 回程包查 `main` 表 → 走 `eth0` 原路返回 |
| **出站连接**（`cli-proxy-api` → LLM API） | ❌ 不打 | 落到 `9002` → 查 `2022` → 走 Mihomo TUN |
| **其它一切**（SSH、别的应用…） | ❌ 不打 | 完全不受影响 |

`connmark` 是**连接级**的标记：只要在连接的第一个包（入站 SYN）上打标，整条连接的后续包（包括回程）都能凭 `conntrack` 把标记取回来。完美匹配需求。

---

## 六、实战

### 6.1 环境假设（请替换成你自己的值）

| 项 | 本文示例值 | 怎么查 |
|---|---|---|
| 物理入口网卡 | `eth0` | `ip -br addr`（带你内网 IP 的那块） |
| 内网 IP / 网关 | `10.110.10.110` / `10.110.10.1` | `ip -br addr` / `ip route show table main` |
| 公网 IP | `203.0.113.10` | H3C 上映射的公网地址 |
| Docker 网桥 | `br-f3c018d58b03` | `docker network inspect <网络名> -f '{{.Id}}'`，网桥名=`br-<Id前12位>` |
| 容器子网 | `172.19.0.0/16` | 同上 inspect 里的 Subnet |
| TUN 接口 / 路由表 | `Mihomo` / `2022` | `ip rule show`、`ip route show table all \| grep -i tun` |
| 服务端口 | `8317`、`18317` | 你的 `compose.yaml` |

### 6.2 第一步：诊断

```bash
ip rule show                                   # 看 Mihomo 的规则和优先级（本例在 9000–9010）
ip -br addr                                     # 找物理网卡、各 docker 网桥
docker network ls                               # 找 compose 网络（一般是 <目录名>_default）
docker network inspect <网络名> -f '{{.Id}}'    # 网桥名 = br-<Id 前 12 位>
ip route show table all | grep -iE 'meta|mihomo|tun'   # 确认 TUN 接口与路由表
```

**务必先确认 `main` 表里有指向物理网卡的默认路由**，否则回程包查了 `main` 也出不去：

```bash
ip route show table main | grep default
# 期望看到：default via 10.110.10.1 dev eth0 ...
```

如果这条不存在，先补上（网关换成你的实际值）：

```bash
ip route add default via 10.110.10.1 dev eth0 table main
```

### 6.3 第二步：三条规则搞定

```bash
# ① 给入站到 8317/18317 的连接打 connmark（只用最低位，避免动 Mihomo 自己的 mark）
iptables -t mangle -A PREROUTING -i eth0 -p tcp -m multiport --dports 8317,18317 \
  -j CONNMARK --set-mark 0x1/0x1

# ② 回程包恢复 connmark 到包标记（不绑接口，靠 connmark 区分；只有被标记的连接会被恢复）
iptables -t mangle -A PREROUTING -j CONNMARK --restore-mark --nfmask 0x1 --ctmask 0x1

# ③ 带标记的包查 main 表，绕过 TUN（pref 100 抢在 Mihomo 的 9002 之前）
ip rule add fwmark 0x1/0x1 lookup main pref 100
ip route flush cache
```

逐条解释：

- **①** 仅匹配「从 `eth0` 进来、去 `8317/18317`」的入站请求，给整条连接打上 `connmark = 0x1`。`/0x1` 掩码表示只动最低位，不会覆盖 Mihomo 在高位用的标记。
- **②** 对所有经过 `PREROUTING` 的包尝试把 `connmark` 还原成 `packet mark`。但只有属于「被①标记过的连接」的包，`connmark` 才是 `0x1`，其它连接 `connmark=0`、还原后仍是 0，**毫无影响**。所以这条**不需要绑定网桥名**（后面会讲为什么这点很重要）。
- **③** 加一条**高优先级**（`pref 100` < Mihomo 的 `9002`）的策略路由：凡 `packet mark` 命中 `0x1` 的，查 `main` 表。于是入站连接的回程包走 `main` → `eth0` → 原路返回。

> 为什么 `pref` 要小于 9002？`ip rule` 按 `pref` 数字从小到大匹配，数字越小越优先。插在 9002 前面，回程包才会先命中我们这条、走 `main`，轮不到 Mihomo 那条把它拖进 TUN。

### 6.4 第三步：验证

```bash
ip rule show | grep 0x1
# 100:    from all fwmark 0x1/0x1 lookup main

iptables -t mangle -L PREROUTING -n -v
# 两条 CONNMARK 规则，pkts 计数随访问增长说明命中了

curl -I http://10.110.10.110:8317/      # 内网
curl -I http://10.110.10.110:18317/
curl -I http://203.0.113.10:18317/      # 公网经 H3C —— 这条通了就成了
```

我这边三条 `curl` 分别返回 `404`（`cli-proxy-api` 根路径本就没内容）、`307 → /management.html`、`307`，**公网入站恢复**。同时 `cli-proxy-api` 的日志里上游 API 调用照常有 `200`，说明它经 TUN 出墙完全不受影响。三个诉求一次性达成。

---

## 七、持久化（含我踩的两个坑）

上面的 `iptables` 和 `ip rule` **重启即失效**，必须固化。这里有两个容易翻车的点。

### 坑 1：`restore-mark` 别绑死网桥名

最初我把第②条写成 `-i br-f3c018d58b03`（精准匹配那个网桥）。看起来更"干净"，其实有两个雷：

- `docker compose down && up` 会**重建网络**，网桥名跟着网络 ID 变（`br-<新ID>`），规则就指向一个不存在的接口失效了；
- `iptables-persistent` 开机恢复规则时，docker 网络可能**还没创建**，引用不存在的网卡名会导致恢复报错。

所以最终改成**不绑接口**的写法（就是 6.3 的第②条）。靠 `connmark` 本身区分，既稳又和 Docker 解耦：

```bash
# 删掉绑定网桥名的旧规则
iptables -t mangle -D PREROUTING -i br-f3c018d58b03 -j CONNMARK --restore-mark --nfmask 0x1 --ctmask 0x1
# 换成不绑接口的
iptables -t mangle -A PREROUTING -j CONNMARK --restore-mark --nfmask 0x1 --ctmask 0x1
```

然后保存 iptables（Debian/Kali 系）：

```bash
apt install -y iptables-persistent
netfilter-persistent save
```

### 坑 2：`netfilter-persistent` 不管 `ip rule`，且 systemd unit 别漏 `[Install]`

`netfilter-persistent` 只持久化 `iptables/ip6tables`，**不会保存策略路由规则**。`ip rule` 得单独用一个 systemd oneshot 固化。

我第一次写的 unit **漏了 `[Install]` 段**，结果 `systemctl enable` 直接报：

```text
The unit files have no installation config (WantedBy=, RequiredBy=, ...).
This means they are not meant to be enabled or disabled using systemctl.
```

——也就是**根本没设成开机自启**。补上 `[Install]` 后才正常。完整正确版本：

```bash
tee /etc/systemd/system/inbound-route-fix.service >/dev/null <<'EOF'
[Unit]
Description=Inbound return traffic bypass Mihomo TUN
After=network-online.target
Wants=network-online.target

[Service]
Type=oneshot
RemainAfterExit=yes
ExecStartPre=-/usr/sbin/ip rule del fwmark 0x1/0x1 lookup main pref 100
ExecStart=/usr/sbin/ip rule add fwmark 0x1/0x1 lookup main pref 100
ExecStartPost=/usr/sbin/ip route flush cache
ExecStop=/usr/sbin/ip rule del fwmark 0x1/0x1 lookup main pref 100

[Install]
WantedBy=multi-user.target
EOF

systemctl daemon-reload
systemctl enable --now inbound-route-fix.service
```

几个细节：

- `ExecStartPre=-...del` 前缀的 `-` 表示忽略失败：先删再加，**幂等**，`--now` 也不会冒出两条 `pref 100`。
- `ip rule add fwmark` 不依赖任何网卡/地址，所以即使网络起得晚也能成功。
- 确认 `ip` 路径：`command -v ip`（Kali/Debian 默认 `/usr/sbin/ip`）。

验证自启真的生效：

```bash
systemctl is-enabled inbound-route-fix.service   # 期望：enabled
ls -l /etc/systemd/system/multi-user.target.wants/inbound-route-fix.service   # 符号链接应存在
```

> 补充：Clash Verge 每次重启 TUN 只会重建它自己 `9000–9010` 那套规则，**不会动我们 `pref 100` 的规则**，所以两者井水不犯河水。

---

## 八、数据包完整路径（修复后）

```text
【入站连接：公网访问 18317】
 客户端 ──> H3C ──> eth0
        │  mangle/PREROUTING ①：dport 18317 → connmark=0x1
        │  mangle/PREROUTING ②：restore-mark → pkt mark=0x1
        │  nat/PREROUTING：DNAT → 172.19.0.x:18317
        ▼
       容器处理，产生响应
        │  响应包经网桥进宿主机
        │  mangle/PREROUTING ②：该连接 connmark=0x1 → restore 到 pkt mark=0x1
        │  路由决策：命中 ip rule (pref 100) → 查 main 表
        │  main: default via 10.110.10.1 dev eth0
        ▼
       nat/POSTROUTING 反向 NAT（源改回 10.110.10.110:18317）
        ──> eth0 ──> H3C ──> 客户端   ✅ 原路返回

【出站连接：cli-proxy-api 访问 LLM】
 容器 172.19.0.x ──> 网桥 ──> 宿主机
        │  mangle/PREROUTING ②：该连接 connmark=0（没被①标记）→ pkt mark=0
        │  路由决策：不命中 pref 100 → 落到 Mihomo 9002 → 查 table 2022
        ▼
       dev Mihomo（TUN）──> 代理节点 ──> 出墙   ✅ 维持现状
```

一张图把"为什么三个约束能同时满足"讲透了：**区分点就是那个 `connmark`**。

---

## 九、一键复用脚本

以后换机器或重装，直接套这个幂等脚本（用 `-C` 检查避免重复添加）：

```bash
#!/usr/bin/env bash
set -euo pipefail

WAN=eth0                 # 物理入口网卡
PORTS=8317,18317         # 需要被公网访问的端口
MARK=0x1

# ① 入站连接打 connmark
iptables -t mangle -C PREROUTING -i "$WAN" -p tcp -m multiport --dports "$PORTS" \
  -j CONNMARK --set-mark ${MARK}/${MARK} 2>/dev/null \
  || iptables -t mangle -A PREROUTING -i "$WAN" -p tcp -m multiport --dports "$PORTS" \
       -j CONNMARK --set-mark ${MARK}/${MARK}

# ② 回程恢复 connmark（不绑接口）
iptables -t mangle -C PREROUTING -j CONNMARK --restore-mark --nfmask ${MARK} --ctmask ${MARK} 2>/dev/null \
  || iptables -t mangle -A PREROUTING -j CONNMARK --restore-mark --nfmask ${MARK} --ctmask ${MARK}

# ③ 策略路由：带标记的包查 main 表
ip rule del fwmark ${MARK}/${MARK} lookup main pref 100 2>/dev/null || true
ip rule add fwmark ${MARK}/${MARK} lookup main pref 100
ip route flush cache

netfilter-persistent save 2>/dev/null || true
echo "done. 记得用 systemd 固化 ip rule（见正文第七节）。"
```

---

## 十、总结

- **现象**：开 Clash/Mihomo TUN 全局代理后，本机（含 Docker）对外服务的公网入站全部超时，出站正常。
- **根因**：TUN 的 `auto-route` 用一条 catch-all 策略路由（`iif != lo → 走 TUN 表`）把**入站连接的回程包**也劫持进了 TUN，形成非对称路由。
- **解法**：用 `CONNMARK` 给「外部发起的入站连接」打标，再用高优先级 `ip rule` 把带标记的回程包导回 `main` 表，绕过 TUN。**出站连接不带标记，照旧走 TUN**，互不干扰。
- **适用范围**：不止 Docker，任何「开了 TUN/全局代理后本机服务外部访问不了」的场景（裸跑的 Web/SSH/RDP 服务同理），改一下端口和入口网卡即可。
- **两个坑**：`restore-mark` 别绑网桥名（`compose down/up` 会变）；systemd unit 记得写 `[Install]`，否则不开机自启。

核心就一句话：**TUN 是无差别劫持，而我们要的是「按连接方向」精准放行——`connmark` 就是那把手术刀。**
