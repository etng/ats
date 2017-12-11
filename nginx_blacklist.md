## 更新黑名单文件

```
cat <<EOT >blacklist.txt
1.2.2.0-1.2.2.255;
1.2.4.0-1.2.5.255;
1.2.8.0-1.2.8.255;
1.4.4.0-1.4.4.255;
1.8.0.0-1.8.0.255;
1.8.2.0-1.8.7.255;
EOT

cat <<'EOT' >ipdeny.py
#!/bin/env python
# encoding=utf-8
import sys
from netaddr import IPRange
def iprange2cidrs(ipr):
    ipr = ipr.replace(';', '').strip()
    return map(str, IPRange(*ipr.split('-')).cidrs())
cidrs = map(iprange2cidrs, open(sys.argv[1], 'r').readlines())
scidrs = set()
for _ in cidrs:
    scidrs.update(_)
with open('denyip.conf', 'w') as f:
    f.write("\n".join(map(lambda _: 'deny %s;' % _, scidrs)))
print 'done'
EOT
chmod +x ipdeny.py
./ipdeny.py blacklist.txt
```

## 更新 nginx 配置文件
```
server{
  include /path/to/denyip.conf
}
```
