![[Pasted image 20260403005912.png]]

### 1 — IPv4 address structure

32 bits total

Every IPv4 address is 32 binary digits. Written as 4 octets (groups of 8 bits) in decimal, e.g. `10.0.0.0`.

**Total possible addresses**

232 = 4,294,967,296 (~4.3 billion) unique addresses in all of IPv4.

### 2 — Prefix length (the /n)

#### Network bits vs host bits

The `/n` splits 32 bits into two parts: the first `n` bits are fixed (identify the network), the remaining `32−n` bits are free (identify individual hosts).

#### Subnet

Any block defined by a `/n` prefix. All addresses in a subnet share the same network bits. Larger `n` = smaller subnet (more bits locked, fewer free).

**Why 32 bits and not just host bits?**

Network bits let routers distinguish between millions of different networks. Host bits identify devices within one network. Together they enable the internet — a network of networks.

### 3 — Reserved addresses

**Network address (host bits all 0)**

Names the subnet itself. Not assignable to any device. e.g. `10.0.0.0` in a `/8`.

**Broadcast address (host bits all 1)**

Delivers a packet to every device in the subnet. Not assignable either. e.g. `10.255.255.255` in a `/8`.

#### 4 — The formula

Usable addresses = $2^{32 − n} − 2$

| Subnet | Host bits | Total      | Usable     |
| ------ | --------- | ---------- | ---------- |
| `/8`   | 24        | 16,777,216 | 16,777,214 |
| `/16`  | 16        | 65,536     | 65,534     |
| `/24`  | 8         | 256        | 254        |
|        |           |            |            |

## Subnet Mask

### 1 — Subnet mask

**What it is**

32 bits where the first n bits are `1` (network) and the rest are `0` (host). Same information as the prefix length, different notation.

**What it does**

When ANDed with any IP address, it zeroes out the host bits and returns the network address — letting a device check if two IPs are on the same network.

**Where it lives**

Stored locally on each device and router. It never travels inside a packet.

`/24 → 11111111.11111111.11111111.00000000 → 255.255.255.0`  
`/20 → 11111111.11111111.11110000.00000000 → 255.255.240.0`

### 2 — How your computer reaches google.com

Your computer's local config (assigned by router via DHCP)

Your IP: 192.168.1.100  
Subnet mask: 255.255.255.0 (/24)  
Default gateway: 192.168.1.1 (your home router)

1- Browser resolves `google.com` to an IP via DNS

→ 142.250.80.46

2- Your computer ANDs both IPs with its subnet mask to compare networks

192.168.1.100 AND 255.255.255.0 = 192.168.1.0 (your network)  
142.250.80.46 AND 255.255.255.0 = 142.250.80.0 (Google's network)

Networks don't match → Google is not local

3- Your computer sends the packet to the default gateway

→ 192.168.1.1 (your home router)

4- Your router forwards it to your ISP. Each router along the way checks its own routing table and forwards toward Google — using its own locally stored subnet masks.

5- Packet arrives at Google's network. Their router delivers it to the correct server.

→ 142.250.80.46 responds back to 192.168.1.100

The subnet mask never travels in the packet. Each device only uses it locally to make one decision — local or not local — then hands the packet off.


## Subnetting
1 — What subnetting is

**Definition**: Splitting one larger subnet into multiple smaller ones by borrowing bits from the host side and adding them to the network side.

**The tradeoff**
Every bit borrowed doubles the number of subnets but halves the addresses in each one.

2 — The mental model

**Block size**

Each subnet occupies a fixed block of addresses. Block size tells you where each subnet starts.

```
block size = 2^(host bits)  
num subnets = 256 / block size  
usable = block size − 2

```

Shortcut from subnet mask

Block size = 256 − last non-255 octet of the mask.

/26 → mask 255.255.255.192 → 256 − 192 = 64 (block size)  
/27 → mask 255.255.255.224 → 256 − 224 = 32 (block size)

3 — Worked example: 192.168.1.0/24 → 4 subnets

1. Need 4 subnets → borrow 2 bits (2² = 4) → new prefix /26

2. Block size = 2^6 = 64. Subnets start at multiples of 64.

3. The 4 subnets:

|Subnet|Range|Usable|Bits|
|---|---|---|---|
|192.168.1.0/26|.1 → .62|62|00|
|192.168.1.64/26|.65 → .126|62|01|
|192.168.1.128/26|.129 → .190|62|10|
|192.168.1.192/26|.193 → .254|62|11|

4. — Reference table

|Prefix|Bits borrowed|Subnets|Block size|Usable each|
|---|---|---|---|---|
|/24|0|1|256|254|
|/25|1|2|128|126|
|/26|2|4|64|62|
|/27|3|8|32|30|
|/28|4|16|16|14|
|/29|5|32|8|6|
|/30|6|64|4|2|
