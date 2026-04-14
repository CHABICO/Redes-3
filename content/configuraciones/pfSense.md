---
title: "Configuración de Firewall pfSense"
date: 2026-04-14
summary: "Implementación de pfSense como gateway de seguridad perimetral, segmentación de zonas (LAN, DMZ, MGMT) y conectividad con el clúster HSRP."
author: "Xavier C., Òscar F., Gerard S."
tags: ["pfSense", "Firewall", "DMZ", "Segmentación", "Seguridad"]
---

## Contexto y Objetivo

Tras estabilizar la alta disponibilidad en el borde con los routers Cisco, el siguiente paso es la integración del firewall **pfSense**. Este dispositivo actúa como el "cerebro" de la red, encargándose de la inspección de paquetes, el filtrado de tráfico entre zonas y la publicación de servicios internos hacia el exterior.

El pfSense se conecta al clúster de routers a través de una red de tránsito y segmenta la infraestructura interna en tres zonas diferenciadas para aplicar políticas de seguridad granulares.

---

## Direccionamiento de Interfaces

| Interfaz | Nombre GNS3 | Etiqueta pfSense | IP / Máscara | Gateway / Notas |
|----------|-------------|------------------|--------------|-----------------|
| **em0** | WAN         | WAN              | 10.10.0.10/24| 10.10.0.1 (VIP HSRP) |
| **em1** | LAN         | LAN              | 10.20.0.1/24 | Puerta enlace Clientes |
| **em5** | DMZ         | DMZ              | 172.16.0.1/24| Zona Servidores Públicos |
| **em2** | ADMIN       | MGMT             | 10.30.0.1/24 | Gestión (WebGUI) |

---

## Configuración de Zona LAN (10.20.0.0/24)

[🚧 ESTAMOS TRABAJANDO EN ELLO 🚧]

---

## Configuración de Zona DMZ (172.16.0.0/24)

[🚧 ESTAMOS TRABAJANDO EN ELLO 🚧]

---

## Servicios y Reglas Globales

### 1. Conectividad WAN y Enrutamiento
* **Default Gateway:** Configurado hacia la VIP del clúster HSRP (`10.10.0.1`).
* **DNS Forwarder:** Configurado con los servidores de Google (`8.8.8.8`) para resolución externa.
* **Regla Bogon/Private:** Desactivada en la interfaz WAN para permitir el tráfico desde el segmento `10.10.0.0/24`.

### 2. Acceso de Gestión (Zona MGMT)
* Se ha habilitado una regla de acceso total (`Allow Any`) desde la red `10.30.0.0/24` hacia la dirección de la interfaz MGMT para permitir la administración vía WebGUI sin depender de la consola.

![ping pfsense a google](/images/recursos/pfsense-ping-google.png)

---

## Próximos Pasos
- [ ] Implementación de reglas de firewall para permitir navegación desde la LAN.
- [ ] Configuración de **Port Forwarding (NAT)** para exponer el servidor web de la DMZ.
- [ ] Verificación de persistencia de sesión tras conmutación (failover) de routers.