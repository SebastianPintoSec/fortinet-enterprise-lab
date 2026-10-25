# LAB 01 — FortiGate Dual WAN y Failover

**Estado:** implementación física en curso · **Fecha de registro:** 2026-10-24  
**Firmware confirmado:** FortiGate 80F PoE, **FortiOS v7.0.12 build 0523 (GA.M)** · operación NAT/Route, VDOM root, standalone.

## Objetivo
Implementar conectividad de dos ISP simulados hacia un FortiGate, verificar enrutamiento y NAT, y posteriormente configurar failover automático basado en SD-WAN y Performance SLA. El laboratorio es didáctico: para cada cambio se explica objetivo, fundamento, sintaxis, verificación, riesgos, fallas y solución.

## Topología actual
```text
Conexión doméstica 192.168.1.0/24
             |
     MikroTik RB4011 (simulador de ISP)
       ether7              ether8
    198.51.100.1/29     203.0.113.1/29
       |                    |
     wan1                 wan2
    198.51.100.2/29     203.0.113.2/29
         \                /
          FortiGate 80F PoE
            | LAN / VLAN
          Ubiquiti USW Pro
              | clientes
```

Ambas WAN simuladas dependen del mismo upstream doméstico: **no representan redundancia física real de operadores**. Los prefijos 198.51.100.0/24 y 203.0.113.0/24 son rangos de documentación RFC 5737, usados aquí exclusivamente en segmentos aislados.

## Estado de ejecución

| Fase | Resultado |
| --- | --- |
| WAN1 y WAN2 configuradas | Confirmado por pruebas y rutas |
| ICMP a gateway ISP-A | 5/5, pérdida 0 % |
| ICMP a gateway ISP-B | 5/5, pérdida 0 % |
| Ruta primaria WAN1 (distancia 10) | Confirmada como ruta activa |
| Ruta secundaria WAN2 (distancia 20) | Configurada; no activa mientras WAN1 se prefiera |
| Internet por WAN1 | 5/5 a 1.1.1.1, forzando WAN1 |
| Internet por WAN2 | 5/5 a 9.9.9.9 con ruta temporal /32 |
| SD-WAN y Performance SLA | **Pendientes** |
| Failover/failback automático y tráfico LAN | **Pendientes de validación** |

## Documentos
- [Arquitectura y direccionamiento](01-arquitectura.md)
- [Enrutamiento y explicación de pruebas](02-enrutamiento-y-validaciones.md)
- [Plan de SD-WAN y criterios de aceptación](03-plan-sdwan.md)

> Seguridad de publicación: no se incluyen credenciales, números de serie, IP públicas operativas ni respaldos completos. La documentación registra únicamente datos necesarios para reproducir el laboratorio.
