# 屏蔽指定 ip 列表访问

## 软件加规则

```
yum install ipset -y
ipset create blacklist hash:ip
iptables -I INPUT -m set --match-set blacklist src -j DROP
iptables -L
chkconfig ipset on
chkconfig --list ipset
```

# 屏蔽指定 ip

```
ipset add blacklist 1.2.3.4
ipset list blacklist
```

# 备份黑名单

```
ipset save > /etc/ipset.conf
```
