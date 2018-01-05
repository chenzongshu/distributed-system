
Corosync为集群心跳控制软件，Pacemaker为资源管理软件，Pcs为控制脚本，以前有用crmsh的，在CentOS下可用pcs

# 1 配置节点名称

在node1和node2上都执行

```
[root@node1 ~]# vim /etc/hosts
127.0.0.1   localhost localhost.localdomain localhost4 localhost4.localdomain4 
::1         localhost localhost.localdomain localhost6 localhost6.localdomain6  
192.168.18.201  node1  
192.168.18.202  node2
```

# 2 配置免密ssh

在node1和node2上都执行,注意配置ntp时间同步,此处略

```
[root@node1 ~]# ssh-keygen  -t rsa -f ~/.ssh/id_rsa  -P ''   
[root@node1 ~]# ssh-copy-id -i .ssh/id_rsa.pub root@node2:/root
```

# 3 安装Corosync+Pacemaker+Pcs

```
yum install pcs pacemaker corosync fence-agents-all
```

# 4 配置corosync,并生成认证文件

在一个节点配置,再拷贝到另外的节点即可

```
[root@node1 corosync]# cat corosync.conf 
# Please read the corosync.conf.5 manual page  
compatibility: whitetank
totem { 
    version: 2  
    secauth: on #启动认证  
    threads: 2  
    interface {  
        ringnumber: 0  
        bindnetaddr: 192.168.18.0 #修改心跳线网段  
        mcastaddr: 226.99.10.1 #组播传播心跳信息  
        mcastport: 5405  
        ttl: 1  
    }  
}
logging { 
    fileline: off  
    to_stderr: no  
    to_logfile: yes  
    to_syslog: no  
    logfile: /var/log/cluster/corosync.log #日志位置  
    debug: off  
    timestamp: on  
    logger_subsys {  
        subsys: AMF  
        debug: off  
    }  
}
amf { 
    mode: disabled  
}
#启用pacemaker
service { 
    ver: 0  
    name: pacemaker  
}
aisexec { 
    user: root  
    group: root  
}
```

> 注意:
1 配置文件中不能出现中文
2 bindnetaddr为双机网段

生成密钥文件

```
[root@node1 corosync]# corosync-keygen  
```

# 5 拷贝并启动

需要在每个节点启动

```
[root@node1 corosync]# scp -p authkey corosync.conf node2:/etc/corosync/

systemctl start corosync
systemctl enable corosync

systemctl start pacemaker
systemctl enable pacemaker

systemctl start pcsd
systemctl enable pcsd
```

# 6 创建一个hacluster用户

两边节点都需要

```
echo redhat | passwd --stdin hacluster
```

# 7 pcs配置互信

两边节点需要配置

```
pcs cluster auth node1 node2 -u hacluster -p redhat --force
```

```
pcs property set pe-warn-series-max=1000 pe-input-series-max=1000 pe-error-series-max=1000 cluster-recheck-interval=5min

pcs property set stonith-enabled=false
```

# 8 启动

```
$ pcs cluster setup --force --name my-cluster node1 node2
$ pcs cluster start --all
$ pcs cluster enable --all
```

# 9 配置VIP

该虚拟ip即为向外界提供使用的ip

```
pcs resource create vip ocf:heartbeat:IPaddr2 params ip="10.xx.xx.145" cidr_netmask="23" op monitor interval="30s"
```

# 10 配置mariadb为高可用资源

因为双机使用的是共同的iscsi存储,所以每次启动之前需要mount一下,iscsi挂载本身复位之后会任然保留

因为iscsi挂载不同主机有可能盘符不同,所以使用ID来创建高可用资源

```
pcs resource create sdisk ocf:heartbeat:Filesystem params device="/dev/disk/by-id/scsi-35009a9dc5730f294-part1" directory="/var/lib/mysql " fstype="xfs" op start timeout=60s op stop timeout=60s op monitor interval=20s timeout=60s
```

```
pcs resource create mysql systemd:mariadb binary="/usr/libexec/mysqld" config="/etc/my.cnf" datadir="/var/lib/mysql" pid="/var/run/mariadb/mariadb.pid" socket="/var/lib/mysql/mysql.sock" op start timeout=180s op stop timeout=180s op monitor interval=20s timeout=60s
```

# 11 增加强制约束

使得mysql和共享存储资源和vip落在同一个主机上

```
pcs constraint colocation add sdisk with vip score=INFINITY
pcs constraint colocation add mysql with vip score=INFINITY
```

可以使用命令查看当前约束:

```
[root@host-206 mysql]# pcs constraint colocation show --full
Colocation Constraints:
  sdisk with vip (score:INFINITY) (id:colocation-sdisk-vip-INFINITY)
  mysql with vip (score:INFINITY) (id:colocation-mysql-vip-INFINITY)
```


# 帮组命令

```
pcs cluster status : 查看集群状态
pcs status : 查看集群状态
pcs resource : 可以看到vip在哪个节点上
```
