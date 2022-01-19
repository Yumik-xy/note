# Smack 4.4.4

## Android 无法进行文件传输

在 `4.4.3` 版本及以前，`smack` 使用3种文件传输形式：**点对点传输**、**代理服务器传输**和**带内文件传输（Base64编码的消息传输）**

但是根据抓包显示，代理服务器传输在传输可选列表中排列在第一位，即优先使用代理服务器传输，其次再使用点对点传输。

// TODO: Smack 4.3.4 抓包数据

因此在2021年9月有人在 `igniterealtime` 发布帖子提问 [Local SOCK5 proxy not running](https://discourse.igniterealtime.org/t/local-sock5-proxy-not-running/90721) ，而管理员很快回答了这个问题，并在 `4.4.4` 版本中修复了这个问题

- [[SMACK-912\] Smack does not start the local SOCKS5 proxy automatically](https://igniterealtime.atlassian.net/browse/SMACK-912)
- [Rework SOCKS5 unit tests so that they can be run in parallel](https://github.com/igniterealtime/Smack/commit/9352225f444b68d1f5fe96567dadb71c43908ce6)

// TODO: Smack 4.4.4 抓包数据

这是个好消息，但是在基于 `smack 4.4.4` 版本的 `Android IM` 开发中，却由于 `getLocalStreamHost()` 方法返回的地址在存在无法检索到主机的地址，导致连接直接中断，文件传输终止

幸运的是，`smack` 为本地`socks5` 提供了可配置选项

```java
// package: org.jivesoftware.smackx.bytestreams.socks5

/**
 * Sets if the local Socks5 proxy should be started. Default is true.
 *
 * @param localSocks5ProxyEnabled if the local Socks5 proxy should be started
 */
public static void setLocalSocks5ProxyEnabled(boolean localSocks5ProxyEnabled) {
    Socks5Proxy.localSocks5ProxyEnabled = localSocks5ProxyEnabled;
}
```

我们可以调整 `localSocks5ProxyEnabled` 为 `False` 从而禁用本地连接，仅使用代理服务器进行文件传输，经过测试该方法可行

并且在实际生产环境中很少会存在 `Android` 设备存在于同一个无防火墙的网段下，因此该禁用有存在意义

 

