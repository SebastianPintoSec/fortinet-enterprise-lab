# LAB 03 — Segmentación Inter-VLAN y acceso de mínimo privilegio

## Objetivo

Validar una política de **segmentación inter-VLAN controlada** en FortiGate, permitiendo que una red de usuarios inalámbricos acceda únicamente a un host específico de la VLAN de management, mientras otras VLAN permanecen aisladas.

El caso de prueba utiliza una Raspberry Pi ubicada en `VLAN100-MGMT` como activo de infraestructura.

## Topología lógica

```text
VLAN20-USUARIOS
10.10.20.0/24
     │
     └── acceso a Raspberry: DENEGADO

VLAN30-WIFI
10.10.30.0/24
     │
     │ policy específica
     ▼
FortiGate 80F
     │
     ▼
VLAN100-MGMT
10.10.100.0/24
     │
     └── Raspberry Pi
         10.10.100.4
```

## Principio aplicado

La política se diseñó bajo **mínimo privilegio**:

- no se permite acceso general de VLAN30 hacia toda VLAN100;
- el destino es un único objeto de host: `RASPBERRY`;
- se permiten únicamente los servicios necesarios;
- no se utiliza NAT entre redes internas;
- el retorno se permite mediante el stateful firewall del FortiGate;
- se habilita logging para todas las sesiones permitidas.

## Política FortiGate

### Parámetros

| Campo | Valor |
|---|---|
| Nombre | VLAN30ARASPBERRY |
| Incoming Interface | VLANWIFI / VLAN30 |
| Outgoing Interface | VLAN100-MGMT |
| Source | VLANWIFI address |
| Destination | RASPBERRY |
| Schedule | always |
| Services | PING, SSH, HTTPS |
| Action | ACCEPT |
| NAT | Disabled |
| Inspection Mode | Flow-based |
| Logging | All Sessions |

### Lógica

```text
VLAN30 -> Raspberry
  ICMP  -> permitido
  TCP/22 -> permitido
  TCP/443 -> permitido

VLAN30 -> resto de VLAN100
  -> no permitido por esta policy

VLAN20 -> Raspberry
  -> no permitido
```

## Motivo para deshabilitar NAT

No se requiere NAT entre VLANs porque ambas redes están directamente conectadas al FortiGate.

Mantener NAT deshabilitado conserva la IP de origen real del cliente, lo que permite:

- trazabilidad;
- logging correcto;
- políticas posteriores por origen;
- troubleshooting más preciso;
- auditoría de sesiones.

## Validaciones realizadas

### Prueba 1 — VLAN30 hacia Raspberry

Cliente de prueba en VLAN30:

```text
Origen:      10.10.30.101
Destino:     10.10.100.4
Policy:      VLAN30ARASPBERRY
```

Resultados observados en **Log & Report > Forward Traffic**:

- PING: permitido;
- HTTPS: sesión permitida por policy;
- Policy ID: `VLAN30ARASPBERRY`.

La presencia del tráfico HTTPS en logs confirma que el FortiGate permitió el intento de conexión, independientemente de que exista o no un servicio web escuchando en TCP/443 en la Raspberry.

### Prueba 2 — VLAN20 hacia Raspberry

Desde un host de VLAN20 se intentó alcanzar la Raspberry.

Resultado:

```text
VLAN20 -> Raspberry
DENEGADO
```

Esto confirma que no existe una regla amplia que permita acceso indiscriminado hacia la VLAN de management.

## Resultado

La prueba demuestra que el FortiGate puede aplicar **microsegmentación básica entre VLANs** mediante políticas específicas por:

- interfaz de origen;
- interfaz de destino;
- red de origen;
- host de destino;
- servicio permitido.

La VLAN de management permanece protegida y solo se habilita el acceso explícitamente requerido.

## Próximos pasos

- recuperar acceso administrativo completo de la Raspberry;
- validar SSH real desde VLAN30;
- reservar IP de la Raspberry por DHCP/MAC si aún no está reservado;
- centralizar Syslog en la Raspberry;
- agregar monitoreo ICMP/SNMP;
- evaluar FreeRADIUS y 802.1X;
- documentar eventos permitidos y bloqueados como evidencia del laboratorio.

## Evidencia recomendada

Para publicación en GitHub, adjuntar capturas sanitizadas de:

1. policy `VLAN30ARASPBERRY`;
2. Forward Traffic con PING/HTTPS permitidos;
3. prueba fallida desde VLAN20;
4. objeto de dirección `RASPBERRY` como host /32.

No publicar credenciales, tokens ni información sensible.
