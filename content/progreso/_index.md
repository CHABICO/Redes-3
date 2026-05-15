---
title: "Estado del Proyecto"
date: 2026-04-29
summary: "Checklist de requisitos del proyecto cruzado con el estado actual de implementación."
author: "Xavier C., Òscar F., Gerard S."
tags: ["Checklist", "Estado", "Requisitos"]
---

## 1. Diseño jerárquico y escalable

- [x] Topología de tres capas (core, distribución, acceso)
- [x] Justificación del hardware virtualizado (GNS3 + imágenes Cisco c7200, pfSense, FortiGate)
- [x] VLANs (VLAN10, VLAN11)
- [x] Port Security en switches de acceso
- [x] DHCP Snooping
- [x] Dynamic ARP Inspection (DAI)
- [x] HSRP — redundancia de gateway (Rborde1 activo, Rborde2 standby, VIP 192.168.44.201 / 10.10.0.1)
- [x] STP / Spanning Tree Protocol (no hace falta incluir)

---

## 2. Hardening de OSPF

- [x] OSPF básico configurado (Rborde1, Rborde2, pfSense con FRR)
- [x] Autenticación OSPF con SHA-256 (md5 por versión de routers)
- [x] `passive-interface` en interfaces que no forman adyacencia
- [x] Segmentación en áreas OSPF para control de tráfico LSA

---

## 3. BGP con el ISP

- [x] BGP configurado entre routers de borde y Cloud1 (ISP simulado)

---

## 4. Protección perimetral y DMZ

- [x] DMZ creada (172.16.0.0/24) con servidor web Apache (172.16.0.20)
- [x] pfSense como Stateful Firewall perimetral
- [x] Acceso externo a DMZ mediante Port Forward (192.168.44.201:80 → 172.16.0.20)
- [x] Regla que bloquea DMZ → red interna
- [x] Red interna puede iniciar conexiones hacia DMZ
- [x] FortiGate como firewall interno (segundo nivel de seguridad) — *en progreso*
- [ ] Inspección de contenido / filtrado de aplicación en firewall (detección SQLi sobre HTTP)

> ⚠️ La inspección de contenido (SQLi, HTTP inspection) se puede implementar en pfSense con el paquete **Snort/Suricata** o en FortiGate con **Application Control + IPS**.

---

## 5. Hardening de dispositivos

- [x] `no ip http server` en routers
- [x] `no cdp run` en routers
- [x] `no ip source-route` — *pendiente de verificar en todos los dispositivos*
- [x] `no service finger` — *pendiente de verificar en todos los dispositivos*
- [x] Contraseñas con `enable secret` (hash MD5 — SHA-256 requiere IOS 15.3+)
- [x] Acceso por consola y VTY con contraseña
- [x] SSHv2 habilitado, Telnet deshabilitado
- [x] `exec-timeout` configurado
- [x] `service password-encryption`
- [x] Banner MOTD
- [ ] Verificar hardening completo en R1, R2, IOU1 y switches

---

## 6. Monitorización avanzada — IDS/IPS

- [x] Suricata instalado en servidor DMZ (172.16.0.20) — modo IDS
- [x] Suricata instalado en servidor interno (10.40.0.10) — modo IDS
- [x] Reglas de detección activas y verificadas

---

## 7. SIEM

- [x] Wazuh instalado en servidor SIEM (10.40.0.20)
- [x] Logs de pfSense integrados en Wazuh
- [ ] Logs de FortiGate integrados en Wazuh
- [x] Alertas de Suricata integradas en Wazuh
- [ ] Dashboard operativo con eventos de seguridad

---

## 8. Control de acceso capa 2 — 802.1X / RADIUS

- [ ] Servidor RADIUS configurado
- [ ] Autenticación 802.1X en switches de acceso

> ⚠️ Requisito marcado como "dentro de lo posible". Se puede implementar FreeRADIUS en un servidor Linux de la red interna.

---

## 9. Ingeniería de tráfico — MPLS / VRF

- [ ] VPN MPLS configurada
- [ ] Aislamiento mediante VRF

> ⚠️ Requisito avanzado. Requiere IOS con soporte MPLS (c7200 lo soporta). Valorar si entra en el alcance del proyecto.

---

## 10. VPN y cifrado

- [ ] Site-to-Site VPN IPsec (entre dos sedes)
- [ ] Remote Access VPN TLS/SSL (teletrabajadores) — pfSense OpenVPN

---

## 11. SD-WAN

- [ ] Implementación de SD-WAN

> ⚠️ Requisito avanzado. Se puede simular con pfSense + múltiples interfaces WAN y políticas de enrutamiento por aplicación.

---

## 12. Automatización

- [ ] Script Python de hardening automático para nuevos routers
- [ ] Script Python de verificación periódica de configuración de seguridad
- [ ] Integración: alerta IDS → script Python → modificación ACL en switch

---

## 13. Escenarios de ataque documentados

- [ ] **ARP Spoofing** — simulación y mitigación con DHCP Snooping + DAI
- [ ] **SYN Flood / ICMP Flood** — ataque a servidor DMZ, monitorización CPU, mitigación con Rate Limiting en pfSense
- [ ] **Inyección de ruta falsa en OSPF** — verificación que la autenticación SHA-256 bloquea el ataque

---

## Resumen de prioridades

| Prioridad | Tarea | Complejidad |
|-----------|-------|-------------|
| 🟡 Media | Script Python hardening/verificación | Media |
| 🟡 Media | Escenarios de ataque documentados | Media |
| 🟡 Media | Remote Access VPN (OpenVPN en pfSense) | Media |
| 🟢 Opcional | 802.1X / RADIUS | Alta |
| 🟢 Opcional | MPLS / VRF | Alta |
| 🟢 Opcional | SD-WAN | Alta |
| 🟢 Opcional | Site-to-Site VPN IPsec | Alta |