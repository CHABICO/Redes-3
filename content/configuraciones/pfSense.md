---
title: "Configuración de Firewall pfSense"
date: 2026-04-14
summary: "Implementación de pfSense como gateway de seguridad perimetral, segmentación de zonas (WAN, LAN, DMZ, MGMT), reglas de firewall y publicación de servicios hacia el exterior mediante NAT."
author: "Xavier C., Òscar F., Gerard S."
tags: ["pfSense", "Firewall", "DMZ", "NAT", "Segmentación", "Seguridad"]
---

## Contexto y Objetivo

Tras estabilizar la alta disponibilidad en el borde con los routers Cisco, el siguiente paso es la integración del firewall **pfSense**. Este dispositivo actúa como primera línea de defensa interna, encargándose de la inspección de paquetes, el filtrado de tráfico entre zonas y la publicación de servicios hacia el exterior mediante NAT.

pfSense se conecta al clúster HSRP a través del segmento `10.10.0.0/24`, recibiendo el tráfico externo a través de la VIP `10.10.0.1`. Internamente, segmenta la infraestructura en cuatro zonas diferenciadas para aplicar políticas de seguridad granulares.

![topología GNS3 - pfSense](/images/recursos/topología-pfsense.png)

---

## Direccionamiento de Interfaces

| Interfaz | Etiqueta pfSense | IP / Máscara | Notas |
|----------|------------------|--------------|-------|
| **em0** | WAN | `10.10.0.10/24` | Gateway: `10.10.0.1` (VIP HSRP interna) |
| **em1** | LAN | `10.20.0.1/24` | Hacia FortiGate port1 |
| **em5** | DMZ | `172.16.0.1/24` | Zona servidores públicos |
| **em2** | ADMIN | `DHCP` | Gestión CloudNAT |

![cuatro interfaces con sus IPs y estado up](/images/recursos/interfaces-pfsense.png)

---

## Enrutamiento

El enrutamiento estático se configuró en **System → Routing** para que pfSense conozca las redes internas que viven detrás del FortiGate:

| Red destino | Gateway | Descripción |
|-------------|---------|-------------|
| `192.168.10.0/24` | `10.20.0.2` | VLAN10 vía FortiGate |
| `192.168.11.0/24` | `10.20.0.2` | VLAN11 vía FortiGate |
| `10.30.0.0/24` | `10.20.0.2` | Red VLANs (IOU1/R1) vía FortiGate |
| `10.40.0.0/24` | `10.20.0.2` | Servidores internos vía FortiGate |

El gateway `10.20.0.2` corresponde a **port1 del FortiGate**, que actúa como siguiente salto para todas las redes internas.

---

## Reglas de Firewall

### Interfaz WAN

La interfaz WAN tiene desactivadas las opciones **Block private networks** y **Block bogon networks**, necesario porque el segmento `10.10.0.0/24` es una red privada RFC1918 y pfSense la bloquearía por defecto.

| Acción | Protocolo | Origen | Destino | Puerto | Descripción |
|--------|-----------|--------|---------|--------|-------------|
| Pass | TCP | any | WAN address | 80 | Acceso web DMZ |
| Pass | TCP | any | WAN address | 8443 | Acceso WebGUI (temporal) |

![reglas WAN](/images/recursos/reglas-wan-pfsense.png)

### Interfaz DMZ

| Acción | Protocolo | Origen | Destino | Puerto | Descripción |
|--------|-----------|--------|---------|--------|-------------|
| Block | any | `172.16.0.0/24` | `10.20.0.0/24` | any | DMZ no accede a LAN |
| Pass | TCP | `172.16.0.0/24` | any | 80, 443 | DMZ sale a Internet |
| Block | any | `172.16.0.0/24` | any | any | Denegar resto |

### Interfaz ADMIN

Acceso total desde la red de gestión para permitir administración vía WebGUI:

| Acción | Protocolo | Origen | Destino | Descripción |
|--------|-----------|--------|---------|-------------|
| Pass | any | `10.30.0.0/24` | any | Allow gestión |

---

## NAT — Port Forwarding hacia DMZ

Para exponer el servidor web de la DMZ hacia el exterior, se configuró una regla de **Port Forward** en **Firewall → NAT → Port Forward**:

| Campo | Valor |
|-------|-------|
| Interface | WAN |
| Protocol | TCP |
| Dest. Address | WAN address (`10.10.0.10`) |
| Dest. Port | 80 |
| NAT IP | `172.16.0.20` |
| NAT Port | 80 |

El flujo completo del tráfico entrante es el siguiente:

```
PC externo → 192.168.44.201 (VIP HSRP externa)
  → Rborde1 reenvía a 10.10.0.10 (pfSense WAN)
    → Port Forward redirige a 172.16.0.20 (servidor DMZ)
      → Apache responde
```

![regla HTTP hacia DMZ](/images/recursos/portforwarding-pfsense.png)

### NAT Outbound

Configurado en modo **Manual** con dos reglas para permitir salida a Internet:

| Interface | Source | NAT Address | Descripción |
|-----------|--------|-------------|-------------|
| WAN | `172.16.0.0/24` | WAN address | Salida DMZ |
| WAN | `10.20.0.0/24` | WAN address | Salida LAN |

---

## Verificación

La verificación del correcto funcionamiento se realizó accediendo desde un navegador externo a `http://192.168.44.201`, obteniendo la página de bienvenida del servidor Apache en la DMZ.

![navegador mostrando la página web del servidor DMZ accedida desde fuera via 192.168.44.201](/images/recursos/web-dmz.png)

Desde la consola de pfSense se verificó también mediante `tcpdump` que el tráfico entrante en `em0` era correctamente redirigido hacia `em5` (DMZ):

```bash
tcpdump -i em0 tcp port 80
tcpdump -i em5 tcp port 80
```

---

## Incidencias durante la implementación

Durante la configuración se encontraron las siguientes incidencias relevantes documentadas para la memoria:

**Acceso al WebGUI bloqueado:** pfSense por defecto no permite acceso al WebGUI desde interfaces que no sean LAN. Se resolvió habilitando SSH desde la consola y ejecutando `pfctl -d` para desactivar temporalmente el firewall y poder añadir la regla de gestión desde el GUI.

**Block private networks:** La regla estaba bloqueando el tráfico legítimo en la interfaz WAN ya que el segmento `10.10.0.0/24` es RFC1918. Se desactivó explícitamente en la configuración de la interfaz WAN.

**Gateway toWAN sendto error 64:** Error de monitorización del gateway causado por un problema temporal de conectividad durante la configuración. Se resolvió tras reiniciar pfSense una vez completada la configuración de interfaces y rutas.