---
title: "VPN Site-to-Site y Acceso Remoto"
date: 2026-05-15
summary: "Implementación de VPN IPsec site-to-site entre los dos edificios y planificación de acceso remoto mediante OpenVPN para teletrabajadores."
author: "Xavier C., Òscar F., Gerard S."
tags: ["VPN", "IPsec", "OpenVPN", "Site-to-Site", "Teletrabajo", "Seguridad"]
---

## Contexto y Objetivo

Con la infraestructura de dos edificios operativa, el siguiente paso es interconectarlos de forma segura para que los usuarios del Edificio 2 puedan acceder a los recursos del Edificio 1 (servidores internos, SIEM) sin exponer esos recursos directamente a internet.

Se han planteado dos soluciones de acceso remoto:

- **VPN Site-to-Site IPsec** entre los pfSense de ambos edificios, para conectividad permanente entre sedes.
- **VPN de Acceso Remoto OpenVPN** en pfSense Edificio 1, para teletrabajadores externos.

---

## Topología VPN

```bash
Edificio 1                          Edificio 2
pfSense (192.168.44.139) ══════════ pfSense (192.168.44.136)
        │              Túnel IPsec               │
        │                                        │
  Rborde1/2                              Rborde3/4
        │                                        │
   FortiGate Ed1                       FortiGate Ed2
        │                                        │
   R1 → VLAN10 (192.168.10.0/24)      R2 → VLAN20 (192.168.20.0/24)
        └── VLAN11 (192.168.11.0/24)       └── VLAN21 (192.168.21.0/24)
   Servidores internos (10.40.0.0/24)
```

Las redes del Edificio 2 se renombraron de `192.168.10-11.x` a `192.168.20-21.x` para evitar solapamiento con el Edificio 1 en el túnel VPN.

---

## VPN Site-to-Site IPsec

### Configuración Phase 1

La Phase 1 negocia los parámetros del canal IKE entre ambos pfSense:

| Parámetro | Valor |
|-----------|-------|
| Key Exchange | IKEv2 |
| Remote Gateway Ed1 | `192.168.44.136` (pfSense Ed2) |
| Remote Gateway Ed2 | `192.168.44.139` (pfSense Ed1) |
| Authentication | Pre-Shared Key |
| PSK | `VpnApt404!` |
| Encryption | AES 256 bits |
| Hash | SHA256 |
| DH Group | 14 (2048 bit) |

### Configuración Phase 2

Las Phase 2 definen qué redes se comunican a través del túnel:

| Local (Ed1) | Remote (Ed2) | Descripción |
|-------------|--------------|-------------|
| `192.168.10.0/24` | `192.168.20.0/24` | VLAN10 ↔ VLAN20 |
| `192.168.11.0/24` | `192.168.21.0/24` | VLAN11 ↔ VLAN21 |
| `10.40.0.0/24` | `192.168.20.0/24` | Servidores Ed1 ↔ VLAN20 Ed2 |

### Estado del túnel

El túnel IPsec se ha establecido correctamente entre ambos pfSense, confirmado en `Status → IPsec`:

```bash
con1[1]: ESTABLISHED
192.168.44.139[192.168.44.139]...192.168.44.136[192.168.44.136]
IKEv2, AES_CBC_256/HMAC_SHA2_256_128/PRF_HMAC_SHA2_256/MODP_2048
```

![alt text](/images/recursos/conexión-ipsec.png)

---

## Estado actual y dificultades

A pesar de que el túnel IPsec se establece correctamente (Phase 1 y Phase 2 negociadas), el tráfico entre las redes de ambos edificios no fluye con normalidad. Las investigaciones realizadas han identificado los siguientes problemas:

**NAT sobre tráfico VPN:** pfSense aplicaba NAT al tráfico antes de meterlo en el túnel, traduciendo las IPs origen y rompiendo el matching con las Phase 2. Se resolvió parcialmente añadiendo reglas **No NAT** en el Outbound NAT de ambos pfSense:

```
WAN | 192.168.10.0/24 → 192.168.20.0/24 | No NAT
WAN | 192.168.11.0/24 → 192.168.21.0/24 | No NAT
LAN | 192.168.20.0/24 → 192.168.10.0/24 | No NAT
LAN | 192.168.21.0/24 → 192.168.11.0/24 | No NAT
```

**Auto-exclude LAN bypass:** pfSense tenía activa la opción `Auto-exclude LAN address` en `VPN → IPsec → Advanced Settings`, lo que creaba una regla de bypass que impedía que el tráfico LAN entrara en el túnel. Se desactivó en ambos pfSense.

**Routing de retorno a través de FortiGates:** El tráfico VPN tiene que atravesar dos FortiGates (uno por edificio) y los Rborde de cada edificio. Cada dispositivo intermedio necesita rutas hacia las redes del otro edificio, y las políticas de firewall de cada FortiGate deben permitir el tráfico de paso. Se añadieron rutas en Rborde3/4, FortiGate Ed2 y R2 apuntando a las redes del Edificio 1.

**Resultado:** Los paquetes salen del túnel en pfSense Edificio 1 (`2240 bytes_o`) y llegan al pfSense Edificio 2 (`1512 bytes_i`), pero las respuestas no vuelven correctamente por la complejidad del routing asimétrico a través de múltiples dispositivos intermedios.

En un entorno con menos dispositivos intermedios (pfSense conectado directamente a las redes internas sin FortiGate ni Rborde entre medias), el túnel IPsec funcionaría sin estos problemas de routing.

---

## Rutas necesarias para VPN (configuración aplicada)

### Rborde1/2 Edificio 1
```bash
ip route 192.168.20.0 255.255.255.0 10.10.0.1
ip route 192.168.21.0 255.255.255.0 10.10.0.1
```

### FortiGate Edificio 1
```bash
conf router static
 edit 6
  set dst 192.168.20.0 255.255.255.0
  set gateway 10.20.0.2
  set device "port1"
 next
 edit 7
  set dst 192.168.21.0 255.255.255.0
  set gateway 10.20.0.2
  set device "port1"
 next
end
```

### R1 Edificio 1
```bash
ip route 192.168.20.0 255.255.255.0 10.30.0.1
ip route 192.168.21.0 255.255.255.0 10.30.0.1
```

### Rborde3/4 Edificio 2
```bash
ip route 192.168.10.0 255.255.255.0 10.10.0.1
ip route 192.168.11.0 255.255.255.0 10.10.0.1
ip route 10.40.0.0 255.255.255.0 10.10.0.1
```

### FortiGate Edificio 2
```bash
conf router static
 edit 6
  set dst 192.168.10.0 255.255.255.0
  set gateway 10.20.0.2
  set device "port1"
 next
 edit 7
  set dst 192.168.11.0 255.255.255.0
  set gateway 10.20.0.2
  set device "port1"
 next
 edit 8
  set dst 10.40.0.0 255.255.255.0
  set gateway 10.20.0.2
  set device "port1"
 next
end
```

### R2 Edificio 2
```bash
ip route 192.168.10.0 255.255.255.0 10.30.0.1
ip route 192.168.11.0 255.255.255.0 10.30.0.1
ip route 10.40.0.0 255.255.255.0 10.30.0.1
```

---

## VPN de Acceso Remoto — OpenVPN

La VPN de acceso remoto para teletrabajadores está planificada mediante **OpenVPN** en pfSense Edificio 1, aprovechando que pfSense incluye el servidor OpenVPN de forma nativa sin necesidad de instalar paquetes adicionales.

La configuración teórica sería la siguiente:

**Servidor OpenVPN en pfSense Ed1:**

| Parámetro | Valor |
|-----------|-------|
| Protocol | UDP |
| Port | 1194 |
| Tunnel Network | `10.50.0.0/24` (pool para clientes remotos) |
| Local Network | `192.168.10.0/24, 192.168.11.0/24, 10.40.0.0/24` |
| Authentication | TLS + usuario/contraseña |
| Encryption | AES-256-GCM |

Los teletrabajadores conectados recibirían una IP del rango `10.50.0.0/24` y tendrían acceso a las redes internas del Edificio 1 como si estuvieran físicamente en la oficina.

Esta implementación no se ha completado por limitaciones de tiempo del proyecto. En un escenario real, pfSense incluye un asistente de configuración para OpenVPN que simplifica enormemente el proceso, incluyendo la generación de certificados y la exportación del perfil de cliente.
