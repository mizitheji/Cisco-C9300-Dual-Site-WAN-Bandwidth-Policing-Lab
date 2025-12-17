# Cisco C9300 Dual-Site WAN Bandwidth Policing Lab

This repository documents a dual-site WAN connectivity design using Cisco Catalyst 9300 switches.
The objective is to implement **ingress bandwidth policing** with **burst capability**, combined with **IP SLA–based WAN failover**, while keeping the core network free from QoS policies.

---

## Project Objectives

* Limit WAN traffic to **100 Mbps committed rate (CIR)**
* Allow traffic burst up to **200 Mbps (PIR)**
* Apply policing **on edge switches (SW1 & SW2) only**
* Implement **WAN redundancy** using IP SLA tracking
* Validate bandwidth enforcement using `iperf`

---

## Network Topology

```text
    LAN
     |
Core-SW (Site 1)
     |
SW1 (C9300)
     |
PRIMARY / SECONDARY WAN
     |
ISP / NID
     |
PRIMARY / SECONDARY WAN
     |
SW2 (C9300)
     |
Core-SW (Site 2)
     |
    LAN
```

> QoS is applied on **WAN-facing interfaces** of SW1 and SW2
> No QoS policies are configured on Core switches

---

## IP Addressing

### Site 1

| Interface | Description   | IP Address      |
| --------- | ------------- | --------------- |
| Gi1/0/1   | Primary WAN   | 172.26.40.9/30  |
| Gi1/0/2   | Secondary WAN | 172.26.40.13/30 |
| Gi1/0/24  | LAN           | 172.17.1.78/24  |

### Site 2

| Interface | Description   | IP Address        |
| --------- | ------------- | ----------------- |
| Gi1/0/1   | Primary WAN   | 172.26.40.10/30   |
| Gi1/0/2   | Secondary WAN | 172.26.40.14/30   |
| Gi1/0/24  | LAN           | 172.31.230.246/24 |

---

## WAN Redundancy Design (IP SLA)

* Primary WAN monitored using **ICMP echo**
* Probe interval: **5 seconds**
* Static default route tracked using `track 10`
* Secondary WAN used as backup with higher administrative distance

### IP SLA Configuration (Both Sites)

```cisco
ip sla 10
 icmp-echo <remote-wan-ip> source-interface GigabitEthernet1/0/1
 frequency 5
ip sla schedule 10 life forever start-time now
track 10 ip sla 10 reachability
```

---

## QoS / Bandwidth Policing Strategy

### Policy Objective

| Parameter | Value    |
| --------- | -------- |
| CIR       | 100 Mbps |
| PIR       | 200 Mbps |
| Exceed    | Transmit |
| Violate   | Drop     |

### Policy Map

```cisco
policy-map LIMIT-100M-BURST-200M
 class class-default
  police cir 100000000 pir 200000000
   conform-action transmit
   exceed-action transmit
   violate-action drop
```

### Policy Application

Applied **ingress** on WAN interfaces only:

```cisco
interface GigabitEthernet1/0/1
 service-policy input LIMIT-100M-BURST-200M

interface GigabitEthernet1/0/2
 service-policy input LIMIT-100M-BURST-200M
```

> Ingress policing ensures traffic is controlled **before entering the WAN**, preventing provider-side congestion.

---

## Routing Configuration

### Site 1

```SW1 (C9300)
ip route 0.0.0.0 0.0.0.0 172.26.40.10 track 10 name PRIMARY
ip route 0.0.0.0 0.0.0.0 172.26.40.14 250 name SECONDARY
ip route 172.31.4.0 255.255.255.0 172.17.1.67 name LAN
```

### Site 2

```SW2 (C9300)
ip route 0.0.0.0 0.0.0.0 172.26.40.9 track 10 name PRIMARY
ip route 0.0.0.0 0.0.0.0 172.26.40.13 250 name SECONDARY
ip route 172.20.232.0 255.255.255.0 172.31.230.4 name LAN
```

---

## Testing & Validation

### Bandwidth Testing (iperf)

```bash
*Server command*
iperf3 -s

*Client command*
iperf3 -c <remote-ip> -t 30 (TCP Throughput Test)
iperf3 -c <remote-ip> -u -b 200M -t 30 (UDP Burst Test)
```

Expected behavior:

* Throughput capped near **100 Mbps**
* Burst allowed up to **200 Mbps**
* Packet drops observed only when exceeding PIR

### QoS Verification

```cisco
show policy-map interface gigabitEthernet1/0/1
```

---

## Design Notes & Best Practices

* Apply QoS at the **network edge**, not the core
* Prefer **ingress policing** over egress shaping on WAN
* Avoid double-policing in the path
* Validate with both **TCP and UDP** traffic
