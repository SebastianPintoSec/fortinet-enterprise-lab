# 04 — SD-WAN y Performance SLA: implementación y primera prueba

**Fecha:** 2026-10-24 (hora local del laboratorio) · **Plataforma:** FortiGate 80F PoE · **FortiOS:** 7.0.12 build 0523

## Objetivo
Migrar del failover con rutas estáticas a una arquitectura de **SD-WAN principal/respaldo** y comprobar la detección de pérdida de conectividad mediante sondas activas ICMP.

## Arquitectura
- ISP-A simulado: RB4011 ether7, gateway `198.51.100.1` → FortiGate `wan1` (`198.51.100.2/29`).
- ISP-B simulado: RB4011 ether8, gateway `203.0.113.1` → FortiGate `wan2` (`203.0.113.2/29`).
- PC de prueba en LAN `internal` detrás del FortiGate.
- Ambos proveedores simulados comparten upstream doméstico; no equivale a redundancia física real.

## Implementación GUI confirmada

### Respaldo previo
Se generó un backup local antes de migrar. **No se publica el archivo por contener potencialmente secretos e información sensible.**

### Integración de interfaces
Se creó la zona `SDWAN-INTERNET`. En el asistente *Integrate Interface*, se encontraron referencias de una **Firewall Policy** y una **Static Route** en cada WAN, migradas mediante *Replace Instance*. Después fue necesario establecer manualmente los gateways en los miembros:
- Miembro `ISP_A (wan1)` → `198.51.100.1`.
- Miembro `ISP_B (wan2)` → `203.0.113.1`.

**Fundamento:** la IP del puerto identifica el extremo local; el gateway del miembro es el próximo salto. La zona abstrae los enlaces para aplicar políticas y reglas de selección.

### Rutas y políticas
El intento de agregar una ruta SD-WAN manteniendo dos rutas por defecto de interfaces físicas arrojó el error:
`You cannot have duplicated routes on SD-WAN and non SD-WAN interfaces` (código `-9100`).
Se sustituyó el esquema anterior por una **única ruta predeterminada 0.0.0.0/0 asociada a `SDWAN-INTERNET`**, confirmada por GUI. Las políticas `internal → SDWAN-INTERNET` permanecieron en estado ACCEPT con NAT habilitado. Existen dos políticas parecidas; consolidación pendiente tras validar cambios.

**Fundamento:** no pueden coexistir rutas predeterminadas duplicadas hacia miembros físicos y el objeto SD-WAN en este procedimiento. SD-WAN gestiona los gateways desde sus miembros; las firewall policies controlan el acceso, no sustituyen el enrutamiento.

### Regla SD-WAN
- Name: `SDWAN-PRIMARY-BACKUP`
- Source y destination: `all`
- Protocolo: `ANY`
- Interface Selection Strategy: `Manual`
- Orden: `ISP_A (wan1)`, luego `ISP_B (wan2)`
- Habilitada según configuración presentada.

**Fundamento:** la estrategia manual expresa preferencia; la salud del camino se determina aparte mediante health checks.

### Health Check
- Nombre `SLA_INTERNET`
- Active probe, protocolo Ping, servidor `1.1.1.1`
- Participants: wan1 y wan2
- SLA Target desactivado
- Check interval 500 ms; failures before inactive 5; restore after 5 checks
- Update static route activado.

**Fundamento:** cada miembro se sondea individualmente, incluso sin sesiones de usuarios; el monitor indica si un camino está alive/dead. Los parámetros no constituyen una promesa de conmutación en tiempo exacto.

## Validaciones ejecutadas

### Health Check antes de la caída
Salida observada de `diagnose sys sdwan health-check`:
```text
Health Check(SLA_INTERNET):
Seq(1 wan1): state(alive), packet-loss(0.000%) latency(3.034), jitter(0.517) sla_map=0x0
Seq(2 wan2): state(alive), packet-loss(0.000%) latency(3.619), jitter(0.806) sla_map=0x0
```
Fragmento sanitizado; `sla_map=0x0` es coherente con no haber definido SLA Target.

### Fallo de ISP-A / selección de ISP-B
Se ejecutó `ping 1.1.1.1 -t` desde un PC Windows. Captura aportada muestra **dos mensajes consecutivos «Tiempo de espera agotado»** y luego retorno de respuestas ICMP. En la pantalla *Performance SLAs*, durante la incidencia:
- ISP_A (wan1): indicador rojo, sin métricas saludables.
- ISP_B (wan2): indicador verde, pérdida 0.00 %, latencia 3.16 ms, jitter 0.24 ms.

**Resultado:** se confirma recuperación del tráfico del PC al respaldo y reconocimiento de falla del enlace ISP-A por Performance SLA. La prueba está descrita como falla del ISP principal; **no se ha adjuntado evidencia independiente del estado físico de wan1 durante esta incidencia**. Por lo tanto, no se afirma aún que fuera una falla puramente lógica con enlace físico levantado.

El número de timeouts visibles no es una medida exacta de la duración interna del failover. Falta capturar prueba precisa de tiempo, método de interrupción aplicado, ruta/miembro de salida y regreso al principal bajo SD-WAN.

## Próximas pruebas
1. Confirmar *failback* SD-WAN al restaurar ISP-A, monitoreando estado alive y tráfico por WAN1.
2. Simular bloqueo upstream específico de ISP-A **manteniendo ether7 habilitado**, y verificar tanto estado físico *up* como Health Check *dead*; preservar la administración y el laboratorio Cisco.
3. Probar failover de sesiones TCP y cuantificar pérdidas con marca temporal.
4. Consolidar las políticas redundantes una vez verificado tráfico LAN, y documentar pruebas por VLAN.

## Troubleshooting
- Si el SLA muestra `?` en la GUI, consultar `diagnose sys sdwan health-check` antes de asumir caída.
- Si el miembro presenta gateway `0.0.0.0`, revisar configuración estática de su próximo salto.
- Si no hay Internet desde LAN pero sí desde FortiGate, revisar firewall policy, NAT y sesiones.
- Si el ping no conmuta, revisar estado health-check, selección de SD-WAN rule y ruta instalada.

> Solo se reportan resultados observados; ninguna evidencia pública contiene números de serie, credenciales ni archivos de respaldo.


## Evidencia 05: recuperación del ISP-A tras la falla (2026-10-24)

El operador restauró ISP-A y reportó recuperación automática. En capturas de la GUI se observó:

| Medición | ISP-A / wan1 | ISP-B / wan2 |
| --- | --- | --- |
| Indicador de salud | Verde (alive) | Verde (alive) |
| Pérdida mostrada | 78.00 % | 0.00 % |
| Latencia | 3.41 ms | 3.87 ms |
| Jitter | 0.41 ms | 0.56 ms |
| Tráfico visible | Descarga 28.26 kbps / subida 38.86 kbps | Descarga 67.61 kbps / subida 23.44 kbps |

La pérdida de 78 % sobre WAN1 es compatible con la ventana de medición acumulando fallos anteriores; el estado verde indica que las comprobaciones de salud ya consideraban recuperado el camino. Esta captura **no prueba pérdida actual sostenida del 78 %**, ni que todas las sesiones nuevas regresen a WAN1.

**Conclusión:** el *failback de disponibilidad* y la reaparición de tráfico por WAN1 se observan, y el operador confirma recuperación automática. Queda pendiente validar la elección de WAN1 para sesiones nuevas mediante diagnóstico de SD-WAN y confirmar el tipo exacto de falla simulada (estado físico up/down). El gráfico de tráfico simultáneo en WAN1 y WAN2 tampoco demuestra por sí solo balanceo: pueden existir sesiones anteriores y sondas.


## Evidencia 06: regla SD-WAN en modo manual tras recuperación

Salida observada:
```text
Service(1): Address Mode(IPV4) flags=0x200 use-shortcut-sla
  Gen(4), TOS(0x0/0x0), Protocol(0: 1->65535), Mode(manual)
  Members(2):
    1: Seq_num(1 wan1), alive, selected
    2: Seq_num(2 wan2), alive, selected
  Src address(1): 0.0.0.0-255.255.255.255
  Dst address(1): 0.0.0.0-255.255.255.255
```

**Interpretación:** la regla IPv4 1 se encuentra en modo manual con dos miembros saludables y seleccionables. La preferencia se aplica según el orden de miembros configurado: wan1 y luego wan2. El indicador `selected` en ambos miembros **no prueba balanceo de carga ni demuestra qué WAN tomó una sesión particular**. Verificación complementaria pendiente: `diagnose firewall proute list` y registro *Forward Traffic* de una sesión nueva para comprobar `dstintf` efectiva. No confundir elegibilidad con ruta usada por un flujo.

Referencia técnica: documentación Fortinet FortiOS 7.0 New Features Guide y Technical Tip «Working of SD-WAN rule with outgoing interface selection strategy as manual».


## Evidencia 07: regla instalada en policy routing (2026-10-24)

Comando de solo lectura: `diagnose firewall proute list`.

```text
id=2131427329(0x7f0b0001) vwl_service=1(SDWAN-PRIMARY-BACKUP)
vwl_mbr_seq=1 2 ... iif=0(any) ... path(2) oif=5(wan1) oif=6(wan2)
source(1): 0.0.0.0-255.255.255.255
destination(1): 0.0.0.0-255.255.255.255
hit_count=761 last_used=2026-10-24 19:50:48
```

**Interpretación:** `vwl_service=1` identifica la regla manual configurada; `vwl_mbr_seq=1 2` y `path(2)` exponen ambos caminos en orden de preferencia. `hit_count=761` demuestra coincidencias de la regla, no que 761 paquetes o sesiones hayan utilizado WAN1 específicamente. `last_used` conserva la marca del dispositivo sin inferir duración de sesión o conmutación. Se requiere validar la interfaz de salida `dstintf` de una conexión nueva en *Forward Traffic* (o con diagnóstico de sesión) para confirmar selección efectiva por flujo.

**Estado:** regla activa y utilizada, preferencia configurada; inspección de interfaz real de una sesión individual pendiente.
