# Fortinet Enterprise Lab

Laboratorio físico y portafolio técnico orientado a **FortiGate, FortiOS, firewalling, routing, SD-WAN, seguridad perimetral y troubleshooting**.

## Entorno
- **FortiGate 80F PoE** — FortiOS **v7.0.12, build 0523 (GA.M)**, modo NAT/Route.
- **MikroTik RB4011** — simulación independiente de dos gateways ISP.
- **Ubiquiti USW Pro** — capa de acceso downstream.

## Laboratorios
### [LAB 01 — Dual WAN + SD-WAN + Failover](labs/LAB-01-Dual-WAN-SDWAN/README.md)
Topología: RB4011 (ISP-A/ISP-B simulados) → FortiGate 80F → USW Pro. Ya validados ambos gateways y salida a Internet de ambos enlaces mediante pruebas controladas de enrutamiento. **SD-WAN, failover/failback y tráfico LAN aún no ejecutados.**

- [Arquitectura, interfaces y plan IP](labs/LAB-01-Dual-WAN-SDWAN/01-arquitectura.md)
- [Routing, distancias, prefijos y evidencias de validación](labs/LAB-01-Dual-WAN-SDWAN/02-enrutamiento-y-validaciones.md)
- [Diseño SD-WAN, dependencias y plan de pruebas](labs/LAB-01-Dual-WAN-SDWAN/03-plan-sdwan.md)

## Metodología
Cada fase documenta: **objetivo, fundamentos, arquitectura, sintaxis de comandos, efectos esperados, riesgos, validaciones, troubleshooting y resultados reales**. Se distinguen explícitamente las pruebas completadas de las pendientes.

## Publicación segura
No se incluyen contraseñas, tokens, respaldos completos, números de serie ni direcciones de infraestructura real de terceros. Los prefijos TEST-NET empleados en el LAB 01 son direcciones reservadas para documentación. Antes de publicar evidencias se deben sanitizar.

**Estado general:** work in progress · 2026-10-24.
