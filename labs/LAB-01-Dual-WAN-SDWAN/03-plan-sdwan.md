# 03 — Plan de SD-WAN y pruebas de aceptación

**Estado: pendiente de implementación.** Esta página es un plan, no evidencia de failover exitoso.

## Propósito
Las rutas por distancia sirven como respaldo básico cuando una ruta deja de estar disponible. Pero si WAN1 permanece físicamente activa y el camino pierde salida real a Internet, una ruta predeterminada simple puede seguir seleccionándose. SD-WAN más health-check / Performance SLA permite medir salud del camino y aplicar preferencia de enlaces.

## Diseño propuesto (a validar en FortiOS 7.0.12)
- Crear zona SD-WAN de Internet con miembros wan1 y wan2.
- Configurar gateways 198.51.100.1 y 203.0.113.1.
- Regla de selección que privilegie WAN1 y use WAN2 de respaldo.
- Health-check hacia destinos externos con parámetros documentados de pérdida/latencia/jitter y criterios de recuperación.
- Ajustar ruta por defecto para utilizar la zona SD-WAN **solo después** de revisar referencias a interfaces en rutas y políticas.
- Adecuar firewall policies LAN→zona SD-WAN y NAT.
- No incluir en la documentación secretos ni identificadores únicos.

## Preflight obligatorio
```fortios
show system sdwan
show firewall policy
show router static
get router info routing-table all
```
Explicación: las interfaces existentes pueden tener referencias en políticas y rutas. Cambiar rutas o membresías WAN sin revisar dependencias puede cortar la conectividad administrativa y el tráfico LAN.

## Casos de prueba (no ejecutados)
| ID | Escenario | Esperado |
| --- | --- | --- |
| T01 | WAN1 y WAN2 saludables | Tráfico nuevo prefiere WAN1 |
| T02 | Caída física de WAN1 | WAN2 asume tráfico nuevo |
| T03 | Caída lógica de salida WAN1 sin bajar enlace Ethernet | Health-check detecta indisponibilidad y selecciona WAN2 |
| T04 | Recuperación WAN1 | Retorno controlado a WAN1 según reglas |
| T05 | Falla de WAN2 con WAN1 saludable | WAN1 mantiene conectividad |
| T06 | Sesiones activas durante conmutación | Registrar interrupción/reestablecimiento, sin prometer continuidad TCP |
| T07 | Tráfico desde VLAN10/internal | Políticas, ruta, NAT y DNS funcionales por el enlace seleccionado |

## Evidencias a registrar
Capturas sanitizadas de interfaces/miembros SD-WAN, estado SLA, tabla de rutas, política LAN→SD-WAN, secuencia temporal de cada falla, pérdida de ping y pruebas TCP. Incluir resultados fallidos y troubleshooting para reproducibilidad.

## Consideraciones de seguridad
Restringir administración a segmentos dedicados; minimizar servicios permitidos sobre WAN. Respaldar configuración de manera privada antes de cambiar rutas o membresías. Cualquier evidencia pública debe eliminar credenciales, series y datos operativos sensibles.
