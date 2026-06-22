# VPN Hub and Spoke — DMVPN Fase 3 (IKEv2) con EIGRP

---
Link de Youtube: https://youtu.be/kbJSVmgwED8 

## Objetivo

Implementar una VPN **hub-and-spoke punto a multipunto DMVPN Fase 3** utilizando **IPSec IKEv2** y **enrutamiento dinámico EIGRP**. Fase 3 introduce **NHRP redirect/shortcut**: el HUB detecta tráfico subóptimo entre dos spokes y les indica que negocien un túnel directo, lo cual **actualiza dinámicamente el next-hop** en la tabla de rutas. Esto resuelve la limitación de Fase 2, donde el next-hop seguía siendo el HUB incluso tras formar un túnel spoke-to-spoke.

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
<img width="970" height="682" alt="image" src="https://github.com/user-attachments/assets/a1cf1f97-3d70-41f5-adb6-336d9e544d05" />


### Direccionamiento IP (idéntico a Fase 2)

| Dispositivo | Interfaz | Dirección IP        | Descripción              |
|-------------|----------|-----------------------|----------------------------|
| HUB         | e0/1     | 10.11.85.98/30        | WAN → ISP                  |
| HUB         | e0/0     | 10.11.85.65/27        | LAN → PC3                  |
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

## Diferencias clave vs Fase 2

| Característica            | Fase 2 (IKEv1)                  | Fase 3 (IKEv2)                          |
|----------------------------|----------------------------------|-------------------------------------------|
| Protocolo IKE              | IKEv1 (isakmp)                   | IKEv2 (proposal/policy/keyring/profile)   |
| NHRP en HUB                | Sin `ip nhrp redirect`           | **`ip nhrp redirect`**                    |
| NHRP en Spokes             | Sin `ip nhrp shortcut`           | **`ip nhrp shortcut`**                    |
| Next-hop tras spoke-spoke  | No cambia automáticamente        | **Se actualiza dinámicamente**            |
| Comando SA                 | `show crypto isakmp sa`          | `show crypto ikev2 sa`                    |
| Eficiencia con muchos spokes | Menor                          | Mayor                                     |

---

## Parámetros

### IKEv2

| Parámetro              | Valor                       |
|--------------------------|------------------------------|
| Proposal                | PROP-DMVPN                   |
| Cifrado                 | AES-CBC-256                  |
| Integridad               | SHA-256                      |
| Grupo DH                 | 14 (2048-bit)                |
| Keyring                  | KR-DMVPN (wildcard en HUB)   |
| Profile                  | PROF-DMVPN-IKEV2             |
| PSK                      | ITLA2026                     |

### IPSec / NHRP / Túnel

| Parámetro              | Valor                          |
|-------------------------|---------------------------------|
| Transform Set            | TS-DMVPN-IKEV2                  |
| Modo                     | Transport                       |
| IPSec Profile             | PROF-DMVPN-VTI (+ set ikev2-profile) |
| Tunnel mode               | gre multipoint                  |
| NHRP redirect (HUB)        | Habilitado                     |
| NHRP shortcut (Spokes)      | Habilitado                     |

### EIGRP

| Parámetro              | Valor                                          |
|-------------------------|--------------------------------------------------|
| AS Number                | 1                                               |
| Auto-summary              | Deshabilitado (`no auto-summary`)              |
| Extra en HUB                | `no ip split-horizon eigrp 1` + `no ip next-hop-self eigrp 1` |
| Extra en Spokes             | Ninguno                                         |

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
interface Ethernet0/1
 ip address 10.11.85.98 255.255.255.252
 no shutdown
interface Ethernet0/0
 ip address 10.11.85.65 255.255.255.224
 no shutdown
!
ip route 0.0.0.0 0.0.0.0 10.11.85.97
!
crypto ikev2 proposal PROP-DMVPN
 encryption aes-cbc-256
 integrity sha256
 group 14
!
crypto ikev2 policy POL-DMVPN
 proposal PROP-DMVPN
!
crypto ikev2 keyring KR-DMVPN
 peer ANY-SPOKE
  address 0.0.0.0 0.0.0.0
  pre-shared-key ITLA2026
!
crypto ikev2 profile PROF-DMVPN-IKEV2
 match identity remote address 0.0.0.0
 authentication remote pre-share
 authentication local pre-share
 keyring local KR-DMVPN
!
crypto ipsec transform-set TS-DMVPN-IKEV2 esp-aes 256 esp-sha256-hmac
 mode transport
!
crypto ipsec profile PROF-DMVPN-VTI
 set transform-set TS-DMVPN-IKEV2
 set ikev2-profile PROF-DMVPN-IKEV2
!
interface Tunnel0
 ip address 10.10.10.1 255.255.255.224
 no ip redirects
 ip mtu 1400
 ip nhrp authentication ITLA-DMVPN
 ip nhrp map multicast dynamic
 ip nhrp network-id 85185
 ip nhrp holdtime 300
 ip nhrp redirect
 no ip split-horizon eigrp 1
 no ip next-hop-self eigrp 1
 tunnel source Ethernet0/1
 tunnel mode gre multipoint
 tunnel key 85185
 tunnel protection ipsec profile PROF-DMVPN-VTI
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
crypto ikev2 proposal PROP-DMVPN
 encryption aes-cbc-256
 integrity sha256
 group 14
!
crypto ikev2 policy POL-DMVPN
 proposal PROP-DMVPN
!
crypto ikev2 keyring KR-DMVPN
 peer HUB
  address 10.11.85.98
  pre-shared-key ITLA2026
!
crypto ikev2 profile PROF-DMVPN-IKEV2
 match identity remote address 10.11.85.98 255.255.255.255
 authentication remote pre-share
 authentication local pre-share
 keyring local KR-DMVPN
!
crypto ipsec transform-set TS-DMVPN-IKEV2 esp-aes 256 esp-sha256-hmac
 mode transport
!
crypto ipsec profile PROF-DMVPN-VTI
 set transform-set TS-DMVPN-IKEV2
 set ikev2-profile PROF-DMVPN-IKEV2
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
 ip nhrp shortcut
 tunnel source Ethernet0/0
 tunnel mode gre multipoint
 tunnel key 85185
 tunnel protection ipsec profile PROF-DMVPN-VTI
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
crypto ikev2 proposal PROP-DMVPN
 encryption aes-cbc-256
 integrity sha256
 group 14
!
crypto ikev2 policy POL-DMVPN
 proposal PROP-DMVPN
!
crypto ikev2 keyring KR-DMVPN
 peer HUB
  address 10.11.85.98
  pre-shared-key ITLA2026
!
crypto ikev2 profile PROF-DMVPN-IKEV2
 match identity remote address 10.11.85.98 255.255.255.255
 authentication remote pre-share
 authentication local pre-share
 keyring local KR-DMVPN
!
crypto ipsec transform-set TS-DMVPN-IKEV2 esp-aes 256 esp-sha256-hmac
 mode transport
!
crypto ipsec profile PROF-DMVPN-VTI
 set transform-set TS-DMVPN-IKEV2
 set ikev2-profile PROF-DMVPN-IKEV2
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
 ip nhrp shortcut
 tunnel source Ethernet0/0
 tunnel mode gre multipoint
 tunnel key 85185
 tunnel protection ipsec profile PROF-DMVPN-VTI
 no shutdown
!
router eigrp 1
 no auto-summary
 network 10.10.10.0 0.0.0.31
 network 10.11.85.32 0.0.0.31
```

### VPCS

```
! PC4
ip 10.11.85.34 255.255.255.224 10.11.85.33
save

! PC5
ip 10.11.85.2 255.255.255.224 10.11.85.1
save

! PC6
ip 10.11.85.66 255.255.255.224 10.11.85.65
save
```

---

## Verificación — orden recomendado

```
! 1. En ISP: confirmar rutas estáticas
show ip route static

! 2. Conectividad WAN básica
ping 10.11.85.98

! 3. IKEv2 SA (no isakmp)
show crypto ikev2 sa

! 4. Registro NHRP
show ip nhrp
```
<img width="1010" height="565" alt="image" src="https://github.com/user-attachments/assets/bcd8aeaf-c08e-42ae-be6d-9e8cf8c316ed" />
```
! 5. Estado de la nube DMVPN
show dmvpn
```
<img width="991" height="400" alt="image" src="https://github.com/user-attachments/assets/06dab730-9aeb-4b71-9cff-5141572460f5" />

```
! 6. Estado del túnel
show interface Tunnel0

! 7. Vecinos EIGRP
show ip eigrp neighbors

! 8. Rutas EIGRP aprendidas
show ip route eigrp

! 9. IPSec SA + contadores
show crypto ipsec sa

```

---

## Demostración del shortcut 

1. Desde PC1 (tras SPOKE2), hacer ping sostenido a PC2 (tras SPOKE1):
   ```
   ping 10.11.85.2 repeat 30
   ```
2. **Antes** del tráfico sostenido, en SPOKE2:
   ```
   show ip route 10.11.85.0
   ```
   El next-hop debe mostrar la IP de túnel del HUB (`10.10.10.1`).
3. **Después** del tráfico sostenido, repetir el comando: el next-hop debería cambiar a la IP de túnel de SPOKE1 (`10.10.10.2`), demostrando el túnel directo spoke-to-spoke.
4. `show dmvpn` en SPOKE2 mostrará una segunda entrada dinámica, además del HUB.

    <img width="761" height="438" alt="image" src="https://github.com/user-attachments/assets/fe323c49-fde7-4881-8718-94da87796147" />

    <img width="775" height="226" alt="image" src="https://github.com/user-attachments/assets/85c3c186-d10b-4101-87a1-f53fdba8f8cb" />



---
