---
title: "Configuración de Routers de Borde (HSRP)"
date: 2026-04-14
summary: "Despliegue de dos routers de borde Cisco c7200 con alta disponibilidad mediante HSRP y bastionado básico de acceso."
author: "Xavier C., Òscar F., Gerard S."
tags: ["HSRP", "Routers", "Alta Disponibilidad", "OSPF", "Seguridad"]
---

## Contexto y Objetivo

En esta fase del proyecto introducimos una capa de redundancia en el perímetro de la red. Para eliminar el punto único de fallo, desplegamos dos routers de borde (**Rborde1** y **Rborde2**) configurados con el protocolo **HSRP (Hot Standby Router Protocol)**, de forma que si el router activo cae, el standby asume el rol automáticamente sin intervención manual y sin que los dispositivos internos perciban el cambio.

Los routers de borde actúan como intermediarios entre **pfSense** (firewall perimetral, aguas arriba) y el **FortiGate** (firewall interno, aguas abajo), gestionando dos grupos HSRP independientes: uno hacia pfSense y otro hacia FortiGate.

![zona hsrp](/images/recursos/topologia-HSRP.png)

---

## Topología y Direccionamiento

Los routers tienen dos interfaces activas cada uno: `Fa0/0` hacia pfSense y `Fa1/0` hacia FortiGate. Se han configurado dos grupos HSRP, uno por segmento, para garantizar redundancia en ambos sentidos.

| Dispositivo | Interfaz | Red | IP | Rol |
|-------------|----------|-----|----|-----|
| Rborde1 | Fa0/0 | → pfSense | `10.10.0.3/24` | Activo (prioridad 110) |
| Rborde2 | Fa0/0 | → pfSense | `10.10.0.4/24` | Standby (prioridad 90) |
| HSRP VIP grupo 1 | — | `10.10.0.0/24` | `10.10.0.2` | Gateway virtual hacia pfSense |
| Rborde1 | Fa1/0 | → FortiGate | `10.20.0.3/24` | Activo (prioridad 110) |
| Rborde2 | Fa1/0 | → FortiGate | `10.20.0.4/24` | Standby (prioridad 90) |
| HSRP VIP grupo 2 | — | `10.20.0.0/24` | `10.20.0.2` | Gateway virtual hacia FortiGate |

*(Nota: las credenciales de acceso son usuario `cisco` con contraseña `cisco`).*

---

## Configuración HSRP

### Rborde1 — router activo

```bash
interface FastEthernet0/0
 ip address 10.10.0.3 255.255.255.0
 standby version 2
 standby 1 ip 10.10.0.2
 standby 1 priority 110
 standby 1 preempt
 standby 1 track FastEthernet0/0 30
 standby 1 track FastEthernet1/0 30
 no shutdown

interface FastEthernet1/0
 ip address 10.20.0.3 255.255.255.0
 standby version 2
 standby 2 ip 10.20.0.2
 standby 2 priority 110
 standby 2 preempt
 standby 2 track FastEthernet1/0 30
 no shutdown

ip route 0.0.0.0 0.0.0.0 10.10.0.1
ip route 192.168.10.0 255.255.255.0 10.20.0.1
ip route 192.168.11.0 255.255.255.0 10.20.0.1
ip route 10.30.0.0 255.255.255.0 10.20.0.1
ip route 10.40.0.0 255.255.255.0 10.20.0.1
ip route 192.168.20.0 255.255.255.0 10.10.0.1
ip route 192.168.21.0 255.255.255.0 10.10.0.1
```

### Rborde2 — router standby

```bash
interface FastEthernet0/0
 ip address 10.10.0.4 255.255.255.0
 standby version 2
 standby 1 ip 10.10.0.2
 standby 1 priority 90
 standby 1 preempt
 no shutdown

interface FastEthernet1/0
 ip address 10.20.0.4 255.255.255.0
 standby version 2
 standby 2 ip 10.20.0.2
 standby 2 priority 90
 standby 2 preempt
 no shutdown

ip route 0.0.0.0 0.0.0.0 10.10.0.1
ip route 192.168.10.0 255.255.255.0 10.20.0.1
ip route 192.168.11.0 255.255.255.0 10.20.0.1
ip route 10.30.0.0 255.255.255.0 10.20.0.1
ip route 10.40.0.0 255.255.255.0 10.20.0.1
ip route 192.168.20.0 255.255.255.0 10.10.0.1
ip route 192.168.21.0 255.255.255.0 10.10.0.1
```

### Verificación

El comando `show standby brief` ejecutado en Rborde1 confirma el correcto funcionamiento de ambos grupos HSRP:

```
Interface   Grp  Pri P State   Active          Standby         Virtual IP
Fa0/0       1    110 P Active  local           10.10.0.4       10.10.0.2
Fa1/0       2    110 P Active  local           10.20.0.4       10.20.0.2
```

![verificación HSRP](/images/recursos/routers-standby-brief.png)

**Rborde1** aparece como `Active` en ambos grupos con prioridad 110. **Rborde2** actúa como `Standby`. Las VIPs `10.10.0.2` y `10.20.0.2` son las que utilizan pfSense y FortiGate respectivamente como gateway.

![test HSRP](/images/recursos/test-HSRP.png)

---

### Mecanismo de failover

El comando `standby 1 track FastEthernet0/0 30` implementa el seguimiento de interfaz: si una interfaz cae, la prioridad HSRP se reduce en 30 puntos (de 110 a 80), quedando por debajo de Rborde2 (90). Gracias al `preempt`, Rborde2 asume automáticamente el rol activo sin intervención manual.

---

## Enrutamiento Dinámico — OSPF

Como complemento a la redundancia proporcionada por HSRP, se ha configurado OSPF entre Rborde1 y Rborde2 para el intercambio dinámico de rutas en el área 0.

### Configuración OSPF

Ambos routers anuncian los segmentos `10.10.0.0/24` y `10.20.0.0/24` dentro del área backbone (área 0), con `default-information originate` para propagar la ruta por defecto.

| Dispositivo | Router-ID | Redes anunciadas |
|-------------|-----------|-----------------|
| Rborde1 | 1.1.1.1 | 10.10.0.0/24, 10.20.0.0/24 |
| Rborde2 | 2.2.2.2 | 10.10.0.0/24, 10.20.0.0/24 |

```
router ospf 1
 router-id 1.1.1.1
 log-adjacency-changes
 network 10.10.0.0 0.0.0.255 area 0
 network 10.20.0.0 0.0.0.255 area 0
 default-information originate
```

### Autenticación OSPF

El enunciado requería autenticación SHA-256 mediante key chains. Sin embargo, los routers Cisco c7200 en GNS3 ejecutan IOS 12.4, versión que no soporta el comando `cryptographic-algorithm hmac-sha-256`. Como alternativa, se ha implementado autenticación MD5 mediante `message-digest`, que proporciona verificación criptográfica de los paquetes OSPF e impide la inyección de rutas falsas.

```bash
router ospf 1
 area 0 authentication message-digest

interface FastEthernet0/0
 ip ospf message-digest-key 1 md5 Cisco123
 ip ospf authentication message-digest

interface FastEthernet1/0
 ip ospf message-digest-key 1 md5 Cisco123
 ip ospf authentication message-digest
```

### Alcance y limitaciones

OSPF se ha limitado a Rborde1 y Rborde2 porque pfSense no permite la instalación del paquete FRR en el entorno de laboratorio. Sin continuidad OSPF más allá de los Rborde, extender el dominio al FortiGate o R1 no es posible.

`passive-interface` no se ha aplicado porque todas las interfaces activas de ambos routers participan en OSPF con vecinos activos. La segmentación en áreas tampoco se ha implementado: con únicamente dos routers en el dominio, dividir en áreas no aporta ningún valor operativo.

---

## BGP con el ISP

El enunciado contempla la configuración de BGP entre los routers de borde y el ISP simulado (Cloud1). Sin embargo, Cloud1 en el entorno de laboratorio es un nodo Cloud/NAT de GNS3 que actúa únicamente como puente hacia la red del host y no dispone de CLI ni proceso de routing. Al no existir un peer BGP real, la implementación no es posible en este entorno.

Actualmente la salida a internet se gestiona mediante una ruta estática por defecto apuntando a pfSense:

```
ip route 0.0.0.0 0.0.0.0 10.10.0.1
```

Si el entorno dispusiera de un router ISP simulado (AS 100), la configuración eBGP sería la siguiente (AS 200 para los routers de borde):

```bash
router bgp 200
 bgp router-id 1.1.1.1
 neighbor 10.10.0.1 remote-as 100
 network 10.10.0.0 mask 255.255.255.0
 network 10.20.0.0 mask 255.255.255.0
```

---

## Bastionado de Acceso

### 1. Control de acceso y gestión

* **Contraseña de enable:** Configurada con `enable secret cisco` (almacenada con hash MD5).
* **Acceso por consola y VTY:** Protegidos con contraseña y `exec-timeout 10 0` para cerrar sesiones inactivas.
* **Cifrado de contraseñas:** Habilitado `service password-encryption`.

### 2. Reducción de superficie de ataque

* Deshabilitado el servidor HTTP/HTTPS integrado (`no ip http server`).
* Deshabilitado CDP (`no cdp run`).

### 3. Banner de aviso legal

```bash
banner motd #
============================================
   Acceso restringido - Solo personal autorizado
   Proyecto Redes 3 - APT-404
============================================
#
```

---

## Notas de implementación

El modelo **Cisco c7200** en GNS3 presenta una limitación conocida con máscaras **/30**, rechazándolas con el error `Bad mask /30 for address`. La solución adoptada fue utilizar máscara **/24** en todos los segmentos de enlace.

STP no se ha implementado porque toda la topología opera en capa 3, con IPs en cada enlace. La redundancia está gestionada por HSRP. En ausencia de enlaces trunk L2 entre switches, no existen loops de capa 2 que STP deba resolver.

El NAT que originalmente estaba configurado en los Rborde se eliminó al reestructurar la topología, ya que pfSense es ahora el dispositivo responsable de hacer NAT hacia internet. Mantener NAT en los Rborde causaba routing asimétrico y rompía la conectividad entre las redes internas e internet.