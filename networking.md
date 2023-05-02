# Basic networking in Linux system 

## Routing 

Gateway: Determine where packet can be sent 

![](./images/Screenshot%202023-05-02%20at%2006.14.16.png)
- Show interface: `ip link`
- Assign IP to the interface: `ip addr add 192.168.1.10/24 dev eth0`
- Display route table: `route`
- Add route (go to 192.168.2.0 via 192.168.1.1)
```bash 
ip route add 192.168.2.0/24 via 192.168.1.1 
```
- In the case of Internet, we can specify with any packet that don't have specific network will go to default route 
```bash
ip route add default via 192.168.1.1
```

Setup Linux host as a router 

![](./images/Screenshot%202023-05-02%20at%2006.19.30.png)

- Add route for packet from host A -> C:
```bash 
ip route add 192.168.2.0/24 via 192.168.1.6
```
- Add route for packet from host C -> A: 
```bash 
ip route add 192.168.1.0/24 via 192.168.2.6
```

- Allow port forward between 2 ports in host B 
```bash 
echo 1 > /proc/sys/net/ipv4/ip_forward 
vi /etc/sysctl.conf 
>> net.ipv4.ip_forward = 1 
```

## DNS 

- With small system, we can use name solution in `/etc/hosts`
- In the large system, we setup 1 server as DNS server (save every record) and point another server to that server. 

![](./images/Screenshot%202023-05-02%20at%2006.31.14.png)
- Add entry to point to dns server 

```bash
vi /etc/resolv.conf

>>> nameserver 192.168.1.100

```
- `/etc/host` > `/etc/resolv.conf`. Can change the order in `/etc/nsswitch.conf`

## Docker network 
- Type of network:
  - None network
  - Host networking
  - Bridge networking (default)