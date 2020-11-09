### 综合示例
1. iptables防护web服务
分析：单主机服务防护 <br />
注意：
	1. 环回口lo处理
	2. 协议+端口处理
	3. 状态检测处理
```shell
# iptables -t filter -A INPUT -i lo -j ACCEPT
	允许换回口访问
# iptables -t filter -A INPUT -p tcp -m multiport --dport 22,80 -j ACCEPT
	放行ssh服务和http服务
# iptables -t filter -A INPUT -m state --state RELATED,ESTABLISHED -j ACCEPT
	允许 通过ssh服务或http服务衍生出的连接 和 已经建立过连接的流量 通过。状态防火墙能识别tcp或udp会话，非状态防火墙只能根据端口识别，不能识别会话
# iptables -t filter -P INPUT DROP
	更改默认策略为丢弃，拒绝所有。更改默认策略必须要添加按包状态匹配的规则，否则可能出现流量出去回不来的情况
```
注：一般iptables的OUTPUT出口都放行，允许本机主动访问外网 <br /><br />

测试：
```shell
$ ping 10.250.1.11		#ping测试无回包
$ ssh root@10.250.1.11	#ssh测试连接正常

$ nmap 10.250.1.11		#使用nmap扫描一下目标开放的端口
	PORT   STATE  SERVICE
	22/tcp open   ssh
	80/tcp closed http

$ telnet 10.250.1.11 80	#服务端打开http服务后，客户端使用telnet测试正常
Trying 10.250.1.11...
Connected to 10.250.1.11.
Escape character is '^]'.

# iptables -nL			#查看防火墙条目
```

2. 