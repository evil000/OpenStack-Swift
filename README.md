# OpenStack-Swift
OpenStack Swift安装

###OpenStack Swift单节点部署
部署环境
VMware Workstation 1 (推荐版本，低版本亦可，只要能安装Ubuntu 12.04)
Ubuntu 12.04 Server 32 bit
OpenStack Swift v1.8 (Grizzly)
说明
此文档为基于官方修改的只针对Ubuntu 12.04，完整的参阅官方文档
本部署为开发环境，Swift配置为以root权限运行，生产环境推荐新建用户swift
选择Ubuntu 12.04因其默认内核为3.5，减少部署时的麻烦
Ubuntu Server安装时不选其它组件，最小化安装
以下安装命令全部以root身份运行
Ubuntu 更改为国内源（非必需）:
sudo vi /etc/apt/sources.list
%s/cn.archive.ubuntu.com/mirrors.ustc.edu.cn/g
sudo apt-get update
sudo apt-get upgrade
安装依赖
apt-get update
apt-get install curl gcc memcached rsync sqlite3 xfsprogs git python-setuptools
apt-get install python-coverage python-dev python-nose python-simplejson python-xattr python-eventlet 
python-greenlet python-pastedeploy python-netifaces python-pip python-dnspython python-mock
使用回环设备作为存储，若需要使用一个分区作为存储，参见官方文档
mkdir /srv
truncate -s 1GB /srv/swift-disk
mkfs.xfs /srv/swift-disk
修改/etc/fstab 添加如下行

/srv/swift-disk /mnt/sdb1 xfs loop,noatime,nodiratime,nobarrier,logbufs=8 0 0

mkdir /mnt/sdb1
mount /mnt/sdb1
mkdir /mnt/sdb1/1 /mnt/sdb1/2 /mnt/sdb1/3 /mnt/sdb1/4
chown root:root /mnt/sdb1/*
for x in {1..4}; do ln -s /mnt/sdb1/$x /srv/$x; done
mkdir -p /etc/swift/object-server /etc/swift/container-server /etc/swift/account-server 
mkdir -p /srv/1/node/sdb1 /srv/2/node/sdb2 /srv/3/node/sdb3 /srv/4/node/sdb4 /var/run/swift
chown -R root:root /etc/swift /srv/[1-4]/ /var/run/swift
编辑文件/etc/rc.local，在exit 0 之前添加如下4行:

mkdir -p /var/cache/swift /var/cache/swift2 /var/cache/swift3 /var/cache/swift4
chown <your-user-name>:<your-group-name> /var/cache/swift*
mkdir -p /var/run/swift
chown <your-user-name>:<your-group-name> /var/run/swift
设置Rsync
创建文件/etc/rsyncd.conf，：

uid = root
gid = root
log file = /var/log/rsyncd.log
pid file = /var/run/rsyncd.pid
address = 127.0.0.1

[account6012]
max connections = 25
path = /srv/1/node/
read only = false
lock file = /var/lock/account6012.lock

[account6022]
max connections = 25
path = /srv/2/node/
read only = false
lock file = /var/lock/account6022.lock

[account6032]
max connections = 25
path = /srv/3/node/
read only = false
lock file = /var/lock/account6032.lock

[account6042]
max connections = 25
path = /srv/4/node/
read only = false
lock file = /var/lock/account6042.lock

[container6011]
max connections = 25
path = /srv/1/node/
read only = false
lock file = /var/lock/container6011.lock

[container6021]
max connections = 25
path = /srv/2/node/
read only = false
lock file = /var/lock/container6021.lock

[container6031]
max connections = 25
path = /srv/3/node/
read only = false
lock file = /var/lock/container6031.lock

[container6041]
max connections = 25
path = /srv/4/node/
read only = false
lock file = /var/lock/container6041.lock

[object6010]
max connections = 25
path = /srv/1/node/
read only = false
lock file = /var/lock/object6010.lock

[object6020]
max connections = 25
path = /srv/2/node/
read only = false
lock file = /var/lock/object6020.lock

[object6030]
max connections = 25
path = /srv/3/node/
read only = false
lock file = /var/lock/object6030.lock

[object6040]
max connections = 25
path = /srv/4/node/
read only = false
lock file = /var/lock/object6040.lock
编辑文件/etc/default/rsync，设置参数RSYNC_ENABLE为true:

RSYNC_ENABLE=true
启动rsync服务

service rsync restart
确认rsync启动成功

rsync rsync://pub@localhost/
设置独立日志（非必需，略，见官方文档）
获取代码并设置测试环境
从git上获取swift代码，下载到本地

git clone https://github.com/openstack/swift.git
安装Swift的开发版本

cd ~/swift
git checkout -t origin/stable/grizzly # checkout稳定分支1.8
python setup.py develop
cd ..
从git上获取python-swiftclient代码，下载到本地

git clone https://github.com/openstack/python-swiftclient.git
安装python-swiftclient的开发版本

cd ~/python-swiftclient
python setup.py develop
cd ..
配置各结点
创建文件/etc/swift/proxy-server.conf，

[DEFAULT]
bind_port = 8080
user = root
log_facility = LOG_LOCAL1
eventlet_debug = true
[pipeline:main]
pipeline = healthcheck cache tempauth proxy-logging proxy-server

[app:proxy-server]
use = egg:swift#proxy
allow_account_management = true
account_autocreate = true

[filter:tempauth]
use = egg:swift#tempauth
user_admin_admin = admin .admin .reseller_admin
user_test_tester = testing .admin
user_test2_tester2 = testing2 .admin
user_test_tester3 = testing3

[filter:healthcheck]
use = egg:swift#healthcheck

[filter:cache]
use = egg:swift#memcache

[filter:proxy-logging]
use = egg:swift#proxy_logging
创建文件/etc/swift/swift.conf，

[swift-hash]
# random unique strings that can never change (DO NOT LOSE)
swift_hash_path_prefix = rui
swift_hash_path_suffix = jie
Create /etc/swift/account-server/1.conf:

[DEFAULT]
devices = /srv/1/node
mount_check = false
disable_fallocate = true
bind_port = 6012
user = root
log_facility = LOG_LOCAL2
recon_cache_path = /var/cache/swift
eventlet_debug = true

[pipeline:main]
pipeline = recon account-server

[app:account-server]
use = egg:swift#account

[filter:recon]
use = egg:swift#recon

[account-replicator]
vm_test_mode = yes

[account-auditor]

[account-reaper]
Create /etc/swift/account-server/2.conf:

[DEFAULT]
devices = /srv/2/node
mount_check = false
disable_fallocate = true
bind_port = 6022
user = root
log_facility = LOG_LOCAL3
recon_cache_path = /var/cache/swift2
eventlet_debug = true

[pipeline:main]
pipeline = recon account-server

[app:account-server]
use = egg:swift#account

[filter:recon]
use = egg:swift#recon

[account-replicator]
vm_test_mode = yes

[account-auditor]

[account-reaper]
Create /etc/swift/account-server/3.conf:

[DEFAULT]
devices = /srv/3/node
mount_check = false
disable_fallocate = true
bind_port = 6032
user = root
log_facility = LOG_LOCAL4
recon_cache_path = /var/cache/swift3
eventlet_debug = true

[pipeline:main]
pipeline = recon account-server

[app:account-server]
use = egg:swift#account

[filter:recon]
use = egg:swift#recon

[account-replicator]
vm_test_mode = yes

[account-auditor]

[account-reaper]
Create /etc/swift/account-server/4.conf:

[DEFAULT]
devices = /srv/4/node
mount_check = false
disable_fallocate = true
bind_port = 6042
user = root
log_facility = LOG_LOCAL5
recon_cache_path = /var/cache/swift4
eventlet_debug = true

[pipeline:main]
pipeline = recon account-server

[app:account-server]
use = egg:swift#account

[filter:recon]
use = egg:swift#recon

[account-replicator]
vm_test_mode = yes

[account-auditor]

[account-reaper]
Create /etc/swift/container-server/1.conf:

[DEFAULT]
devices = /srv/1/node
mount_check = false
disable_fallocate = true
bind_port = 6011
user = root
log_facility = LOG_LOCAL2
recon_cache_path = /var/cache/swift
eventlet_debug = true

[pipeline:main]
pipeline = recon container-server

[app:container-server]
use = egg:swift#container

[filter:recon]
use = egg:swift#recon

[container-replicator]
vm_test_mode = yes

[container-updater]

[container-auditor]

[container-sync]
Create /etc/swift/container-server/2.conf:

[DEFAULT]
devices = /srv/2/node
mount_check = false
disable_fallocate = true
bind_port = 6021
user = root
log_facility = LOG_LOCAL3
recon_cache_path = /var/cache/swift2
eventlet_debug = true

[pipeline:main]
pipeline = recon container-server

[app:container-server]
use = egg:swift#container

[filter:recon]
use = egg:swift#recon

[container-replicator]
vm_test_mode = yes

[container-updater]

[container-auditor]

[container-sync]
Create /etc/swift/container-server/3.conf:

[DEFAULT]
devices = /srv/3/node
mount_check = false
disable_fallocate = true
bind_port = 6031
user = root
log_facility = LOG_LOCAL4
recon_cache_path = /var/cache/swift3
eventlet_debug = true

[pipeline:main]
pipeline = recon container-server

[app:container-server]
use = egg:swift#container

[filter:recon]
use = egg:swift#recon

[container-replicator]
vm_test_mode = yes

[container-updater]

[container-auditor]

[container-sync]
Create /etc/swift/container-server/4.conf:

[DEFAULT]
devices = /srv/4/node
mount_check = false
disable_fallocate = true
bind_port = 6041
user = root
log_facility = LOG_LOCAL5
recon_cache_path = /var/cache/swift4
eventlet_debug = true

[pipeline:main]
pipeline = recon container-server

[app:container-server]
use = egg:swift#container

[filter:recon]
use = egg:swift#recon

[container-replicator]
vm_test_mode = yes

[container-updater]

[container-auditor]

[container-sync]
Create /etc/swift/object-server/1.conf:

[DEFAULT]
devices = /srv/1/node
mount_check = false
disable_fallocate = true
bind_port = 6010
user = root
log_facility = LOG_LOCAL2
recon_cache_path = /var/cache/swift
eventlet_debug = true

[pipeline:main]
pipeline = recon object-server

[app:object-server]
use = egg:swift#object

[filter:recon]
use = egg:swift#recon

[object-replicator]
vm_test_mode = yes

[object-updater]

[object-auditor]
Create /etc/swift/object-server/2.conf:

[DEFAULT]
devices = /srv/2/node
mount_check = false
disable_fallocate = true
bind_port = 6020
user = root
log_facility = LOG_LOCAL3
recon_cache_path = /var/cache/swift2
eventlet_debug = true

[pipeline:main]
pipeline = recon object-server

[app:object-server]
use = egg:swift#object

[filter:recon]
use = egg:swift#recon

[object-replicator]
vm_test_mode = yes

[object-updater]

[object-auditor]
Create /etc/swift/object-server/3.conf:

[DEFAULT]
devices = /srv/3/node
mount_check = false
disable_fallocate = true
bind_port = 6030
user = root
log_facility = LOG_LOCAL4
recon_cache_path = /var/cache/swift3
eventlet_debug = true

[pipeline:main]
pipeline = recon object-server

[app:object-server]
use = egg:swift#object

[filter:recon]
use = egg:swift#recon

[object-replicator]
vm_test_mode = yes

[object-updater]

[object-auditor]
Create /etc/swift/object-server/4.conf:

[DEFAULT]
devices = /srv/4/node
mount_check = false
disable_fallocate = true
bind_port = 6040
user = root
log_facility = LOG_LOCAL5
recon_cache_path = /var/cache/swift4
eventlet_debug = true

[pipeline:main]
pipeline = recon object-server

[app:object-server]
use = egg:swift#object

[filter:recon]
use = egg:swift#recon

[object-replicator]
vm_test_mode = yes

[object-updater]

[object-auditor]
创建Swift运行脚本
mkdir ~/bin

创建脚本~/bin/resetswift

因未创建rsyslog作为独立日志，已移除find /var/log/swift…这一行
如果使用的是单独分区存储需要将/srv/swift-disk替换为/dev/sdb1

#!/bin/bash
swift-init all stop
sudo umount /mnt/sdb1
sudo mkfs.xfs -f /srv/swift-disk
sudo mount /mnt/sdb1
sudo mkdir /mnt/sdb1/1 /mnt/sdb1/2 /mnt/sdb1/3 /mnt/sdb1/4
sudo chown root:root /mnt/sdb1/*
mkdir -p /srv/1/node/sdb1 /srv/2/node/sdb2 /srv/3/node/sdb3 /srv/4/node/sdb4
sudo rm -f /var/log/debug /var/log/messages /var/log/rsyncd.log /var/log/syslog
find /var/cache/swift* -type f -name *.recon -exec rm -f {} \;
sudo service rsyslog restart
sudo service memcached restart
创建脚本~/bin/remakerings

#!/bin/bash
cd /etc/swift
rm -f *.builder *.ring.gz backups/*.builder backups/*.ring.gz
swift-ring-builder object.builder create 10 3 1
swift-ring-builder object.builder add r1z1-127.0.0.1:6010/sdb1 1
swift-ring-builder object.builder add r1z2-127.0.0.1:6020/sdb2 1
swift-ring-builder object.builder add r1z3-127.0.0.1:6030/sdb3 1
swift-ring-builder object.builder add r1z4-127.0.0.1:6040/sdb4 1
swift-ring-builder object.builder rebalance
swift-ring-builder container.builder create 10 3 1
swift-ring-builder container.builder add r1z1-127.0.0.1:6011/sdb1 1
swift-ring-builder container.builder add r1z2-127.0.0.1:6021/sdb2 1
swift-ring-builder container.builder add r1z3-127.0.0.1:6031/sdb3 1
swift-ring-builder container.builder add r1z4-127.0.0.1:6041/sdb4 1
swift-ring-builder container.builder rebalance
swift-ring-builder account.builder create 10 3 1
swift-ring-builder account.builder add r1z1-127.0.0.1:6012/sdb1 1
swift-ring-builder account.builder add r1z2-127.0.0.1:6022/sdb2 1
swift-ring-builder account.builder add r1z3-127.0.0.1:6032/sdb3 1
swift-ring-builder account.builder add r1z4-127.0.0.1:6042/sdb4 1
swift-ring-builder account.builder rebalance
创建~/bin/startmain

#!/bin/bash
swift-init main start
创建~/bin/startrest

#!/bin/bash
swift-init rest start
chmod +x ~/bin/*

编辑~/.bashrc，在最后添加如下两行

export SWIFT_TEST_CONFIG_FILE=/etc/swift/test.conf
export PATH=${PATH}:~/bin

<!-- lang: shell -->
. ~/.bashrc
remakerings
cp ~/swift/test/sample.conf /etc/swift/test.conf
开启Swift

startmain
检查Swift工作

swift -A http://127.0.0.1:8080/auth/v1.0 -U test:tester -K testing stat
正确情况下，应该输出以下信息

Account: AUTH_test
Containers: 0
Objects: 0
Bytes: 0
Accept-Ranges: bytes
重启main服务

swift-init main restart
swift-init

swift-init命令有许多参数，具体查看帮助
