PolarisMesh 的路由是如何实现的？



## 0. 规则

### 0.1 规则下发

路由规则的下发与上文中“服务发现”的逻辑是一样的。

### 0.2 规则信息

每个服务可以配置两个规则：被调规则和主调规则，存储在 **routing_config** 表中。

以 polaris-go 中的 example 为例，设置了一个“被调规则”：

```shell
mysql> select * from routing_config \G
*************************** 1. row ***************************
        id: 82c9f27067da4827849f66286d480e32
 in_bounds: [{"sources":[{...
out_bounds: null
  revision: e0b0ab3239bd47f1b0856205e818fa93
      flag: 0
     ctime: 2022-03-17 18:07:56
     mtime: 2022-03-17 18:07:56
1 row in set (0.00 sec)
```

“被调”也就是 in_bounds 入口流量，具体的一系列规则存储为一个 json 数组：

```json
[
  {
    "sources": [
      {
        "service": {
          "value": "*"
        },
        "namespace": {
          "value": "*"
        },
        "metadata": {
          "env": {
            "value": {
              "value": "dev"
            }
          }
        }
      }
    ],
    "destinations": [
      {
        "service": {
          "value": "polaris_go_provider"
        },
        "namespace": {
          "value": "default"
        },
        "metadata": {
          "env": {
            "value": {
              "value": "test"
            }
          }
        },
        "priority": {},
        "weight": {
          "value": 100
        },
        "isolate": {
          "value": true
        }
      }
    ]
  }
]
```

“主调” out_bounds 的规则格式是一样的。区别主要在于 in_bounds 的 destinations.service 是确定的，out_bounds 的 sources.service 也是确定的，也就是服务自身，简而言之就是视角不一样。

对于上面这条“被调”规则，翻译成中文就是：对于任意来源（通分配星号），如果请求中带有 env:dev 标签，那么 100% 路由到我的带 env:test 标签的实例。

// todo isolate 隔离的意思是？







