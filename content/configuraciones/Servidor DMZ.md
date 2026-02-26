---
title: "Implementación de DMZ y Servicios con Docker"
date: 2026-02-26
summary: "Diseño de una arquitectura de 3 zonas y despliegue de servicios corporativos (DNS, Web, Correo) mediante contenedores en un servidor Ubuntu alojado en la DMZ."
author: "Xavier C., Òscar F., Gerard S."
---



[Image of DMZ network architecture]


## Arquitectura de 3 Zonas (Inside, Outside, DMZ)

Para dotar a la red de un nivel de seguridad empresarial, evolucionamos la topología integrando una **Zona Desmilitarizada (DMZ)** en el firewall FortiGate. El objetivo de esta zona aislada es alojar los servicios públicos de la empresa (Web, DNS, Mail) de forma que sean accesibles desde el exterior (Outside) sin comprometer la seguridad de las redes internas de los usuarios (Inside).

## Preparación del Servidor Host (Ubuntu)

En lugar de usar los dispositivos virtuales básicos de GNS3, optamos por un entorno real. Desplegamos una máquina virtual con **Ubuntu Server** (obtenida mediante *Osboxes*) y adaptamos las interfaces de red de VMware para integrarla transparentemente dentro de la topología de GNS3.

Para la gestión de los servicios, apostamos por una arquitectura basada en contenedores:
1. Instalación del motor **Docker** y **Docker-Compose**.
2. Despliegue de **Portainer** para disponer de una interfaz gráfica de gestión de contenedores.
   * *Acceso de administración:* Usuario `admin` con contraseña `admin1234567`.

## Despliegue de Servicios Corporativos

Dentro del servidor Ubuntu, levantamos los siguientes servicios para dar soporte a nuestro dominio ficticio corporativo: `novatech.cat`.

### 1. Servidor DNS Autoritativo (BIND9)
Implementamos un contenedor con BIND9 para gestionar la resolución de nombres de nuestra red. 

* **Configuración previa del Host:** Por defecto, las distribuciones modernas de Linux utilizan `systemd-resolved`, el cual ocupa el puerto 53. Para que nuestro contenedor DNS pudiera escuchar peticiones, tuvimos que deshabilitar este servicio local y liberar el puerto 53 en el host.
* **IP Asignada:** El servidor DNS quedó establecido en la dirección `200.40.4.10`.

### 2. Servidor de Correo Electrónico
Implementamos la infraestructura de mensajería electrónica utilizando la imagen `docker-mailserver`, una solución robusta e integrada.

* **Protocolos habilitados:** * **SMTP (Puerto 25):** Para el enrutamiento y envío de correos.
  * **IMAP (Puerto 143):** Para la sincronización y lectura de buzones por parte de los clientes.
* **Persistencia de datos:** Para evitar la pérdida de correos si el contenedor se reinicia, mapeamos volúmenes locales en el host para `maildata` (que almacena el cuerpo de los mensajes) y `mailstate` (que guarda las configuraciones y estados de los usuarios).
* **Cuentas creadas:** Se inicializó el sistema con la cuenta administradora `admin@novatech.cat`.



## Resolución de Incidencias (Troubleshooting)

### El problema del "Falso Servidor Web" (Error de Resolución DNS)
Durante las pruebas de validación de la DMZ, intentamos acceder a la web corporativa local ejecutando `curl novatech.cat` desde la consola. En lugar de nuestra página, la terminal devolvió un error *HTTP 301* asociado a un servidor "OpenResty" que no habíamos configurado.

**Diagnóstico:**
Descubrimos que el comando `curl` estaba utilizando los servidores DNS globales del sistema operativo (configurados por defecto en Netplan para salir a internet), en lugar de consultar a nuestro contenedor BIND9 recién creado. Casualmente, `novatech.cat` es un dominio real registrado en Internet. Nuestra petición salía por la WAN, el DNS público resolvía la IP del propietario real en internet, y nos conectábamos a su servidor en lugar de al nuestro alojado en la DMZ.

**Solución:**
Para validar que nuestro contenedor web funcionaba, usamos primero `curl http://localhost`, lo que devolvió correctamente nuestra web corporativa. Para solucionar el problema a nivel de red, reconfiguramos **Netplan** en el servidor y ajustamos el servicio DHCP de las redes internas (VLANs) para forzar que todos los equipos usaran nuestra IP `200.40.4.10` como DNS primario, priorizando así la resolución local antes de salir a Internet.