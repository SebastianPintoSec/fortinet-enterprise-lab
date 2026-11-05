# LAB 02 — FortiGate + UniFi: Trunk 802.1Q, VLAN de gestión y WiFi segmentado

## Objetivo

Integrar un **FortiGate 80F PoE** con un **Ubiquiti USW Pro 48 PoE** y un **UniFi AP**, utilizando al FortiGate como gateway, servidor DHCP y firewall, mientras UniFi se utiliza para switching, tagging 802.1Q y gestión de los dispositivos.

El objetivo principal fue validar:

- uplink trunk FortiGate ↔ USW;
- red nativa temporal para evitar perder gestión durante la migración;
- VLAN 20 para usuarios cableados;
- VLAN 30 para clientes WiFi;
- VLAN 100 para administración de infraestructura;
- migración del management del USW y AP hacia VLAN 100;
- DHCP y salida a Internet por cada segmento;
- troubleshooting de DHCP sobre enlaces tagged.

## Equipamiento

- FortiGate 80F PoE — FortiOS 7.0.12 build0523
- Ubiquiti USW Pro 48 PoE
- UniFi AP
- UniFi CloudKey Plus
- MikroTik RB4011 como upstream / simulación ISP

## Diseño lógico

```text
FortiGate 80F
└── internal3
    ├── Native / untagged : 10.10.10.0/24
    ├── VLAN 20           : 10.10.20.0/24
    ├── VLAN 30           : 10.10.30.0/24
    └── VLAN 100          : 10.10.100.0/24
           │
           ▼
       USW Pro 48 PoE
       ├── Management: VLAN100
       └── Puerto AP
           ├── Native: VLAN100-MGMT
           └── Tagged: VLAN30-WIFI
```

## FortiGate

### Interfaz física del trunk

```text
internal3
IP nativa temporal: 10.10.10.1/24
```

La red nativa se agregó deliberadamente para mantener un camino de administración funcional durante la migración del USW. Esto evitó repetir el escenario inicial donde el switch perdía conectividad al cambiar directamente su management a VLAN100.

### VLANs

| Segmento | VLAN | Gateway |
|---|---:|---|
| Usuarios cableados | 20 | 10.10.20.1/24 |
| WiFi | 30 | 10.10.30.1/24 |
| Management | 100 | 10.10.100.1/24 |

Las tres VLAN tienen como interfaz padre `internal3`.

### DHCP

- Native/internal3: 10.10.10.100–10.10.10.200
- VLAN20: 10.10.20.100–10.10.20.200
- VLAN30: 10.10.30.100–10.10.30.200
- VLAN100: 10.10.100.2–10.10.100.254

### Políticas

Se habilitó salida hacia la zona SD-WAN con NAT para:

- red nativa de `internal3`;
- VLAN100-MGMT;
- VLAN30-WIFI.

Las políticas se validaron por conectividad IP y navegación real desde clientes finales.

## UniFi USW Pro

### Uplink hacia FortiGate

El puerto de uplink quedó configurado como trunk:

- Native VLAN: Default / tráfico untagged
- Tagged VLANs permitidas: VLAN20, VLAN30 y VLAN100

Durante la migración, el USW recibió inicialmente una dirección de la red nativa `10.10.10.0/24`. Solo después de comprobar DHCP, routing e Internet en VLAN100 se aplicó el Network Override.

### Management

Resultado final validado:

```text
USW Pro 48 PoE
Management VLAN: 100
IP observada: 10.10.100.3
Estado UniFi: Online
```

La dirección puede reservarse por MAC en el DHCP del FortiGate para mantener administración predecible.

## UniFi AP

El AP se adoptó localmente y posteriormente se migró su administración a VLAN100.

El puerto del USW hacia el AP opera como trunk:

- Native VLAN: VLAN100-MGMT
- Tagged VLAN: VLAN30-WIFI

El SSID de prueba utiliza VLAN30.

Resultado:

```text
AP management  -> VLAN100
Cliente WiFi   -> VLAN30
DHCP VLAN30    -> OK
Internet WiFi  -> OK
```

## Troubleshooting relevante

### 1. El USW desaparecía al mover management directamente a VLAN100

Síntoma:
- el switch dejaba de verse en UniFi después del Network Override.

Causa de diseño:
- el uplink definitivo era un trunk puro y no existía una red nativa temporal previamente validada.

Corrección:
1. configurar `internal3` con una red nativa temporal;
2. validar DHCP e Internet directamente;
3. mover el USW manteniendo management untagged;
4. validar VLAN100 con un cliente real;
5. recién entonces migrar management a VLAN100.

### 2. Cliente WiFi no obtenía IP en VLAN30

El sniffer del FortiGate mostró:

```text
DHCP Discover  -> recibido por VLAN30
DHCP Offer     -> generado por FortiGate
```

Esto descartó inicialmente al servidor DHCP y dirigió el troubleshooting hacia la configuración del puerto USW ↔ AP.

Tras ajustar el puerto del AP para transportar correctamente VLAN30 tagged, el cliente obtuvo DHCP y navegación.

## Validaciones completadas

- [x] DHCP en red nativa de internal3
- [x] Navegación desde red nativa
- [x] USW conectado al trunk definitivo
- [x] VLAN100 transportada por el trunk
- [x] DHCP en VLAN100 desde un puerto access del USW
- [x] Internet desde VLAN100
- [x] Management USW migrado a VLAN100
- [x] AP adoptado
- [x] Management AP en VLAN100
- [x] SSID con VLAN30
- [x] DHCP para clientes WiFi
- [x] Internet desde VLAN30

## Resultado

El laboratorio confirma una integración funcional entre **FortiGate como gateway/firewall/DHCP** y **UniFi como plataforma de switching y WLAN**, manteniendo una VLAN dedicada de management y separación de tráfico de usuarios.

La principal lección operacional fue no migrar la administración de un dispositivo UniFi a una VLAN etiquetada hasta validar previamente el trunk, DHCP, routing y retorno de tráfico con un cliente independiente.

## Próxima fase

Evaluar **adopción remota Layer 3 de dispositivos UniFi** para clientes que utilizan ONT/router del ISP y pueden encontrarse detrás de CGNAT, comparando CloudKey on-premise, Official UniFi Hosting y UniFi OS Server en VPS con IPv4 pública.
