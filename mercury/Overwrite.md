[Network Abstraction Layer - Mercury (mercury-hpc.github.io)](https://mercury-hpc.github.io/user/na/)

# Overview

Mercury is composed of three main layers:

1. the [network abstraction layer](https://mercury-hpc.github.io/user/na/), 它在较低层次的网络结构之上提供高性能的通信接口。
2. the [RPC layer](https://mercury-hpc.github.io/user/hg/), 它为用户提供了发送和接收RPC元数据(小消息)所需的组件。这包括函数参数的序列化和反序列化;
3. the [bulk layer](https://mercury-hpc.github.io/user/hg_bulk/),它提供了处理大型参数的必要组件——这意味着通过RMA传输大量数据;
4. the (*optional*) [high-level RPC layer](https://mercury-hpc.github.io/user/hg_macros/), 它旨在提供一个方便的API，构建在较低的层之上，并提供用于生成RPC存根的宏以及序列化和反序列化函数。

These three main layers can be summarized in the following diagram:

![Overview](assets/overview.svg+xml)

根据定义，RPC调用由一个进程发起，称为* *origin* *，并转发到另一个进程，后者将执行该调用，称为* *target* *。每一方，origin和target，都使用RPC * *processor* *来序列化和反序列化通过接口发送的参数。调用参数相对较小的函数会导致使用网络抽象层公开的短消息传递机制，而包含大数据参数的函数则会额外使用远程内存访问(remote memory access, RMA)机制。请注意，当批量数据足够小时，如果可以容纳，Mercury会自动将其与元数据一起嵌入。