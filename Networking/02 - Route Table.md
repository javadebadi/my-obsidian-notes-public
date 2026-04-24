In vm1, I ran the `ip route` **(routing table)** — decides _where_ to send traffic. "Traffic to 192.168.2.0/24 goes directly, everything else goes to the gateway." Without this, the kernel doesn't know where to forward packets.
Without `ip route` → kernel doesn't know _where_ to send traffic

![[networking-ip-route.png]]

Let's investigate the first line:
$\text{\color{red}{default} } \text{\color{black}{via }} {\color{blue}{192.168.2.1}} \text{\color{magenta}{ dev enp0s1 }} \text{\color{green}{ proto  dhcp }} \text{\color{orange}{ src 192.168.2.2 }} \text{\color{brown}{metric 100}}$

- $\text{\color{red}{default} }$ - matches any ==destination==  that doesn't have a more specific route. If the kernel looks at its routing table and finds no specific match, it falls back to this line. That's why it's called the **default gateway**.
- $\text{\color{black}{via }} {\color{blue}{192.168.2.1}}$ - the next hop. When traffic matches this rule, send it to ${\color{blue}{192.168.2.1}}$ first. That machine (the Multipass virtual router in the lab case) then figures out where to send it next. You don't talk directly to `google.com` - you hand the packet to your gateway and it takes it from there.
- $\text{\color{magenta}{ dev enp0s1 }}$ - which network interface to send traffic out of. $\text{\color{magenta}{ enp0s1 }}$ is your VM's ethernet card (virtual in this case). This name is also seeable in `ip addr` results.
- $\text{\color{green}{ proto  dhcp }}$ - who created this route. $\text{\color{green}{dhcp }}$ means DHCP server told your machine about this gateway automatically when it booted. You didn't type it manually
- $\text{\color{orange}{ src 192.168.2.2 }}$ - when sending traffic out this route, use this as your source IP. So packets leaving this machine will show $\text{\color{orange}{192.168.2.2 }}$ as their origin.
- $\text{\color{brown}{metric 100}}$ - priority. If two routes match the same destination, the one with the lower metric wins. 100 is just a default number here.

And the second line is:
192.168.2.0/24 dev enp0s1 proto kernel scope link src 192.168.2.2 metric 100

- 192.168.2.0/24 - matches any ip address from 192.168.2.2 to 192.168.2.254
- proto kernel - this time it wasn't DHCP that created this route. The kernel created it automatically when you were assigned the IP 192.168.2.2/24. The kernel thought: "if I have an IP in this range, I can obviously reach other IPs in the same range directly."
- scope link - this is the key difference from line 1. `link` means "the destination is directly reachable on this network segment, no gateway needed." You don't hand the packet to anyone — you send it straight to the destination. This is why vm1 and vm2 can ping each other directly.
And line 3:
192.168.2.1 dev enp0s1 proto dhcp scope link src 192.168.2.2 metric 100
- makes sure the default gateway is always reachable no matter what
- without 3rd line the first line is useless
- Third rule matches with second line and it is a safety guarantee


## Add new route
> Tell vm1 that to reach the `10.10.10.0/24` network, it should go via the Multipass gateway.

```shell
sudo ip route add 10.10.10.0/24 via 192.168.2.1 dev enp0s1
```
![[networking-ip-route.mp4]]
```shell
sudo ip route del 10.10.10.0/24
```