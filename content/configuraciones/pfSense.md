---
title: "Configuración de Firewall pfSense"
date: 2026-04-14
summary: "Implementación de pfSense como gateway de seguridad perimetral, segmentación de zonas (WAN, LAN, DMZ, MGMT), reglas de firewall y publicación de servicios hacia el exterior mediante NAT."
author: "Xavier C., Òscar F., Gerard S."
tags: ["pfSense", "Firewall", "DMZ", "NAT", "Segmentación", "Seguridad"]
---

## Contexto y Objetivo

## Contexto y Objetivo

pfSense actúa como primera línea de defensa perimetral de la infraestructura. Recibe el tráfico directamente desde Cloud1 (ISP simulado) por la interfaz WAN, y lo distribuye hacia los routers de borde (Rborde1/Rborde2) por la interfaz LAN. También gestiona la zona DMZ donde se aloja el servidor público, y aplica NAT para que las redes internas puedan salir a internet.

![topología GNS3 - pfSense](/images/recursos/topología-pfsense.png)

---

## Direccionamiento de Interfaces

| Interfaz | Etiqueta pfSense | IP / Máscara | Notas |
|----------|------------------|--------------|-------|
| **em0** | WAN | Estática (`192.168.44.139/24`) | Conectada a Cloud1 (ISP simulado) |
| **em1** | LAN | `10.10.0.1/24` | Hacia Switch1 → Rborde1/Rborde2 |
| **em5** | DMZ | `172.16.0.1/24` | Zona servidor público |

![interfaces con sus IPs y estado up](/images/recursos/interfaces-pfsense.png)

---

## Enrutamiento

pfSense conoce las redes internas mediante rutas estáticas configuradas en **System → Routing → Static Routes**, todas apuntando al VIP HSRP de los Rborde (`10.10.0.2`) como siguiente salto:

| Red destino | Gateway | Descripción |
|-------------|---------|-------------|
| `192.168.10.0/24` | `GW_VIP - 10.10.0.2` | VLAN10 vía Rborde → FortiGate → R1 |
| `192.168.11.0/24` | `GW_VIP - 10.10.0.2` | VLAN11 vía Rborde → FortiGate → R1 |
| `10.20.0.0/24` | `GW_VIP - 10.10.0.2` | Red FortiGate-Rborde |
| `10.30.0.0/24` | `GW_VIP - 10.10.0.2` | Red FortiGate-R1 |
| `10.40.0.0/24` | `GW_VIP - 10.10.0.2` | Servidores internos |

El gateway por defecto para salida a internet (`WAN_DHCP`) está definido en **System → Routing → Gateways** apuntando al gateway de Cloud1.

---

## Reglas de Firewall

### Interfaz WAN

Las opciones **Block private networks** y **Block bogon networks** están desactivadas en la interfaz WAN.

| Acción | Protocolo | Origen | Destino | Puerto | Descripción |
|--------|-----------|--------|---------|--------|-------------|
| Pass | TCP | any | WAN address | 80 | HTTP hacia servidor DMZ |
| Pass | TCP | any | WAN address | 8888 | Acceso GUI FortiGate |
| Pass | TCP | any | WAN address | 9999 | Acceso GUI pfSense |
| Pass | TCP | any | WAN address | 7777 | Acceso dashboard Wazuh |
| Pass | TCP | any | WAN address | 22 | SSH administración |

### Interfaz LAN

| Acción | Protocolo | Origen | Destino | Puerto | Descripción |
|--------|-----------|--------|---------|--------|-------------|
| Pass | any | any | any | any | Permitir redes internas hacia internet |
| Pass | IPv4 TCP | `10.10.0.0/24` | any | any | LAN hacia cualquier destino |
| Pass | IPv4 * | LAN subnets | any | any | Default allow LAN to any rule |

### Interfaz DMZ

| Acción | Protocolo | Origen | Destino | Puerto | Descripción |
|--------|-----------|--------|---------|--------|-------------|
| Pass | IPv4 * | `172.16.0.0/24` | `10.40.0.20` | 1514, 1515 | DMZ hacia SIEM Wazuh |
| Block | IPv4 TCP | `172.16.0.0/24` | `10.10.0.0/24` | any | DMZ no accede a LAN |
| Pass | IPv4 * | `172.16.0.0/24` | any | any | DMZ sale a Internet |

---

## NAT

### Port Forwarding

Se han configurado cuatro reglas de port forwarding para acceder a los servicios internos desde el exterior:

| Puerto externo | Destino interno | Descripción |
|----------------|-----------------|-------------|
| `80` | `172.16.0.20:80` | Servidor web DMZ |
| `8888` | `10.20.0.1:443` | GUI FortiGate |
| `9999` | `10.10.0.1:443` | GUI pfSense |
| `7777` | `10.40.0.20:443` | Dashboard Wazuh |

[🔴 imagen mostrando las reglas de port forwarding en Firewall → NAT → Port Forward]

### NAT Outbound

Configurado en modo **Hybrid** para combinar reglas automáticas con reglas manuales adicionales. Las reglas manuales garantizan que todas las redes internas reciben NAT correctamente al salir por la WAN:

| Interface | Source | NAT Address | Descripción |
|-----------|--------|-------------|-------------|
| WAN | `192.168.10.0/24` | WAN address | NAT VLAN10 |
| WAN | `192.168.11.0/24` | WAN address | NAT VLAN11 |
| WAN | `172.16.0.0/24` | WAN address | NAT DMZ |
| WAN | `10.20.0.0/24` | WAN address | NAT red FortiGate |
| WAN | `10.30.0.0/24` | WAN address | NAT red R1 |
| WAN | `10.40.0.0/24` | WAN address | NAT servidores internos |

Adicionalmente se han configurado reglas **No NAT** para excluir el tráfico VPN site-to-site del proceso de traducción, garantizando que los paquetes entre edificios mantienen sus IPs originales al entrar en el túnel IPsec:

| Interface | Source | Destination | Acción |
|-----------|--------|-------------|--------|
| WAN | `192.168.10.0/24` | `192.168.20.0/24` | No NAT |
| WAN | `192.168.11.0/24` | `192.168.21.0/24` | No NAT |

---

## Incidencias durante la implementación

**Conflicto WAN/LAN en misma subred:** Al reestructurar la topología, pfSense tenía WAN y LAN en la misma red `192.168.44.0/24`, lo que impedía el enrutamiento correcto. Se resolvió asignando `10.10.0.1/24` a la LAN.

**NAT no aplicaba a redes internas:** pfSense en modo automático solo hace NAT para redes directamente conectadas. Se resolvió cambiando a modo Hybrid y añadiendo reglas manuales para cada subred interna.

**Routing asimétrico con FortiGate:** El FortiGate hacía NAT con su propia IP antes de mandar tráfico a los Rborde. Se resolvió eliminando el NAT en las políticas del FortiGate hacia port1.

**Anti-spoofing pfSense:** pfSense bloqueaba silenciosamente paquetes que llegaban por LAN con origen en redes no directamente conectadas. Se resolvió añadiendo rutas estáticas explícitas para cada red interna.

**Routing de retorno desde los Rborde:** Los Rborde no tenían rutas de vuelta hacia las redes internas, por lo que las respuestas no llegaban a los clientes. Se resolvió añadiendo rutas estáticas en los Rborde apuntando al FortiGate (`10.20.0.1`) para cada red interna.