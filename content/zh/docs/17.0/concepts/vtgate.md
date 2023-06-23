---
title: VTGate 
---

VTGate 是一个轻量级代理服务器，它将流量路由到正确的 [VTTablet](../tablet) 服务器并将合并的结果返回给客户端。 它支持 MySQL 协议和 Vitess gRPC 协议。 因此，您的应用程序可以连接到 VTGate，就像它是 MySQL 服务器一样。

当将查询路由到适当的 VTablet 服务器时，VTGate 会考虑分片方案、所需的延迟以及表及其底层 MySQL 实例的可用性。

**Related Vitess Documentation**

* [Execution Plans](../execution-plans)