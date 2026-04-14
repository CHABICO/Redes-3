---
title: "Configuración de Routers de Borde (HSRP)"
date: 2026-04-14
summary: "Despliegue de dos routers de borde Cisco c7200 con alta disponibilidad mediante HSRP y bastionado básico de acceso."
author: "Xavier C., Òscar F., Gerard S."
tags: ["HSRP", "Routers", "Alta Disponibilidad", "Seguridad"]
---

## Contexto y Objetivo

En esta fase del proyecto introducimos una capa de redundancia en el perímetro WAN de la red. Hasta este punto, la topología disponía de un único camino de salida hacia Internet a través del FortiGate. Para eliminar ese punto único de fallo, desplegamos dos routers de borde (**Rborde1** y **Rborde2**) configurados con el protocolo **HSRP (Hot Standby Router Protocol)**, de forma que si el router activo cae, el standby asume el rol automáticamente sin intervención manual y sin que los dispositivos internos perciban el cambio.

Esta arquitectura también sienta las bases para la integración con **pfSense**, que actuará como firewall perimetral en la siguiente fase, y que apuntará a la IP virtual HSRP como su gateway WAN.


![zona hsrp](/images/recursos/topologia-HSRP.png)

---

## Topología y Direccionamiento

Los dos routers se conectan al mismo segmento WAN a través de **Switch1** (switch L2 sin configuración adicional), que actúa como troncal entre **Cloud1** y ambos routers. El enlace hacia pfSense comparte el segmento `10.10.0.0/24`.

| Dispositivo | Interfaz | Red      | IP           | Rol                   |
|-------------|----------|----------|--------------|-----------------------|
| Rborde1     | Fa0/0    | WAN      | 200.20.2.3   | Activo (prioridad 110)|
| Rborde1     | Fa1/0    | → pfSense| 10.10.0.3    | Enlace interno        |
| Rborde2     | Fa0/0    | WAN      | 200.20.2.4   | Standby (prioridad 90)|
| Rborde2     | Fa1/0    | → pfSense| 10.10.0.4    | Enlace interno        |
| HSRP VIP    | —        | WAN      | 200.20.2.1   | Gateway virtual       |

*(Nota: las credenciales de acceso son usuario `cisco` con contraseña `cisco`).*

---

## Configuración HSRP

HSRP opera sobre la interfaz WAN (`Fa0/0`) de ambos routers. La IP virtual `200.20.2.1` es la que utilizan los dispositivos aguas abajo como gateway — nunca la IP real de ninguno de los dos routers.

### Rborde1 — router activo

```
interface FastEthernet0/0
 ip address 200.20.2.3 255.255.255.0
 standby version 2
 standby 1 ip 200.20.2.1
 standby 1 priority 110
 standby 1 preempt
 standby 1 track FastEthernet0/0 30
 no shutdown

interface FastEthernet1/0
 ip address 10.10.0.3 255.255.255.0
 no shutdown

ip route 0.0.0.0 0.0.0.0 200.20.2.1
```

### Rborde2 — router standby

```
interface FastEthernet0/0
 ip address 200.20.2.4 255.255.255.0
 standby version 2
 standby 1 ip 200.20.2.1
 standby 1 priority 90
 standby 1 preempt
 no shutdown

interface FastEthernet1/0
 ip address 10.10.0.4 255.255.255.0
 no shutdown

ip route 0.0.0.0 0.0.0.0 200.20.2.1
```

### Verificación

El comando `show standby brief` ejecutado en Rborde1 confirma el correcto funcionamiento:

```
Interface   Grp  Pri P State   Active          Standby         Virtual IP
Fa0/0       1    110 P Active  local           200.20.2.4      200.20.2.1
```

![verificación HSRP](/images/recursos/routers-standby-brief.png)


**Rborde1** aparece como `Active` con prioridad 110 y la `P` de preempt habilitada. **Rborde2** actúa como `Standby` con IP `200.20.2.4`. La VIP `200.20.2.1` es la que recibirá pfSense como gateway.

![zona hsrp](/images/recursos/test-HSRP.gif)

### Mecanismo de failover

El comando `standby 1 track FastEthernet0/0 30` en Rborde1 implementa el seguimiento de interfaz: si la interfaz WAN de Rborde1 cae, su prioridad HSRP se reduce en 30 puntos (de 110 a 80), quedando por debajo de Rborde2 (90). Gracias al `preempt`, Rborde2 asume automáticamente el rol activo sin intervención manual.

---

## Bastionado de Acceso

Aunque se trata de un entorno de laboratorio, aplicamos las medidas básicas de seguridad en ambos routers para simular buenas prácticas reales.

### 1. Control de acceso y gestión

* **Contraseña de enable:** Configurada con `enable secret cisco` (almacenada con hash MD5, más segura que `enable password`).
* **Acceso por consola y VTY:** Protegidos con contraseña y `exec-timeout 10 0` para cerrar sesiones inactivas tras 10 minutos.
* **Cifrado de contraseñas:** Habilitado `service password-encryption` para ofuscar las contraseñas en texto plano del archivo de configuración.

### 2. Reducción de superficie de ataque

* Deshabilitado el servidor HTTP/HTTPS integrado (`no ip http server`).
* Deshabilitado CDP (`no cdp run`) para no exponer información del dispositivo a la red.

### 3. Banner de aviso legal

```
banner motd #
============================================
   Acceso restringido - Solo personal autorizado
   Proyecto Redes 3 - APT-404
============================================
#
```

---

## Notas de implementación

El modelo **Cisco c7200** en GNS3 presenta una limitación conocida con máscaras **/30** en interfaces PA-GE y PA-FE-TX, rechazándolas con el error `Bad mask /30 for address`. La solución adoptada fue utilizar máscara **/24** (`255.255.255.0`) en el segmento de enlace hacia pfSense, lo que no afecta al funcionamiento del laboratorio pero debe tenerse en cuenta si la topología se traslada a equipos físicos o a imágenes IOSv, donde las máscaras /30 funcionan sin restricciones.
