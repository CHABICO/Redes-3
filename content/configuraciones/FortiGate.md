---
title: "Configuración del Firewall FortiGate"
date: 2026-02-26
summary: "Implementación de un Firewall Fortinet como núcleo de seguridad para gestionar el enrutamiento inter-sedes, acceso a WAN y publicación segura de servicios web."
author: "Xavier C., Òscar F., Gerard S."
tags: ["FortiGate", "Firewall", "Seguridad", "Routing", "Políticas"]
---

## Rol del FortiGate en la Topología

El FortiGate actúa como firewall interno situado entre los routers de borde (Rborde1/Rborde2) y las redes internas del Edificio 1. Gestiona tres zonas diferenciadas mediante sus puertos físicos y aplica políticas de seguridad para controlar el tráfico entre ellas.

```
Rborde1/Rborde2 (10.20.0.x)
        ↓
   FortiGate port1 (10.20.0.1)
        ├── port2 → R1 (10.30.0.0/24) → VLANs internas
        └── port3 → Switch3 → Servidores internos (10.40.0.0/24)
```

---

## Configuración de Interfaces

| Puerto | Red | IP | Descripción |
|--------|-----|----|-------------|
| port1 | `10.20.0.0/24` | `10.20.0.1` | Hacia Rborde1/Rborde2 (WAN interna) |
| port2 | `10.30.0.0/24` | `10.30.0.1` | Hacia R1 y VLANs |
| port3 | `10.40.0.0/24` | `10.40.0.1` | Hacia servidores internos y SIEM |

```bash
conf system interface
 edit port1
  set mode static
  set ip 10.20.0.1 255.255.255.0
  set allowaccess ping https ssh
 next
 edit port2
  set mode static
  set ip 10.30.0.1 255.255.255.0
  set allowaccess ping
 next
 edit port3
  set mode static
  set ip 10.40.0.1 255.255.255.0
  set allowaccess ping
 next
end
```

## Enrutamiento Estático

El FortiGate conoce todas las redes de la infraestructura mediante rutas estáticas:

| Destino | Gateway | Interfaz | Descripción |
|---------|---------|----------|-------------|
| `0.0.0.0/0` | `10.20.0.2` | port1 | Ruta por defecto hacia Rborde VIP |
| `192.168.10.0/24` | `10.30.0.2` | port2 | VLAN10 vía R1 |
| `192.168.11.0/24` | `10.30.0.2` | port2 | VLAN11 vía R1 |
| `192.168.20.0/24` | `10.20.0.2` | port1 | VLAN20 Edificio 2 vía Rborde → pfSense |
| `192.168.21.0/24` | `10.20.0.2` | port1 | VLAN21 Edificio 2 vía Rborde → pfSense |

## Políticas de Firewall

Las políticas controlan qué tráfico está permitido entre las distintas zonas. El FortiGate deniega implícitamente todo lo que no esté explícitamente permitido.

| ID | Origen | Destino | Acción | Servicio | Descripción |
|----|--------|---------|--------|----------|-------------|
| 1 | port2 | port3 | Accept | ALL | VLANs → Servidores internos |
| 2 | port2 | port1 | Accept | ALL | VLANs → Internet |
| 3 | port3 | port1 | Accept | ALL | Servidores → Internet |
| 4 | port3 | port2 | Accept | ALL | Servidores → VLANs (retorno) |
| 5 | port1 | port2 | Accept | ALL | Tráfico entrante → VLANs |
| 6 | port1 | port3 | Accept | HTTPS, WAZUH | Acceso externo → Servidores (Wazuh) |

El servicio personalizado `WAZUH` se creó para permitir los puertos de comunicación de los agentes:

```bash
conf firewall service custom
 edit WAZUH
  set protocol TCP/UDP
  set tcp-portrange 1514-1515
  set udp-portrange 1514
 next
end
```
## Incidencias durante la implementación

**port1 en modo DHCP:** Al arrancar, port1 estaba en modo DHCP y no cogía IP. Se resolvió forzando el modo estático con `set mode static` antes de asignar la IP.

**Políticas bloqueando tráfico interno:** Sin políticas configuradas, el FortiGate bloqueaba todo el tráfico aunque las rutas estuvieran bien. El síntoma era que los pings llegaban al FortiGate pero no salían hacia las redes internas. Se resolvió creando las políticas de accept entre zonas.

**NAT en políticas hacia port1:** Las políticas que permitían tráfico de las VLANs hacia internet tenían `set nat enable`, lo que hacía que el FortiGate tradujera las IPs origen a `10.20.0.1` antes de mandar a los Rborde. Esto rompía el NAT de pfSense y el routing asimétrico. Se resolvió eliminando `nat enable` de las políticas.

**Objetos VPN residuales:** Al duplicar la topología para el Edificio 2, quedaron objetos VPN residuales del wizard de FortiGate (fases IPsec, direcciones, políticas). Algunos no pudieron eliminarse por dependencias cruzadas pero no interfieren con el funcionamiento actual.
