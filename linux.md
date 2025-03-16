# linux

## networking

- `ip link` for see the interfaces for host
- `ip addr add 192.168.1.10/24 dev eth0` add ip to an interface
- `ip route` to see route list
- `ip route add 192.168.2.0/24 via 192.168.1.1` to add route
- `ip route add default via 102.168.2.1` add a default gateway
- `nslookup` and `dig` skips `/etc/hosts` file

- to see if a host forward network packages between interfaces check `/prod/sys/net/ipv4/ip_forward`
- to persist the changes for ip forward, add `net.ipv4.ip_forward = 1` to `/etc/sysctl.conf`

### dns
- basic is adding alias `/etc/hosts`. This will not check dns
- to point a host to dns add `nameserver 192.168.1.00` to `/etc/resolv.conf`
- you can change the order of **hosts** file or **dns** in the `/etc/nsswitch.conf`
- to search a domain name add `search example.com prod.example.com` to `/etc/resolv.conf`, this will add **example.com** when you ping **web**

#### record types
- **a** record stores ipv4
- **aaaa** record stores ipv6
- **cname** record stores mapping one name to another

### network namespaces
- to create network namespace, run `ip netns add red`
- to list network namespaces, run `ip netns`
- to list interfaces in namespaces, run `ip netns exec red ip link` or `ip -n red link`
- to connect two namespaces, create virtual interface `ip link add veth-red type veth peer name veth-blue`
  - to attach virtual interface, it `ip link set veth-red netns red` and `ip link set veth-blue netns blue`
  - to assign ip address, `ip -n red addr add 192.168.15.1 dev veth-red` and `ip -n blue addr add 192.168.15.2 dev veth-blue`
  - to bring up to interfaces, `ip -n red link set veth-red up` and `ip -n blue link set veth-blue up`
  - to delete this connection, `ip -n red link del veth-red` this will delete both.
- to connect more than two you need a virtual switch
  - add new interface to host via `ip link add v-net-o type bridge` and `ip link set dev v-net-0 up` to bring it up. it is interface to **host** and **switch** to **namespaces**
  - create new interface to link namespace to switch `ip link add veth-red type veth peer name veth-red-br` and `ip link add veth-blue type veth peer name veth-blue-br`
  - to attach it to a namespace `ip link set veth-red netns red`
  - to attach it to a bridge `ip link set veth-red-br master v-net-0`
  - then assing ip address to namespace interfaces and bring them up
- to reach bridge from the host, attach an ip address `ip addr add 192.168.15.5/24 dev v-net-0`
- to reach outsite network from the namespaces
  - first we need to add a route `ip netns exec blue ip route add 192.168.1.0/24 via 192.168.15.5`
  - then we need to enable **nat** on the host with **iptables**
    - `iptables -t nat -A POSTROUTING -s 192.168.15.0/24 -j MASQUERADE`
- to reach internet from the namespaces
  - add a default route `ip netns exec blue ip route add default via 192.168.15.5`
- to reach to namespaces from outside network we need to add port forwarding to the host
  - `iptables -t nat -A PREROUTING --dport 80 --to-destination 192.168.15.2:80 -j DNAT`