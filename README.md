# Fortinet Enterprise Lab

Laboratorio físico y portafolio técnico orientado a **FortiGate, FortiOS, firewalling, routing, SD-WAN, seguridad perimetral, segmentación y troubleshooting**.

## Entorno
- **FortiGate 80F PoE** — FortiOS **v7.0.12, build 0523 (GA.M)**, modo NAT/Route.
- **MikroTik RB4011** — simulación independiente de dos gateways ISP.
- **Ubiquiti USW Pro** — capa de acceso downstream.
- **UniFi CloudKey Plus** — gestión local UniFi.
- **UniFi AP** — WLAN segmentada por VLAN.

## Laboratorios

### [LAB 01 — Dual WAN + SD-WAN + Failover](labs/LAB-01-Dual-WAN-SDWAN/README.md)
Topología: RB4011 (ISP-A/ISP-B simulados) → FortiGate 80F → USW Pro. Ya validados ambos gateways y salida a Internet de ambos enlaces. **Failover y failback con rutas estáticas verificados desde un PC**, con interrupción de algunos segundos; **SD-WAN y Performance SLA configurados; primera conmutación a WAN2 observada, failback de conectividad reportado y recuperación de WAN1 observada; aún falta validar selección de nuevas sesiones y falla lógica pura**.

- [Arquitectura, interfaces y plan IP](labs/LAB-01-Dual-WAN-SDWAN/01-arquitectura.md)
- [Routing, distancias, prefijos y evidencias de validación](labs/LAB-01-Dual-WAN-SDWAN/02-enrutamiento-y-validaciones.md)
- [Diseño SD-WAN, dependencias y plan de pruebas](labs/LAB-01-Dual-WAN-SDWAN/03-plan-sdwan.md)
- [SD-WAN: implementación, Performance SLA y resultados](labs/LAB-01-Dual-WAN-SDWAN/04-sdwan-performance-sla-validaciones.md)

### [LAB 02 — FortiGate + UniFi: VLAN, Management y WiFi](labs/LAB-02-UniFi-VLAN-Segmentation/README.md)
Integración FortiGate ↔ USW Pro ↔ UniFi AP utilizando **internal3 como trunk 802.1Q**, red nativa temporal, **VLAN20 para usuarios**, **VLAN30 para WiFi** y **VLAN100 para management**.

Validado:
- DHCP y salida a Internet sobre el trunk;
- migración del management del USW a VLAN100;
- adopción y management del AP en VLAN100;
- SSID sobre VLAN30;
- DHCP e Internet para clientes WiFi;
- troubleshooting de DHCP y tagging UniFi/FortiGate.

**Próxima fase:** evaluar adopción remota Layer 3 para clientes detrás de ONT/CGNAT mediante UniFi OS Server en VPS u Official UniFi Hosting.

## Metodología
Cada fase documenta: **objetivo, fundamentos, arquitectura, sintaxis de comandos, efectos esperados, riesgos, validaciones, troubleshooting y resultados reales**. Se distinguen explícitamente las pruebas completadas de las pendientes.

## Publicación segura
No se incluyen contraseñas, tokens, respaldos completos, números de serie ni direcciones de infraestructura real de terceros. Los prefijos TEST-NET empleados en el LAB 01 son direcciones reservadas para documentación. Antes de publicar evidencias se deben sanitizar.

**Estado general:** work in progress · 2026-11-05.
