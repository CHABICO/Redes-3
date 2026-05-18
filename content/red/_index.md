---
title: "Red"
---
# Topología de Red
## Proximamente 
![topología de red nueva](__/images/recursos/topologia.png__)
---
# Direccionamiento de Red

## Edificio 1

### Routers de Borde
| Dispositivo | Interfaz | Dirección IP    | Red           | Destino              |
|-------------|----------|-----------------|---------------|----------------------|
| Rborde1     | Fa0/0    | 10.10.0.3 /24   | 10.10.0.0     | pfSense LAN          |
| Rborde1     | Fa1/0    | 10.20.0.3 /24   | 10.20.0.0     | FortiGate Port1      |
| Rborde2     | Fa0/0    | 10.10.0.4 /24   | 10.10.0.0     | pfSense LAN          |
| Rborde2     | Fa1/0    | 10.20.0.4 /24   | 10.20.0.0     | FortiGate Port1      |
| HSRP VIP    | —        | 10.10.0.2 /24   | 10.10.0.0     | Gateway pfSense→Rborde |
| HSRP VIP    | —        | 10.20.0.2 /24   | 10.20.0.0     | Gateway Rborde→FortiGate |

### Firewall pfSense — Edificio 1
| Interfaz | Dirección IP        | Red             | Conexión          |
|----------|---------------------|-----------------|-------------------|
| WAN (em0)| 192.168.44.139 /24  | 192.168.44.0    | Cloud1 (Internet) |
| LAN (em1)| 10.10.0.1 /24       | 10.10.0.0       | Switch1 → Rborde1/2 |
| DMZ (em5)| 172.16.0.1 /24      | 172.16.0.0      | Servidor DMZ      |

### Firewall FortiGate — Edificio 1
| Puerto | Dirección IP    | Red          | Conexión              |
|--------|-----------------|--------------|-----------------------|
| Port1  | 10.20.0.1 /24   | 10.20.0.0    | Rborde1/2 (WAN interna)|
| Port2  | 10.30.0.1 /24   | 10.30.0.0    | R1                    |
| Port3  | 10.40.0.1 /24   | 10.40.0.0    | Servidores internos   |

### Router R1 — Edificio 1
| Interfaz    | Dirección IP      | Red            | Destino           |
|-------------|-------------------|----------------|-------------------|
| Fa0/0       | 10.30.0.2 /24     | 10.30.0.0      | FortiGate Port2   |
| Fa0/1.10    | 192.168.10.1 /24  | 192.168.10.0   | IOU1 (VLAN10)     |
| Fa0/1.11    | 192.168.11.1 /24  | 192.168.11.0   | IOU1 (VLAN11)     |

### VLANs — Edificio 1
| VLAN    | Red               | Dispositivos       | Asignación IP |
|---------|-------------------|--------------------|---------------|
| VLAN 10 | 192.168.10.0 /24  | PC1, DebianV10     | DHCP          |
| VLAN 11 | 192.168.11.0 /24  | PC2                | DHCP          |

### DMZ — Edificio 1
| Dispositivo         | Interfaz | Dirección IP    | Red          |
|---------------------|----------|-----------------|--------------|
| Ubuntu-Server-DMZ-1 | ens33    | 172.16.0.20 /24 | 172.16.0.0   |

### Servidores Internos — Edificio 1
| Dispositivo              | Dirección IP    | Red          | Servicio          |
|--------------------------|-----------------|--------------|-------------------|
| Ubuntu-Server-Interno-1  | 10.40.0.10 /24  | 10.40.0.0    | DNS, Correo, IDS  |
| Ubuntu-Server-SIEM-1     | 10.40.0.20 /24  | 10.40.0.0    | Wazuh SIEM        |

---

## Edificio 2

### Routers de Borde
| Dispositivo | Interfaz | Dirección IP    | Red           | Destino                  |
|-------------|----------|-----------------|---------------|--------------------------|
| Rborde3     | Fa0/0    | 10.10.0.3 /24   | 10.10.0.0     | pfSense LAN              |
| Rborde3     | Fa1/0    | 10.20.0.3 /24   | 10.20.0.0     | FortiGate Port1          |
| Rborde4     | Fa0/0    | 10.10.0.4 /24   | 10.10.0.0     | pfSense LAN              |
| Rborde4     | Fa1/0    | 10.20.0.4 /24   | 10.20.0.0     | FortiGate Port1          |
| HSRP VIP    | —        | 10.10.0.2 /24   | 10.10.0.0     | Gateway pfSense→Rborde   |
| HSRP VIP    | —        | 10.20.0.2 /24   | 10.20.0.0     | Gateway Rborde→FortiGate |

### Firewall pfSense — Edificio 2
| Interfaz | Dirección IP        | Red             | Conexión            |
|----------|---------------------|-----------------|---------------------|
| WAN (em0)| 192.168.44.136 /24  | 192.168.44.0    | Cloud1 (Internet)   |
| LAN (em1)| 10.10.0.1 /24       | 10.10.0.0       | Switch1 → Rborde3/4 |

### Firewall FortiGate — Edificio 2
| Puerto | Dirección IP    | Red          | Conexión               |
|--------|-----------------|--------------|------------------------|
| Port1  | 10.20.0.1 /24   | 10.20.0.0    | Rborde3/4 (WAN interna)|
| Port2  | 10.30.0.1 /24   | 10.30.0.0    | R2                     |

### Router R2 — Edificio 2
| Interfaz    | Dirección IP      | Red            | Destino           |
|-------------|-------------------|----------------|-------------------|
| Fa0/0       | 10.30.0.2 /24     | 10.30.0.0      | FortiGate Port2   |
| Fa0/1.10    | 192.168.20.1 /24  | 192.168.20.0   | IOU2 (VLAN20)     |
| Fa0/1.11    | 192.168.21.1 /24  | 192.168.21.0   | IOU2 (VLAN21)     |

### VLANs — Edificio 2
| VLAN    | Red               | Dispositivos       | Asignación IP |
|---------|-------------------|--------------------|---------------|
| VLAN 20 | 192.168.20.0 /24  | PC3, DebianV20     | DHCP          |
| VLAN 21 | 192.168.21.0 /24  | PC4                | DHCP          |

---

## VPN Site-to-Site
| Parámetro       | Valor                    |
|-----------------|--------------------------|
| Protocolo       | IPsec IKEv2              |
| Extremo Ed1     | 192.168.44.139 (pfSense) |
| Extremo Ed2     | 192.168.44.136 (pfSense) |
| Redes Ed1       | 192.168.10.0/24, 192.168.11.0/24, 10.40.0.0/24 |
| Redes Ed2       | 192.168.20.0/24, 192.168.21.0/24 |