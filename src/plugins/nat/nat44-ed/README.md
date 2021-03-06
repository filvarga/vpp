# NAT44-ED: NAT44 Endpoint Dependent

## TBD
- Describe Port preserving port allocation
- Describe attack vectors and "protections"
  Max-sessions. How max-sessions relate to multiple workers.
  Add description of static mapping being stateless.
- Static mappings
- How to determine if the NAT is running out of ports?
- Describe session table, how LRU works
- Describe connection tracking
- CLI vs API. Or should we just create a supported CLI instead?
- Future: Drawing of NAT pools, instances, per-VRF 
- Description of running NAT on inside interface and outside interface or only on outside.

## Introduction

NAT44-ED is the IPv4 endpoint dependent network address translation plugin.
The component implements an address and port-dependent mapping and address and port-dependent filtering NAT as
described in [RFC4787](https://tools.ietf.org/html/rfc4787). 

The outside address and port (X1':x1') is reused for internal hosts (X:x) for different values of Y:y.
A flow is matched by {source address, destination address, protocol, transport source port, transport destination port,
fib index}. As long as all these are unique the mapping is valid. While a single outside address in theory allows for
2^16 source ports * 2^32 destination IP addresses * 2^16 destination ports = 2^64 sessions, this number is much smaller
in practice. Few destination ports are generally used (80, 443) and a fraction of the IP address space is available.
The limitation is 2^16 bindings per outside IP address to a single destination address and port (Y:y).

The implementation is split, a control-plane / slow-path and a data-plane / fast-path. Essentially acting as a flow
router. The data-plane does a 6-tuple flow lookup (SA, DA, P, SP, DP, FIB) and on a match runs the per-flow packet
handling instructions on the packet. On a flow lookup miss, the packet is punted to the slow-path, where depending
on policy new sessions are created.

The support set of packet handling instructions is ever-increasing. Currently, the implementation supports rewrite
of SA, DA, SP, DP and TCP MSS. The fast-path also does connection tracking and expiry of older sessions.

NAT44-ED uses 6 tuple`(src address, src port, dst address, dst port,
protocol and fib)`for matching communication.

Supported features:

- NAT Overload - PAT (Port Address Translation)
- Dynamic NAT (Network address translation)
- Static NAT (static mappings)
  - address only mapping
    - 1-1 translation withouth ports
  - twice-nat
    - double-nat, translation of source and destination
  - self-twice-nat
    - double-nat, translation of source and destination, where external
      host address is the same as local host address
  - out2in-only
    - session is created only from outside nat interface
- identity mapping
  - exceptions to translations
- load-balanced static mapping
  - feature used for translating one outside frontend (`addr`:`port`)
    to multiple backends (`addr`:`port`)
- mss-clamping
- syslog
- ipfix logging
- pre routing / post routing translations
  - pre routing is the default behaviour, nat processing happens before
    ip4-lookup node
  - using output-feature moves processing of nat traffic for the specific
    nat interface after ip4-lookup node
  - output-features enables translating traffic after routing

## Terminology

## Configuraiton

### Enable/disable plugin

> nat44 enable sessions `max-number` [static-mapping-only [connection-tracking]]
[inside-vrf `vrf-id`] [outside-vrf `vrf-id`]

> nat44 disable

### Enable/disable feature on interface

> set interface nat44 in `intfc` out `intfc` [output-feature] [del]

inside, outside interface index.

### Enable/disable forwarding

> nat44 forwarding enable|disable

### Add outside address

> nat44 add interface address `interface` [twice-nat] [del]\
nat44 add address `ip4-range-start` [- `ip4-range-end`]
[tenant-vrf `vrf-id`] [twice-nat] [del]

### Configure protocol timeouts

> set nat timeout [udp `sec` | tcp-established `sec` tcp-transitory `sec`
| icmp `sec` | reset]

### Additional configuration

> set nat frame-queue-nelts `number`\
set nat workers `workers-list`

### Configure logging

> nat set logging level `level`\
nat ipfix logging [domain `domain-id`] [src-port `port`] [disable]

### Configure mss-clamping

> nat mss-clamping `mss-value`|disable

### Add static mapping

> nat44 add static mapping tcp|udp|icmp local `addr` [`port|icmp-echo-id`]
external `addr` [`port|icmp-echo-id`] [vrf `table-id`]
[twice-nat|self-twice-nat] [out2in-only] [exact `pool-addr`] [del]

### Add identity mapping

> nat44 add identity mapping `ip4-addr`|external `interface`
[`protocol` `port`] [vrf `table-id`] [del]

### Configure load balanced static mapping

> nat44 add load-balancing back-end protocol tcp|udp external `addr`:`port`
local `addr`:`port` [vrf `table-id`] probability `n` [del]

> nat44 add load-balancing static mapping protocol tcp|udp  external
`addr`:`port` local `addr`:`port` [vrf `table-id`] probability `n`
[twice-nat|self-twice-nat] [out2in-only] [affinity `timeout-seconds`] [del]

### Configure per vrf session limits

> set nat44 session limit `limit` [vrf `table-id`]

### Delete session

> nat44 del session in|out `addr`:`port` tcp|udp|icmp [vrf `id`]
[external-host `addr`:`port`]

### Show commands

```
show nat workers
show nat timeouts
show nat44 summary
show nat44 sessions
show nat44 addresses
show nat mss-clamping
show nat44 interfaces
nat44 show static mappings
show nat44 interface address
show nat44 hash tables [detail|verbose]
```

## Examples

### PAT - Port Address Translation (NAT Overload)


### Identity NAT

TODO: identity NAT implementation is not what is expected
  it should create session and it should do translations to
  itself

Can be initiated only from inside

TODO: rewrite this one

When you do Identity NAT you are translating a packet to "itself"
(pre-NAT packet looks the same as post-NAT packet) so you'll see
an entry in the XLATE table.

You would use Identity NAT when you want to traffic from your inside
interface to flow through to your outside interface without changing
the address. An example scenario would be a private MPLS cloud with
separate clients. Each client has a unique address space so NATing
is not necessary. Using Identity NAT is the solution because it
provides us with the privacy of only allowing inside hosts to
initiate communication with outside hosts.

### TWICE-NAT

Twice NAT lets you translate both the source and destination address
in a single rule. Currently, twice NAT44 is supported only for local
network service session initiated from outside network.
Twice NAT static mappings can only get initiated (create sessions)
from outside network.

##### Topology

```
+--------------------------+
| 10.0.0.2/24 (local host) |
+--------------------------+
            |
+---------------------------------+
| 10.0.0.1/24 (eth0) (nat inside) |
| 20.0.0.1/24 (eth1) (nat outside)|
+---------------------------------+
            |
+---------------------------+
| 20.0.0.2/24 (remote host) |
+---------------------------+
```

In this example traffic will be initiated from remote host.
Remote host will be accessing local host via twice-nat mapping.

##### Translation will occur as follows:

###### outside to inside translation:
> src address: 20.0.0.2 -> 192.168.160.101\
dst address: 20.0.0.1 -> 10.0.0.2

###### inside to outside translation:
> src address: 10.0.0.2 -> 20.0.0.1\
dst address: 192.168.160.101 -> 20.0.0.2

#### Configuration

###### Enable nat44-ed plugin:
```
nat44 enable sessions 1000
```

###### Configure inside interface:
```
set int state eth0 up
set int ip address eth0 10.0.0.1/24
set int nat44 in eth0
```

###### Configure outside interface:
```
set int state eth1 up
set int ip address eth1 20.0.0.1/24
set int nat44 out eth1
```

###### Configure nat address pools:
```
nat44 add address 20.0.0.1
nat44 add address 192.168.160.101 twice-nat
```
- alternatively we could use `nat44 add interface address eth1`
- both pools are required
- pool `20.0.0.1` is used for out2in incoming traffic
- special twice-nat pool `192.168.160.101` is used for secondary translation

###### Finally, add twice-nat mapping:
> nat44 add static mapping tcp local 10.0.0.2 5201
external 20.0.0.1 5201 twice-nat

### SELF TWICE-NAT

Self twice NAT works similar to twice NAT with few exceptions.
Self twice NAT is a feature that lets client and service running
on the same host to communicate via NAT device. This means that
external address is the same address as local address.
Self twice NAT static mappings can only get initiated
(create sessions) from outside network.

##### Topology

```
+--------------------------+
| 10.0.0.2/24 (local host) |
+--------------------------+
            |
+-------------------------------------------+
| 10.0.0.1/24 (eth0) (nat inside & outside) |
+-------------------------------------------+
```

In this example traffic will be initiated from local host.
Local host will be accessing itself via self-twice-nat mapping.

##### Translation will occur as follows:

###### outside to inside translation:
> src address: 10.0.0.2 -> 192.168.160.101\
dst address: 10.0.0.1 -> 10.0.0.2

###### inside to outside translation:
> src address: 10.0.0.2 -> 10.0.0.1\
dst address: 192.168.160.101 -> 10.0.0.2

#### Configuration

###### Enable nat44-ed plugin:
```
nat44 enable sessions 1000
```

###### Configure NAT interface:
```
set int state eth0 up
set int ip address eth0 10.0.0.1/24
set int nat44 in eth0
set int nat44 out eth0
```

###### Configure nat address pools:
```
nat44 add address 10.0.0.1
nat44 add address 192.168.160.101 twice-nat
```

###### Finally, add self-twice-nat mapping:
> nat44 add static mapping tcp local 10.0.0.2 5201
external 10.0.0.1 5201 self-twice-nat