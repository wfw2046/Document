#  一、安装openshift
host文件如下：
``` 
# Create an OSEv3 group that contains the masters and nodes groups
[OSEv3:children]
masters
nodes

# Set variables common for all OSEv3 hosts
[OSEv3:vars]
# SSH user, this user should allow ssh based auth without requiring a password
ansible_ssh_user=root
openshift_release=v1.3.2

# If ansible_ssh_user is not root, ansible_become must be set to true
#ansible_become=true

deployment_type=origin

# uncomment the following to enable htpasswd authentication; defaults to DenyAllPasswordIdentityProvider
#openshift_master_identity_providers=[{'name': 'htpasswd_auth', 'login': 'true', 'challenge': 'true', 'kind': 'HTPasswdPasswordIdentityProvider', 'filename': '/etc/origin/master/htpasswd'}]

# host group for masters
[masters]
192.168.10.145 containerized=true openshift_ip=192.168.10.145

# host group for nodes, includes region info
[nodes]
192.168.10.145 openshift_node_labels="{'region': 'infra', 'zone': 'default'}" containerized=true  openshift_use_openshift_sdn=False
192.168.10.146 openshift_node_labels="{'region': 'infra', 'zone': 'east'}" containerized=true  openshift_use_openshift_sdn=False
192.168.10.147 openshift_node_labels="{'region': 'primary', 'zone': 'west'}" containerized=true  openshift_use_openshift_sdn=False
```
==openshift_use_openshift_sdn=False==：表示不启用openshift自带SDN插件。

```
[root@node-r1 ~] git clone https://github.com/openshift/openshift-ansible.git
[root@node-r1 ~] ansible-playbook -i hosts openshift-ansible/playbooks/byo/config.yml
```

#  二、安装calico。
##  2、安装etcd

```
[root@etcd ~]# ETCD_VER=v3.1.0-rc.1
[root@etcd ~]# DOWNLOAD_URL=https://github.com/coreos/etcd/releases/download
[root@etcd ~]# curl -L ${DOWNLOAD_URL}/${ETCD_VER}/etcd-${ETCD_VER}-linux-amd64.tar.gz -o /tmp/etcd-${ETCD_VER}-linux-amd64.tar.gz
[root@etcd ~]# mkdir -p /tmp/test-etcd && tar xzvf /tmp/etcd-${ETCD_VER}-linux-amd64.tar.gz -C /tmp/test-etcd --strip-components=1
[root@etcd ~]# /tmp/test-etcd/etcd --listen-peer-urls 'http://192.168.1.110:2380' --listen-client-urls 'http://192.168.1.110:2379' --advertise-client-urls 'http://192.168.1.110:2379' &
```
##  3、添加calico-node系统服务。

```
[root@node-r1 ~]# vim /etc/systemd/system/calico-node.service
    [Unit]
    Description=calico node
    After=docker.service
    Requires=docker.service
    
    [Service]
    User=root
    Environment=ETCD_ENDPOINTS=http://192.168.1.110：2379
    PermissionsStartOnly=true
    ExecStart=/usr/bin/docker run --net=host --privileged --name=calico-node \
      -e ETCD_ENDPOINTS=${ETCD_ENDPOINTS} \
      -e NODENAME=${HOSTNAME} \
      -e IP= \
      -e NO_DEFAULT_POOLS= \
      -e AS= \
      -e CALICO_LIBNETWORK_ENABLED=true \
      -e IP6= \
      -e CALICO_NETWORKING_BACKEND=bird \
      -v /var/run/calico:/var/run/calico \
      -v /lib/modules:/lib/modules \
      -v /run/docker/plugins:/run/docker/plugins \
      -v /var/run/docker.sock:/var/run/docker.sock \
      -v /var/log/calico:/var/log/calico \
      calico/node:v1.0.0
    ExecStop=/usr/bin/docker rm -f calico-node
    Restart=always
    RestartSec=10
    
    [Install]
    WantedBy=multi-user.target
```
启动服务

```
systemctl enable  calico-node.service 
service calico-node.service start
```

##  4、下载calico插件
```
wget -N -P /opt/cni/bin https://github.com/projectcalico/calico-cni/releases/download/v1.5.5/calico
wget -N -P /opt/cni/bin https://github.com/projectcalico/calico-cni/releases/download/v1.5.5/calico-ipam
chmod +x /opt/cni/bin/calico /opt/cni/bin/calico-ipam
```
##  5、创建openshift插件文件

```
[root@node-r2 ~]# mkdir -p /usr/libexec/kubernetes/kubelet-plugins/net/exec
[root@node-r2 ~]# cat >/usr/libexec/kubernetes/kubelet-plugins/net/exec/calico.conf <<EOF
{
    "name": "calico-k8s-network",
    "type": "calico",
    "etcd_endpoints": "http://192.168.1.110:2379",
    "log_level": "info",
    "ipam": {
        "type": "calico-ipam"
    },
    "policy": {
        "type": "k8s"
    },
    "kubernetes": {
        "kubeconfig": "/etc/origin/node/system:node:192.168.10.146.kubeconfig"
    }
}
EOF
```
下载lookback

```
[root@node-r2 ~]# wget https://github.com/containernetworking/cni/releases/download/v0.3.0/cni-v0.3.0.tgz
[root@node-r2 ~]# tar -zxvf cni-v0.3.0.tgz
sudo cp loopback /opt/cni/bin/
```

##  6、配置calico的profile。


```
[root@node-r1 ~]# cat profile.yaml
apiVersion: v1
kind: profile
metadata:
  name: k8s_ns.default
  labels:
    profile: profile1 
spec:
  ingress:
  - action: allow
    source:
      net: 172.18.1.0/24
  egress:
  - action: allow 
[root@node-r1 ~]# ETCD_ENDPOINTS=http://192.168.1.110:2379 /root/calicoctl  create -f profile.yaml
```

#  三、openshift配置
##  1、node节点配置

```
[root@node-r2 ~]# cat  /etc/origin/node/node-config.yaml 
allowDisabledDocker: false
apiVersion: v1
dnsDomain: cluster.local
dnsIP: 192.168.10.146
dockerConfig:
  execHandlerName: ""
iptablesSyncPeriod: "30s"
imageConfig:
  format: openshift/origin-${component}:${version}
  latest: false
kind: NodeConfig
kubeletArguments: 
  node-labels:
  - region=infra
  - zone=east
masterClientConnectionOverrides:
  acceptContentTypes: application/vnd.kubernetes.protobuf,application/json
  contentType: application/vnd.kubernetes.protobuf
  burst: 200
  qps: 100
masterKubeConfig: system:node:192.168.10.146.kubeconfig
# networkConfig struct introduced in origin 1.0.6 and OSE 3.0.2 which
# deprecates networkPluginName above. The two should match.
networkConfig:
   mtu: 1450
   networkPluginName: cni
nodeName: 192.168.10.146
podManifestConfig:
servingInfo:
  bindAddress: 0.0.0.0:10250
  certFile: server.crt
  clientCA: ca.crt
  keyFile: server.key
volumeDirectory: /var/lib/origin/openshift.local.volumes
proxyArguments:
  proxy-mode:
     - iptables
volumeConfig:
  localQuota:
    perFSGroup:
```
其中networkPluginName设置为==cni==，代表使用cni网络插件作为openshift网络插件。

##  2、更改origin-node服务系统启动

```
[root@node-r2 ~]# cat /etc/systemd/system/origin-node.service 
[Unit]
After=origin-master.service
After=docker.service
After=openvswitch.service
PartOf=docker.service
Requires=docker.service
Wants=origin-master.service
Requires=origin-node-dep.service
After=origin-node-dep.service

[Service]
EnvironmentFile=/etc/sysconfig/origin-node
EnvironmentFile=/etc/sysconfig/origin-node-dep
ExecStartPre=-/usr/bin/docker rm -f origin-node
ExecStart=/usr/bin/docker run --name origin-node --rm --privileged --net=host --pid=host --env-file=/etc/sysconfig/origin-node -v /:/rootfs:ro -e CONFIG_FILE=${CONFIG_FILE} -e OPTIONS=${OPTIONS} -e HOST=/rootfs -e HOST_ETC=/host-etc -v /var/lib/origin:/var/lib/origin:rslave -v /etc/origin/node:/etc/origin/node -v /etc/localtime:/etc/localtime:ro -v /etc/machine-id:/etc/machine-id:ro -v /run:/run -v /sys:/sys:rw -v /usr/bin/docker:/usr/bin/docker:ro -v /var/lib/docker:/var/lib/docker -v /lib/modules:/lib/modules -v /etc/origin/openvswitch:/etc/openvswitch -v /etc/origin/sdn:/etc/openshift-sdn -v /etc/systemd/system:/host-etc/systemd/system -v /var/log:/var/log -v /dev:/dev -v /opt/cni/bin:/opt/cni/bin -v /usr/libexec:/usr/libexec  $DOCKER_ADDTL_BIND_MOUNTS openshift/node:${IMAGE_VERSION}
ExecStartPost=/usr/bin/sleep 10
ExecStop=/usr/bin/docker stop origin-node
SyslogIdentifier=origin-node
Restart=always
RestartSec=5s

[Install]
WantedBy=docker.service
```
主要增加以下配置：对应的是calico的路径

```
-v /opt/cni/bin:/opt/cni/bin -v /usr/libexec:/usr/libexec
```




