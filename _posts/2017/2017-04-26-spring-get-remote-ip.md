---
title: 部署在阿里云 SLB 后面的spring 应用如何获得用户真实 IP
date: '2017-04-26 00:11'
categories: 开发
tag: 阿里云 SLB 真实IP RemoteIpFilter Spring
---

如果你的应用是部署在阿里云上面的, 往往在应用的前面会架设一个SLB(负载均衡). 如果 SLB配置成四层转发, 那么你的应用看到的 http 连接的对端地址为真实的用户 IP, `HttpServletRequest.getRemoteAddr()`能返回正确的用户 IP. 但是如果 SLB 配置成`七层转发`, `HttpServletRequest.getRemoteAddr()`返回的是 SLB对内地址, 比如: `100.109.*.*`. 在这种情况下如何获取真实 IP 呢? 其实 SLB 在转发 http 请求时, 会增加一个 header: `X-FORWARDED-FOR`. 真实 IP 就放在这个头中. 所一个简单的获取真实 IP 的方法就是读这个头:
```java
String realIP = request.getHeader("X-FORWARDED-FOR");
```

上面的代码假设`X-FORWARDED-FOR` header 中只有一个 IP, 实际情况可能会存在多个 IP, IP之间用逗号隔开. 为什么会有多个 IP 呢? 因为用户机器和你的服务器之间可能存在多个 http 代理或者负载均衡, 每个代理都可能会在`X-FORWARDED-FOR`中加上一个 IP. 另外, 除了获取真实 IP, 我们有时候也想获取真实协议(http 或者 https, 比如我们往往在SLB层把 https 请求 变成 http 请求再转发到后端服务器, 这样后端服务器收到的都是 http 请求). `org.apache.catalina.filters.RemoteIpFilter`就是用来处理这种情况的. 下面代码为 spring 下使用 RemoteIpFilter的例子:

```java
@Configuration
public class RemoteIpConfig {
    static final String internalProxyPattern =
      "10\\.\\d{1,3}\\.\\d{1,3}\\.\\d{1,3}|" +
      "192\\.168\\.\\d{1,3}\\.\\d{1,3}|" +
      "169\\.254\\.\\d{1,3}\\.\\d{1,3}|" +
      "127\\.\\d{1,3}\\.\\d{1,3}\\.\\d{1,3}|" +
      "172\\.1[6-9]{1}\\.\\d{1,3}\\.\\d{1,3}|" +
      "172\\.2[0-9]{1}\\.\\d{1,3}\\.\\d{1,3}|" +
      "172\\.3[0-1]{1}\\.\\d{1,3}\\.\\d{1,3}|" +
      "100\\.10[0-9]{1}\\.\\d{1,3}\\.\\d{1,3}";

      @Bean
      public RemoteIpFilter remoteIpFilter() {
        RemoteIpFilter ipFilter = new RemoteIpFilter();
        ipFilter.setProtocolHeader("X-Forwarded-Proto");
        ipFilter.setInternalProxies(internalProxyPattern);
        return ipFilter;
      }
```

其中 internalProxyPattern为内网代理 IP 模式. 前面几行是 RemoteIpFilter的默认配置, 主要是增加了最后一行, `"100\\.10[0-9]{1}\\.\\d{1,3}\\.\\d{1,3}"`, 这是 阿里云SLB对内地址段. 实际上整个`100.64.0.0/10`网段都可能会被阿里云用作SLB 的对内地址, 如果你发现SLB 对内地址不在`"100\\.10[0-9]{1}\\.\\d{1,3}\\.\\d{1,3}"`中, 那么需要再增加相关的正则表达式.
