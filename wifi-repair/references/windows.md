# Windows 10 / 11 操作手册

在出问题的电脑上打开 **Windows PowerShell**。下面的小节依次执行，在同一个窗口中保留变量。前四项是诊断，通常可在普通用户会话完成。位置权限、设备驱动和组织策略会影响可见字段；修改网卡配置则可能要求管理员授权。

**本页顺序**　无线网卡与扫描 → 网关、公网与五跳 → DNS 计时 → 浏览器负载测速 → DNS 设置与恢复。

## 1 · 找到真正连接 Wi-Fi 的网卡

```powershell
netsh wlan show interfaces
```

找到“已连接”无线接口，记下名称、无线类型、频段、信道、信号、接收与发送速率。虚拟网卡、以太网和 Wi-Fi 的结果要分开。根据实际名称设置变量，名称可能为 WLAN、Wi-Fi 2 或其他名称：

```powershell
$wifiName = 'Wi-Fi'
$wifi = Get-NetAdapter -Name $wifiName -ErrorAction Stop
$wifiIndex = $wifi.ifIndex
$config = Get-NetIPConfiguration -InterfaceIndex $wifiIndex
$gw = $config.IPv4DefaultGateway.NextHop | Select-Object -First 1
$config | Format-List InterfaceAlias,InterfaceIndex,IPv4DefaultGateway,DNSServer
Get-DnsClientServerAddress -InterfaceIndex $wifiIndex
```

如果没有网关或有多个候选，运行 `ipconfig /all`，仅核对这个无线适配器的 IPv4 网关、DHCP 和 DNS 行，不要复制整份配置到公开位置。静态或 IPv6-only 的网络可能没有对应 IPv4 网关；在没有 IPv4 路径时，将指定的 IPv4 测试标为不适用。

然后单独扫描附近网络：

```powershell
netsh wlan show networks mode=bssid
```

| macOS 字段 | Windows 上怎么记 |
|---|---|
| RSSI / Noise / SNR | 如果有 RSSI dBm 就记原值；只有 Signal 百分比时保留百分比。噪声未提供就写“不可见”，不要从百分比推算 dBm 或 SNR。 |
| 当前信道宽度 | 同时依赖系统版本、驱动和网卡。高级驱动属性里的“首选宽度”是配置偏好，不能代替协商宽度。宽度未知时可统计主信道，排序须注明局限。 |
| 频段 / PHY / 速率 | 优先记驱动报告的实际频段。只看到信道编号时留意 6 GHz 与旧频段可有同号信道。收发速率属于无线协商速率。 |
| 周边网络 | 某些新版 Windows 会限制带 BSSID 的扫描。在“设置 → 隐私和安全性 → 位置”核对限制，用户愿意时可使用系统提供的位置访问选项；组织策略限制时采用路由器的信道利用率或 AP 管理页。 |
| 网卡错误 | 使用接口计数的前后差值，保留接收和发送错误、丢弃。驱动未提供时标记缺失。 |

## 2 · 网关、外网、五跳，逐项运行

以下第一块只进行一次网关 ping，前后各取一次网卡计数。确认变量 gw 已指向无线网关再执行。

```powershell
function Show-WifiCounters {
    Get-NetAdapterStatistics -Name $wifiName |
        Select-Object ReceivedPacketErrors,OutboundPacketErrors,ReceivedDiscardedPackets,OutboundDiscardedPackets
}
Show-WifiCounters | Format-List
ping.exe -4 -n 30 $gw
Show-WifiCounters | Format-List
```

结束后再执行公网 ping：

```powershell
ping.exe -4 -n 30 1.1.1.1
```

最后做最多五跳：

```powershell
tracert.exe -4 -d -h 5 -w 2000 1.1.1.1
```

Windows 的 -n 表示 ping 次数。用 tracert 的 -d 关闭反查，-h 5 限制最多五跳，-w 的单位是毫秒。Windows tracert 默认发 ICMP Echo；macOS 默认 traceroute 发 UDP。在 VPN、组网设备或过滤规则下，即便同一地点两者也可能收到不同回包。Windows ping 摘要通常没有标准差，小于 1 ms 的读数不要编造小数。需要确认出口时，单独看 `Find-NetRoute -RemoteIPAddress 1.1.1.1`；它描述本机 IP 选路，无法涵盖应用层代理的全部行为。

## 3 · DNS 对比，不用安装 dig

确认路由器 DNS 地址（以下暂用网关，可替换）。每次向一个指定地址查询一个 A 记录；相同域名、相同顺序，全部串行。预先加载模块，减少第一次调用的启动开销。

```powershell
Import-Module DnsClient
$routerDns = $gw
$domains = 'example.org', 'www.iana.org', 'www.microsoft.com'
foreach ($dns in @('1.1.1.1', '8.8.8.8', $routerDns)) {
    foreach ($domain in $domains) {
        $clock = [Diagnostics.Stopwatch]::StartNew()
        try {
            $reply = Resolve-DnsName -Name $domain -Type A -Server $dns -DnsOnly -NoHostsFile -QuickTimeout -ErrorAction Stop
            $clock.Stop()
            $answer = $reply | Where-Object { $_.Type -eq 'A' -and $_.IPAddress } | Select-Object -First 1
            if ($answer) { $result = $answer.IPAddress } else { $result = '无 A 记录' }
        } catch {
            $clock.Stop()
            $result = $_.FullyQualifiedErrorId
        }
        '{0,-15} {1,-18} {2,7:N1} ms  {3}' -f $dns, $domain, $clock.Elapsed.TotalMilliseconds, $result
    }
}
```

Windows 上记录的是本地命令执行耗时，包括系统调用开销，-QuickTimeout 也不等于 dig 的“等待 2 秒、只尝试一次”。跨平台比较可用性和相对趋势；严禁把失败等待时间混入成功延迟的中位数。对返回冲突或时间差很小的结果，换查询顺序复核并标明缓存、浏览器安全 DNS、VPN 分流等影响。缺少此模块时可单次执行 `nslookup example.org 1.1.1.1` 核对返回，再替换为另两台服务；只有答案时只报告可用性。

## 4 · 替代 networkQuality

Windows 10/11 没有内置的 Apple networkQuality。在问题设备的浏览器打开 [Cloudflare Speed Test](https://speed.cloudflare.com/)，保持当前代理路径，只启动这一项测速。依页面记录下载、上传 Mbps，空闲延迟、下载负载延迟、上传负载延迟（ms），以及提供时的抖动、丢包与测试节点。页面未给出的值留空。记录工具、时间和浏览器，复测时使用同一工具和路径。浏览器代理与命令行的出口可能不同。

两套工具使用不同端点、负载和统计方法。Windows 报告保留原始毫秒和吞吐量，**不要把毫秒换算成 Apple RPM 或 Medium/High 评级**。企业网络禁止外部测速时将本项记为未测；局域网文件传输只能补充 LAN 吞吐，不能替代外网响应性。

## 5 · 修复、验证、回退

1. 仅在这台设备的 DNS 对照支持修改时，保存当前无线接口的 IPv4 和 IPv6 DNS 项、顺序和“自动/手动”状态。可在 PowerShell 查看 `netsh interface ipv4 show dnsservers name="$wifiName"`；不要凭当前地址猜测是否由 DHCP 分配。
2. Windows 11 可在“设置 → 网络和 Internet → Wi-Fi → 已连接网络的属性/硬件属性 → DNS 服务器分配 → 编辑”更改。Windows 10 或界面不同，按 Win+R，输入 `ncpa.cpl`，右键对应无线网卡 → 属性 → Internet 协议版本 4 (TCP/IPv4) → 属性，只改 DNS 区域。IP 地址分配、IPv6 与原代理策略各自保留。
3. 用同一无线接口再执行 `Get-DnsClientServerAddress -InterfaceIndex $wifiIndex` 核对设置。单独执行 `Resolve-DnsName example.org -Type A -DnsOnly` 检查默认系统解析。前面的 -Server 显式测试即使成功，也不能证明浏览器使用了该上游。
4. 用原应用复测首开等待或视频卡顿。无改善就在同一界面恢复：原来自动则恢复自动，原来手动则按原顺序填写原地址。组织管理的设备遵守其 DNS、加密 DNS 和 VPN 策略。

**资料**：[微软无线排障](https://learn.microsoft.com/en-us/troubleshoot/windows-client/networking/wireless-network-connectivity-issues-troubleshooting) · [Wi-Fi 位置权限](https://learn.microsoft.com/en-us/windows/win32/nativewifi/wi-fi-access-location-changes) · [网卡计数](https://learn.microsoft.com/en-us/powershell/module/netadapter/get-netadapterstatistics) · [ping](https://learn.microsoft.com/en-us/windows-server/administration/windows-commands/ping) · [tracert](https://learn.microsoft.com/en-us/windows-server/administration/windows-commands/tracert) · [Resolve-DnsName](https://learn.microsoft.com/en-us/powershell/module/dnsclient/resolve-dnsname) · [Cloudflare 测速原理](https://blog.cloudflare.com/how-does-cloudflares-speed-test-really-work/)
