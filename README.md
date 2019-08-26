# node-local-coredns

node-local-coredns是一个dns caching agent，并以daemonset的方式运行在集群中的节点上。节点上的pod会直接查询node-local-coredns拿到dns解析结果，以此避免iptables DNAT和链路跟踪conntrack，如果cache没有命中，node-local-coredns再进一步查询集群中的kube-dns。

#### 动机

参考：https://kubernetes.io/docs/tasks/administer-cluster/nodelocaldns/#motivation

其中的部分DNS查询延迟的问题可以参考：https://tencentcloudcontainerteam.github.io/tke-handbook/damn/dns-lookup-delay.html

#### 实现细节

设计的目标是不修改节点上的kubelet参数，具体是指--cluster-dns参数，当节点上有或者没有运行dns cache agent都将不会影响节点上的dns查询。

node-local-coredns首先会将kube-dns的clusterip（以172.27.255.78为例）直接绑到网卡cbr0，并在corefile中设置bind 172.27.255.78，这样节点上的dns查询都可以访问到node-local-coredns。但是集群中kube-proxy有为kube-dns设置dnat规则，访问172.27.255.78时将会被nat到kube-dns的一个endpoint对象上，因此，我们需要设置NOTRACK规则，避免conntrack；另外，还需要设置filter表规则避免包被drop，规则设置如下：

    iptables -t raw -A PREROUTING -p tcp -d 172.27.255.78 --dport 53 -j NOTRACK
    iptables -t raw -A PREROUTING -p udp -d 172.27.255.78 --dport 53 -j NOTRACK
    
    iptables -t filter -A INPUT -p tcp -d 172.27.255.78 --dport 53 -j ACCEPT
    iptables -t filter -A INPUT -p udp -d 172.27.255.78 --dport 53 -j ACCEPT
    
    iptables -t raw -A OUTPUT -p tcp -s 172.27.255.78 --sport 53 -j NOTRACK
    iptables -t raw -A OUTPUT -p udp -s 172.27.255.78 --sport 53 -j NOTRACK
    
    iptables -t filter -A OUTPUT -p tcp -s 172.27.255.78 --sport 53 -j ACCEPT
    iptables -t filter -A OUTPUT -p udp -s 172.27.255.78 --sport 53 -j ACCEPT

接下来，node-local-coredns还需要解决cache没有命中的问题。当node-local-coredns中没有查到，需要forward到集群中的kube-dns中进一步查询，因此，我们需要再新建一个service对象，如下：

    apiVersion: v1
    kind: Service
    metadata:
      name: node-local-coredns
      namespace: kube-system
      labels:
        k8s-app: kube-dns
    spec:
      selector:
        k8s-app: kube-dns
      ports:
      - name: dns
        port: 53
        protocol: UDP
      - name: dns-tcp
        port: 53

上面的service与集群中的kube-dns有着相同的endpoint，node-local-coredns将通过它来访问到kube-dns的endpoint。

更多参考：https://github.com/kubernetes/enhancements/blob/master/keps/sig-network/20190424-NodeLocalDNS-beta-proposal.md

