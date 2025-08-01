# FreeBSD port of AmneziaWG kernel module

## Installation

Download and build kernel module as:

```shell
git clone https://github.com/vgrebenschikov/amneziawg-kmod
sudo make install -C amneziawg-kmod
```

Download and build userland tools:

```shell
git clone https://github.com/vgrebenschikov/amneziawg-tools
sudo make install -C amneziawg-tools
```

## Using Kernel AmneziaWG module

Install [net/amneziawg-kmod](https://github.com/vgrebenschikov/amneziawg-kmod)

```shell
kldload -n if_awg
```

## Configuration with awg tool

/usr/local/bin/awg tool can be used to configure awg device:

```shell

bash

if=$(ifconfig awg create inet 192.168.1.1/24 up)
ifconfig $if
  awg0: flags=10080c1<UP,RUNNING,NOARP,MULTICAST,LOWER_UP> metric 0 mtu 1420
	options=80000<LINKSTATE>
	inet 192.168.1.1 netmask 0xffffff00
	groups: awg
	nd6 options=109<PERFORMNUD,IFDISABLED,NO_DAD>

awg set $if jc 7 jmin 150 jmax 1000 s1 117 s2 321 h1 2008066467 h2 2351746464 h3 3053333659 h4 1789444460
awg set $if listen-port 12345 private-key <(awg genkey) peer $(awg genkey | awg pubkey) allowed-ips 192.168.1.2/32

awg show $if
  interface: awg0
    public key: yyAHM...
    private key: (hidden)
    listening port: 12345
    jc: 7
    jmin: 150
    jmax: 1000
    s1: 117
    s2: 321
    h1: 2008066467
    h2: 2351746464
    h3: 3053333659
    h4: 1789444460

peer: bdfTF..
  allowed ips: 192.168.1.2/32
```

## Configuration with awg-quick

Generally - similar way as you will configure with wg-quick from net/wireguard-tools:
With configuration file, like:

```shell
# cd /usr/local/etc/amnezia/amneziawg/
# cat > awg0.conf << EOF
[Interface]
PrivateKey = $(awg genkey)
ListenPort = 12345
Address = 192.168.1.1/24
Description = Test AmneziaWG

Jc = 7
Jmin = 150
Jmax = 1000
S1 = 117
S2 = 321
H1 = 2008066467
H2 = 2351746464
H3 = 3053333659
H4 = 1789444460

[Peer]
PublicKey = $(awg genkey | awg pubkey)
AllowedIPs = 192.168.1.2/32
EOF
```

Then start:

```shell
awg-quick up awg0
[#] ifconfig awg create name awg0 description Test AmneziaWG
[#] awg setconf awg0 /dev/stdin
[#] ifconfig awg0 inet 192.168.1.1/24 alias
[#] ifconfig awg0 mtu 1420
[#] ifconfig awg0 up
[#] route -q -n add -inet 192.168.1.2/32 -interface awg0
[+] Backgrounding route monitor

awg show
interface: awg0
  public key: CI...
  private key: (hidden)
  listening port: 12345
  jc: 7
  jmin: 150
  jmax: 1000
  s1: 117
  s2: 321
  h1: 2008066467
  h2: 2351746464
  h3: 3053333659
  h4: 1789444460

peer: kue...
  allowed ips: 192.168.1.2/32
```

To setup autostart (amneziawg rc.d script will load if_awg module):

```shell
# sysrc amneziawg_enable=YES amneziawg_interfaces="awg0"
```

## Amnezia Wireguard config options

## Jc

Number of junk packets before handshake.

1–128 (recomended 3–10)

## Jmin

Minimum size of junk packets.

Jmin: < Jmax (recomended ~ 50)

## Jmax

Maximum size of junk packets.

Jmax: ≤ 1280 (recomended ~ 1000)

## S1

Size of handshake initiation packet prepend junk. Should be the same on both ends.

0–1280 (recomended 15–150), S1 != S2

## S2

Size of handshake response packet prepend junk. Should be the same on both ends.

0–1280 (recomended 15–150), S1 != S2

## H1-H4

Custom identifiers for initiation/response/cookie/data packets. Should be the same on both ends.

The unique value in range of 5 - 4,294,967,295 (0x5 - 0xFFFFFFFF), H1 != H2 != H3 != H4

## Additional config options

### Description

```config
[Interface]
...
Description = Some Text
```

Will setup interface description visible in ifconfig and SNMP.

### UserLand

Enforce to use amnezia-go instead of kernel driver, you can use port
[net/amnezia-wireguard-go](https://github.com/vgrebenschikov/amnezia-wireguard-go) to install it.

```config
[Interface]
...
UserLand = true
...
```

### Routes

List of routes for the peer to be installed into FIB - that option provides a way to have AllowedIPs list wider then routes installed. Empty list is allowed.

That is useful if routing protocol will work over the link. 
But remember that internal wireguard routing will happen according to AllowedIPs anyway.
Suggested use in case dynamic route - one interface -> one link.

```config
...

[Peer]
PublicKey = ...peer.public.key.here...
AllowedIPs = 0.0.0.0/0
Routes = 192.168.1.2/32
```

and after start - only routes limited in Routes config section:

```shell
awg-quick up awg0
...

netstat -rn | fgrep awg0
192.168.1.0/24     link#3             U              awg0
192.168.1.2        link#3             UHS            awg0
```
