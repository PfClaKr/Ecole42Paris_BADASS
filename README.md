# Ecole42Paris_BADASS

### Environment
`Ubuntu LTS 24.04.2` in Virtual box


### Prerequisites
[gns3 installation](https://docs.gns3.com/docs/getting-started/installation/linux/) \
[docker installation](https://docs.docker.com/engine/install/ubuntu/)


### Summary Table

| Component | Role |
|----------|------|
| Loopback Interface | Stable IP used for BGP sessions and VTEP identity |
| OSPF | Ensures IP-level reachability between devices |
| BGP EVPN | Distributes MAC/IP/VTEP info across Leafs |
| VXLAN | Layer 2 overlay tunnel across Layer 3 network |
| EVPN Type 2 | MAC/IP Advertisement |
| EVPN Type 3 | BUM (Broadcast/Unknown/Multicast) traffic distribution |


## EVPN-VXLAN Based Spine-Leaf Architecture Communication Flow

### 1. Network Overview

Architecture Summary:
- A single Route Reflector (RR) at the top (Spine)
- Multiple `Leaf` switches connected to the `RR` via `iBGP EVPN`
- Each `Leaf` switch connects to a `Host` via `bridge` and `VXLAN`
- `OSPF` is used between `Leaf` and Host for IP reachability
- `VXLAN` tunnels are dynamically created between `Leaf` switches using `EVPN` control-plane


### 2. Pre-Configuration Steps

#### a. Loopback Configuration & Reachability
- Each `RR` and `Leaf` has a unique loopback interface
- `BGP` sessions use the loopback interface as `update-source`
- Reachability between loopback IPs is achieved via `OSPF`

#### b. BGP EVPN Session Setup
- `iBGP EVPN` sessions are formed between each `Leaf` and the `RR`
- Typical `Leaf` configuration includes:
  - `advertise-all-vni`
  - `neighbor x.x.x.x route-reflector-client`

#### c. VXLAN Tunnel Auto-Creation
- Using `EVPN` control plane, each `Leaf` learns the `VTEP` (loopback IP) of other `Leafs`
- `VXLAN` tunnels are dynamically created based on the `VNI` (Virtual Network Interface)
- No static `VXLAN` configuration required between `Leafs`

#### d. Host-to-Leaf Connectivity
- Each `Leaf` bridges:
  - Its physical interface (e.g., `eth1`)
  - The `VXLAN` interface (e.g., `vxlan10`)
  - Both interfaces are part of a Linux software `bridge` (e.g., `br0`)
- `OSPF` is used between `Leaf` and directly connected `Host`


### 3. Communication Flow (HostA -> HostB: Initial Ping)

#### Step 1: HostA sends Ping to HostB
- `HostA` does not have `HostB`’s `MAC`
- `ARP Request` is broadcast on the local segment

#### Step 2: Leaf1 receives ARP Request
- Since `HostB` is not on the local `Leaf1`, the `ARP Request` is replicated to all known `VTEPs` via `EVPN Type 3` `BUM` (Broadcast, Unknown, Multicast)
- `VXLAN` encapsulation is performed with `VNI=10` and sent to all remote `VTEPs` (`Leaf2`, `Leaf3`)

#### Step 3: Leaf2 responds with ARP Reply
- `Leaf2` is directly connected to `HostB` and knows its `MAC/IP`
- `Leaf2` sends an `ARP Reply` back to `Leaf1`, encapsulated in `VXLAN`

#### Step 4: Leaf1 receives ARP Reply
- Learns `HostB`’s `MAC/IP/VTEP` information
- Sets entry in the local `MAC/IP` table
- Sends `EVPN Type 2` route (`MAC/IP Advertisement`) to the `RR`

#### Step 5: RR reflects the information to other Leafs
- `Leaf2` and `Leaf3` receive `HostA`'s `MAC/IP/VTEP` info from the `RR`
- `Full mesh` `MAC` learning via `control plane` is achieved

#### Step 6: VXLAN Tunnel Forwarding Begins
- `Leaf1` now encapsluates future traffic for `HostB` in `VXLAN`
- `VNI` = 10; `VTEP` Src = `Leaf1` loopback, `VTEP` Dst = `Leaf2` loopback
- `HostA` `MAC/IP` -> `HostB` `MAC/IP`

#### Step 7: Leaf2 decapsulates and delivers to HostB
- `Leaf2` decapsulates `VXLAN` and forwards the Ethernet frame to `HostB`
- `HostB` replies via `ICMP` -> `Leaf2` -> `VXLAN` -> `Leaf1` -> `HostA`

---

### Helpful informations

#### When gns3 get a problem with root permission with docker, use this command 
```sh
sudo usermod -aG docker $USER
```
#### then reboot or logout/login the session, and you must have docker in the groups.
```sh
groups
```
#### How to build the dockerfile to image.
```sh
sudo docker build -t [container name] -f [dockerfile] .
```
#### Useful debug command line
```sh
# see arp table
arp -n
# delete arp table in interface like eth0 vxlan10
ip neigh flush dev [interface]
```
