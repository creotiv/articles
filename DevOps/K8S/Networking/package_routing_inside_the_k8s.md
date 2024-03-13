# Networking & Package routing inside the Kubernetes

## One pod: container to container connection

**Scheme**
```
[Container 1]---\
                 \
[Container 2]-----> [ns_pod] -- [veth_pod]<->[brveth_pod] -- [br0] -- [External Network]
                 /
[Container 3]---/
```

#### Creating Network Namespace
All pods networking running inside network namespaces to isolate them from each other.
```bash
sudo ip netns add ns_pod
```

#### Creating node bridge
Bridge needed to be able containers and pods to connect to outside world
```bash
sudo ip link add name br0 type bridge
sudo ip link set br0 up
```

#### Creating Veth pair for pod
Veth - is basically virtual connection, like virtual cable which connects one thing with another
```bash
sudo ip link add veth_pod type veth peer name brveth_pod
```

#### Connecting one end to the node bridge
```bash
sudo ip link set brveth_pod master br0
sudo ip link set brveth_pod up
```

#### Seting another end into pod namespace
```bash
sudo ip link set veth_pod netns ns_pod
```

#### Setting up IP address for the external pod connections
All containers inside the pod running on localhost and thus can't use same ports.
THey like services in the system.
```bash
sudo ip netns exec ns_pod ip addr add 10.1.1.1/24 dev veth_pod
sudo ip netns exec ns_pod ip link set veth_pod up
sudo ip netns exec ns_pod ip link set lo up
```

#### Turning on IP Forwarding
This needed to make system route packages between ip ranges
```bash
sudo sysctl -w net.ipv4.ip_forward=1
# wlo1 - external interface 
sudo iptables -t nat -A POSTROUTING -o wlo1 -j MASQUERADE
```

#### Chcking ICMP ping
`sudo ip netns exec` - using the same way as `docker exec`, just used to run commands inside network namespace
```bash
sudo ip netns exec ns_pod ping 10.1.1.1
```

#### Checking HTTP
```bash
sudo ip netns exec ns_pod python -m http.server --bind 10.1.1.1 8000
sudo ip netns exec ns_pod curl 10.1.1.1
```


