iptables操作命令
===============
将iptables操作分为几个类别

#### 查看规则
示例：查看详细规则
```shell
# iptables -t filter -nL
	-v：显示详细信息，包括每条规则的匹配包数量、匹配字节数和网卡接口
	-n：只显示IP地址和端口号，不显示域名和服务名称
	-x：在-v的基础上，禁止单位自动换算(K、M)
# service iptables save		保存对iptables做的操作
```

#### 添加规则
示例：拒绝所有人以任何协议访问
```shell
# iptables -t filter -A INPUT -j DROP
```

#### 删除规则
示例：删除filter表INPUT链中的第7条规则
```shell
# iptables -t filter -D INPUT 7
```
注意：
```diff
- 1. 若规则列表中有多条相同的规则时，按内容匹配则只删除序号最小的一条
- 2. 按号码匹配删除时，确保规则号码小于已有规则数
- 3. 按内容匹配删除时，确保规则存在
```
示例：按内容删除规则
```shell
# iptables -t filter -D INPUT -s 192.168.0.1 -j DROP
```

#### 设置规则链默认策略
示例：设置filter表的INPUT链默认策略为拒绝
```shell
# iptables -t filter -P INPUT DROP
```

#### 清空规则
示例：清空指定链的规则
```shell
# iptables -t filter -F INPUT
```
注：
```diff
- 1. 如果已经将INPUT链的默认策略改为非ACCEPT，那清空规则的操作就要小心。
- 2. 不指定链时，清空所有链的规则
- 3. -F仅清空链中规则，不影响默认规则
```

#### 匹配条件
匹配条件分4种： 
* 流入、流出接口：-i、-o 
* 源、目的地址：-s、-d 
* 协议类型：-p 
* 源、目的端口：--sport、--dport 

按网络接口匹配： <br />
-i [匹配数据进入的网卡]	#此参数主要用于nat表，做DNAT <br />
-o [匹配数据流出的网卡] <br /><br />

按源、目的地址匹配： <br />
-s [匹配源地址]	#可以是IP、NET(网段)、DOMAIN，也可空，空表示任意地址 <br />
-o [匹配目的地址]	 <br /><br />

按协议类型匹配：<br />
-p [匹配协议类型]		#可以是TCP、UDP、ICMP等，也可空<br /><br />

按源、目的端口匹配：<br />
--sport [匹配源端口]		#可以是个别端口、可以是端口范围<br />
    --sport 1000:3000
--dport [匹配目的端口]<br />
注：--sport 和 --dport 必须配置 -p 使用<br />

#### 动作（处理方式）
几种处理方式：
* ACCEPT 	允许
* DROP	 	丢弃，不给对端任何回应
* REJECT 	拒绝，给对端回应
* SNAT	 
* DNAT		端口映射
* MASQUERADE 伪装IP地址
* REDIRECT	端口重定向
* MARK		打标签

6种处理方式示例：
```shell
# iptables -t filter -A INPUT -j ACCEPT
	允许所有数据访问本机

# iptables -t filter -A FORWARD -s 192.168.80.39 -j DROP
	阻止源地址为192.168.80.39的流量经过本机

# iptables -t nat -A POSTROUTING -s 192.168.0.0/24 -j SNAT --to 1.1.1.1
# iptables -t nat -A POSTROUTING -s 192.168.0.0/24 -j SNAT --to 1.1.1.1-1.1.1.10
	SNAT仅支持nat表的postrouting链，SNAT支持转换为单IP，也支持转换到IP地址池

# iptables -t nat -A PREROUTING -i eth0 -p tcp --dport 80 -j DNAT --to 192.168.0.1
# iptables -t nat -A PREROUTING -i eth0 -p tcp --dport 81 -j DNAT --to 192.168.0.1:81
	DNAT仅支持nat表的prerouting链，DNAT支持内、外网主机间端口映射

# iptables -t nat -A POSTROUTING -s 192.168.0.0/24 -o eth0 -j MASQUERADE 
	将源地址为192.168.0.0/24网段的流量进行地址伪装，转换成eth0上的IP地址
```

### iptables扩展模块
4个附加模块：
* 包状态匹配：state
* 源mac匹配：mac
* 包速率匹配：limit
* 多端口匹配：multiport

按包状态匹配<br />
-m state --state 状态<br />
状态：NEW、RELATED、ESTABLISHED、INVALID<br />
	NEW：有别于tcp的syn。刚开始建立连接
	ESTABLISHED：连接态。连接完成或已经建立过连接
	RELATED：衍生态，与conntrack关联（FTP）。tcp衍生链接
	INVALID：不能被识别属于哪个连接或没有任何状态
示例：
```shell
# iptables -t filter -A INPUT -m state --state RELATED,ESTABLISHED -j ACCEPT
	允许衍生态或已经建立过连接的流量通过
```

按源mac匹配<br />
-m mac --mac-source MAC<br />
示例：
```shell
# iptables -t filter -A FORWARD -m mac --mac-source 28:6e:d4:89:1b:c5 -j DROP
	报文经过路由后，数据包中原有的mac信息会被替换，所以在POSTROUTING后的iptables中使用mac模块是无意义的
```

按包速率匹配<br />
-m limit --limit 匹配速率 [--burst 缓冲数量]<br />
示例：
```shell
# iptables -A FORWARD -d 192.168.0.1 -m limit --limit 50/s -j ACCEPT
# iptables -A FORWARD -d 192.168.0.1 -j DROP
	按照包速率限速流量，仅允许每秒发送50个数据包，做到限制需要两条规则
```

多端口匹配<br />
-m multiport [--sports|--dports|--ports] 端口1[,端口n...]<br />
一次性匹配多个端口，可区分源端口，目的端口或不指定端口<br />
示例：
```shell
# iptables -A INPUT -p tcp -m multiport --dports 21,22,25,80,110 -j ACCEPT
	允许所有客户端访问本机多个端口，必须与-p参数一起使用
```