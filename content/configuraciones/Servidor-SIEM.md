---
title: "Monitorización — IDS y SIEM"
date: 2026-05-15
summary: "Despliegue de Suricata en modo IDS en los servidores DMZ e Interno, e integración con Wazuh como plataforma SIEM centralizada."
author: "Xavier C., Òscar F., Gerard S."
tags: ["IDS", "SIEM", "Suricata", "Wazuh", "Monitorización", "Seguridad"]
---

## Contexto y Objetivo

En esta fase del proyecto se implementa una capa de monitorización y detección de intrusiones sobre la infraestructura del Edificio 1. El objetivo es disponer de visibilidad sobre el tráfico en las zonas más sensibles de la red — la DMZ y la red de servidores internos — y centralizar las alertas en un SIEM para su análisis.

Se ha optado por implementar únicamente el modo **IDS (Intrusion Detection System)** en lugar de IPS, dado que el entorno es de laboratorio y el modo inline (bloqueo activo) requeriría recursos adicionales y una fase de ajuste de reglas para evitar falsos positivos que interrumpan el tráfico legítimo. Esta decisión se documenta como limitación del entorno y no del diseño.

---

## Arquitectura de Monitorización

La solución se compone de tres elementos:

- **Suricata** como motor de detección, instalado en los servidores de la DMZ y de la red interna.
- **Wazuh Agent** en cada servidor monitorizado, responsable de recoger los logs de Suricata y enviarlos al manager.
- **Wazuh Manager + Indexer + Dashboard** en el servidor SIEM, como plataforma centralizada de correlación y visualización.

```
Ubuntu-Server-DMZ-1 (172.16.0.20)
  └── Suricata IDS → eve.json → Wazuh Agent → Wazuh Manager (10.40.0.20)

Ubuntu-Server-Interno-1 (10.40.0.10)
  └── Suricata IDS → eve.json → Wazuh Agent → Wazuh Manager (10.40.0.20)

Wazuh Manager + Indexer + Dashboard (10.40.0.20)
  └── Dashboard accesible via https://192.168.44.136:7777
```

![interfaz wazuh](/images/recursos/gui-wazuh.png)

---

## Suricata IDS


### Instalación

Suricata se ha instalado en ambos servidores desde los repositorios oficiales de Ubuntu:

```bash
sudo apt update
sudo apt install -y suricata
sudo suricata-update
```

### Configuración

En cada servidor se ha editado `/etc/suricata/suricata.yaml` para ajustar dos parámetros principales:

La interfaz de red a monitorizar:

```yaml
af-packet:
  - interface: ens33   # nombre de la interfaz real de cada servidor
```

Y la red local a proteger:

```yaml
# Servidor DMZ
HOME_NET: "[172.16.0.0/24]"

# Servidor Interno
HOME_NET: "[10.40.0.0/24]"
```

### Reglas personalizadas

Se han añadido reglas propias en `/etc/suricata/rules/local.rules` para demostrar la capacidad de detección personalizada:

```
alert icmp any any -> $HOME_NET any (msg:"ICMP Ping detectado"; sid:1000001; rev:1;)
alert tcp any any -> $HOME_NET 80 (msg:"Acceso HTTP detectado"; sid:1000002; rev:1;)
```

![local.rules suricata](/images/recursos/local.rules-suricata.png)

### Verificación


La detección se ha verificado accediendo a `http://testmyids.com` desde un cliente de la red, lo que genera una alerta inmediata en `/var/log/suricata/fast.log`:

![verificación suricata](/images/recursos/verificacion-suricata.png)

### Limitaciones

La transición a modo IPS (bloqueo activo mediante modo inline) no se ha implementado por limitaciones del entorno de laboratorio. En un entorno productivo, una vez validadas las reglas de detección, se activaría el modo inline en Suricata para bloquear tráfico malicioso de forma activa.

---

## Wazuh SIEM

### Instalación

Wazuh se ha instalado en el servidor SIEM (`10.40.0.20`) mediante el instalador oficial todo-en-uno, que despliega automáticamente el manager, el indexer (basado en OpenSearch) y el dashboard:

```bash
curl -sO https://packages.wazuh.com/4.7/wazuh-install.sh
sudo bash wazuh-install.sh -a -i
```

El flag `-i` se ha usado para omitir la verificación de versión del sistema operativo, ya que el servidor ejecuta Ubuntu 24.04 y el instalador oficial soporta hasta 22.04. La instalación ha funcionado correctamente.

Tras la instalación se han ampliado los recursos de disco del servidor expandiendo el volumen lógico LVM:

```bash
sudo lvextend -l +100%FREE /dev/mapper/ubuntu--vg-ubuntu--lv
sudo resize2fs /dev/mapper/ubuntu--vg-ubuntu--lv
```

### Servicios activos


```bash
sudo systemctl status wazuh-manager    # active (running)
sudo systemctl status wazuh-indexer    # active (running)
sudo systemctl status wazuh-dashboard  # active (running)
```

### Agentes desplegados

Se han instalado agentes Wazuh en ambos servidores monitorizados:

| Agente | IP | Estado |
|--------|----|--------|
| IDS-DMZ | 172.16.0.20 | Active |
| IDS-Interno | 10.40.0.10 | Active |

Instalación del agente:

```bash
sudo WAZUH_MANAGER='10.40.0.20' WAZUH_AGENT_NAME='IDS-DMZ' dpkg -i wazuh-agent_4.7.5-1_amd64.deb
sudo systemctl enable wazuh-agent
sudo systemctl start wazuh-agent
```
![agentes wazuh](/images/recursos/siem-agentes-wazuh.png)

### Integración con Suricata

En cada servidor monitorizado se ha configurado el agente Wazuh para leer los logs de Suricata en formato JSON y enviarlos al manager. Se ha añadido el siguiente bloque en `/var/ossec/etc/ossec.conf`:

```xml
<localfile>
  <log_format>json</log_format>
  <location>/var/log/suricata/eve.json</location>
</localfile>
```

Con esta configuración, las alertas generadas por Suricata aparecen automáticamente en el dashboard de Wazuh bajo `Security Events`.

![alerta de suricata en wazuh](/images/recursos/alerta-suricata-en-wazuh.png)


### Acceso al dashboard

El dashboard es accesible desde la red externa mediante port forwarding configurado en pfSense:

```
https://192.168.44.136:7777 → 10.40.0.20:443
```

### Consideraciones de seguridad

La comunicación entre los agentes y el manager utiliza los puertos `1514` (TCP/UDP) y `1515` (TCP). Se han configurado las políticas de firewall necesarias en pfSense y FortiGate para permitir únicamente este tráfico desde la DMZ hacia el SIEM, sin dar acceso general de la DMZ a la red de servidores internos, evitando así posibles movimientos laterales en caso de compromiso del servidor DMZ.

Los logs de pfSense y FortiGate no se han integrado en Wazuh por limitaciones de tiempo del proyecto. En un entorno productivo, pfSense enviaría logs via syslog al manager de Wazuh, y FortiGate haría lo propio mediante su funcionalidad de log remoto.