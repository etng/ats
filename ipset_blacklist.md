# 屏蔽指定 ip 列表访问
## 安装软件

```
yum install ipset -y
chkconfig ipset on
chkconfig --list ipset
```

## 初始化防火墙

```
ipset create blacklist hash:net hashsize 6553608 maxelem 655360000
# 封禁 80 端口
iptables -I INPUT -m set --match-set blacklist src -p tcp --dport 80 -j DROP
# 封禁 443 端口
iptables -I INPUT -m set --match-set blacklist src -p tcp --dport 443 -j DROP
iptables -L
```

##  查看黑名单数据统计

```
ipset list --terse
```

## 封禁指定 ip 或 ip 段

注意以下几种格式都可以

```
ipset add blacklist 192.168.1.100-192.168.1.200
ipset add blacklist 192.168.2.1/24
ipset add blacklist 192.168.1.1/16
ipset add blacklist 10.141.25.36
```
# 解封指定 ip

```
export ip2unblock=192.168.1.199
ipset del blacklist ${ip2unblock} -exist && ipset add blacklist ${ip2unblock} nomatch
```

# 查看封禁情况

* `ipset test blacklist 192.168.1.188`
看到`is in set blacklist.` 表示已经封禁
* `ipset test blacklist 192.168.1.199`
看到`is NOT in set blacklist.` 表示已经解封
