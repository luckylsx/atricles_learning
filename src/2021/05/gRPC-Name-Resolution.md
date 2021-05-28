>- 原文地址：https://github.com/grpc/grpc/blob/master/doc/naming.md

## gRPC名称解析

### 概述

gRPC支持将DNS作为默认名称系统。 在各种部署中使用了许多备用名称系统。 我们支持足够通用的API，以支持各种名称系统和名称的相应语法。 各种语言的gRPC客户端库将提供一种插件机制，以便可以插入用于不同名称系统的解析器。

### 详细设计

#### 名称语法

用于gRPC通道构造的完全限定的自包含名称使用RFC 3986中定义的URI语法。

URI scheme 指示要使用的解析程序插件。 如果未指定方案前缀或方案未知，则默认使用dns方案。

URI路径指示要解析的名称。

大多数gRPC实现都支持以下URI schemes：

* dns:[//authority/]host[:port] -- DNS (默认)
    * host 是要通过DNS解析的主机。
    * port 是每个地址要返回的端口。 如果未指定，则使用443（但对于不安全的通道，某些实现默认为80）。
    * authority 指定要使用的DNS服务器，尽管只有某些实现才支持。 （在C-core中，默认的DNS解析器不支持此功能，但是基于c-ares的解析器支持以 IP:port 的形式指定此名称。）
* unix:path, unix://absolute_path -- Unix domain sockets (仅Unix系统)
    * path 表示所需套接字的位置。
    * 在第一种形式中，路径可以是相对的，也可以是绝对的。 在第二种形式中，路径必须是绝对斜线（即，实际上将存在三个斜线，两个斜线在路径之前，另一个斜线用于开始绝对路径）。
* unix-abstract：abstract_path-抽象名称空间中的Unix域套接字（仅Unix系统）
    * abstract_path 表示抽象名称空间中的名称。
    * 该名称与文件系统路径名没有任何关系。
    * 没有权限将应用于套接字-任何进程/用户都可以访问套接字。
    * Abstract套接字的基础实现使用空字节（'\ 0'）作为第一个字符； 实现将在此null之前添加。 不要在abstract_path中包含null。
    * abstract_path不能包含空字节
    * TODO（[https://github.com/grpc/grpc/issues/24638](https://github.com/grpc/grpc/issues/24638)）：Unix允许抽象套接字名称包含空字节，但是gRPC C核心实现不支持此功能。


gRPC C核心实现支持以下方案，但其他语言可能不支持以下方案：

* ipv4:address[:port][,address[:port],...] -- IPv4 地址
    * 可以指定格式为 address[:port]的多个逗号分隔地址
        * address 是要使用的IPv4地址。
        * port 是要使用的端口。 如果未指定，则使用443。
* ipv6:address[:port][,address[:port],...] -- IPv6 地址
    * 可以指定格式为 address[:port]的多个逗号分隔地址：
    * address是要使用的IPv6地址。 要与端口一起使用，该地址必须放在方括号（[and]）中。 例如：ipv6:[2607:f8b0:400e:c00::ef]:443或ipv6:[::]:1234
    * port 是要使用的端口。 如果未指定，则使用443。

将来，其他的scheme 例如像 etcd 也会被添加。

### 解析器插件

gRPC客户端库将使用指定的scheme 来选择正确的解析器插件，并将完整的名称字符串传递给它。

解析程序应该能够与授权机构联系，并获得解决方案，以便他们返回gRPC客户端库。 返回的内容包括：

* 解析地址列表（IP地址和端口）。 每个地址都可以具有一组与之关联的任意属性（键/值对），这些属性可以用于将信息从解析器传递到[负载平衡](https://github.com/grpc/grpc/blob/master/doc/load-balancing.md)策略。
* [服务配置](https://github.com/grpc/grpc/blob/master/doc/service_config.md).

插件API允许解析器持续监视 endpoint 并根据需要返回更新的解析数据。
