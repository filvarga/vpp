# DET44: Deterministic Network Address Translation (CGNAT)

## Introduction

Carrier-grade NAT allows pools with preallocated sessions and
predetermined translations from inside to outside with port.

Inside user is statically mapped to a set of outside ports with the
purpose of enabling deterministic NAT to reduce logging and to achieve
high scale/high performance in CGN deployments. Support endpoint dependent
mapping to deal with overloading of the outside ports. Prealocate 1000
session slots for each inside user. Use sequential port range assignment
algorithm (the first block goes to address 1, the second block to address 2, etc.)

##### Supported protocols:
 - TCP
 - UDP
 - ICMP

##### NAT Session Refresh:
 - NAT Outbound refresh behavior
 - default UDP idle-timeout 5 minutes
 - default TCP established connection idle-timeout 2 hours 4 minutes
 - default TCP transitory connection idle-timeout 4 minutes
 - default ICMP idle-timeout 60 seconds
 - TCP session close detection
 - configurable idle-timeouts

##### Memory requirements

Deterministic NAT prealocate vector with 1000 session slots (one session 15B)
for each host from inside network range. Heap size is configured via heapsize
parameter. 

## Configuration

### Enable/Disable DET44 plugin

> det44 plugin <enable [inside `vrf`] [outside `vrf`] | disable>

vrf id

### Enable/Disable DET44 feature on interface

> set interface det44 inside `intfc` outside `intfc` [del]

interface index.

### Add DET44 mapping

> det44 add in `addr`/`plen` out `addr`/`plen` [del]

IPv4 address and IPv4 suffix.

### Configure protocol timeouts

> set det44 timeouts <[udp `sec`] [tcp established `sec`]
[tcp transitory `sec`] [icmp `sec`] | reset>

number of seconds.

### Manualy close sessions

> det44 close session out `out_addr`:`out_port` `ext_addr`:`ext_port`\
det44 close session in `in_addr`:`in_port` `ext_addr`:`ext_port`

in_addr: inside IPv4 address, in_port: inside TCP/UDP port\
out_addr: outside IPv4 address, out_port: outside TCP/UDP port

### Get coresponding outside address based on inside address

> det44 forward `addr`

IPv4 address

### Get coresponding inside address based on outside address and port

> det44 reverse `addr`:`port`

IPv4 address and TCP/UDP port

### Show commands

```
show det44 interfaces
show det44 sessions
show det44 timeouts
show det44 mappings
```

## Notes

Deterministic NAT currently preallocates 1000 sessions per user.
