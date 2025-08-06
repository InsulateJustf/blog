---
title: macOS 上 tailscale 与代理软件共存技巧
published: 2025-08-04
description: ''
image: ''
tags: [selfhost,proxy,tailscale]
category: 'Note'
draft: false 
lang: ''
---
macOS 上虽然 tailscale 和“网络调试工具“ 等同时启动和使用，但一般都需要设置 skip-proxy 等通用字段使用，配置麻烦不说，一份配置不能到处用，遇到奇怪的需求重新配置也是十分麻烦的。所以我们需要一种手段让 tailscale 成为网络调试工具的一部份。

### 安装 golang 和 tailscale/tailscaled

在 macOS 下那当然是 `brew install tailscale` 一把梭，这应该不是什么有疑问的问题。

> [!NOTE]
> 如果你不喜欢 HomeBrew ，那么一定要从 https://go.dev/dl/ 下载最新版本的 golang 进行安装，记得下载对应的架构的版本，否则可能会出现编译 tailscale 失败的情况（别问我怎么知道的x）。
> 下载安装完，就可以开始编译 tailscale 和 tailscaled 了：
> `go install tailscale.com/cmd/tailscale{,d}@main` 
> 编译完的文件在 `$HOME/go/bin` 目录下, 不是很优雅，我们给它换个地方：
>
> ```shell
> cp $HOME/go/bin/tailscale /usr/local/bin/
> cp $HOME/go/bin/tailscaled /usr/local/bin/
> ```



### 配置 tailscale

根据官方手册，在终端执行 `sudo tailscaled install-system-daemon`，此时会在 `/Library/LaunchDaemons` 生成 `com.tailscale.tailscaled.plist` 文件并运行，我们先停止 tailscaled :
`sudo launchctl unload /Library/LaunchDaemons/com.tailscale.tailscaled.plist` 

我们需要修改它，让它以 userspace networking 模式启动，在 `<string>/usr/local/bin/tailscaled</string>` 下方补充三行：
```xml
<string>--tun=userspace-networking</string>
<string>--socks5-server=localhost:10555</string>
<string>--outbound-http-proxy-listen=localhost:10555</string>
```
第一行让 tailscale 以 userspace networking 模式启动，第二行启动 socks5 服务器监听，第三行启动 http 代理服务器监听。
以上的 10555 可以换成你想用的任意不被其他程序占用的端口，保存然后关闭文件。
（此处你使用 sudo 跑一个 vscode 修改，或者 `sudo vi` 也行，能改并且改对就行）

执行 `sudo launchctl load /Library/LaunchDaemons/com.tailscale.tailscaled.plist` 重新加载 tailscaled 的 plist 配置文件并重启 tailscaled 。
使用 `netstat -an | grep 10555` 确保 tailscaled 后台运行并且正在监听设置的端口。
最后执行 `tailscale up --accept-dns=false --accept-routes`。这样 tailscale 就成功以一个 socks5 服务器运行了。然后就是在代理软件里添加`127.0.0.1:10555`为代理节点，写好路由规则了，此处不赘述。

至此大功告成。但还是有存在一些小问题，如果有部署了内网 DNS 解析内网域名，此时是无法通过代理节点解析内网域名的，暂不清楚是谁的问题。即使如此对我而言已经算是非常好用了。



> 参考文献：
>
> ###### [**Tailscale 配合 Mihomo(Clash.Meta) TUN/Quantumult X VPN 共存使用技巧**](https://blog.ichr.me/post/tailscale-mihomo-quantumult-x/#%E5%AE%8C%E6%95%B4-Tailscale-%E4%B8%8E%E4%BB%A3%E7%90%86%E5%B7%A5%E5%85%B7%E5%85%B1%E5%AD%98)
>
> ###### [**Tailscale 和 clash 共存**](https://jiz4oh.com/2024/09/tailscale-with-clash/)
>
> [**Tailscaled on macOS**](https://github.com/tailscale/tailscale/wiki/Tailscaled-on-macOS)