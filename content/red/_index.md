---
title: "Red"
---

# Topología de Red

![topología de red](/images/recursos/topologia_20-3.png)

---

# Direccionamiento de Red

## Routers

| Dispositivo | Interfaz | Dirección IP     | Red           | Destino          |
|-------------|----------|------------------|---------------|------------------|
| R1          | f0/1     | 192.168.10.1 /24 | 192.168.10.0  | IOU1 (VLAN 10/11)|
| R1          | f0/0     | 200.20.2.2 /24   | 200.20.2.0    | FortiGate Port2  |
| R2          | f0/1     | 192.168.20.1 /24 | 192.168.20.0  | IOU2 (VLAN 20/21)|
| R2          | f0/0     | 200.30.3.2 /24   | 200.30.3.0    | FortiGate Port3  |

## Firewall FortiGate

| Puerto | Dirección IP    | Red          | Conexión        |
|--------|-----------------|--------------|-----------------|
| Port1  | DHCP (WAN)      | —            | Internet (Cloud1)|
| Port2  | 200.20.2.1 /24  | 200.20.2.0   | R1              |
| Port3  | 200.30.3.1 /24  | 200.30.3.0   | R2              |
| Port4  | 200.40.4.1 /24  | 200.40.4.0   | DMZ (Ubuntu)    |

## VLANs

| VLAN    | Red               | Dispositivos          | Asignación IP |
|---------|-------------------|-----------------------|---------------|
| VLAN 10 | 192.168.10.0 /24  | PC1, DebianV10        | DHCP          |
| VLAN 11 | 192.168.11.0 /24  | PC2                   | DHCP          |
| VLAN 20 | 192.168.20.0 /24  | PC3                   | DHCP          |
| VLAN 21 | 192.168.21.0 /24  | PC4, DebianV21        | DHCP          |

## DMZ

| Dispositivo    | Interfaz | Dirección IP    | Red          |
|----------------|----------|-----------------|--------------|
| UbuntuServer-1 | e0       | 200.40.4.10 /24 | 200.40.4.0   |