#Fortuna

Fortuna is a tool designed to simulate common network problems like latency, bandwidth restrictions, and dropped/reordered/corrupted packets.  Testing distributed systems under hard failures like network partitions and instance termination is critical, but it's also important we test them under [less catastrophic conditions](http://www.bravenewgeek.com/sometimes-kill-9-isnt-enough/) because this is what they most often experience.

It works by wrapping up some system tools in a way. On BSD-derived systems such as OSX, we use tools like `ipfw` and `pfctl` to inject failure. On Linux, we use `iptables` and `tc`. Fortuna is a thin wrapper around these controls. Windows support is not currently added.

## Installation

```
$ go get github.com/sbberk/fortuna
```

## Usage

Currently (on Linux), Fortuna supports several options: device, latency, target/default bandwidth, packet loss, protocol, and port number

```
$ fortuna --device=eth0 --latency=250 --target-bw=1000 --default-bw=1000000 --packet-loss=10% --target-addr=8.8.8.8,10.0.0.0/24 --target-proto=tcp,udp,icmp --target-port=80,22,1000:2000
```

On OSX/BSD (with `ipfw`), Fortuna currently supports only: device, latency, target bandwidth, packet loss.
This will cease to be the case in a future (soon<sup>TM</sup>) update.

```
$ fortuna --device=eth0 --latency=250 --target-bw=1000 --packet-loss=10%
```

This will add 250ms of latency, limit bandwidth to 1Mbps, and drop 10% of packets to the targeted destination addresses using the specified protocols on the specified port numbers. The default bandwidth specified will apply to all egress traffic. To turn this off, run the following:

```
$ fortuna --mode stop
```

### Linux

On Linux, you can use `iptables` to drop incoming and outgoing packets.

```
$ iptables -A INPUT -m statistic --mode random --probability 0.1 -j DROP
$ iptables -A OUTPUT -m statistic --mode random --probability 0.1 -j DROP
```

Alternatively, you can use `tc` which supports some additional options.

```
$ tc qdisc add dev eth0 root netem delay 50ms 20ms distribution normal
$ tc qdisc change dev eth0 root netem reorder 0.02 duplicate 0.05 corrupt 0.01
```

To reset:

```
$ tc qdisc del dev eth0 root netem
```

### BSD/OSX

To shape traffic in BSD-derived systems, create an `ipfw` pipe and configure it. You can control incoming and outgoing traffic separately as well as which hosts are affected if you want.

```
$ ipfw add 1 pipe 1 ip from me to any
$ ipfw add 2 pipe 1 ip from any to me
$ ipfw pipe 1 config delay 500ms bw 1Mbit/s plr 0.1
```

To reset:

```
$ ipfw delete 1
```

*Note: `ipfw` was removed in OSX Yosemite in favor of `pfctl`.*

## Network Condition Profiles

Here's a list of network conditions with values that you can plug into Fortuna.

Name | Latency | Bandwidth | Packet-loss
3G/HSDPA (good) | 250 | 750 | 1.5
DIAL-UP (good) | 185 | 40 | 2
DSL (poor) | 70 | 2000 | 2
DSL (good) | 40 | 8000 | 0.5
WIFI (good) | 40 | 30000 | 0.2
Satellite | 1500 | - | 0.2
