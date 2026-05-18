---
title: "Configuración de DMZ"
date: 2026-02-26
summary: "Diseño de una arquitectura de 3 zonas y despliegue de servicios corporativos (DNS, Web, Correo) mediante contenedores en un servidor Ubuntu alojado en la DMZ."
tags: ["DMZ", "Ubuntu", "Apache", "IDS", "Suricata", "Seguridad"]
---


# Configuración VMware

<details>
<summary>Configuración VMware - Adaptadores de Red</summary>

![conf server vmware](/images/recursos/ubuntu-server-vmware.png)

</details>

<details>
<summary>Panel de Control - Configuración de Red</summary>

![Network Connections](/images/recursos/network-connections.png)

</details>

<br>

## Contexto y Objetivo

La DMZ (Zona Desmilitarizada) es una zona de red aislada que permite alojar servicios accesibles desde el exterior sin comprometer la seguridad de las redes internas. En esta infraestructura, la DMZ está conectada directamente a pfSense mediante la interfaz `em5`, con la red `172.16.0.0/24`.

El servidor Ubuntu-Server-DMZ-1 (`172.16.0.20`) aloja el servicio web corporativo y ejecuta Suricata en modo IDS para monitorizar el tráfico de la zona.

```
Internet → pfSense WAN (port 80)
    → Port Forward → 172.16.0.20:80 (servidor DMZ)
        → Apache responde
```

---

## Configuración de Red

El servidor tiene configuración estática en `/etc/netplan/`:

```yaml
network:
  version: 2
  ethernets:
    ens33:
      addresses:
        - 172.16.0.20/24
      routes:
        - to: default
          via: 172.16.0.1
      nameservers:
        addresses: [8.8.8.8]
```

El gateway `172.16.0.1` corresponde a la interfaz DMZ de pfSense.

---

## Servicio Web

El servidor web se basa en **Apache2**, instalado directamente en el sistema operativo:

```bash
sudo apt update
sudo apt install -y apache2
sudo systemctl enable apache2
sudo systemctl start apache2
```

La página web es accesible desde el exterior mediante el port forwarding configurado en pfSense:

```
http://192.168.44.139:80 → 172.16.0.20:80
```

![navegador accediendo a la web del servidor DMZ desde fuera](/images/recursos/web-dmz.png)


---

## Suricata IDS
Suricata está instalado y configurado en modo IDS para monitorizar el tráfico que entra y sale de la DMZ.

### Instalación

```bash
sudo apt install -y suricata
sudo suricata-update
```

### Configuración

En `/etc/suricata/suricata.yaml`:

```yaml
af-packet:
  - interface: ens33

vars:
  address-groups:
    HOME_NET: "[172.16.0.0/24]"
```

### Reglas personalizadas

En `/etc/suricata/rules/local.rules`:

```
alert icmp any any -> $HOME_NET any (msg:"ICMP Ping detectado"; sid:1000001; rev:1;)
alert tcp any any -> $HOME_NET 80 (msg:"Acceso HTTP detectado"; sid:1000002; rev:1;)
```

---

## Integración con Wazuh

El agente Wazuh está instalado y conectado al manager SIEM (`10.40.0.20`) para centralizar las alertas de Suricata:

```bash
sudo WAZUH_MANAGER='10.40.0.20' WAZUH_AGENT_NAME='IDS-DMZ' dpkg -i wazuh-agent_4.7.5-1_amd64.deb
sudo systemctl enable wazuh-agent
sudo systemctl start wazuh-agent
```

En `/var/ossec/etc/ossec.conf` se ha añadido la integración con los logs de Suricata:

```xml
<localfile>
  <log_format>json</log_format>
  <location>/var/log/suricata/eve.json</location>
</localfile>
```

---

## Seguridad de la zona DMZ

pfSense aplica las siguientes reglas para aislar la DMZ:

* La DMZ **no puede acceder a la red LAN** (`10.10.0.0/24`), evitando movimiento lateral si el servidor es comprometido.
* La DMZ **solo puede comunicarse con el SIEM** (`10.40.0.20`) por los puertos 1514 y 1515, para enviar alertas de Suricata.
* La DMZ **sí puede salir a internet** para actualizaciones del sistema operativo y descarga de reglas de Suricata.

---

## Notas de implementación

**Conectividad para instalación de paquetes:** Durante la instalación, el servidor DMZ tuvo problemas de conectividad por la topología de laboratorio. Se resolvió temporalmente conectando un adaptador NAT de VMware para la descarga de paquetes, y reconectando a la topología una vez completada la instalación.
