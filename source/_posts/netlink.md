---
title: 网络拓扑图
date: 2024-11-20 21:09:00
hidden: true
noindex: true
tags:
- 网络拓扑关系
categories: 网络
---

## 专业名词介绍

| 名词    | 描述                                                         | 备注 |
| ------- | ------------------------------------------------------------ | ---- |
| BGP     | BGP广泛应用于互联网服务提供商（移动、联动、电信）之间的网络互联，以及大型企业或组织的内部网络互联。主要用于互联网AS（自治系统）之间的互联，控制路由的传播和选择最好的路由。 |      |
| DGW     | DIDI Gateway，是一套实现多网统一接入，支持自动负载均衡的系统。四层接入（tcp udp），通过VIP+VPORT+PROTOCOL 来区分不同的业务。 |      |
| route   | 参考文档：https://base.xiaojukeji.com/docs/helm/787          |      |
| inroute | 参考文档：https://base.xiaojukeji.com/docs/helm/788          |      |



## IDC 机房分布

> 滴滴线上机房互联及网络资源情况：

> http://wiki.intra.xiaojukeji.com/pages/viewpage.action?pageId=1167769820

![img](https://s3-cooper.xiaojukeji.com/didoc-upload-image-prod/17298640467780dc7984def15ff70bd7a6d70c631f3ee/image.png)

## 接入层链路

> http://wiki.intra.xiaojukeji.com/pages/viewpage.action?pageId=202549036

用户端请求到 DGW 的链路如下图所示

![img](https://cooper.didichuxing.com/uploader/f/v0K9bZfFSL1xcmqB.png?accessToken=eyJhbGciOiJIUzI1NiIsImtpZCI6ImRlZmF1bHQiLCJ0eXAiOiJKV1QifQ.eyJleHAiOjE3Mjk4NzAyMDAsImZpbGVHVUlEIjoibDc4d3h1WndmUDkyZjluVyIsImlhdCI6MTcyOTg2OTYwMCwiaXNzIjoidXBsb2FkZXJfYWNjZXNzX3Jlc291cmNlIiwidXNlcklkIjoxMDAwMDAzNTI0NH0.3Apk38ZvMUHnVgSKgBMXsdQCJRm8kP3Dh9_-qVekSLs)

## 网络链路

> router 和 inroute 接入层

> [http://wiki.intra.xiaojukeji.com/pages/viewpage.action?pageId=116794551](http://wiki.intra.xiaojukeji.com/pages/viewpage.action?pageId=116794551#id-接入层集群列表-Inrouter)

![img](https://cooper.didichuxing.com/cooper_gateway/cn/shimo-images/Yf2L2GjL4MgBpSJm/%E7%BD%91%E7%BB%9C%E9%93%BE%E8%B7%AF%E6%8B%93%E6%89%91%E5%9B%BE.png)

![img](https://s3-cooper.xiaojukeji.com/didoc-upload-image-prod/17298661645208cc25d7af2e69e47492dd9b3881be78f/image.png)



![img](https://s3-cooper.xiaojukeji.com/didoc-upload-image-prod/1729865650633a2cb0b7c8bd1e3831fe3d6d19fbddf66/image.png)



![img](https://s3-cooper.xiaojukeji.com/didoc-upload-image-prod/1729868055139f9e851f7cfaa2cc5b8a46006de1514fb/image.png)





## 请求解析链路

举例说明：

以星海获取资质列表请求为例：https://page-biz.xiaojukeji.com/xinghai-sale-car/req-proxy/xinghai-sale-car/dop/dmp/company/model/list?version=2&businessId=2&companyId=517879

1. DNS 协议解析 CNAME：[hna-router-gs-v6-static.udache.com](http://hna-router-gs-v6-static.udache.com). （管理灵活，方便做切流、容灾，就近读等优势）

1. ![img](https://s3-cooper.xiaojukeji.com/didoc-upload-image-prod/1730273172312ddf48136d05d373a16073e40ca10a603/image.png)

1. [hna-router-gs-v6-static.udache.com](http://hna-router-gs-v6-static.udache.com). 解析A 记录地址为：157.255.76.19、157.255.76.17 （联通、移动、电信 都有配置，共 15+ 地址）。

1. ![img](https://s3-cooper.xiaojukeji.com/didoc-upload-image-prod/173027316083595d36299b10cf769e0498a28a34566fc/image.png)

1. 分析 VIP 地址，反查是处于华新园机房，odin 节点是 didi.op.dfe.router.gs.gz01（[详见 router 明细](http://wiki.intra.xiaojukeji.com/pages/viewpage.action?pageId=123096872)）

1. ![img](https://s3-cooper.xiaojukeji.com/didoc-upload-image-prod/1730361019474b2f27053b0acea76e4b5a8d208e68f53/image.png)

1. router 配置在海姆平台上，根据域名可以配置转发到对应 USN 服务节点上（odin 上一个服务的 USN 唯一不变）

1. ![img](https://s3-cooper.xiaojukeji.com/didoc-upload-image-prod/1730273598207f16e825fc981ff447f107fb1ee2f695d/image.png)

1. 找到对应后端服务，biz-gs-node-fe-biz_agent_main，可以看见是先请求到 node 前端

1. ![img](https://s3-cooper.xiaojukeji.com/didoc-upload-image-prod/173027371024026e493eacdc3c40bf3585d6cbdfca656/image.png)

1. 前端 node 中间层后台：https://biz-mc.intra.xiaojukeji.com/，识别出 [/dop/dmp/company/model/list](https://page-biz.xiaojukeji.com/xinghai-sale-car/req-proxy/xinghai-sale-car/dop/dmp/company/model/list?version=2&businessId=2&companyId=517879)，按照path配置进行转发（有 sso、passport、登录不校验）。转发Host配置成对应后端 inroute 配置 10.88.128.16:8000

1. ![img](https://s3-cooper.xiaojukeji.com/didoc-upload-image-prod/17302743396301da0681b341952e5ab52f2cbddd4f4c1/image.png)

1. hnb VIP inroute 识别 /dop/dmp 前缀进行转发到 odin 节点：hnb-v.dmp-enter.dmp.driver-operating.biz.didi.com，负载均衡到其中一台机器。



TODO：上面补充一个流程图，每个环节使用的平台功能介绍。



### 参考文献

流量接入和网络拓扑：http://wiki.intra.xiaojukeji.com/pages/viewpage.action?pageId=901735655

CMDB 系统对外开放接口：http://wiki.intra.xiaojukeji.com/pages/viewpage.action?pageId=89667089