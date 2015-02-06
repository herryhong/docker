**为什么选择ovs：**

需求：

1）当前机器是线上机器，已经完成网络的配置，ip地址已经分配好，很多管理运维系统依赖该ip，所以需要保留该ip，用于机器的正常运维和管理；

2）host当前ip段已经在线上分配，很难有同网段的ip供容器自由使用，容器只能使用新申请的网段，这样的结果就是同一台host上的ips不能保证在同一网段，
至少host ip和容器ip不在同一网段；

基于上述原因，需要使用vlan来实现host ip网段和容器ip网段的互联，而linux
bridge还不支持vlan，因此采用当前使用比较多的openVswitch（简称ovs），支持vlan；

3）由于必须使用vlan，更进一步，我们也就可以灵活配置同一host上容器所在的ip网段，不同容器（同一host上）可以为不同的vlan，支持的vlan个数
，需要和接入交换机配合，太多可能带来网络性能的影响，需要在灵活性和性能做平衡。


**安装ovs:**

优先参考$OVS_HOME/INSTALL.RHEL

[install ovs on centos7.0](https://n40lab.wordpress.com/2015/01/25/centos-7
-installing-openvswitch-2-3-1-lts/) and [install ovs on centos6.5](https://n40la
b.wordpress.com/2014/01/11/centos-6-5-openvswitch-1-9-3-lts-installation/)

****

**配置ovs网桥**：

需求：

host和container 网络都需要走vlan，即下面的方式2）。从网络组得知，一个交换机端口可以配置成三种方式：

1）access模式：该端口不支持vlan tag，只有同网段的数据包才会下放；

2）trunk模式：经过该端口的数据包都需标记vlan tag，一个vlan tag对应一个网段，可以配置哪些vlan tag可以下放；

3）混合模式：可以配置某一网段（比如host所在网段）为access，其余部分为trunk模式。

第3）种方式比较复杂，网络组建议线上环境不要使用，测试时，为了方便可以使用，比如当前host所在网段配置成access，而host上的容器所在的不同网段配置成trunk。线上建议使用trunk模式。

配置：

具体参考$OVS_HOME/rhel/README.RHEL

1）配置eth0（物理网卡对应的网络接口）：把eth0加入ovs网桥，这儿网桥的名字叫ovs0，注意：当eth0加入网桥ovs0后，分配给它的ip将失效，需
要在加入ovs0前，把其ip迁移到其它的网络接口（参考[http://openvswitch.org/support/config-cookbooks
/vlan-configuration-cookbook/](http://openvswitch.org/support/config-cookbooks
/vlan-configuration-cookbook/)），否则将无法通过该ip链接

cat /etc/sysconfig/network-scripts/ifcfg-eth0

>

> DEVICE=eth0

>

> ONBOOT=yes

>

> HWADDR=C8:1F:66:DA:48:11

>

> DEVICETYPE=ovs

>

> TYPE=OVSPort

>

> OVS_BRIDGE=ovs0

>

> BOOTPROTO=none

>

> HOTPLUG=no

>

>

2）配置ovs网桥ovs0：

配置ovs0网桥时，ovs会自动生成相同名字的网桥端口（port）、和网络接口（interface）：

> #ovs-vsctl show

>

> Bridge "ovs0"

> 

> Port "ovs0"

>

> Interface "ovs0"

>

> type: internal

>

> ovs_version: "2.3.1"

a）当上联交换机的端口配置成混合模式时，可以把eth0原来的ip（原host
ip）配置在接口ovs0上，此时端口ovs0还是工作在access模式下，只有容器对应的端口工作在trunk（vlan）下。

![Image](http://git.intra.weibo.com/uploads/platform/docker-service/4a5afa9f2f/Image.png)

 cat /etc/sysconfig/network-scripts/ifcfg-ovs0

> DEVICE=ovs0

>

> ONBOOT=yes

>

> DEVICETYPE=ovs

>

> TYPE=OVSBridge

>

> BOOTPROTO=static

>

> IPADDR=10.77.109.105

>

> NETMASK=255.255.255.0

>

> GATEWAY=10.77.109.1

>

> HOTPLUG=no

>

>

b）当上联交换机的端口配置成trunk模式时，目前了解到的是，无法给端口ovs0标记vlan tag，所以如果直接把原host
ip分配给接口ovs0，会导致网络不通。我还测试过，即使手动的把端口ovs0标记vlan tag：

ovs-vsctl set port ovs0 tag=349 -- 交换机为10.77.109.1/24网段标记的vlan tag为349

网络还是不通。所以我理解自建的ovs端口不支持标记vlan tag（求验证或被挑战）。因此在上联交换机端口工作在trunk模式下，为了保证原host
ip可达，ovs0接口也不配置ip，通过新建接口vlan349，把原ip赋予给它，同时对其端口设置vlan tag；此时网络拓扑看起来是这样子的：

![Image](http://git.intra.weibo.com/uploads/platform/docker-service/3ae05d51fd/Image.png)

cat /etc/sysconfig/network-scripts/ifcfg-ovs0

> DEVICE=ovs0

>

> ONBOOT=yes

>

> DEVICETYPE=ovs

>

> TYPE=OVSBridge

>

> BOOTPROTO=none

>

> HOTPLUG=no


并且需要配置vlan349接口：原host ip赋予接口vlan349，同时该接口加入网桥ovs0：

cat /etc/sysconfig/network-scripts/ifcfg-vlan349

> DEVICE=vlan349

> ONBOOT=yes

>

> DEVICETYPE=ovs

>

> TYPE=OVSIntPort

>

> BOOTPROTO=static

>

> IPADDR=10.77.109.105

>

> NETMASK=255.255.255.0

>

> GATEWAY=10.77.109.1

>

> OVS_BRIDGE=ovs0

>

> OVS_OPTIONS="tag=349"

>

> OVS_EXTRA="set Interface $DEVICE external-ids:iface-id=$(hostname -s
)-$DEVICE-vif"

>

> HOTPLUG=no

>

3）重启网络服务： service network restart


4）配置交换机端口为trunk模式，设置vlan tags，使得下放host ip所在网段、以及host上容器所有可能的网段数据包；


5）验证：能ping通host ip，同时在host上也能ping通网关，至此，ovs网桥结构如下：

> #ovs-vsctl show

>

> Bridge "ovs0"

>

> Port "vlan349"

>

> tag: 349

>

> Interface "vlan349"

>

> type: internal

>

> Port "eth0"

>

> Interface "eth0"

>

> Port "ovs0"

>

> Interface "ovs0"

>

> type: internal

>

> ovs_version: "2.3.1"


注意：

通过ovs-vsctl命令方式添加端口（port）和接口（interface）时，当网络服务重启（service network
restart）或服务器重启时，接口信息将丢失，所以需要采用上面的ifcfg-xxx文件来固化网络设置。


**配置容器网络**：

采用[pipework](https://github.com/jpetazzo/pipework)来配置容器网络，pipework脚本在centos6.5
上不work，主要是找不到ip netns命令，修改了脚本，使用[nsenter](https://github.com/jpetazzo/nsenter)
来替换。centos7上脚本对ovs也不work，主要是判断网桥类型时有问题，把ovs网桥识别为linux网桥，修改下判断顺序就好了。

1）启动docker daemon时，关闭网络设置选项：如网桥、iptable、ip forward：--ip-forward=false--
bridge=none--iptables=false

2）启动容器：由于daemon启动时关闭了网络设置，所以此时启动的容器网络都是空的，只有lo：

> #docker exec -it test3 ifconfig

> lo Link encap:Local Loopback

> inet addr:127.0.0.1 Mask:255.0.0.0

> inet6 addr: ::1/128 Scope:Host

> UP LOOPBACK RUNNING MTU:16436 Metric:1

> RX packets:0 errors:0 dropped:0 overruns:0 frame:0

> TX packets:0 errors:0 dropped:0 overruns:0 carrier:0

> collisions:0 txqueuelen:0

> RX bytes:0 (0.0 b) TX bytes:0 (0.0 b)

3）配置容器网络：

./pipework ovs0 -i eth0 <container_name or cid>
<container_ip>/<subnet>@default_gateway @vlan_id，比如：

> ./pipework ovs0 -i eth0 test310.13.160.10/24@10.13.160.1@860

4）验证：

docker exec -it test3 ifconfig: eth0的ip地址应该为10.13.160.10，

>docker exec -it test3 ip route: eth0的默认网关应该为：default via 10.13.160.1 dev eth0

ovs-vsctl show: 容器网络都是通过veth pair来实现的，一头在容器里面，也就是上面容器里面的eth0，称之为guest接口，一头在hos
t上创建一个网络接口，叫本地接口（如veth0pl2978），本地接口应该在网桥ovs0里面，并且端口对应的port为上面设置的860：

> #ovs-vsctl show

> ac95789e-1918-4557-b238-27ded44675a5

>

> Bridge "ovs0"

>

> Port "eth0"

>

> Interface "eth0"

>

> Port "vlan349"

>

> tag: 349

>

> Interface "vlan349"

>

> type: internal

>

> Port "ovs0"

>

> Interface "ovs0"

>

> type: internal

>

> Port "veth0pl2978"

>

> tag: 860

>

> Interface "veth0pl2978"

>

> ovs_version: "2.3.1"

> docker exec -it test3 ping 10.13.160.1：网络应该都能通。

5）注意事项：

a）host网络重启时，由于veth
pair中的本地接口命令配置的，没有固化下来，可能导致本地接口丢失或者容器网络不可达；host网络重启后，需要重新启动容器并配置网络；

b）容器网络是定制的，并且在容器启动后才初始化，所以原来容器内进程的启动顺序需要控制在网络初始化之后才能开始；

c）当前部分服务器的管理卡和业务网卡是接入同一个交换机接口，通过虚拟技术来实现隔离的，这样需要对管理卡也进行vlan tag改造，网络才能正常。

**References:**

[Isolating VM Traffic Using VLANs](http://openvswitch.org/support/config-
cookbooks/vlan-configuration-cookbook/)


[ovs official site](http://openvswitch.org/)、[configure vlan by
ovs](http://networkstatic.net/configuring-vxlan-and-gre-tunnels-on-
openvswitch/) and [four ways to connect docker
containers](http://blog.oddbit.com/2014/08/11/four-ways-to-connect-a-docker/)


[The fake OVS bridge is a very good solution to allowing VMs interact with
VLANS. Using fake OVS bridges that act and feel just like real bridges is
better solution than using tagged ports (which act just like cisco access
ports). If you use tagged ports, you would have to create one for each VM.
With fake OVS bridge that is tied to a particular VLAN, one fake bridge can be
shared among several VMs.](https://fvtool.wordpress.com/tag/openvswitch/)
