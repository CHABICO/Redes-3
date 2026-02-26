---
title: Proyecto Redes
description: 
---

## Contexto

"NovaTech Solutions" es una empresa tecnológica en pleno crecimiento dedicada al desarrollo de software y consultoría. Recientemente han ampliado sus oficinas y su plantilla, por lo que han tenido que rediseñar su arquitectura de red para garantizar la seguridad de sus datos internos, permitir el trabajo remoto de sus empleados y alojar sus propios servicios públicos de manera segura.


### Justificación de la Arquitectura de Red:
### 
    VLANs y Routers (Redes Internas):
La empresa está dividida en dos grandes bloques.

- Router 1 (VLANs Administrativas): Gestiona las redes de departamentos como Recursos Humanos y Contabilidad. Manejan datos muy sensibles (nóminas, facturación), por lo que están en sus propias VLANs aisladas.

- Router 2 (VLANs Operativas): Gestiona las redes del departamento de Desarrollo/IT y Ventas. Necesitan ancho de banda y acceso a servidores internos de pruebas.

###
    El Firewall:

Dado que hay departamentos críticos, el firewall actúa como el "cerebro" que decide quién habla con quién. Por ejemplo, permite que IT acceda a Contabilidad para dar soporte, pero bloquea que Ventas pueda ver los servidores de Recursos Humanos.

###
    La DMZ (Zona Desmilitarizada):

NovaTech quiere tener el control total de su presencia online sin depender de terceros, pero sin arriesgar su red interna. Por ello, han creado una DMZ.

Aquí reside el Servidor Web (la página corporativa para clientes), el Servidor DNS (para resolver los nombres de su propio dominio) y el Servidor de Mail (para las comunicaciones oficiales).
- Si un atacante logra comprometer la página web desde Internet, se quedará atrapado en la DMZ gracias al Firewall, y no podrá saltar a las VLANs donde están los datos de los clientes y empleados.