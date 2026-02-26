---
title: "Configuración del Firewall FortiGate"
date: 2026-02-26
summary: "Implementación de un Firewall Fortinet como núcleo de seguridad para gestionar el enrutamiento inter-sedes, acceso a WAN y publicación segura de servicios web."
author: "Xavier C., Òscar F., Gerard S."
---

## Rol del Firewall en la Topología

Para gobernar el tráfico entre nuestras dos sedes principales (R1 y R2) y gestionar el perímetro hacia el exterior, desplegamos un Firewall Fortinet (FortiGate). Este dispositivo actúa como punto central de inspección y enrutamiento entre las distintas redes internas y la salida a Internet (WAN).

## Configuración de Interfaces

Se definieron tres zonas lógicas principales mapeadas a los puertos físicos del FortiGate, habilitando accesos administrativos como Ping, HTTPS y SSH para su gestión:

* **Port1 (WAN):** Interfaz de salida hacia Internet, configurada por DHCP.
* **Port2 (LAN 1):** Conexión hacia la Sede R1. Dirección IP `200.20.2.1 /24`.
* **Port3 (LAN 2):** Conexión hacia la Sede R2. Dirección IP `200.30.3.1 /24`.

*(Nota: Las credenciales administrativas del FortiGate son `admin` / `admin`)*.

## Enrutamiento Estático

Para que el Firewall conociera cómo alcanzar las redes segmentadas detrás de los routers R1 y R2, implementamos rutas estáticas:
* Para alcanzar las **VLAN 10 (192.168.10.0/24)** y **VLAN 11 (192.168.11.0/24)**, el tráfico se envía a través del Gateway `200.20.2.2` (Router 1).
* Para alcanzar las **VLAN 20 (192.168.20.0/24)** y **VLAN 21 (192.168.21.0/24)**, el tráfico se envía a través del Gateway `200.30.3.2` (Router 2).

## Políticas de Seguridad (Policies) y NAT

El núcleo del firewall se basa en la creación de políticas estrictas que dictan qué tráfico está permitido:

1.  **Acceso a Internet (Internet Access):** Se crearon políticas para permitir que el tráfico originado en LAN 1 y LAN 2 salga hacia la WAN, habilitando **NAT** para enmascarar las direcciones IP privadas.
2.  **Comunicación Inter-Sedes (VLAN-to-VLAN):** Se establecieron reglas de confianza bidireccionales para permitir el tráfico necesario entre las redes locales (LAN 1 <-> LAN 2).
3.  **Publicación de Servicios (Port Forwarding):** Para alojar nuestros propios servicios públicos de manera segura, configuramos *Virtual IPs* en el FortiGate. Esto permite mapear el tráfico entrante de la IP pública (Puerto 80) hacia la IP interna privada de nuestras máquinas servidoras Debian (alojadas en las VLANs 10 y 21).

## Resolución de Incidencias (Troubleshooting)

Durante el despliegue, nos encontramos con varios retos técnicos que solucionamos exitosamente:

* **Bloqueo por Spanning-Tree:** Los switches genéricos de GNS3 enviaban tramas BPDU que causaban bucles y el bloqueo de la VLAN 11. Lo resolvimos sustituyéndolos por **Open vSwitch**, lo que nos dio mayor control sobre STP.
* **Servicios Web:** Los equipos virtuales (VPCS) por defecto no soportan servidores web, por lo que desplegamos máquinas virtuales completas con Debian para alojar los servicios.
* **Enrutamiento Asimétrico:** Al duplicar la topología y añadir la segunda sede, los paquetes del servidor web no sabían cómo volver, causando timeouts. Lo solucionamos añadiendo las rutas estáticas de retorno pertinentes en el FortiGate hacia las nuevas VLANs.

> **Próximos Pasos (Work in Progress):** Actualmente, el equipo está investigando la implementación de una **VPN IPSec / SSL** en el FortiGate. El objetivo es permitir el acceso remoto y seguro de clientes externos a las VLANs internas sin necesidad de exponer más puertos a través de Port Forwarding.