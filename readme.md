iptables基础
============

概述：ip包过滤系统，实际由netfilter和iptables组成 <br />
netfilter/iptables关系：
- netfilter是内核的一部分，由包过滤表组成，这些表包含内核用来控制包过滤处理的规则集
- iptables是用户空间的一种工具，简化插入、修改、删除包过滤表中的规则

```diff
iptables内置三表filter,nat,mangle五链postrouting,prerouting,input,output,forward，所有规则配置后立即生效
```

### 三表
- filter：负责过滤数据包，包扩input,output,forward三个规则链
- nat：负责网络地址转换，包括prerouting,postrouting,output三个规则链
- mangle：负责修改数据包内容，用于流量整形，给数据包打标，默认包括五链

```diff
+ iptables中还有一个raw表，用于处理异常，包括prerouting,output两个规则链，使用较少
```


### 五链
- input：匹配目标IP是本机的数据
- forward：匹配流经本机的数据
- prerouting：修改DIP，用于外网访问内网，做DNAT。路由判断之前的nat，地址转换后再匹配路由表，此表通常自动配合postrouting使用
- postrouting：修改SIP，用于内网访问外网，做SNAT。路由判断之后的nat，只有先匹配到目的路由之后才会做地址转换，做了SNAT后数据包回来时会自动做DNAT
- output:出口数据包

补充：在私有主机普及的情况下解决ipv4上网的方案只有两种:nat 和 proxy <br />

示例：查看nat表
```shell
# iptables -t nat -L
	-t：指定要查看的表
	-L：表示list
```

iptables过滤封包流程
-------------------
```shell
  --> PREROUTING --> [ROUTE] --> FORWARD --> POSTROUTING -->
        mangle          |        mangle   ↑     mangle
         nat            |        filter   |       nat
                        ↓                 |
                      INPUT             OUTPUT
                        |mangle           ↑
                        |filter           |mangle
                        |                 | nat
                        ↓----> local ---->|filter
```
整体数据包分两类：
1. 发送给防火墙本身的数据包
	此类数据包将经过**除了forward链**以外的四个链，可以通过在不同的位置调整不同的表中的规则对数据包设置规则
2. 经过防火墙的包
	经过防火墙的包不会经过input和output两个链

```diff
- 既然一个链可以在多个表中生效，那么四个表必然有优先顺序：
- raw > mangle > nat > filter
```

#### 链间的匹配顺序(根据过滤封包流程看)
- 入站数据：prerouting、input
- 出站数据：output、postrouting
- 转发数据：prerouting、forward、postrouting

#### 链内的匹配顺序
- 自上向下按顺序匹配规则，匹配到相应规则即停止
- 链内无相匹配的规则时，按照该链的默认策略处理（未修改情况下，默认策略为允许）

#### iptables命令语法格式
iptables [-t 表名] 管理选项 [链名] [条件匹配] [-j 目标动作或跳转]
	注：iptables命令区分大小写
```diff
- 不指定表名时，默认为filter表
- 不指定链名时，默认为该表内所有链
- 除非设置规则链的缺省策略，否则需要指定匹配条件
```