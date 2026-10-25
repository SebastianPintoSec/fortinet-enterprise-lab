# 02 — Enrutamiento y validaciones reproducibles

**Fecha:** 2026-10-24 · **FortiOS:** 7.0.12 build 0523.

## Fundamento: tabla de rutas y distancia administrativa
En FortiOS hay diferencia entre rutas **configuradas** y rutas **activas**. Las rutas estáticas por defecto hacia WAN1 y WAN2 se crearon con distancias 10 y 20. Para el mismo destino 0.0.0.0/0, la menor distancia favorece WAN1. La ruta de respaldo aparece en configuración, pero puede no estar instalada en el FIB mientras WAN1 esté disponible.

Configuración observada (extracto sanitizado):
```fortios
config router static
    edit 1
        set gateway 198.51.100.1
        set device "wan1"
    next
    edit 2
        set gateway 203.0.113.1
        set distance 20
        set device "wan2"
    next
end
```

Ruta activa observada:
```text
S* 0.0.0.0/0 [10/0] via 198.51.100.1, wan1
```

## Evidencia 01: gateways
- WAN1 (origen 198.51.100.2) → 198.51.100.1: **5/5**, pérdida **0 %**, RTT 0.1/0.1/0.3 ms (min/prom/max, salida del equipo).
- WAN2 (origen 203.0.113.2) → 203.0.113.1: **5/5**, pérdida **0 %**, RTT 0.1/0.2/0.4 ms.

Esto valida conectividad L3 al gateway, no Internet.

## Evidencia 02: diferencia entre source IP e interfaz
```fortios
execute ping-options reset
execute ping-options interface wan1
execute ping-options source 198.51.100.2
execute ping-options view-settings
execute ping 1.1.1.1
```
Resultado **WAN1**: 5/5, 0 % pérdida, RTT min/prom/max **4.1/4.3/4.6 ms**.

Repetido con `interface wan2`, `source 203.0.113.2`, destino 1.1.1.1: **5 mensajes `sendto failed`**, 0/5 respuestas. Las opciones indicaban `Use SD-WAN: no`.

Interpretación: indicar `source` fija la IP de origen, pero no instala una ruta de salida. Incluso al especificar `interface wan2`, el intento requiere enrutamiento válido. El fallo por sí solo **no acredita caída física ni problema NAT**.

## Evidencia 03: ruta temporal /32 por WAN2
Para probar el camino secundario sin cambiar la preferencia global se creó esta ruta:
```fortios
config router static
    edit 50
        set dst 9.9.9.9 255.255.255.255
        set gateway 203.0.113.1
        set device "wan2"
        set distance 10
    next
end
```
La tabla mostró:
```text
S 9.9.9.9/32 [10/0] via 203.0.113.1, wan2
```
Y se verificó con:
```fortios
execute ping-options reset
execute ping-options interface wan2
execute ping-options source 203.0.113.2
execute ping 9.9.9.9
```
**Resultado real:** 5/5, pérdida 0 %, RTT min/prom/max **3.1/6.0/13.9 ms**.

### Por qué funcionó: Longest Prefix Match
`9.9.9.9/32` es más específica que `0.0.0.0/0`. Para ese destino se elige la ruta /32 hacia WAN2 aunque WAN1 conserve la ruta por defecto de menor distancia. Esto demuestra que WAN2 tiene conectividad de salida bajo una selección de ruta válida; **no demuestra failover automático**.

### Limpieza verificada
```fortios
config router static
    delete 50
end
execute ping-options reset
```
La ruta de diagnóstico fue eliminada. Se recomienda consultar de nuevo la tabla al iniciar la siguiente sesión para confirmar el estado final.

## Troubleshooting de referencia
| Síntoma | Hipótesis a verificar | Prueba |
| --- | --- | --- |
| Gateway no responde | Cableado, IP/máscara, interfaz, firewall upstream | ping al gateway y estado de interfaz |
| `sendto failed` por WAN2 | Ruta no instalada o selección de interfaz incompatible | `get router info routing-table all` |
| Gateway responde, Internet no | Ruta, NAT o filtros del MikroTik, retorno | ruta específica /32 y prueba controlada |
| Ping a Internet responde | Conectividad L3/ICMP | No presupone políticas LAN→WAN ni sesiones existentes |

No publicar logs de producción completos sin sanitización.
