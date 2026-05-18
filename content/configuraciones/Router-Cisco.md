---
title: "Configuración de Routers, Switches y VLANs"
date: 2026-02-26
summary: "Evolución de la topología de red: desde el aislamiento básico mediante VLANs hasta el enrutamiento inter-sedes y la fortificación de la capa 2."
author: "Xavier C., Òscar F., Gerard S."
tags: ["Router", "Switch", "VLANs", "DHCP", "Seguridad", "Capa2"]
---

## Contexto y Topología

R1 (Edificio 1) y R2 (Edificio 2) actúan como gateways de las VLANs internas de cada edificio. Cada router tiene una interfaz hacia el FortiGate y subinterfaces hacia los switches IOU que gestionan las VLANs de usuarios.

```
FortiGate port2 (10.30.0.1)
        ↓
   R1 Fa0/0 (10.30.0.2)
        ↓
   IOU1 (switch L2)
        ├── Fa0/1.10 → VLAN10 (192.168.10.0/24)
        └── Fa0/1.11 → VLAN11 (192.168.11.0/24)
```

El Edificio 2 replica esta estructura con R2 e IOU2, usando las redes `192.168.20.0/24` y `192.168.21.0/24` para evitar solapamiento con el Edificio 1.

---

## Direccionamiento

### Edificio 1 — R1

| Interfaz | IP | Descripción |
|----------|----|-------------|
| `Fa0/0` | `10.30.0.2/24` | Hacia FortiGate port2 |
| `Fa0/1.10` | `192.168.10.1/24` | Gateway VLAN10 |
| `Fa0/1.11` | `192.168.11.1/24` | Gateway VLAN11 |

### Edificio 2 — R2

| Interfaz | IP | Descripción |
|----------|----|-------------|
| `Fa0/0` | `10.30.0.2/24` | Hacia FortiGate port2 |
| `Fa0/1.10` | `192.168.20.1/24` | Gateway VLAN20 |
| `Fa0/1.11` | `192.168.21.1/24` | Gateway VLAN21 |

---

## Configuración R1 — Edificio 1

```bash
interface FastEthernet0/0
 ip address 10.30.0.2 255.255.255.0
 no shutdown

interface FastEthernet0/1.10
 encapsulation dot1Q 10
 ip address 192.168.10.1 255.255.255.0

interface FastEthernet0/1.11
 encapsulation dot1Q 11
 ip address 192.168.11.1 255.255.255.0

ip route 0.0.0.0 0.0.0.0 10.30.0.1
ip route 192.168.20.0 255.255.255.0 10.30.0.1
ip route 192.168.21.0 255.255.255.0 10.30.0.1
ip route 10.40.0.0 255.255.255.0 10.30.0.1
```

## Configuración R2 — Edificio 2

```bash
interface FastEthernet0/0
 ip address 10.30.0.2 255.255.255.0
 no shutdown

interface FastEthernet0/1.10
 encapsulation dot1Q 10
 ip address 192.168.20.1 255.255.255.0

interface FastEthernet0/1.11
 encapsulation dot1Q 11
 ip address 192.168.21.1 255.255.255.0

ip route 0.0.0.0 0.0.0.0 10.30.0.1
ip route 192.168.10.0 255.255.255.0 10.30.0.1
ip route 192.168.11.0 255.255.255.0 10.30.0.1
```

---

## DHCP

El servicio DHCP está configurado en cada router para asignar IPs dinámicamente a los clientes de cada VLAN.

### R1 — Edificio 1

```bash
ip dhcp excluded-address 192.168.10.1
ip dhcp excluded-address 192.168.11.1

ip dhcp pool VLAN_10
 network 192.168.10.0 255.255.255.0
 default-router 192.168.10.1
 dns-server 8.8.8.8

ip dhcp pool VLAN_11
 network 192.168.11.0 255.255.255.0
 default-router 192.168.11.1
 dns-server 8.8.8.8
```

### R2 — Edificio 2

```bash
ip dhcp excluded-address 192.168.20.1
ip dhcp excluded-address 192.168.21.1

ip dhcp pool VLAN_20
 network 192.168.20.0 255.255.255.0
 default-router 192.168.20.1
 dns-server 8.8.8.8

ip dhcp pool VLAN_21
 network 192.168.21.0 255.255.255.0
 default-router 192.168.21.1
 dns-server 8.8.8.8
```

---

## Bastionado de Capa 2 — IOU1/IOU2

Los switches IOU gestionan el tronco 802.1Q hacia los routers y los puertos de acceso hacia los clientes. Se han aplicado medidas de seguridad en la capa de acceso.

### Control de acceso y gestión

* **Gestión segura:** Accesos administrativos restringidos mediante SSH con `exec-timeout` para cerrar sesiones inactivas.
* **Cifrado de contraseñas:** Habilitado `service password-encryption`.
* **Banner de aviso legal:** Implementado `banner motd` para advertir sobre acceso no autorizado.

### Port Security

```bash
switchport port-security maximum 2
switchport port-security violation restrict
switchport port-security mac-address sticky
```

Límite de 2 MACs por puerto. En modo `restrict`, el tráfico no autorizado se descarta y se genera un log sin apagar el puerto.

### Mitigación de ataques de spoofing

```bash
# DHCP Snooping
ip dhcp snooping
ip dhcp snooping vlan 10,11

# Dynamic ARP Inspection
ip arp inspection vlan 10,11

# IP Source Guard (en puertos de acceso)
ip verify source
```

### Prevención de bucles

```bash
# Portfast en puertos de acceso
spanning-tree portfast

# BPDU Guard para evitar inyección de tramas BPDU
spanning-tree bpduguard enable
```

### Gestión de puertos no utilizados

```bash
# Apagar puertos libres y asignarlos a VLAN blackhole
interface range FastEthernet0/X - Y
 switchport access vlan 999
 shutdown
 switchport nonegotiate
```

---

## Cambio de IPs en Edificio 2

Al implementar la VPN site-to-site entre edificios, se detectó que ambos edificios usaban las mismas redes (`192.168.10.0/24` y `192.168.11.0/24`), lo que causaría solapamiento en el túnel IPsec. Se renombraron las VLANs del Edificio 2:

| Red original | Red nueva | Motivo |
|--------------|-----------|--------|
| `192.168.10.0/24` | `192.168.20.0/24` | Evitar solapamiento VPN |
| `192.168.11.0/24` | `192.168.21.0/24` | Evitar solapamiento VPN |

El cambio implicó actualizar: subinterfaces de R2, pools DHCP de R2, rutas estáticas en FortiGate Edificio 2, rutas estáticas en Rborde3/Rborde4 y reglas NAT en pfSense Edificio 2.