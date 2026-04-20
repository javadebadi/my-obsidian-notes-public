
To learn networking, I wanted to have at least two VMs and experiment with configuring networking. I asked Claude about what solution I have to emulate that and it suggested to install two VMs in my system.

My computer is running a MacOS and to have two VMs for this goal I had to install `multipass`:
```shell
brew install multipass # or snap install multipass on Linux
```

Then I was able to spin up two VMs easily:
```shell
# Spin up two VMs
multipass launch --name vm1
multipass launch --name vm2
```

![[networking-setup-lab-multipass-vms.png]]

And then I was able to open a shell in VMs:
```shell
# Shell into them
multipass shell vm1
multipass shell vm2
```
![[networking-setup-lab-multipass-shell-into-vms.png]]

In the above screenshot, I have two shells side by side and I how shelled in vm1 on the left side and in vm2 on the right side.

In the next step, I wanted to test basic connection between these two VMs. For that, I needed their IP address. In my MacOS terminal (not those VM shells), I run `multipass list` to find their IP addresses:
``

![[networking-setup-lab-multipass-list.png]]

And I see I have following information for two VMs:

| Name | State   | IPv4        | Image            |
| ---- | ------- | ----------- | ---------------- |
| vm1  | Running | 192.168.2.2 | Ubuntu 24.04 LTS |
| vm2  | Running | 192.168.2.3 | Ubuntu 24.04 LTS |
## Ping
Now use Ping to check connectivity between two VMs:
```shell
ping -c 4 192.168.2.2
```

![[networking-setup-lab-multipass-ping.mp4]]
