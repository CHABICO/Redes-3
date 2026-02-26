---
title: "Configuración de Routers, Switches y VLANs"
date: 2026-02-26
summary: "Evolución de la topología de red: desde el aislamiento básico mediante VLANs hasta el enrutamiento inter-sedes y la fortificación de la capa 2."
author: "Xavier C., Òscar F., Gerard S."
---

## Contexto y Evolución de la Topología

El proyecto comenzó con el objetivo de configurar una red segura básica estableciendo conectividad entre dispositivos utilizando un router y un switch de Cisco. Inicialmente, el tráfico se segmentó en dos redes virtuales (VLAN 10 y VLAN 11). 

Posteriormente, la topología evolucionó hacia un entorno multi-sede. Desplegamos dos enrutadores principales (R1 y R2) que actúan como puertas de enlace para distintas VLANs, separados e interconectados mediante un Firewall central.

## Direccionamiento y Enrutamiento (Routers R1 y R2)

Para garantizar la correcta segmentación, asignamos un esquema de direccionamiento IPv4 para cada interfaz y subinterfaz de los routers:

* **Router 1 (Sede A):**
    * `f0/0` (Hacia el Firewall): 200.20.2.2 /24
    * `f0/1` (VLAN 10 - Red Interna 1): 192.168.10.1 /24
    * `f1/0` (VLAN 11 - Red Interna 2): 192.168.11.1 /24
* **Router 2 (Sede B):**
    * `f0/0` (Hacia el Firewall): 200.30.3.2 /24
    * `f0/1` (VLAN 20 - Red Interna 3): 192.168.20.1 /24
    * `f1/0` (VLAN 21 - Red Interna 4): 192.168.21.1 /24

*(Nota: Las credenciales de acceso para R1 y R2 son `r1/r2` con la contraseña `cisco`)*.

## Fortificación de Capa 2 (Switches)

La seguridad en la capa de acceso es vital. Por ello, aplicamos un bastionado ('hardening') a los switches siguiendo buenas prácticas de la industria:

### 1. Control de Acceso y Gestión
* **Gestión Segura:** Accesos administrativos restringidos mediante SSH (cifrado) y configuración de `exec-timeout` para cerrar sesiones inactivas.
* **Avisos Legales:** Implementación de un `banner motd` para advertir sobre el acceso no autorizado.
* **Protección de Credenciales:** Habilitado `service password-encryption` para ofuscar contraseñas en los archivos de configuración.

### 2. Seguridad en Puertos (Port Security)
* **Límite de MACs:** Restringimos el número de direcciones MAC permitidas a 2 por puerto.
* **Modo de Violación:** Configurado en `Restrict`, de forma que si se detecta una intrusión, se descarta el tráfico no autorizado y se genera un log, pero el puerto no se apaga completamente.
* **Aprendizaje:** Uso de `mac-address sticky` para el aprendizaje dinámico y retención de las direcciones legítimas.

### 3. Mitigación de Ataques de Spoofing
Para evitar ataques Man-in-the-Middle donde un atacante falsifica su identidad en la red, implementamos:
* **DHCP Snooping:** Valida los mensajes DHCP para evitar que servidores no autorizados entreguen configuraciones falsas.
* **Dynamic ARP Inspection (DAI):** Inspecciona los paquetes ARP cruzándolos con bases de datos de confianza para descartar correspondencias MAC/IP inválidas.
* **IP Source Guard:** Vincula la IP, la MAC y el puerto, bloqueando tráfico de hosts con IPs no asignadas legítimamente.

### 4. Estabilidad y Prevención de Bucles
* Habilitamos `spanning-tree portfast` para acelerar la convergencia en puertos de acceso y `bpduguard` para prevenir la inyección maliciosa de tramas BPDU que alteren la topología STP.
* **Gestión de Puertos Libres:** Todos los puertos no utilizados se configuraron en modo `shutdown` y se asignaron a una "Blackhole VLAN" (una VLAN sin tráfico), desactivando además el protocolo DTP para evitar negociaciones troncales indeseadas.