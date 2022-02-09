
PolarisMesh 的注册发现与健康检查是如何实现的？

## 0. SDK 的封装

在分析控制面逻辑之前，我们先看一下客户端 SDK 的逻辑，以 [polaris-go](https://github.com/polarismesh/polaris-go) 为例：针对 RPC 调用关系中的两个角色，Golang SDK 分别提供的了 `ProviderAPI` 和 `ConsumerAPI` 两个 interface，这两个接口的实现逻辑只是薄薄的一层，与控制面的交互以及核心的处理逻辑都是封装在了 model 包中，其中最重要的一个 interface 就是 `Engine`。

![](./assets/polaris-gosdk-1.png) 

本文主要关注图中标红的几个方法：注册、注销和发现的逻辑比较直观，就是与控制面进行一次 GRPC 交互；发现的逻辑稍微复杂一些，其中多了一个本地缓存。

## 1. 注册注销

### 1.1 注册

注册调的 GRPC 方法是 `rpc RegisterInstance(Instance) returns(Response)`。

请求参数 `Instance` 的字段很多，对于注册这个场景，
```
#必填字段
namespace //命名空间
service //服务名
host //实例ip
port //实例端口

#开启健康检查，必填字段
health_check //目前只支持HEARTBEAT，需要设置一个 TTL

# 重要的可选字段
metadata //实例标签，examples 中规则匹配的“标签”就是这里设置的
service_token // 认证信息，需要鉴权时设置
```

控制面收到注册请求后，处理逻辑位于 `service/instance.go` 中的 `CreateInstance` 方法。

![](./assets/polaris-register-1.png)

本质上就是写三张表：instance 实例信息表，health_check 健康检查配置表和 instance_metadata 实例标签信息表。

因为写库的成本比较高，为了提高吞吐量加了一个批处理逻辑，这部分逻辑用 Golang 写很简洁，核心逻辑位于 `service/batch/instance.go` 的 `mainLoop` 中。

![](./assets/polaris-bc-1.png)

在批处理的 `batch.registerHandler` 处理逻辑中有个小 BUG，没有在事务中对服务加共享锁（对应串行处理中的 `rlockServiceWithID`），极端情况下会有产生脏数据的风险。

**关于隔离的问题**

在数据表 instance 中是通过 isolate 字段标识实例是否隔离，所以在注册流程中首先会通过 instance id 查询实例是否存在，如果已存在并且是隔离状态，那么需要继续保持隔离状态。管理台是通过 /instances/isolate/host 这个 http 接口进行隔离相关的操作，每次对实例进行操作都会更新 revision 字段，实现方式是 UUID，表示 instance 信息（包含 instance、health_check 和 instance_metadata 这三张表）的版本。

返回的消息是 `Response`，字段很多，对于注册这个场景使用这三个字段，
```
code //六位状态码，前三位参照 HTTP Status，后三位业务自定义
info
instance // 实例信息（带上了id）
```

### 1.2 注销

注册调的 GRPC 方法是 `rpc DeregisterInstance(Instance) returns(Response)`。

请求参数是也是 `Instance`，对于注销这个场景，
```

```


## 2. 监控检查

## 3. 服务发现