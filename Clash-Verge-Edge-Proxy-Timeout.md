# Clash Verge 系统代理已开启，但 Edge 仍 `ERR_CONNECTION_TIMED_OUT`

## 现象

- Clash Verge 中 **System Proxy（系统代理）已开启**
- **TUN Mode 未开启**
- 国内网站可正常访问
- 访问 Google / YouTube 等被墙网站失败，浏览器报错：

```text
ERR_CONNECTION_TIMED_OUT
```

- 表面上看像“梯子没生效”，但实际不一定是节点或 Clash 本身的问题

## 环境

- Windows 11
- Clash Verge
- 混合端口（mixed-port）：`7890`
- 浏览器：Microsoft Edge（问题浏览器）
- Chrome 在强制指定代理后可正常访问

## 排查结论

问题 **不是** 节点延迟，也 **不是** Clash 核心没工作。

真正原因是：

> **Edge 没有走系统代理，而是直连（DIRECT）外网。**

因此访问 Google 时等于裸连，最终超时。

### 关键证据

1. **Clash 代理端口本身是通的**

```bash
curl -I --proxy http://127.0.0.1:7890 https://www.google.com
```

返回正常（例如 HTTP 200），说明：

- Clash 在监听 `127.0.0.1:7890`
- 当前节点可用
- 代理链路本身没问题

2. **Windows 系统代理配置正确**

```text
ProxyEnable = 1
ProxyServer = 127.0.0.1:7890
```

3. **Edge 实际生效的代理却是直连**

打开：

```text
edge://net-internals/#proxy
```

看到：

```text
Effective settings
Proxy server: DIRECT
```

这说明 Edge **没有使用** `127.0.0.1:7890`。

4. **用强制代理参数启动 Chrome 可以打开 Google**

```text
chrome.exe --proxy-server="http://127.0.0.1:7890" https://www.google.com
```

进一步证明 Clash 没问题，问题在浏览器是否真正走代理。

## 解决方法

### 1. 先确认 Clash 侧正常

- System Proxy：**开**
- TUN Mode：**关**（如你只想用系统代理）
- 模式可先切到 **全局（Global）** 方便验证
- 手动选择一个延迟正常的节点，不要只依赖超时节点 / 异常的自动选择

### 2. 确认 Windows 代理指向 Clash 端口

路径：

`设置 → 网络和 Internet → 代理`

确认：

- 使用代理服务器：**开**
- 地址：`127.0.0.1`
- 端口：`7890`（以你的 Clash 实际端口为准）

> 如果刚改过端口（例如从 `7897` 改到 `7890`），建议把 Clash 的 System Proxy 开关重新拨一次，确保系统设置同步。

### 3. 修复 Edge 未接管系统代理

这是本次问题的核心。

#### 3.1 关闭 Edge 后台驻留

打开：

```text
edge://settings/system
```

关闭：

- **启动增强（Startup boost）**
- **关闭 Microsoft Edge 后继续运行后台扩展和应用**

#### 3.2 彻底结束 Edge 进程

打开任务管理器，结束所有：

- `Microsoft Edge`
- `msedge.exe`

> 若后台进程没清干净，后续带 `--proxy-server` 启动时，新参数可能不会真正生效。

#### 3.3 用强制代理启动 Edge 验证

```text
"%ProgramFiles(x86)%/Microsoft/Edge/Application/msedge.exe" --proxy-server="http://127.0.0.1:7890" --new-window https://www.google.com
```

如果路径不同，再试：

```text
"%ProgramFiles%/Microsoft/Edge/Application/msedge.exe" --proxy-server="http://127.0.0.1:7890" --new-window https://www.google.com
```

#### 3.4 再次检查 Edge 实际代理

打开：

```text
edge://net-internals/#proxy
```

确认不再是：

```text
Proxy server: DIRECT
```

而应类似：

```text
Proxy server: 127.0.0.1:7890
```

或显示通过 `PROXY 127.0.0.1:7890` 访问。

### 4. 建议同时关闭的 Edge 干扰项

- `edge://settings/privacy` → **安全 DNS** 先关闭
- `edge://flags/#enable-quic` → 设为 **Disabled**，然后重启 Edge
- 暂时禁用代理类扩展（如 SwitchyOmega 等）

## 为什么会误判

常见误判路径：

1. 看到 Clash 系统代理是绿的 → 以为浏览器一定走代理  
2. 看到部分节点超时 → 以为是节点问题  
3. 浏览器报 `ERR_CONNECTION_TIMED_OUT` → 以为网络或订阅挂了  

但在本例中：

- 超时是因为 **浏览器直连被墙网站**
- 不是因为 Clash 代理端口挂了
- 也不是因为选中的节点一定不可用

判断关键点只有一个：

> 去浏览器的 `net-internals/#proxy` 看 **Effective settings** 到底是 `DIRECT` 还是 `127.0.0.1:端口`

## 快速自检清单

遇到同类问题时，按这个顺序查：

1. `curl --proxy http://127.0.0.1:7890 https://www.google.com` 是否成功  
2. Windows 代理是否为 `127.0.0.1:实际端口`  
3. `edge://net-internals/#proxy` 是否为 `DIRECT`  
4. Edge 是否有后台进程残留  
5. 强制 `--proxy-server` 启动后是否恢复  

如果第 1 步成功、第 3 步却是 `DIRECT`，就可以基本判定：

**Clash 正常，浏览器没走代理。**

## 结果

完成 Edge 后台清理，并确保其真正走 `127.0.0.1:7890` 后，Google 恢复可访问。

## 一句话总结

> Clash Verge 系统代理开着，不代表 Edge 真的在用它；若 `edge://net-internals/#proxy` 显示 `DIRECT`，就会出现外网 `ERR_CONNECTION_TIMED_OUT`。先验证 Clash 端口可用，再让 Edge 重新接管系统代理。
