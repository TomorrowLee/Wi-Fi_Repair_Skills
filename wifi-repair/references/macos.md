# macOS 操作手册

在发生问题的 Mac 上打开“终端”。各小节依次执行，命令块结束后再进入下一块。如果协助者的终端提示 `Operation not permitted`，将这次记录为“未测”，由用户在本机完成当前一步。保留故障发生时的位置和代理状态。

## 1 · 无线连接、扫描与接口

按住 **Option ⌥** 点击菜单栏 Wi-Fi 图标，在当前网络下面读 Channel（主信道、实际频段、MHz）、RSSI、Noise、Tx Rate 和 PHY Mode。有 MCS、NSS 可一并记录。信噪比用 RSSI 减噪声，单位 dB。Wi-Fi 名称里的“5G”不能代替实际频段。Tx Rate 是本机当时的发送协商速率。

扫描可用“无线诊断 → 窗口 → 扫描”；如果系统版本或权限未显示该入口，可读取系统信息列出的附近网络：

```sh
LC_ALL=C system_profiler SPAirPortDataType | grep -E 'Current Network Information|Other Local Wi-Fi Networks|Channel:|Signal / Noise:'
```

“Current”是本机，“Other”下面是其他扫描记录。该输出可能来自系统最近的扫描；若无输出，检查图形界面或不加英文过滤器查看本机语言的字段。某行只有信道就只记录信道。主信道不同也可能落在同一个 80/160 MHz 块；扫描中的一个名称或记录并不一定代表一个独立物理路由器。实际信道忙碌时间需要利用率读数或复测。只考虑路由器按当地地区设置提供的信道。

用下面的命令查看 Wi-Fi 对应的 Device，再设置变量。示例中的 en0 替换为实际值。在同一个终端中运行本页后续命令。

```sh
networksetup -listallhardwareports
```

```sh
wifi_if=en0
gw=$(ipconfig getoption "$wifi_if" router)
printf 'Wi-Fi gateway: %s\n' "$gw"
ipconfig getoption "$wifi_if" domain_name_server
```

如果网关为空，在“系统设置 → Wi-Fi → 已连接网络的详细信息 → TCP/IP”读取后手动给 `gw` 赋值。IPv4 步骤要求该网络确有 IPv4 网关。VPN 的默认路由可能是 utun 接口，不能把它当作 Wi-Fi 网关。DNS 面板与 `scutil --dns` 分别用于查看接口设置和系统各解析器；有分流时还要看域名范围、接口和应用内加密 DNS。

## 2 · 家庭网关、公网和前五跳

先取网卡计数，再 ping，结束后再次取计数。只比较同一接口的 Link 行。Ierrs、Oerrs 是接口收发错误，不包含所有无线重传。

```sh
netstat -in -I "$wifi_if" | awk 'NR==1 || /<Link/'
ping -n -q -c 30 "$gw"
netstat -in -I "$wifi_if" | awk 'NR==1 || /<Link/'
```

下面两条分开执行。30 包不足以排除间歇性故障，记录最小/平均/最大/标准差以及丢包数。

```sh
ping -n -q -c 30 1.1.1.1
```

```sh
traceroute -n -m 5 -q 3 -w 2 1.1.1.1
```

macOS 默认路由追踪发送 UDP，ping 使用 ICMP Echo。目标可能提前到达。遇到星号、重复地址或公网 RTT 比网关小很多，记录现象，再用 `route -n get 1.1.1.1` 查看本机选中的接口。不能据此推断每个应用或每个协议的完整路径。

## 3 · DNS 对比

先确认路由器是否提供 DNS 转发（网络 DNS 设置、DHCP 信息或路由器配置）。以下块把网关作为**待测的路由器 DNS**；如服务在其他地址，把 router_dns 改成已确认的地址。所有查询前台串行，测 IPv4 A 记录，记录状态、耗时及第一个 A 答案。短时未应答仅表示本轮未取得响应。

```sh
router_dns="$gw"
for dns in 1.1.1.1 8.8.8.8 "$router_dns"; do
  for domain in example.org www.iana.org www.microsoft.com; do
    printf '%-15s %-18s ' "$dns" "$domain"
    dig @"$dns" "$domain" A +time=2 +tries=1 +noall +comments +answer +stats |
      awk '/status:/ {s=$6; sub(/,$/, "", s)}
           $4=="A" && !ip {ip=$5}
           /^;; Query time:/ {t=$4; measured=1}
           END {if (measured) printf "%s  %s ms  %s\n",s,t,ip;
                else print "无有效计时，请查看原始输出"}'
  done
done
```

NOERROR 且有预期类型的答案才计入成功样本；区分 NXDOMAIN、SERVFAIL、REFUSED、无有效 A 记录、超时和本机错误。缺少计时时，单独把失败的那条 dig 去掉管道重跑一次看原文。三次样本只作筛查；差距很小时调换服务器顺序另测一轮。缓存和代理的 DNS 接管均会影响比较。

## 4 · 吞吐和负载下的响应

```sh
networkQuality -v
```

保留 SUMMARY 内的上下行 Mbps、Idle Latency、Responsiveness、RPM 和评级；详细输出中有接口、端点或准确性就一并记录。Mbps 越大表示吞吐能力越高，RPM 越大表示每分钟完成的往返越多。测试覆盖的是本机到测试端点的路径，VPN 可能在其中。若本机没有该命令，可用 Windows 手册中的浏览器测试，记录工具及各项毫秒数。

## 5 · 只改一项，并检查是否有效

若症状与 DNS 等待一致且路由器 DNS 实测可用，在“系统设置 → Wi-Fi → 当前网络的详细信息 → DNS”记录原模式、服务器顺序及 IPv6 项。做一次局部对照：原来手动指定公共 DNS 的，可临时改为已验证的路由器地址；若准备恢复自动，可删除手动项并核对自动填入的实际地址。灰色的自动地址、链路本地 IPv6 地址也可能是正常配置。

保存后重新打开 DNS 面板，再查看 `scutil --dns`。显式 dig 直查成功、系统默认解析成功、浏览器首开变快是三个独立观察。用 `dscacheutil -q host -a name example.org` 可检查系统能否取得一个答案（可能命中缓存）；用平常的浏览器复核原症状。若无改善，原为自动就删除新增项，原为手动就按原顺序还原，复核后再改其他变量。

**资料**：[Apple Wi-Fi 状态菜单](https://support.apple.com/guide/mac-help/mchlfad426fa/mac) · [无线诊断](https://support.apple.com/guide/mac-help/mchlf4de377f/mac) · [路由器设置](https://support.apple.com/en-us/102766) · [Apple 的网络响应性测量](https://developer.apple.com/videos/play/wwdc2021/10239/) · [BIND dig 手册](https://bind9.readthedocs.io/en/stable/manpages.html#dig-dns-lookup-utility)
