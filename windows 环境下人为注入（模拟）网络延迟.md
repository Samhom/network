# windows 环境下**人为注入（模拟）网络延迟**

如果想 **人为注入（模拟）网络延迟**（例如增加特定 IP + 端口的延迟，用于测试网络应用在高延迟环境下的表现），可以通过 clumsy 工具实现。

[Clumsy](https://jagt.github.io/clumsy/) 是一个开源的 Windows 工具，可以模拟延迟、丢包、乱序等网络问题。

**步骤：**

1. 在 **Filter** 输入规则（如 `ip.DstAddr == 1.2.3.4 and tcp.DstPort == 80`）。
2. 在 **Lag** 选项卡设置延迟（如 `100ms`）。
3. 点击 **Start**，所有匹配的流量都会被注入延迟。

其中 Presets 可以按需配置，如果不配置则匹配所有 协议。

注入后可以通过 powersheel 测试：

```bash
> Measure-Command { Test-NetConnection 1.2.3.4 -Port 80 -InformationLevel Quiet }
Days              : 0
Hours             : 0
Minutes           : 0
Seconds           : 0
Milliseconds      : 105
Ticks             : **
TotalDays         : **
TotalHours        : **
TotalMinutes      : **
TotalSeconds      : **
TotalMilliseconds : 100.9746
```

