# Informe de configuración de DMZ con Cisco Packet Tracer

---

### 1. Objetivo del laboratorio

Configurar una red segmentada en tres zonas (LAN interna, DMZ y red externa) utilizando un router Cisco ISR 2911 como firewall perimetral en Cisco Packet Tracer. Se aplicó NAT estático para exponer el servidor web de la DMZ hacia Internet de forma controlada, y ACLs para restringir el tráfico entre zonas, simulando las políticas de seguridad de un entorno empresarial real.

---

### 2. Topología implementada

- **Cantidad de redes:** 3 (LAN interna, DMZ, Red externa)
- **Dispositivos usados:** Router Cisco ISR 2911 (`Router_FW`), 3 switches Cisco 2960 (`SW_Internal`, `SW_DMZ`, `SW_External`), 1 PC cliente interno (`PC_Internal`), 1 servidor web (`Web_DMZ`), 1 PC cliente externo (`PC_External`)

| Zona | Función |
|------|---------|
| **LAN Interna** (`192.168.1.0/24`) | Red privada de usuarios. Puede iniciar conexiones hacia la DMZ y el exterior, pero está protegida de accesos no autorizados desde ambas zonas. |
| **DMZ** (`192.168.2.0/24`) | Zona neutral que aloja el servidor web público. Está aislada de la LAN interna: no puede iniciar conexiones hacia ella, solo responder a las que la LAN inicia. |
| **Red Externa** (`192.168.3.0/24`) | Simula Internet. Solo puede acceder al servidor web vía HTTP a través del NAT del router; el resto del tráfico está bloqueado. |

```
                        ┌─────────────┐
                        │  Router_FW  │
                        │  ISR 2911   │
                        └──┬────┬────┘
                Gi0/0      │    │     Gi0/2
          ┌────────────────┘    └────────────────┐
          │           Gi0/1                       │
          │      ┌────────┐                       │
          ▼      ▼        │                       ▼
     SW_Internal  SW_DMZ  │              SW_External
          │          │    │                       │
     PC_Internal  Web_DMZ │                PC_External
    192.168.1.10  192.168.2.10            192.168.3.10
```

---

### 3. Plan de direccionamiento IP

| Dispositivo             | IP              | Máscara           | Gateway         |
|-------------------------|-----------------|-------------------|-----------------|
| PC_Internal             | 192.168.1.10    | 255.255.255.0     | 192.168.1.1     |
| Web_DMZ (Server)        | 192.168.2.10    | 255.255.255.0     | 192.168.2.1     |
| PC_External             | 192.168.3.10    | 255.255.255.0     | 192.168.3.1     |
| Router_FW Gi0/0 (LAN)   | 192.168.1.1     | 255.255.255.0     | —               |
| Router_FW Gi0/1 (DMZ)   | 192.168.2.1     | 255.255.255.0     | —               |
| Router_FW Gi0/2 (Ext)   | 192.168.3.1     | 255.255.255.0     | —               |

---

### 4. Configuración aplicada (resumen)

#### 4.1 Interfaces del router

```bash
interface GigabitEthernet0/0
 ip address 192.168.1.1 255.255.255.0
 duplex auto
 speed auto
!
interface GigabitEthernet0/1
 ip address 192.168.2.1 255.255.255.0
 ip nat inside
 ip access-group BLOQUEAR_DMZ_A_LAN in
 duplex auto
 speed auto
!
interface GigabitEthernet0/2
 ip address 192.168.3.1 255.255.255.0
 ip nat outside
 ip access-group 101 in
 duplex auto
 speed auto
```

#### 4.2 NAT estático

Permite que el servidor web en la DMZ sea accesible desde la red externa a través del puerto 80, sin exponer su IP privada directamente.

```bash
ip nat inside source static tcp 192.168.2.10 80 192.168.3.1 80
```

#### 4.3 ACLs

**ACL 101** — Aplicada en la interfaz externa (`Gi0/2`), permite únicamente tráfico HTTP entrante hacia el router:

```bash
access-list 101 permit tcp any host 192.168.3.1 eq www
```

**ACL BLOQUEAR_DMZ_A_LAN** — Aplicada en la interfaz DMZ (`Gi0/1`), aísla la DMZ de la LAN interna permitiendo solo tráfico de retorno:

```bash
ip access-list extended BLOQUEAR_DMZ_A_LAN
 remark Permitir trafico de retorno si la conexion se inicio en la LAN
 permit tcp 192.168.2.0 0.0.0.255 192.168.1.0 0.0.0.255 established
 remark Denegar el resto del trafico iniciado por la DMZ hacia la LAN
 deny ip 192.168.2.0 0.0.0.255 192.168.1.0 0.0.0.255
 permit ip any any
```

---

### 5. Verificaciones realizadas

| Prueba | Origen | Destino | Protocolo | Resultado |
|--------|--------|---------|-----------|-----------|
| Conectividad básica LAN → DMZ | PC_Internal | Web_DMZ | ICMP (ping) | ✅ Exitoso |
| Acceso web externo al servidor | PC_External | Router_FW :80 → Web_DMZ | HTTP | ✅ Exitoso (NAT) |
| Bloqueo externo → LAN | PC_External | PC_Internal | ICMP (ping) | ✅ Bloqueado (ACL 101) |
| Bloqueo DMZ → LAN | Web_DMZ | PC_Internal | ICMP (ping) | ✅ Bloqueado (ACL DMZ) |
| Conectividad LAN → Externa | PC_Internal | PC_External | ICMP (ping) | ✅ Exitoso |

> Las capturas de evidencia (pings, acceso HTTP, salidas de `show ip nat translations`) se encuentran en el directorio `/capturas` del repositorio.

---

### 6. Conclusiones y recomendaciones

- La arquitectura DMZ de tres zonas logra un aislamiento efectivo: incluso si el servidor web fuera comprometido, el atacante no puede alcanzar la red interna gracias a la ACL `BLOQUEAR_DMZ_A_LAN`.
- El NAT estático es la técnica adecuada para exponer servicios públicos sin revelar la topología interna ni abrir rangos amplios de puertos.
- La palabra clave `established` en la ACL es clave: permite que la LAN consuma recursos de la DMZ sin que la DMZ pueda iniciar conexiones de forma autónoma.
- **Recomendación:** Verificar conectividad IP básica (ping entre interfaces del router) antes de aplicar ACLs, ya que un error de direccionamiento previo puede hacer que las reglas parezcan incorrectas cuando el problema es otro.
- **Recomendación:** En un entorno real, complementar esta configuración con un IDS/IPS en la DMZ y logs de las ACLs (`ip access-list log`) para auditoría.

---

### 7. Capturas de evidencia

Las siguientes capturas se adjuntan como evidencia del funcionamiento correcto de la práctica:

| Archivo | Descripción |
|---------|-------------|
| `capturas/pc_internal_ip.png` | Configuración IP de PC_Internal |
| `capturas/web_dmz_ip.png` | Configuración IP del servidor Web_DMZ |
| `capturas/pc_external_ip.png` | Configuración IP de PC_External |
| `capturas/router_config.png` | Configuración de interfaces y ACLs en Router_FW |
| `capturas/topologia.png` | Diagrama de topología en Cisco Packet Tracer |

---

*Práctica de laboratorio — Redes y Seguridad Informática · Cisco Packet Tracer*
