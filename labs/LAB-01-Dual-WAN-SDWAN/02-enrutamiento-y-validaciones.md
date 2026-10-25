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


## Evidencia 04: failover básico y failback con rutas estáticas — 2026-10-24

**Topología:** PC conectado por LAN al FortiGate; RB4011 simula ISP-A (ether7) e ISP-B (ether8). Prueba de tráfico con `ping 1.1.1.1 -t` desde Windows.

### Configuración mostrada en GUI
| Ruta | Gateway | Administrative Distance | Priority |
| --- | --- | --- | --- |
| ISP-A / wan1 | 198.51.100.1 | 10 | 1 |
| ISP-B / wan2 | 203.0.113.1 | 20 | 10 |

Se constató una política de salida `internal → wan1` con ACCEPT/NAT y se añadió `internal → wan2` también con ACCEPT/NAT. Durante el troubleshooting, el contador de WAN2 aparecía inicialmente en 0 B.

### Falla inicial y diagnóstico
Tras interrumpir ISP-A, el ping del PC perdió conectividad sin recuperarse. En el FortiGate se observó que la ruta por defecto continuaba siendo `0.0.0.0/0 [10/0] via 198.51.100.1, wan1`; tampoco respondía un ping local del FortiGate a 1.1.1.1. Esto mostró que crear una política WAN2 no bastaba mientras el equipo conservaba su ruta activa hacia WAN1.

### Resultado posterior verificado por el operador
Después de ajustar **Priority** en las rutas (1 para WAN1, 10 para WAN2), al deshabilitar `ether7` en el MikroTik, **el ping continuo del PC perdió conectividad unos pocos segundos y después recuperó respuestas usando ISP-B**. Al restaurar ISP-A, volvió automáticamente al principal. El operador confirmó ambas transiciones sin efectuar cambios adicionales durante la prueba.

**Alcance y cautela causal:** la configuración final contiene distancias 10/20 y prioridades 1/10. No se ha aislado experimentalmente si el cambio de Priority fue, por sí solo, la causa de la recuperación; la distancia administrativa ya establecía preferencia por WAN1. Tampoco se capturó una tabla de rutas durante la conmutación exitosa ni una medición precisa de segundos perdidos. No se atribuyen a esta prueba detección de pérdida lógica upstream con enlace físico activo ni garantías de continuidad de sesiones TCP.

### Conclusión
**Failover y failback de conectividad del PC: validados funcionalmente** bajo la falla simulada. **SD-WAN/Performance SLA: no configurados**. Próximas evidencias sugeridas: tabla de rutas durante ambas transiciones, número de paquetes perdidos, tráfico y logs, y ensayo de pérdida de upstream sin caída de portadora Ethernet.
