# Configuración VPN — Laboratorio (ISP R1 + R2/R3/R4)

## Esquema de direccionamiento

| Segmento | Red | Gateway | Host |
|---|---|---|---|
| VLAN 10 — Windows10-1 (Cliente, tras R2) | 10.13.67.0/24 | R2 f0/1 = 10.13.67.1 | 10.13.67.10 |
| VLAN 20 — WindowsServer2022-1 (Server, tras R3) | 20.13.67.0/24 | R3 f0/1 = 20.13.67.1 | 20.13.67.10 |
| VLAN 30 — Pc1/VPCS (Cliente2, tras R4) | 30.13.67.0/24 | R4 f0/1 = 30.13.67.1 | 30.13.67.10 |
| Público (ISP R1) | 200.13.67.0/24 en /30 | R1↔R2: .0/30 · R1↔R3: .4/30 · R1↔R4: .8/30 | — |

**VPN:**
- **Client-to-Site L2TP/IPsec**: Windows10-1 → R3 (LNS). Usuario `2025` / clave `1367`.
- **Site-to-Site IKEv1**: túnel permanente R3 ↔ R4 (VLAN20 ↔ VLAN30).
- R1 es solo tránsito ISP, sin protocolo de enrutamiento y sin participar en ninguna VPN.

---

# 1. Dispositivos Intermediarios (Routers y Switches)

## R1 — ISP

```
enable
configure terminal
hostname R1

interface f0/0
 ip address 200.13.67.1 255.255.255.252
 no shutdown

interface f0/1
 ip address 200.13.67.5 255.255.255.252
 no shutdown

interface f1/0
 ip address 200.13.67.9 255.255.255.252
 no shutdown

end
write memory
```

## R2 — Router Cliente (VLAN 10)

```
enable
configure terminal
hostname R2

interface f0/0
 ip address 200.13.67.2 255.255.255.252
 ip nat outside
 no shutdown

interface f0/1
 ip address 10.13.67.1 255.255.255.0
 ip nat inside
 no shutdown

ip route 0.0.0.0 0.0.0.0 200.13.67.1

access-list 1 permit 10.13.67.0 0.0.0.255
ip nat inside source list 1 interface f0/0 overload

ip dhcp excluded-address 10.13.67.1
ip dhcp pool POOL_WIN10
 network 10.13.67.0 255.255.255.0
 default-router 10.13.67.1

end
write memory
```

## R3 — Router Server (VLAN 20) + LNS L2TP + extremo IKEv1

```
enable
configure terminal
hostname R3

interface f0/0
 ip address 200.13.67.6 255.255.255.252
 no shutdown

interface f0/1
 ip address 20.13.67.1 255.255.255.0
 no shutdown

ip route 0.0.0.0 0.0.0.0 200.13.67.5

! --- VPN Site-to-Site IKEv1 (R3 <-> R4) ---
crypto isakmp policy 10
 encr aes 256
 hash sha
 authentication pre-share
 group 2
crypto isakmp key Clave_S2S_123 address 200.13.67.10

crypto ipsec transform-set TS_S2S esp-aes 256 esp-sha-hmac

access-list 101 permit ip 20.13.67.0 0.0.0.255 30.13.67.0 0.0.0.255

crypto map MAPA_VPN 10 ipsec-isakmp
 set peer 200.13.67.10
 set transform-set TS_S2S
 match address 101

! --- VPN Client-to-Site L2TP/IPsec (Windows10-1 -> R3) ---
aaa new-model
aaa authentication ppp VPDN_AUTH local
aaa authorization network VPDN_AUTH local
username 2025 password 1367

vpdn enable
vpdn-group L2TP_GRUPO
 accept-dialin
  protocol l2tp
  virtual-template 1
 no l2tp tunnel authentication

crypto isakmp policy 20
 encr 3des
 hash sha
 authentication pre-share
 group 2
crypto isakmp key ClaveL2TP_123 address 0.0.0.0 0.0.0.0

crypto ipsec transform-set TS_L2TP esp-3des esp-sha-hmac
 mode transport

crypto dynamic-map MAPA_DIN 5
 set transform-set TS_L2TP

crypto map MAPA_VPN 20 ipsec-isakmp dynamic MAPA_DIN

interface f0/0
 crypto map MAPA_VPN

ip local pool POOL_L2TP 20.13.67.100 20.13.67.150

interface Virtual-Template1
 ip unnumbered f0/1
 peer default ip address pool POOL_L2TP
 ppp authentication ms-chap-v2 VPDN_AUTH

end
write memory
```

## R4 — Router Cliente2 (VLAN 30) + extremo IKEv1

```
enable
configure terminal
hostname R4

interface f0/0
 ip address 200.13.67.10 255.255.255.252
 no shutdown

interface f0/1
 ip address 30.13.67.1 255.255.255.0
 no shutdown

ip route 0.0.0.0 0.0.0.0 200.13.67.9

crypto isakmp policy 10
 encr aes 256
 hash sha
 authentication pre-share
 group 2
crypto isakmp key Clave_S2S_123 address 200.13.67.6

crypto ipsec transform-set TS_S2S esp-aes 256 esp-sha-hmac

access-list 101 permit ip 30.13.67.0 0.0.0.255 20.13.67.0 0.0.0.255

crypto map MAPA_VPN 10 ipsec-isakmp
 set peer 200.13.67.6
 set transform-set TS_S2S
 match address 101

interface f0/0
 crypto map MAPA_VPN

ip dhcp excluded-address 30.13.67.1
ip dhcp pool POOL_PC1
 network 30.13.67.0 255.255.255.0
 default-router 30.13.67.1

end
write memory
```

## Switch1 — tras R2 (VLAN 10)

```
enable
configure terminal
hostname Switch1

vlan 10
 name VLAN10_Cliente

interface e0/0
 switchport mode access
 switchport access vlan 10
 no shutdown

interface e0/1
 switchport mode access
 switchport access vlan 10
 no shutdown

end
write memory
```

## Switch2 — tras R3 (VLAN 20)

```
enable
configure terminal
hostname Switch2

vlan 20
 name VLAN20_Server

interface e0/0
 switchport mode access
 switchport access vlan 20
 no shutdown

interface e0/1
 switchport mode access
 switchport access vlan 20
 no shutdown

end
write memory
```

## Switch3 — tras R4 (VLAN 30)

```
enable
configure terminal
hostname Switch3

vlan 30
 name VLAN30_Cliente2

interface e0/0
 switchport mode access
 switchport access vlan 30
 no shutdown

interface e0/1
 switchport mode access
 switchport access vlan 30
 no shutdown

end
write memory
```

**Verificación rápida de esta sección:**
```
show ip interface brief
show interfaces status
show crypto isakmp sa
show crypto ipsec sa
show vpdn session
```

---

# 2. Seguridad

> No aplica a R1 (solo ISP de tránsito). Todo lo demás va aquí, agrupado por tema.

## MOTD + endurecimiento general + acceso administrativo — R2

```
enable
configure terminal
hostname R2

banner motd #
ACCESO RESTRINGIDO - Laboratorio VPN
Solo personal autorizado.
#

service password-encryption
no ip http server
no ip http secure-server
enable secret ClaveEnableR2_123

ip domain-name miguel.local
crypto key generate rsa
! 2048
ip ssh version 2

username miguel privilege 15 secret miguel1367

line con 0
 exec-timeout 5 0
line vty 0 4
 transport input ssh
 login local
 exec-timeout 5 0

login block-for 120 attempts 3 within 60
no cdp run

end
write memory
```
> R2 no tiene ruta al NPS (20.13.67.0/24) porque R1 no enruta redes privadas — por eso queda con login local, sin RADIUS.

## MOTD + endurecimiento + RADIUS + ACL servidor — R3

```
enable
configure terminal
hostname R3

banner motd #
ACCESO RESTRINGIDO - Laboratorio VPN
Solo personal autorizado.
#

service password-encryption
no ip http server
no ip http secure-server
enable secret ClaveEnableR3_123

ip domain-name miguel.local
crypto key generate rsa
! 2048
ip ssh version 2

username miguel privilege 15 secret miguel1367

radius-server host 20.13.67.10 auth-port 1812 acct-port 1813 key RadiusKey123
aaa group server radius RADIUS_NPS
 server 20.13.67.10 auth-port 1812 acct-port 1813

aaa authentication login VTY_AUTH group RADIUS_NPS local
aaa authorization exec VTY_AUTH group RADIUS_NPS local

line con 0
 exec-timeout 5 0
line vty 0 4
 transport input ssh
 login authentication VTY_AUTH
 authorization exec VTY_AUTH
 exec-timeout 5 0

login block-for 120 attempts 3 within 60

! Solo HTTP e ICMP llegan al servidor (RADIUS permitido explícitamente)
access-list 110 permit udp any host 20.13.67.10 eq 1812
access-list 110 permit udp any host 20.13.67.10 eq 1813
access-list 110 permit tcp any host 20.13.67.10 eq 80
access-list 110 permit icmp any host 20.13.67.10
access-list 110 deny ip any host 20.13.67.10
access-list 110 permit ip any any

interface f0/1
 ip access-group 110 out

no cdp run

end
write memory
```

## MOTD + endurecimiento + RADIUS — R4

```
enable
configure terminal
hostname R4

banner motd #
ACCESO RESTRINGIDO - Laboratorio VPN
Solo personal autorizado.
#

service password-encryption
no ip http server
no ip http secure-server
enable secret ClaveEnableR4_123

ip domain-name miguel.local
crypto key generate rsa
! 2048
ip ssh version 2

username miguel privilege 15 secret miguel1367

aaa new-model
radius-server host 20.13.67.10 auth-port 1812 acct-port 1813 key RadiusKey123
aaa group server radius RADIUS_NPS
 server 20.13.67.10 auth-port 1812 acct-port 1813

aaa authentication login VTY_AUTH group RADIUS_NPS local
aaa authorization exec VTY_AUTH group RADIUS_NPS local

ip radius source-interface FastEthernet0/1

line con 0
 exec-timeout 5 0
line vty 0 4
 transport input ssh
 login authentication VTY_AUTH
 authorization exec VTY_AUTH
 exec-timeout 5 0

login block-for 120 attempts 3 within 60
no cdp run

end
write memory
```

## Cliente SSH en Windows (IOS antiguo → cifrados obsoletos)

```powershell
notepad $env:USERPROFILE\.ssh\config
```
Contenido:
```
Host 20.13.67.1 30.13.67.1 200.13.67.6 200.13.67.10 200.13.67.2
    KexAlgorithms +diffie-hellman-group1-sha1
    HostKeyAlgorithms +ssh-rsa
    Ciphers +aes128-cbc,3des-cbc,aes192-cbc,aes256-cbc
```

## Port-Security + VTP + DTP + DHCP Snooping + CDP — Switch1

```
enable
configure terminal
hostname Switch1

banner motd #
ACCESO RESTRINGIDO - Laboratorio VPN
Solo personal autorizado.
#

vtp mode transparent
no cdp run

ip dhcp snooping
ip dhcp snooping vlan 10
no ip dhcp snooping information option

interface e0/0
 switchport port-security
 switchport port-security maximum 1
 switchport port-security mac-address sticky
 switchport port-security violation restrict
 switchport nonegotiate
 ip dhcp snooping trust

interface e0/1
 switchport port-security
 switchport port-security maximum 1
 switchport port-security mac-address sticky
 switchport port-security violation restrict
 switchport nonegotiate
 ip dhcp snooping limit rate 10

end
write memory
```

## Port-Security + VTP + DTP + DHCP Snooping + CDP — Switch2

```
enable
configure terminal
hostname Switch2

banner motd #
ACCESO RESTRINGIDO - Laboratorio VPN
Solo personal autorizado.
#

vtp mode transparent
no cdp run

ip dhcp snooping
ip dhcp snooping vlan 20
no ip dhcp snooping information option

interface e0/0
 switchport port-security
 switchport port-security maximum 1
 switchport port-security mac-address sticky
 switchport port-security violation restrict
 switchport nonegotiate
 ip dhcp snooping trust

interface e0/1
 switchport port-security
 switchport port-security maximum 1
 switchport port-security mac-address sticky
 switchport port-security violation restrict
 switchport nonegotiate

end
write memory
```

## Port-Security + VTP + DTP + DHCP Snooping + CDP — Switch3

```
enable
configure terminal
hostname Switch3

banner motd #
ACCESO RESTRINGIDO - Laboratorio VPN
Solo personal autorizado.
#

vtp mode transparent
no cdp run

ip dhcp snooping
ip dhcp snooping vlan 30
no ip dhcp snooping information option

interface e0/0
 switchport port-security
 switchport port-security maximum 1
 switchport port-security mac-address sticky
 switchport port-security violation restrict
 switchport nonegotiate
 ip dhcp snooping trust

interface e0/1
 switchport port-security
 switchport port-security maximum 1
 switchport port-security mac-address sticky
 switchport port-security violation restrict
 switchport nonegotiate
 ip dhcp snooping limit rate 10

end
write memory
```

## DNS Spoofing/Poisoning — WindowsServer2022-1 (rol DNS de AD DS)

```powershell
Set-DnsServerPrimaryZone -Name "miguel.local" -DynamicUpdate Secure
Set-DnsServerPrimaryZone -Name "miguel.local" -SecureSecondaries NoTransfer
Set-DnsServerCache -LockingPercent 100
```

**Verificación de esta sección:**
```
show vtp status
show interfaces e0/1 switchport
show port-security
show ip dhcp snooping binding
show access-lists 110
show cdp
show login
```
```powershell
Get-DnsServerPrimaryZone -Name "miguel.local" | Select DynamicUpdate, SecureSecondaries
```

---

# 3. Dispositivos Finales

## Windows10-1 — Cliente

- Adaptador de red: **DHCP automático** (recibe IP de `POOL_WIN10` en R2).
- Conexión VPN L2TP/IPsec hacia R3:

| Campo | Valor |
|---|---|
| Nombre de conexión | `Miguel-L2PT` |
| Servidor | `200.13.67.6` |
| Tipo VPN | L2TP/IPsec con clave precompartida |
| Clave precompartida | `ClaveL2TP_123` |
| Usuario / Contraseña | `2025` / `1367` |

Ruta: Configuración → Red e Internet → VPN → Agregar una conexión VPN → llenar los campos de arriba → Conectar.

## Pc1 (VPCS) — Cliente2

```
PC1> ip dhcp
```

## WindowsServer2022-1 — Server (AD DS + IIS + NPS)

**IP fija obligatoria:**
```powershell
New-NetIPAddress -InterfaceAlias "Ethernet" -IPAddress 20.13.67.10 -PrefixLength 24 -DefaultGateway 20.13.67.1
Set-DnsClientServerAddress -InterfaceAlias "Ethernet" -ServerAddresses 127.0.0.1
```

**Instalación de roles:** Administrador del servidor → Agregar roles y características → marcar:
- Servicios de dominio de Active Directory (AD DS)
- Servidor web (IIS)
- Servicios de acceso y directivas de redes (NPS)

### AD DS

Promover a controlador de dominio → nuevo bosque `miguel.local` → NetBIOS `MIGUEL`.

```powershell
New-ADOrganizationalUnit -Name "Red" -Path "DC=miguel,DC=local"

New-ADGroup -Name "GG_Nivel15" -GroupScope Global -Path "OU=Red,DC=miguel,DC=local"
New-ADGroup -Name "GG_Nivel10" -GroupScope Global -Path "OU=Red,DC=miguel,DC=local"
New-ADGroup -Name "GG_Nivel1"  -GroupScope Global -Path "OU=Red,DC=miguel,DC=local"

New-ADUser -Name "Miguel Ramirez" -SamAccountName "mramirez" -UserPrincipalName "mramirez@miguel.local" `
  -Path "OU=Red,DC=miguel,DC=local" -AccountPassword (ConvertTo-SecureString "Segura2121..." -AsPlainText -Force) -Enabled $true
New-ADUser -Name "Emily Torres" -SamAccountName "etorres" -UserPrincipalName "etorres@miguel.local" `
  -Path "OU=Red,DC=miguel,DC=local" -AccountPassword (ConvertTo-SecureString "Nivel10Pass!" -AsPlainText -Force) -Enabled $true
New-ADUser -Name "Carlos Gomez" -SamAccountName "cgomez" -UserPrincipalName "cgomez@miguel.local" `
  -Path "OU=Red,DC=miguel,DC=local" -AccountPassword (ConvertTo-SecureString "Nivel1Pass!" -AsPlainText -Force) -Enabled $true

Add-ADGroupMember -Identity "GG_Nivel15" -Members "mramirez"
Add-ADGroupMember -Identity "GG_Nivel10" -Members "etorres"
Add-ADGroupMember -Identity "GG_Nivel1"  -Members "cgomez"
```

| Usuario | Grupo | Nivel Cisco |
|---|---|---|
| mramirez | GG_Nivel15 | 15 |
| etorres | GG_Nivel10 | 10 |
| cgomez | GG_Nivel1 | 1 |

En cada usuario, pestaña **Marcado**: permiso de acceso = "Controlar el acceso a través de NPS".

### IIS

Ya queda instalado con el rol. Verificar: `http://localhost` (local) y `http://20.13.67.10` (remoto).

### NPS

1. `nps.msc` → clic derecho NPS (Local) → **Registrar servidor en Active Directory**.
2. Clientes RADIUS:

| Nombre | IP | Secreto |
|---|---|---|
| R3 | 20.13.67.1 | RadiusKey123 |
| R4 | 30.13.67.1 | RadiusKey123 |

3. Crear 3 directivas de red (orden: 15 → 10 → 1), cada una con:
   - Condición: grupo de Windows correspondiente (`GG_Nivel15`/`10`/`1`)
   - Métodos de autenticación: **MS-CHAP v2** + **PAP/SPAP** (imprescindible para el login SSH)
   - Atributo Cisco `Cisco-AV-Pair` = `shell:priv-lvl=15` / `10` / `1`

```powershell
Get-NetFirewallRule -DisplayGroup "*RADIUS*" | Select DisplayName, Enabled, Profile
# Si faltan:
New-NetFirewallRule -DisplayName "RADIUS Auth" -Direction Inbound -Protocol UDP -LocalPort 1812 -Action Allow
New-NetFirewallRule -DisplayName "RADIUS Acct" -Direction Inbound -Protocol UDP -LocalPort 1813 -Action Allow
```

---

# 4. Verificación / Entregables

```
! Tracert del cliente al servidor
tracert 20.13.67.10                    ! Windows10-1 (VPN conectada)
PC1> trace 20.13.67.10                 ! Pc1

! Direccionamiento IP de la VPN en los clientes
ipconfig /all                          ! Windows10-1 -> adaptador PPP, rango 20.13.67.100-150
PC1> show ip                           ! Pc1 -> 30.13.67.x

! Prueba de la ACL
ping 20.13.67.10                       ! debe responder (ICMP permitido)
! http://20.13.67.10                  ! debe cargar (HTTP permitido)
telnet 20.13.67.10 23                  ! debe fallar (bloqueado)
show access-lists 110                  ! en R3, ver contadores permit/deny

! Autenticación con los 3 usuarios de AD (en R3 y R4)
ssh mramirez@200.13.67.6
ssh etorres@200.13.67.6
ssh cgomez@200.13.67.6
show privilege                         ! 15 / 10 / 1 respectivamente

! Running config de todos los routers
show running-config
```

Acceso al IIS desde Cliente Windows: navegador → `http://20.13.67.10` con la VPN `Miguel-L2PT` conectada.
