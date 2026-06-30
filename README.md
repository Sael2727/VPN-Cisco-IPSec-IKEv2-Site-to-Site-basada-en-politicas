# 🔐 VPN Cisco IPSec IKEv2 Site-to-Site — Basada en Políticas

<div align="center">

![Cisco](https://img.shields.io/badge/Cisco-IOS-blue?style=for-the-badge&logo=cisco)
![IPSec](https://img.shields.io/badge/IPSec-IKEv2-green?style=for-the-badge)
![Policy](https://img.shields.io/badge/VPN-Policy--Based-purple?style=for-the-badge)
![Platform](https://img.shields.io/badge/Platform-PNETLab-orange?style=for-the-badge)
![License](https://img.shields.io/badge/Uso-Educativo-red?style=for-the-badge)

**Sael Germán García** | Matrícula: `2025-0725`  
Asignatura: Seguridad de Redes | Profesor: Jonathan Rondón  
Instituto Tecnológico de las Américas — ITLA | 2026

</div>

---

## 📋 Descripción

Configuración y verificación de una **VPN Cisco IPSec IKEv2 Site-to-Site basada en políticas** para permitir comunicación segura entre dos LANs remotas a través de un router ISP que simula la red pública. A diferencia de una VPN basada en rutas, aquí el tráfico interesante se define mediante una **ACL** referenciada en el crypto map; cuando el tráfico coincide con la ACL, se activa la negociación **IKEv2** y posteriormente **IPSec** cifra los paquetes entre los peers.

> 💡 **Clave técnica:** No existe interfaz de túnel (Tunnel0). La protección se aplica directamente sobre la interfaz WAN (Ethernet0/0) mediante el `crypto map`, y la ACL `VPN-TRAFFIC` decide qué tráfico LAN-a-LAN se cifra.

---

## 🗺️ Topología de Red

La topología conserva dos routers peers (R1 y R2), un router ISP al centro y una LAN en cada extremo. El ISP solo brinda conectividad entre R1 y R2, sin participar en la VPN.

### 📊 Direccionamiento IP

| Dispositivo | Interfaz | IP / Máscara | Función |
|:-----------:|:--------:|:-------------:|---------|
| R1 | Ethernet0/0 | 10.7.25.1/30 | WAN hacia ISP |
| ISP | Ethernet0/0 | 10.7.25.2/30 | Enlace hacia R1 |
| ISP | Ethernet0/1 | 10.7.25.6/30 | Enlace hacia R2 |
| R2 | Ethernet0/0 | 10.7.25.5/30 | WAN hacia ISP |
| R1 | Ethernet0/1 | 10.7.25.65/27 | Gateway LAN-A |
| VPC1 | eth0 | 10.7.25.66/27 | Cliente LAN-A |
| R2 | Ethernet0/1 | 10.7.25.97/27 | Gateway LAN-B |
| VPC2 | eth0 | 10.7.25.98/27 | Cliente LAN-B |

### 📡 Segmentos

| Segmento | Red | Descripción |
|:--------:|:---:|-------------|
| WAN R1-ISP | 10.7.25.0/30 | Enlace entre R1 e ISP |
| WAN ISP-R2 | 10.7.25.4/30 | Enlace entre ISP y R2 |
| LAN-A | 10.7.25.64/27 | Red local conectada a R1 |
| LAN-B | 10.7.25.96/27 | Red local conectada a R2 |

---

## ⚙️ Parámetros de la VPN

| Parámetro | Valor |
|:---------:|-------|
| Versión IKE | IKEv2 |
| Tipo de VPN | Site-to-Site basada en políticas |
| Selección de tráfico | ACL VPN-TRAFFIC |
| Aplicación de IPSec | Crypto map VPN-MAP en Ethernet0/0 |
| Autenticación | Pre-Shared Key |
| Clave compartida | VPN12345 |
| Cifrado IKEv2 | AES-CBC-256 |
| Integridad IKEv2 | SHA256 |
| Grupo Diffie-Hellman | Grupo 5 |
| Transform-set IPSec | TS-IKEV2: esp-aes 256 esp-sha256-hmac |
| Modo IPSec | Tunnel |

---

## 🔍 Funcionamiento

Esta VPN es **basada en políticas** porque utiliza una ACL para identificar el tráfico interesante entre las LANs, en lugar de una interfaz de túnel dedicada:

- En **R1**, la ACL permite tráfico desde `10.7.25.64/27` hacia `10.7.25.96/27`.
- En **R2**, la ACL está invertida: permite tráfico desde `10.7.25.96/27` hacia `10.7.25.64/27`.

Cuando el tráfico coincide con la ACL, el `crypto map` aplicado en la interfaz WAN Ethernet0/0 activa la negociación **IKEv2**. Luego **IPSec** cifra el tráfico entre los peers `10.7.25.1` y `10.7.25.5`, protegiendo la comunicación entre LAN-A y LAN-B.

---

## 🚀 Scripts de Configuración

### ISP
```cisco
hostname ISP
interface Ethernet0/0
 description ENLACE_A_R1
 ip address 10.7.25.2 255.255.255.252
 no shutdown
interface Ethernet0/1
 description ENLACE_A_R2
 ip address 10.7.25.6 255.255.255.252
 no shutdown
```

### R1 — Configuración Principal
```cisco
hostname R1
interface Ethernet0/0
 description WAN_HACIA_ISP
 ip address 10.7.25.1 255.255.255.252
 no shutdown
interface Ethernet0/1
 description LAN_A
 ip address 10.7.25.65 255.255.255.224
 no shutdown
ip route 0.0.0.0 0.0.0.0 10.7.25.2

crypto ikev2 proposal IKEV2-PROP
 encryption aes-cbc-256
 integrity sha256
 group 5

crypto ikev2 policy IKEV2-POLICY
 proposal IKEV2-PROP

crypto ikev2 keyring IKEV2-KEYRING
 peer R2
  address 10.7.25.5
  pre-shared-key local VPN12345
  pre-shared-key remote VPN12345

crypto ikev2 profile IKEV2-PROFILE
 match identity remote address 10.7.25.5 255.255.255.255
 identity local address 10.7.25.1
 authentication remote pre-share
 authentication local pre-share
 keyring local IKEV2-KEYRING

crypto ipsec transform-set TS-IKEV2 esp-aes 256 esp-sha256-hmac
 mode tunnel

ip access-list extended VPN-TRAFFIC
 permit ip 10.7.25.64 0.0.0.31 10.7.25.96 0.0.0.31

crypto map VPN-MAP 10 ipsec-isakmp
 description VPN_IKEV2_HACIA_R2
 set peer 10.7.25.5
 set transform-set TS-IKEV2
 set pfs group5
 set ikev2-profile IKEV2-PROFILE
 match address VPN-TRAFFIC

interface Ethernet0/0
 crypto map VPN-MAP
```

### R2 — Configuración Principal
```cisco
hostname R2
interface Ethernet0/0
 description WAN_HACIA_ISP
 ip address 10.7.25.5 255.255.255.252
 no shutdown
interface Ethernet0/1
 description LAN_B
 ip address 10.7.25.97 255.255.255.224
 no shutdown
ip route 0.0.0.0 0.0.0.0 10.7.25.6

crypto ikev2 proposal IKEV2-PROP
 encryption aes-cbc-256
 integrity sha256
 group 5

crypto ikev2 policy IKEV2-POLICY
 proposal IKEV2-PROP

crypto ikev2 keyring IKEV2-KEYRING
 peer R1
  address 10.7.25.1
  pre-shared-key local VPN12345
  pre-shared-key remote VPN12345

crypto ikev2 profile IKEV2-PROFILE
 match identity remote address 10.7.25.1 255.255.255.255
 identity local address 10.7.25.5
 authentication remote pre-share
 authentication local pre-share
 keyring local IKEV2-KEYRING

crypto ipsec transform-set TS-IKEV2 esp-aes 256 esp-sha256-hmac
 mode tunnel

ip access-list extended VPN-TRAFFIC
 permit ip 10.7.25.96 0.0.0.31 10.7.25.64 0.0.0.31

crypto map VPN-MAP 10 ipsec-isakmp
 description VPN_IKEV2_HACIA_R1
 set peer 10.7.25.1
 set transform-set TS-IKEV2
 set pfs group5
 set ikev2-profile IKEV2-PROFILE
 match address VPN-TRAFFIC

interface Ethernet0/0
 crypto map VPN-MAP
```

### Configuración de VPCs
```bash
# VPC1
ip 10.7.25.66 255.255.255.224 10.7.25.65

# VPC2
ip 10.7.25.98 255.255.255.224 10.7.25.97
```

---

## ✅ Verificación del Túnel

```cisco
show ip interface brief
show running-config | section crypto
show access-lists
show crypto ikev2 sa
show crypto ipsec sa
show crypto session
```

| Comando | Estado esperado |
|:-------:|------------------|
| `show crypto ikev2 sa` | READY |
| `show crypto ipsec sa` | encaps/decaps activos |
| `show crypto session` | UP-ACTIVE |
| `show access-lists` | Matches en VPN-TRAFFIC |

> 💡 **Nota sobre traceroute:** El mensaje "Destination port unreachable" al final del trace es normal en VPCS — usa paquetes UDP, y al llegar al host final este responde que el puerto no está disponible. Esto también confirma que el destino fue alcanzado.

---

## 📊 Resultados

| Prueba | Resultado |
|:------:|:---------:|
| Interfaces WAN y LAN (R1, R2, ISP) | ✅ up/up |
| Configuración IKEv2 (proposal, policy, keyring, profile) | ✅ Presente en ambos peers |
| ACL VPN-TRAFFIC (LAN-A ↔ LAN-B) | ✅ Confirmada en R1 y R2 |
| IKEv2 SA | ✅ READY |
| IPSec SA | ✅ Encaps/decaps activos |
| Sesión crypto | ✅ UP-ACTIVE |
| Ping VPC1 → VPC2 | ✅ Exitoso |
| Ping VPC2 → VPC1 | ✅ Exitoso |

---

## 📁 Archivos del Repositorio

| Archivo | Descripción |
|:-------:|-------------|
| [`VPN-Cisco-IPSec-IKEv2-Site-to-Site-basada-en-políticas-script.txt`](VPN-Cisco-IPSec-IKEv2-Site-to-Site-basada-en-pol%C3%ADticas-script.txt) | Scripts de configuración Cisco IOS |
| [`SaelGerman_2025-0725_VPN-IPSec-IKEv2-Site-to-Site-basada-en-politicas_P2.pdf`](SaelGerman_2025-0725_VPN-IPSec-IKEv2-Site-to-Site-basada-en-politicas_P2.pdf) | Documentación técnica completa |

---

## 🖼️ Capturas de Pantalla

- 📸 [Figura 1 — Topología general de la VPN IKEv2 policy-based](SaelGerman_2025-0725_Capturas_IKEV2_Policy_GitHub/01_topologia_ikev2_policy.png)
- 📸 [Figura 2 — Interfaces del ISP](SaelGerman_2025-0725_Capturas_IKEV2_Policy_GitHub/02_isp_show_ip_interface_brief.png)
- 📸 [Figura 3 — Interfaces de R1](SaelGerman_2025-0725_Capturas_IKEV2_Policy_GitHub/03_r1_show_ip_interface_brief.png)
- 📸 [Figura 4 — Interfaces de R2](SaelGerman_2025-0725_Capturas_IKEV2_Policy_GitHub/04_r2_show_ip_interface_brief.png)
- 📸 [Figura 5 — Configuración crypto en R1](SaelGerman_2025-0725_Capturas_IKEV2_Policy_GitHub/05_r1_show_running_config_section_crypto.png)
- 📸 [Figura 6 — ACL VPN-TRAFFIC en R1](SaelGerman_2025-0725_Capturas_IKEV2_Policy_GitHub/06_r1_show_access_lists.png)
- 📸 [Figura 7 — Configuración crypto en R2](SaelGerman_2025-0725_Capturas_IKEV2_Policy_GitHub/07_r2_show_running_config_section_crypto.png)
- 📸 [Figura 8 — ACL VPN-TRAFFIC en R2](SaelGerman_2025-0725_Capturas_IKEV2_Policy_GitHub/08_r2_show_access_lists.png)
- 📸 [Figura 9 — Prueba desde VPC1 hacia VPC2](SaelGerman_2025-0725_Capturas_IKEV2_Policy_GitHub/09_vpc1_show_ip_ping_trace_to_vpc2.png)
- 📸 [Figura 10 — Prueba desde VPC2 hacia VPC1](SaelGerman_2025-0725_Capturas_IKEV2_Policy_GitHub/10_vpc2_show_ip_ping_trace_to_vpc1.png)
- 📸 [Figura 11 — Estado IKEv2 en R1](SaelGerman_2025-0725_Capturas_IKEV2_Policy_GitHub/11_r1_show_crypto_ikev2_sa.png)
- 📸 [Figura 12 — Estado IKEv2 en R2](SaelGerman_2025-0725_Capturas_IKEV2_Policy_GitHub/12_r2_show_crypto_ikev2_sa.png)
- 📸 [Figura 13 — Contadores IPSec en R1](SaelGerman_2025-0725_Capturas_IKEV2_Policy_GitHub/13_r1_show_crypto_ipsec_sa.png)
- 📸 [Figura 14 — Contadores IPSec en R2](SaelGerman_2025-0725_Capturas_IKEV2_Policy_GitHub/14_r2_show_crypto_ipsec_sa.png)
- 📸 [Figura 15 — Sesión crypto activa en R1](SaelGerman_2025-0725_Capturas_IKEV2_Policy_GitHub/15_r1_show_crypto_session.png)
- 📸 [Figura 16 — Sesión crypto activa en R2](SaelGerman_2025-0725_Capturas_IKEV2_Policy_GitHub/16_r2_show_crypto_session.png)

---

## 📎 Recursos

📄 **Documentación Técnica:** [Ver Informe PDF](SaelGerman_2025-0725_VPN-IPSec-IKEv2-Site-to-Site-basada-en-politicas_P2.pdf)  
▶️ **Video Demostración:** [Ver en YouTube](https://youtu.be/rXf3tFYHeEU)

---

## 📚 Referencias

1. Cisco Systems. *Configuring Site-to-Site IPSec VPN using IKEv2*. Documentación oficial Cisco IOS.
2. Reconocimiento especial: Troubleshooting, base del script y documentación apoyado en Inteligencia Artificial.

---

<div align="center">

*Este laboratorio fue desarrollado exclusivamente con fines académicos y educativos.*

</div>
