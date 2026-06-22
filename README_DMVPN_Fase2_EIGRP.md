# VPN Hub and Spoke — DMVPN Fase 2 (IKEv1) con EIGRP

**Estudiante:** Euni | **Matrícula:** 2024-1185  
**Institución:** Instituto Tecnológico de las Américas (ITLA)  
**Asignatura:** Seguridad de Redes

---

## Objetivo

Implementar una VPN **hub-and-spoke punto a multipunto DMVPN Fase 2** utilizando **IPSec IKEv1** y **enrutamiento dinámico EIGRP**. DMVPN Fase 2 introduce **mGRE (multipoint GRE)** y **NHRP** (Next Hop Resolution Protocol), permitiendo que el HUB administre dinámicamente las direcciones públicas de los spokes. EIGRP se encarga de propagar las rutas de las LANs entre HUB y Spokes a través del túnel.

---

## Topología

```
                  [PC2]
                    │ e0
                 [SPOKE1]
                e0/1 │ │ e0/0
            10.11.85.1   │
                          │
[PC1]──[SPOKE2]────────[ISP]────────[HUB]──[PC3]
   e0  e0/1  e0/0      e0/0  e0/1  e0/0  e0/1  e0
       10.11.85.33              10.11.85.98  10.11.85.65
```

### Direccionamiento IP

| Dispositivo | Interfaz | Dirección IP        | Descripción              |
|-------------|----------|-----------------------|----------------------------|
| HUB         | e0/0     | 10.11.85.98/30        | WAN → ISP                  |
| HUB         | e0/1     | 10.11.85.65/27        | LAN → PC3                  |
| SPOKE1      | e0/0     | 10.11.85.102/30       | WAN → ISP                  |
| SPOKE1      | e0/1     | 10.11.85.1/27         | LAN → PC2                  |
| SPOKE2      | e0/0     | 10.11.85.106/30       | WAN → ISP                  |
| SPOKE2      | e0/1     | 10.11.85.33/27        | LAN → PC1                  |
| ISP         | e0/0     | 10.11.85.97/30        | Hacia HUB                  |
| ISP         | e0/1     | 10.11.85.101/30       | Hacia SPOKE1                |
| ISP         | e0/2     | 10.11.85.105/30       | Hacia SPOKE2                |
| PC1         | e0       | 10.11.85.34/27        | GW: 10.11.85.33 (SPOKE2)   |
| PC2         | e0       | 10.11.85.2/27         | GW: 10.11.85.1 (SPOKE1)    |
| PC3         | e0       | 10.11.85.66/27        | GW: 10.11.85.65 (HUB)      |

### Red NBMA del túnel

| Dispositivo | IP Tunnel0    | Rol en DMVPN          |
|-------------|----------------|--------------------------|
| HUB         | 10.10.10.1/27  | NHS (Next Hop Server)   |
| SPOKE1      | 10.10.10.2/27  | Spoke                    |
| SPOKE2      | 10.10.10.3/27  | Spoke                    |

---

## ⚠️ Causa raíz de fallos comunes (corregido en esta versión)

| Síntoma | Causa | Solución aplicada |
|---|---|---|
| No hay ping ISP↔HUB, ISP↔Spokes, ni entre PCs | El ISP no tenía rutas hacia las LANs (`10.11.85.0/27`, `.32/27`, `.64/27`) | Se agregaron 3 rutas estáticas en el ISP apuntando a la IP WAN de cada router |
| Ping ISP↔WAN sí funcionaba | El ISP solo conocía sus interfaces directamente conectadas | — |
| Riesgo de rutas resumidas incorrectamente | EIGRP hace auto-resumen por defecto (classful) | Se agregó `no auto-summary` en los 3 routers |

---

## Parámetros

### IKEv1 (ISAKMP)

| Parámetro       | Valor              |
|-----------------|--------------------|
| Política        | 10                 |
| Cifrado         | AES-256            |
| Hash            | SHA-256            |
| Autenticación   | PSK (wildcard `0.0.0.0 0.0.0.0`) |
| Grupo DH        | 14 (2048-bit)      |
| Lifetime        | 86400 segundos     |
| Clave PSK       | `ITLA2024`         |

> La clave ISAKMP usa wildcard porque el HUB no conoce de antemano la IP pública de cada spoke que se conectará.

### IPSec / NHRP / Túnel

| Parámetro              | Valor                          |
|-------------------------|---------------------------------|
| Transform Set            | TS-DMVPN                       |
| Modo                     | Transport                      |
| IPSec Profile             | PROF-DMVPN                     |
| Tunnel mode                | gre multipoint                 |
| NHRP network-id            | 85185                          |
| NHRP authentication        | ITLA-DMVPN                     |
| Tunnel key                  | 85185                          |

### EIGRP

| Parámetro              | Valor                                          |
|-------------------------|--------------------------------------------------|
| AS Number                | 1                                               |
| Auto-summary              | Deshabilitado (`no auto-summary`)              |
| Comandos extra en HUB      | `no ip split-horizon eigrp 1` + `no ip next-hop-self eigrp 1` en Tunnel0 |
| Comandos extra en Spokes   | Ninguno (solo van en el HUB)                   |

> **¿Por qué estos dos comandos solo en el HUB?**
> - `no ip split-horizon eigrp 1`: por defecto, EIGRP no reenvía por la misma interfaz una ruta aprendida por ella. En DMVPN, el HUB aprende la LAN de SPOKE1 por Tunnel0 y necesita reenviarla a SPOKE2 por la MISMA interfaz Tunnel0 — sin desactivar split-horizon, esto no ocurre.
> - `no ip next-hop-self eigrp 1`: por defecto, el HUB se anuncia a sí mismo como next-hop de las rutas que reenvía. Esto rompería la posibilidad de spoke-to-spoke directo, porque el spoke pensaría que debe enviar el tráfico al HUB en vez de directamente al otro spoke (vía NHRP).

---

## Configuración

### ISP

```
hostname ISP
!
interface Ethernet0/0
 ip address 10.11.85.97 255.255.255.252
 no shutdown
interface Ethernet0/1
 ip address 10.11.85.101 255.255.255.252
 no shutdown
interface Ethernet0/2
 ip address 10.11.85.105 255.255.255.252
 no shutdown
!
ip route 10.11.85.64 255.255.255.224 10.11.85.98
ip route 10.11.85.0  255.255.255.224 10.11.85.102
ip route 10.11.85.32 255.255.255.224 10.11.85.106
```

### HUB

```
hostname HUB
!
interface Ethernet0/0
 ip address 10.11.85.98 255.255.255.252
 no shutdown
interface Ethernet0/1
 ip address 10.11.85.65 255.255.255.224
 no shutdown
!
ip route 0.0.0.0 0.0.0.0 10.11.85.97
!
crypto isakmp policy 10
 encr aes 256
 hash sha256
 authentication pre-share
 group 14
 lifetime 86400
crypto isakmp key ITLA2024 address 0.0.0.0 0.0.0.0
!
crypto ipsec transform-set TS-DMVPN esp-aes 256 esp-sha256-hmac
 mode transport
!
crypto ipsec profile PROF-DMVPN
 set transform-set TS-DMVPN
!
interface Tunnel0
 ip address 10.10.10.1 255.255.255.224
 no ip redirects
 ip mtu 1400
 ip nhrp authentication ITLA-DMVPN
 ip nhrp map multicast dynamic
 ip nhrp network-id 85185
 ip nhrp holdtime 300
 no ip split-horizon eigrp 1
 no ip next-hop-self eigrp 1
 tunnel source Ethernet0/0
 tunnel mode gre multipoint
 tunnel key 85185
 tunnel protection ipsec profile PROF-DMVPN
 no shutdown
!
router eigrp 1
 no auto-summary
 network 10.10.10.0 0.0.0.31
 network 10.11.85.64 0.0.0.31
```

### SPOKE1

```
hostname SPOKE1
!
interface Ethernet0/0
 ip address 10.11.85.102 255.255.255.252
 no shutdown
interface Ethernet0/1
 ip address 10.11.85.1 255.255.255.224
 no shutdown
!
ip route 0.0.0.0 0.0.0.0 10.11.85.101
!
crypto isakmp policy 10
 encr aes 256
 hash sha256
 authentication pre-share
 group 14
 lifetime 86400
crypto isakmp key ITLA2024 address 0.0.0.0 0.0.0.0
!
crypto ipsec transform-set TS-DMVPN esp-aes 256 esp-sha256-hmac
 mode transport
!
crypto ipsec profile PROF-DMVPN
 set transform-set TS-DMVPN
!
interface Tunnel0
 ip address 10.10.10.2 255.255.255.224
 no ip redirects
 ip mtu 1400
 ip nhrp authentication ITLA-DMVPN
 ip nhrp map 10.10.10.1 10.11.85.98
 ip nhrp map multicast 10.11.85.98
 ip nhrp network-id 85185
 ip nhrp holdtime 300
 ip nhrp nhs 10.10.10.1
 tunnel source Ethernet0/0
 tunnel mode gre multipoint
 tunnel key 85185
 tunnel protection ipsec profile PROF-DMVPN
 no shutdown
!
router eigrp 1
 no auto-summary
 network 10.10.10.0 0.0.0.31
 network 10.11.85.0 0.0.0.31
```

### SPOKE2

```
hostname SPOKE2
!
interface Ethernet0/0
 ip address 10.11.85.106 255.255.255.252
 no shutdown
interface Ethernet0/1
 ip address 10.11.85.33 255.255.255.224
 no shutdown
!
ip route 0.0.0.0 0.0.0.0 10.11.85.105
!
crypto isakmp policy 10
 encr aes 256
 hash sha256
 authentication pre-share
 group 14
 lifetime 86400
crypto isakmp key ITLA2024 address 0.0.0.0 0.0.0.0
!
crypto ipsec transform-set TS-DMVPN esp-aes 256 esp-sha256-hmac
 mode transport
!
crypto ipsec profile PROF-DMVPN
 set transform-set TS-DMVPN
!
interface Tunnel0
 ip address 10.10.10.3 255.255.255.224
 no ip redirects
 ip mtu 1400
 ip nhrp authentication ITLA-DMVPN
 ip nhrp map 10.10.10.1 10.11.85.98
 ip nhrp map multicast 10.11.85.98
 ip nhrp network-id 85185
 ip nhrp holdtime 300
 ip nhrp nhs 10.10.10.1
 tunnel source Ethernet0/0
 tunnel mode gre multipoint
 tunnel key 85185
 tunnel protection ipsec profile PROF-DMVPN
 no shutdown
!
router eigrp 1
 no auto-summary
 network 10.10.10.0 0.0.0.31
 network 10.11.85.32 0.0.0.31
```

### VPCS

```
! PC1
ip 10.11.85.34 255.255.255.224 10.11.85.33
save

! PC2
ip 10.11.85.2 255.255.255.224 10.11.85.1
save

! PC3
ip 10.11.85.66 255.255.255.224 10.11.85.65
save
```

---

## Verificación — orden recomendado

```
! 1. En ISP: confirmar las 3 rutas estáticas
show ip route static

! 2. Conectividad WAN básica (router a router, sin túnel)
ping 10.11.85.98     ! desde SPOKE1/SPOKE2 -> HUB

! 3. IKE Fase 1 establecida
show crypto isakmp sa

! 4. Registro NHRP (en HUB deben verse 2 spokes)
show ip nhrp

! 5. Estado de la nube DMVPN
show dmvpn

! 6. Estado del túnel
show interface Tunnel0

! 7. Vecinos EIGRP (en HUB: SPOKE1 10.10.10.2 y SPOKE2 10.10.10.3)
show ip eigrp neighbors

! 8. Rutas EIGRP aprendidas
show ip route eigrp

! 9. Pruebas finales end-to-end
ping 10.11.85.66     ! PC1 -> PC3
ping 10.11.85.2      ! PC1 -> PC2 (spoke-to-spoke)

! 10. Hora para el video
show clock
```

---

## Guía de diagnóstico (si algo no conecta)

| Paso | Verificación | Si falla, revisar |
|---|---|---|
| 1 | `show ip route static` en ISP | ¿Existen las 3 rutas hacia las LANs? |
| 2 | Ping WAN pública (HUB↔Spoke) | Rutas default + rutas del ISP |
| 3 | `show crypto isakmp sa` → `QM_IDLE` | PSK idéntica + wildcard `0.0.0.0 0.0.0.0` en los 3 routers |
| 4 | `show ip nhrp` en HUB | `ip nhrp nhs 10.10.10.1` en spokes + `tunnel key 85185` idéntico |
| 5 | `show ip eigrp neighbors` | `network` con wildcard correcta (`0.0.0.31` = /27) en los 3 routers |
| 6 | Rutas LAN-a-LAN entre spokes | `no ip split-horizon eigrp 1` **y** `no ip next-hop-self eigrp 1` en Tunnel0 del HUB (ambos comandos, no solo uno) |

---

## Contramédidas

| Amenaza                       | Contramédida                                                |
|--------------------------------|---------------------------------------------------------------|
| Interceptación de tráfico       | AES-256 cifra todo el tráfico GRE entre spokes y hub          |
| Modificación de paquetes        | HMAC-SHA-256 verifica integridad                              |
| Suplantación de spokes          | NHRP authentication (`ITLA-DMVPN`) + PSK + tunnel key 85185   |
| Acceso no autorizado al túnel   | PSK requerida + autenticación IKE antes de admitir el spoke   |
| Saturación de NHS (HUB)         | NHRP holdtime limita registros obsoletos (300s)               |

---

*Documento generado para fines académicos — ITLA 2024-1185*
