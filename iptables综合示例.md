### 综合示例
1. iptables防护web服务
分析：
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

2. iptables路由SNAT使内网PC上网
分析：网络拓扑
```shell
private <---------------> iptables <---------------> public
  pc      10.250.2.0/24    router    10.250.1.0/24   network
```
配置：为iptables主机添加两块网卡，一个放在内网，一个放在公网。启用内核路由转发功能
```shell
# echo "1" > /proc/sys/net/ipv4/ip_forward		#临时开启路由转发
# echo "net.ipv4.ip_forward = 1" >> /etc/sysctl.conf 	#永久开启路由转发
# sysctl -p 	#重新加载sysctl.conf文件

# iptables -t nat -A POSTROUTING -s 10.250.2.0/24 -j SNAT --to 10.250.1.11
	配置SNAT，将所有内网数据转到10.250.1.11这个IP上
```
注：以上操作都是在iptables主机上配置的，对于内网内的主机，需要更改一下IP和网关，将网关地址设置为iptables主机的内网网卡IP <br />
测试：使用内网主机ping百度正常

3. 拒绝访问服务器本身/拒绝通过服务器访问别的机器(网络拓扑还是采用示例2的)
分析：
	1. iptables每个链的作用
	2. filter表的过滤位置在那个链
	3. 匹配条件和处理动作
```shell

```