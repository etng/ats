# CentOS 6.9 + Apache Traffic Server 7.1.1 安装记录

注意下文中，步骤1和2可以同时进行，完毕后再进行第3步，第4步可选，最后第5步。

## 1. 下载并编译安装

```bash
# 同步下系统时间
yum install -y ntpdate
ntpdate time1.aliyun.com

# 各种依赖包
yum install -y gcc gcc-c++ pkgconfig pcre-devel tcl-devel expat-devel openssl-devel perl-ExtUtils-MakeMaker bzip2
yum install -y gcc gcc-c++ pkgconfig pcre-devel tcl-devel expat-devel openssl-devel bzip2
yum install -y libcap libcap-devel hwloc hwloc-devel ncurses-devel libcurl-devel nwind libunwind-devel autoconf automake libtool
yum install centos-release-scl -y
yum install devtoolset-3-gcc* -y
scl enable devtoolset-3 bash
# gcc4.9 needed withc c++ 11

# 下载、解压、安装
wget http://www-us.apache.org/dist/trafficserver/trafficserver-7.1.1.tar.bz2
tar -jxvf trafficserver-7.1.1.tar.bz2
cd trafficserver-7.1.1
groupadd ats
useradd -g ats ats
./configure --prefix=/ --with-user=ats --with-group=ats --enable-experimental-plugins
# 如果有多核的机器不要浪费
make -j 8
make install
```

## 2. 优化系统设置

### 2.1 并发

```bash
cat << 'EOT' >> /etc/sysctl.conf
fs.file-max=655350
net.ipv4.tcp_max_tw_buckets = 300000
net.ipv4.tcp_sack = 1
net.ipv4.tcp_window_scaling = 1
net.ipv4.tcp_max_syn_backlog = 65536
net.core.netdev_max_backlog = 32768
net.core.somaxconn = 32768
net.core.rmem_default=98304
net.core.wmem_default=98304
net.core.rmem_max=2097152
net.core.wmem_max=2097152
net.ipv4.tcp_rmem=4096 98304 2097152
net.ipv4.tcp_wmem=4096 98304 2097152
net.ipv4.tcp_low_latency=1
net.ipv4.tcp_slow_start_after_idle=0
net.ipv4.tcp_timestamps = 1
net.ipv4.tcp_fin_timeout = 20
net.ipv4.tcp_synack_retries = 2
net.ipv4.tcp_syn_retries = 2
net.ipv4.tcp_syncookies = 0
#net.ipv4.tcp_tw_len = 1
net.ipv4.tcp_tw_reuse = 1
net.ipv4.tcp_tw_recycle = 1
net.ipv4.tcp_mem = 94500000 915000000 927000000
net.ipv4.tcp_max_orphans = 3276800
net.ipv4.ip_local_port_range = 1024 65000
net.nf_conntrack_max = 655350
net.netfilter.nf_conntrack_max = 655350
net.netfilter.nf_conntrack_tcp_timeout_close_wait = 60
net.netfilter.nf_conntrack_tcp_timeout_fin_wait = 120
net.netfilter.nf_conntrack_tcp_timeout_time_wait = 120
net.netfilter.nf_conntrack_tcp_timeout_established = 3600
EOT
sysctl -p /etc/sysctl.conf
```

### 2.2 突破系统打开文件限制

```bash
cat << 'EOT' >> /etc/security/limits.d/nofile.conf
* soft nofile 655350
* hard nofile 655350
EOT
cat <<EOF>>/etc/rc.local
#open files
ulimit -HSn 655350
#stack size
ulimit -s 655350
EOF
```

### 2.3 关闭 iptables 防火墙

```bash
#清理防火墙规则
iptables -F
#查看防火墙规则
iptables -L
#保存防火墙配置信息
/etc/init.d/iptables save
```

## 3. 更新配置文件

```bash
cd /etc/trafficserver/
mkdir /home/ats/cache
chown ats.ats /home/ats/cache
mv storage.config{,.bak}
cat << EOT >> storage.config
/home/ats/cache 3000GB
EOT

cat << EOT >> cache.config
url_regex=.* suffix=xml  ttl-in-cache=1d
url_regex=.* suffix=ts  ttl-in-cache=1d
url_regex=.* suffix=jpeg  ttl-in-cache=1d
url_regex=.* suffix=mp4  ttl-in-cache=1d
url_regex=.* suffix=zip  ttl-in-cache=1d
url_regex=.* suffix=gif  ttl-in-cache=1d
url_regex=.* suffix=ppt  ttl-in-cache=1d
url_regex=.* suffix=jpg  ttl-in-cache=1d
url_regex=.* suffix=swf  ttl-in-cache=1d
url_regex=.* scheme=http  ttl-in-cache=1h

EOT

mv records.config{,.bak}
cat << EOT >> records.config
##############################################################################
# *NOTE*: All options covered in this file should be documented in the docs:
#
#    https://docs.trafficserver.apache.org/records.config
##############################################################################
##############################################################################
# Thread configurations. Docs:
#    https://docs.trafficserver.apache.org/records.config#thread-variables
##############################################################################
CONFIG proxy.config.exec_thread.autoconfig INT 1
CONFIG proxy.config.exec_thread.autoconfig.scale FLOAT 1.500000
CONFIG proxy.config.exec_thread.limit INT 2
CONFIG proxy.config.accept_threads INT 1
CONFIG proxy.config.task_threads INT 2
CONFIG proxy.config.cache.threads_per_disk INT 8
CONFIG proxy.config.exec_thread.affinity INT 1
##############################################################################
# Specify server addresses and ports to bind for HTTP and HTTPS. Docs:
#    https://docs.trafficserver.apache.org/records.config#proxy.config.http.server_ports
##############################################################################
CONFIG proxy.config.http.server_ports STRING 80 443:ssl
##############################################################################
# Via: headers. Docs:
#     https://docs.trafficserver.apache.org/records.config#proxy-config-http-insert-response-via-str
##############################################################################
CONFIG proxy.config.http.insert_request_via_str INT 0
CONFIG proxy.config.http.insert_response_via_str INT 2
##############################################################################
# Parent proxy configuration, in addition to these settings also see parent.config. Docs:
#    https://docs.trafficserver.apache.org/records.config#parent-proxy-configuration
#    https://docs.trafficserver.apache.org/en/latest/admin-guide/files/parent.config.en.html
##############################################################################
CONFIG proxy.config.http.parent_proxy_routing_enable INT 0
CONFIG proxy.config.http.parent_proxy.retry_time INT 300
CONFIG proxy.config.http.parent_proxy.connect_attempts_timeout INT 30
CONFIG proxy.config.http.forward.proxy_auth_to_parent INT 0
CONFIG proxy.config.http.uncacheable_requests_bypass_parent INT 1
##############################################################################
# HTTP connection timeouts (secs). Docs:
#    https://docs.trafficserver.apache.org/records.config#http-connection-timeouts
##############################################################################
CONFIG proxy.config.http.keep_alive_no_activity_timeout_in INT 60
CONFIG proxy.config.http.keep_alive_no_activity_timeout_out INT 60
CONFIG proxy.config.http.transaction_no_activity_timeout_in INT 20
CONFIG proxy.config.http.transaction_no_activity_timeout_out INT 20
CONFIG proxy.config.http.transaction_active_timeout_in INT 900
CONFIG proxy.config.http.transaction_active_timeout_out INT 0
CONFIG proxy.config.http.accept_no_activity_timeout INT 60
CONFIG proxy.config.net.default_inactivity_timeout INT 86400
##############################################################################
# Origin server connect attempts. Docs:
#    https://docs.trafficserver.apache.org/records.config#origin-server-connect-attempts
##############################################################################
CONFIG proxy.config.http.connect_attempts_max_retries INT 3
CONFIG proxy.config.http.connect_attempts_max_retries_dead_server INT 1
CONFIG proxy.config.http.connect_attempts_rr_retries INT 3
CONFIG proxy.config.http.connect_attempts_timeout INT 30
CONFIG proxy.config.http.post_connect_attempts_timeout INT 1800
CONFIG proxy.config.http.down_server.cache_time INT 60
CONFIG proxy.config.http.down_server.abort_threshold INT 10
##############################################################################
# Negative response caching, for redirects and errors. Docs:
#    https://docs.trafficserver.apache.org/records.config#negative-response-caching
##############################################################################
CONFIG proxy.config.http.negative_caching_enabled INT 1
CONFIG proxy.config.http.negative_caching_lifetime INT 10
##############################################################################
# Proxy users variables. Docs:
#    https://docs.trafficserver.apache.org/records.config#proxy-user-variables
##############################################################################
CONFIG proxy.config.http.insert_client_ip INT 1
CONFIG proxy.config.http.insert_squid_x_forwarded_for INT 1
##############################################################################
# Security. Docs:
#    https://docs.trafficserver.apache.org/records.config#security
##############################################################################
CONFIG proxy.config.http.push_method_enabled INT 0
##############################################################################
# Enable / disable HTTP caching. Useful for testing, but also as an
# overridable (per remap) config
##############################################################################
CONFIG proxy.config.http.cache.http INT 1
##############################################################################
# Cache control. Docs:
#    https://docs.trafficserver.apache.org/records.config#cache-control
#    https://docs.trafficserver.apache.org/en/latest/admin-guide/files/cache.config.en.html
##############################################################################
CONFIG proxy.config.http.cache.ignore_client_cc_max_age INT 1
CONFIG proxy.config.http.normalize_ae_gzip INT 1
CONFIG proxy.config.http.cache.cache_responses_to_cookies INT 1
CONFIG proxy.config.http.cache.cache_urls_that_look_dynamic INT 1
    # https://docs.trafficserver.apache.org/records.config#proxy-config-http-cache-when-to-revalidate
CONFIG proxy.config.http.cache.when_to_revalidate INT 0
    # https://docs.trafficserver.apache.org/records.config#proxy-config-http-cache-required-headers
CONFIG proxy.config.http.cache.required_headers INT 0
##############################################################################
# Heuristic cache expiration. Docs:
#    https://docs.trafficserver.apache.org/records.config#heuristic-expiration
##############################################################################
CONFIG proxy.config.http.cache.heuristic_min_lifetime INT 3600
CONFIG proxy.config.http.cache.heuristic_max_lifetime INT 86400
CONFIG proxy.config.http.cache.heuristic_lm_factor FLOAT 0.100000
##############################################################################
# Network. Docs:
#    https://docs.trafficserver.apache.org/records.config#network
##############################################################################
CONFIG proxy.config.net.connections_throttle INT 80000
CONFIG proxy.config.net.max_connections_in INT 80000
CONFIG proxy.config.net.max_connections_active_in INT 30000
##############################################################################
# RAM and disk cache configurations. Docs:
#    https://docs.trafficserver.apache.org/records.config#ram-cache
#    https://docs.trafficserver.apache.org/en/latest/admin-guide/files/storage.config.en.html
##############################################################################
CONFIG proxy.config.cache.ram_cache.size INT 103079215104
CONFIG proxy.config.cache.ram_cache_cutoff INT 4194304
    # https://docs.trafficserver.apache.org/records.config#proxy-config-cache-limits-http-max-alts
CONFIG proxy.config.cache.limits.http.max_alts INT 5
    # https://docs.trafficserver.apache.org/records.config#proxy-config-cache-max-doc-size
CONFIG proxy.config.cache.max_doc_size INT 0
CONFIG proxy.config.cache.min_average_object_size INT 8000
##############################################################################
# Logging Config. Docs:
#    https://docs.trafficserver.apache.org/records.config#logging-configuration
#    https://docs.trafficserver.apache.org/en/latest/admin-guide/files/logging.config.en.html
##############################################################################
CONFIG proxy.config.log.logging_enabled INT 1
CONFIG proxy.config.log.max_space_mb_for_logs INT 5000
CONFIG proxy.config.log.max_space_mb_headroom INT 1000
CONFIG proxy.config.log.rolling_enabled INT 1
CONFIG proxy.config.log.rolling_interval_sec INT 86400
CONFIG proxy.config.log.rolling_size_mb INT 100
CONFIG proxy.config.log.auto_delete_rolled_files INT 1
CONFIG proxy.config.log.periodic_tasks_interval INT 5
##############################################################################
# These settings control remapping, and if the proxy allows (open) forward proxy or not. Docs:
#    https://docs.trafficserver.apache.org/records.config#url-remap-rules
#    https://docs.trafficserver.apache.org/en/latest/admin-guide/files/remap.config.en.html
##############################################################################
CONFIG proxy.config.url_remap.remap_required INT 1
    # https://docs.trafficserver.apache.org/records.config#proxy-config-url-remap-pristine-host-hdr
CONFIG proxy.config.url_remap.pristine_host_hdr INT 1
    # https://docs.trafficserver.apache.org/records.config#reverse-proxy
CONFIG proxy.config.reverse_proxy.enabled INT 1
##############################################################################
# SSL Termination. Docs:
#    https://docs.trafficserver.apache.org/records.config#client-related-configuration
#    https://docs.trafficserver.apache.org/en/latest/admin-guide/files/ssl_multicert.config.en.html
##############################################################################
CONFIG proxy.config.ssl.client.verify.server INT 0
CONFIG proxy.config.ssl.client.CA.cert.filename STRING NULL
CONFIG proxy.config.ssl.server.cipher_suite STRING ECDHE-ECDSA-AES256-GCM-SHA384:ECDHE-RSA-AES256-GCM-SHA384:ECDHE-ECDSA-AES128-GCM-SHA256:ECDHE-RSA-AES128-GCM-SHA256:DHE-RSA-AES256-GCM-SHA384:DHE-DSS-AES256-GCM-SHA384:DHE-RSA-AES128-GCM-SHA256:DHE-DSS-AES128-GCM-SHA256:ECDHE-ECDSA-AES256-SHA384:ECDHE-RSA-AES256-SHA384:ECDHE-ECDSA-AES256-SHA:ECDHE-RSA-AES256-SHA:ECDHE-ECDSA-AES128-SHA256:ECDHE-RSA-AES128-SHA256:ECDHE-ECDSA-AES128-SHA:ECDHE-RSA-AES128-SHA:DHE-RSA-AES256-SHA256:DHE-DSS-AES256-SHA256:DHE-RSA-AES128-SHA256:DHE-DSS-AES128-SHA256:DHE-RSA-AES256-SHA:DHE-DSS-AES256-SHA:DHE-RSA-AES128-SHA:DHE-DSS-AES128-SHA:AES256-GCM-SHA384:AES128-GCM-SHA256:AES256-SHA256:AES128-SHA256:AES256-SHA:AES128-SHA:DES-CBC3-SHA:!aNULL:!eNULL:!EXPORT:!DES:!RC4:!MD5:!PSK:!aECDH:!EDH-DSS-DES-CBC3-SHA:!EDH-RSA-DES-CBC3-SHA:!KRB5-DES-CBC3-SHA
##############################################################################
# Debugging. Docs:
#    https://docs.trafficserver.apache.org/records.config#diagnostic-logging-configuration
##############################################################################
CONFIG proxy.config.diags.debug.enabled INT 0
CONFIG proxy.config.diags.debug.tags STRING http.*|dns.*
# ToDo: Undocumented
CONFIG proxy.config.dump_mem_info_frequency INT 0
CONFIG proxy.config.http.slow.log.threshold INT 0

CONFIG proxy.config.http_ui_enabled INT 3
CONFIG proxy.config.http.enable_http_info INT 1
CONFIG proxy.config.http.server_max_connections INT 0
CONFIG proxy.config.http.origin_max_connections INT 0
CONFIG proxy.config.http.cache.ignore_accept_mismatch INT 1
CONFIG proxy.config.http.cache.ignore_accept_language_mismatch INT 1
CONFIG proxy.config.http.cache.ignore_accept_encoding_mismatch INT 1
CONFIG proxy.config.http.cache.ignore_accept_charset_mismatch INT 1
CONFIG proxy.config.http.insert_squid_x_forwarded_for INT 1
CONFIG proxy.config.http.anonymize_insert_client_ip INT 1
CONFIG proxy.config.log.logging_enabled INT 1
CONFIG proxy.config.diags.debug.enabled INT 0
CONFIG proxy.config.log.logfile_dir STRING /var/log/trafficserver
EOT

mv remap.config{,.bak}
cat << EOT >> remap.config
map http://aae.cdn1-youku.com/   http://67.229.100.210/

EOT

cat <<EOF>>/etc/rc.local
/bin/trafficserver start
EOF

```

## 4. 设置裸盘

注意：

* 下面的 `dm-2` 需要替换成你对应的设备，可以通过 `fdisk -l` 查看并找到对应的块设备
* 另外，如果该设备或其软连接已经挂载，请修改 `/etc/fstab` 删除对应的行并补充好对应的内容后再 `mount -a` 重新挂载
* 由于使用的是整个磁盘，所以可以不设置大小，直接上

```bash
cat << EOT > /etc/udev/rules.d/99-ats.rules
SUBSYSTEM=="block", KERNEL=="dm-2", MODE="0660", OWNER="ats", GROUP="ats"
EOT

udevadm trigger –subsystem-match=block

cat << EOT >> storage.config
/dev/dm-2
EOT

```

## 5. 重启生效

    init 6

