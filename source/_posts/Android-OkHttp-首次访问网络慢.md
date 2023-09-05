---
title: Android OkHttp 首次访问网络慢
date: 2023-09-05 10:19:14
tags: [Android, Network]
---

## 问题

最近开发了一个 Android 项目，发现第一个接口返回得特别慢，后面的接口就没问题。用浏览器打开链接，访问速度很快，手机连上 Charles 抓包，访问速度也很快。打印 OkHttp 的日志发现第一个接口请求耗时100多秒，后面的接口都只有几十秒，多次实验结果都一样。

```log
200 OK https://fundgz.1234567.com.cn/js/162605.js (100092ms, unknown-length body)
```

## 问题分析

通过查看 OkHttp 源码，发现 OkHttp 默认的连接超时时间是 10 秒。修改 connectTimeout 的值后测试，果然第一个接口返回的时间都是 connectTimeout 的 10 倍。

经过排查后发现是 Dns 的问题。OkHttp 中默认的 Dns 配置代码如下：

```kotlin
interface Dns {

  @Throws(UnknownHostException::class)
  fun lookup(hostname: String): List<InetAddress>

  companion object {

    @JvmField
    val SYSTEM: Dns = DnsSystem()
    private class DnsSystem : Dns {
      override fun lookup(hostname: String): List<InetAddress> {
        try {
          return InetAddress.getAllByName(hostname).toList()
        } catch (e: NullPointerException) {
          throw UnknownHostException("Broken system behaviour for dns lookup of $hostname").apply {
            initCause(e)
          }
        }
      }
    }
  }
}
```

加上日志打印出获取到的 IP 地址如下：

```log
fundgz.1234567.com.cn/2409:8c60:2600:2:0:1:0:16d
fundgz.1234567.com.cn/2409:8c60:2600:2:0:1:0:167
fundgz.1234567.com.cn/2409:8c60:2600:2:0:1:0:16c
fundgz.1234567.com.cn/2409:8c60:2600:2:0:1:0:16b
fundgz.1234567.com.cn/2409:8c60:2600:2:0:1:0:169
fundgz.1234567.com.cn/2409:8c60:2600:2:0:1:0:168
fundgz.1234567.com.cn/2409:8c60:2600:2:0:1:0:16a
fundgz.1234567.com.cn/2409:8c60:2600:2:0:1:0:16f
fundgz.1234567.com.cn/2409:8c60:2600:2:0:1:0:166
fundgz.1234567.com.cn/2409:8c60:2600:2:0:1:0:16e
fundgz.1234567.com.cn/221.178.98.184
fundgz.1234567.com.cn/221.178.98.179
fundgz.1234567.com.cn/221.178.98.188
fundgz.1234567.com.cn/221.178.98.187
fundgz.1234567.com.cn/221.178.98.185
fundgz.1234567.com.cn/221.178.98.182
fundgz.1234567.com.cn/221.178.98.181
fundgz.1234567.com.cn/221.178.98.180
fundgz.1234567.com.cn/221.178.98.183
fundgz.1234567.com.cn/221.178.98.186
```

可以看到前面 10 个是 IPv6 地址，OkHttp 按顺序一个个解析，IPv6 的都解析超时了，所有请求时间刚好是 connectTimeout 的 10 倍再多一点。

## 解决问题

我们修改 Dns 的实现为下面的代码，优先解析 IPv4 地址，最后测试问题解决。

```kotlin
val client = OkHttpClient.Builder()
    .dns(ApiDns())
    .build()

class ApiDns : Dns {
    @Throws(UnknownHostException::class)
    override fun lookup(hostname: String): List<InetAddress> {
        try {
            // IPv4 first
            return InetAddress.getAllByName(hostname).sortedBy {
                Inet6Address::class.java.isInstance(it)
            }
        } catch (e: NullPointerException) {
            throw UnknownHostException("Broken system behaviour for dns lookup of $hostname").apply {
                initCause(e)
            }
        }
    }
}
```
