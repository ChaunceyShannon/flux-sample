如果使用ipvs, 需要修改kube-proxy的arp设置

```
kubectl edit configmap -n kube-system kube-proxy
```

设置

```yaml
apiVersion: kubeproxy.config.k8s.io/v1alpha1
kind: KubeProxyConfiguration
mode: "ipvs"
ipvs:
  strictARP: true
```

查看kube-proxy的当前运行模式

```
$ k logs -n kube-system kube-proxy-vw2xf | head
E1217 05:01:02.239470       1 node.go:152] Failed to retrieve node info: Get "https://172.18.0.1:6443/api/v1/nodes/laptop": dial tcp 172.18.0.1:6443: connect: connection refused
I1217 05:01:04.624867       1 node.go:163] Successfully retrieved node IP: 172.18.0.1
I1217 05:01:04.625992       1 server_others.go:138] "Detected node IP" address="172.18.0.1"
I1217 05:01:04.626231       1 server_others.go:561] "Unknown proxy mode, assuming iptables proxy" proxyMode=""
I1217 05:01:04.704161       1 server_others.go:206] "Using iptables Proxier" <====== 这里
I1217 05:01:04.704316       1 server_others.go:213] "kube-proxy running in dual-stack mode" ipFamily=IPv4
I1217 05:01:04.704399       1 server_others.go:214] "Creating dualStackProxier for iptables"
I1217 05:01:04.704723       1 server_others.go:491] "Detect-local-mode set to ClusterCIDR, but no IPv6 cluster CIDR defined, , defaulting to no-op detect-local for IPv6"
I1217 05:01:04.707875       1 server.go:656] "Version info" version="v1.23.1"
I1217 05:01:04.717689       1 conntrack.go:52] "Setting nf_conntrack_max" nf_conntrack_max=393216
```

-----

配置地址池, 直接指定就行, 它会自动选择对应的网卡.

例如我有这么多网络接口

```
1: lo: <LOOPBACK,UP,LOWER_UP> mtu 65536 qdisc noqueue state UNKNOWN group default qlen 1000
    link/loopback 00:00:00:00:00:00 brd 00:00:00:00:00:00
    inet 127.0.0.1/8 scope host lo
       valid_lft forever preferred_lft forever
2: eno2: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc fq_codel state UP group default qlen 1000
    link/ether 0c:9d:92:05:91:db brd ff:ff:ff:ff:ff:ff
    altname enp3s0
    inet 192.168.2.224/24 brd 192.168.2.255 scope global noprefixroute eno2
       valid_lft forever preferred_lft forever
3: wlo1: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc noqueue state UP group default qlen 1000
    link/ether 04:33:c2:bb:e9:b9 brd ff:ff:ff:ff:ff:ff
    altname wlp0s20f3
    inet 192.168.1.152/24 brd 192.168.1.255 scope global dynamic noprefixroute wlo1
       valid_lft 24922sec preferred_lft 24922sec
4: virbr0: <NO-CARRIER,BROADCAST,MULTICAST,UP> mtu 1500 qdisc noqueue state DOWN group default qlen 1000
    link/ether 52:54:00:78:b5:91 brd ff:ff:ff:ff:ff:ff
    inet 192.168.122.1/24 brd 192.168.122.255 scope global virbr0
       valid_lft forever preferred_lft forever
5: virbr0-nic: <BROADCAST,MULTICAST> mtu 1500 qdisc fq_codel master virbr0 state DOWN group default qlen 1000
    link/ether 52:54:00:78:b5:91 brd ff:ff:ff:ff:ff:ff
6: docker0: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc noqueue state UP group default 
    link/ether 02:42:c7:ca:63:6c brd ff:ff:ff:ff:ff:ff
    inet 172.20.0.1/16 brd 172.20.255.255 scope global docker0
       valid_lft forever preferred_lft forever
7: br-6d542897a716: <NO-CARRIER,BROADCAST,MULTICAST,UP> mtu 1500 qdisc noqueue state DOWN group default 
    link/ether 02:42:09:09:74:fd brd ff:ff:ff:ff:ff:ff
    inet 172.18.0.1/16 brd 172.18.255.255 scope global br-6d542897a716
       valid_lft forever preferred_lft forever
10: flannel.1: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1450 qdisc noqueue state UNKNOWN group default 
    link/ether 26:90:bd:11:68:36 brd ff:ff:ff:ff:ff:ff
    inet 10.244.0.0/32 brd 10.244.0.0 scope global flannel.1
       valid_lft forever preferred_lft forever
11: cni0: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1450 qdisc noqueue state UP group default qlen 1000
    link/ether 1e:af:99:de:97:a6 brd ff:ff:ff:ff:ff:ff
    inet 10.244.0.1/24 brd 10.244.0.255 scope global cni0
       valid_lft forever preferred_lft forever
14: vmnet1: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc fq_codel state UNKNOWN group default qlen 1000
    link/ether 00:50:56:c0:00:01 brd ff:ff:ff:ff:ff:ff
    inet 192.168.31.1/24 brd 192.168.31.255 scope global vmnet1
       valid_lft forever preferred_lft forever
15: vmnet8: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc fq_codel state UNKNOWN group default qlen 1000
    link/ether 00:50:56:c0:00:08 brd ff:ff:ff:ff:ff:ff
    inet 192.168.6.1/24 brd 192.168.6.255 scope global vmnet8
       valid_lft forever preferred_lft forever
```

其中`wlo1`是主要用来上网的无线网卡, `eno2`是有线网卡.

我设置地址池在`values.yaml`里面

```yaml
configInline: 
  address-pools:
    - name: default
      protocol: layer2
      addresses:
      - 192.168.1.10-192.168.1.100
```

我开虚拟机桥接到vmnet或者eno2, 并设置1网段的静态IP, 是不能访问LB的IP的, 但是桥接到wlo1可以访问

