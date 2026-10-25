# 01 — Arquitectura y direccionamiento

## Intención del diseño
El RB4011 hace **emulación de dos ISP**: expone dos segmentos de tránsito separados. El FortiGate es el equipo de borde que decidirá entre WAN1 (principal) y WAN2 (respaldo). El switch Ubiquiti está aguas abajo del firewall, como capa de acceso.

| Rol | Interfaz | Dirección | Próximo salto |
| --- | --- | --- | --- |
| ISP-A emulado en MikroTik | ether7 | 198.51.100.1/29 | Salida upstream del RB4011 |
| FortiGate ISP-A | wan1 | 198.51.100.2/29 | 198.51.100.1 |
| ISP-B emulado en MikroTik | ether8 | 203.0.113.1/29 | Salida upstream del RB4011 |
| FortiGate ISP-B | wan2 | 203.0.113.2/29 | 203.0.113.1 |
| FortiGate LAN | internal | 192.168.10.0/24 | Directamente conectada |
| FortiGate VLAN | VLAN10 | 10.10.10.0/24 | Directamente conectada |

Las dos últimas filas reflejan redes conectadas observadas en la tabla de rutas, **no validan aún el cableado al switch ni las políticas LAN→Internet**.

## Versión y contexto de plataforma
- FortiGate 80F PoE — FortiOS 7.0.12, build 0523, modo NAT/Route.
- MikroTik RB4011 — simulador de operadores, con interfaces ether7 y ether8 ya preparadas.
- Ubiquiti USW Pro — switch de acceso planificado.
- Las IP de tránsito son prefijos TEST-NET documentales, **no públicos enrutables**.

## Criterios de diseño
1. Mantener caminos WAN lógicamente distintos.
2. Preferir WAN1, usar WAN2 cuando falle el camino principal.
3. Verificar fallas que no bajan el enlace Ethernet; para ello se evaluará SD-WAN health-check.
4. No alterar otros laboratorios existentes en el RB4011.
5. Evitar exponer administración sobre las interfaces WAN; aplicar políticas mínimas y guardar evidencia sanitizada.

## Limitaciones
Las dos WAN comparten el upstream físico doméstico. Si falla esa única conexión, ninguna WAN de laboratorio garantizará salida a Internet. La redundancia aquí evalúa lógica de selección y detección, no alta disponibilidad del proveedor físico.
