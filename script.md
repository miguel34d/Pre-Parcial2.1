# Laboratorio Final - Configuracion
**Estudiante:** Miguel Ramirez Meli - Matricula 2025-1367

---

## 1. Direccionamiento IP

| Enlace / Red | Direccion | Interfaz |
|---|---|---|
| ISP (R1) <-> Peer-A (R2) | 200.13.67.0/30 | R1 f0/0 = .1 / R2 f0/0 = .2 |
| ISP (R1) <-> Peer-B (R3) | 200.13.67.4/30 | R1 f0/1 = .5 / R3 f0/0 = .6 |
| ISP (R1) <-> Internet (Cloud1) | 10.10.10.0/24 | R1 f1/0 = .2 |
| LAN Servidor (VLAN 10) | 10.13.67.0/24 | R2 f0/1 = .1 (gateway) |
| LAN Cliente (VLAN 20) | 172.13.67.0/24 | R3 f0/1 = .1 (gateway) |
| WindowsServer2022-2 | 10.13.67.10/24 | gateway 10.13.67.1 |
| Windows10-1 (NIC1) | DHCP (pool 172.13.67.0/24) | gateway 172.13.67.1 |
| Tunel GRE | 100.13.67.0/30 | Tu0 R2=.1 / Tu0 R3=.2 |
| Tunel L2TPv3 | 100.13.67.8/30 | Tu1 R2=.9 / Tu1 R3=.10 |
| Usuario / Clave VPN | 2025 / 1367 | (matricula 2025-1367) |

### Claves de dispositivos

| Dispositivo | Enable secret | Consola | Usuario SSH | Clave SSH |
|---|---|---|---|---|
| R1 (ISP) | 20251367 | 20251367 | miguel | 20251367 |
| R2 (Peer-A) | 20251367 | 20251367 | miguel | 20251367 |
| R3 (Peer-B) | 20251367 | 20251367 | miguel | 20251367 |
| Switch1 | 20251367 | 20251367 | miguel | 20251367 |
| Switch2 | 20251367 | 20251367 | miguel | 20251367 |

---

## 2. ISP (R1) - direccionamiento publico, sin protocolo de enrutamiento

```
enable
configure terminal
hostname R1
no ip domain-lookup
banner motd # ACCESO SOLO AUTORIZADO - ISP #
enable secret 20251367
```

```
line console 0
 password 20251367
 login
exit
```

```
interface f0/0
 description Enlace a Peer-A
 ip address 200.13.67.1 255.255.255.252
 no shutdown
exit

interface f0/1
 description Enlace a Peer-B
 ip address 200.13.67.5 255.255.255.252
 no shutdown
exit

interface f1/0
 description Enlace hacia Internet
 ip address 10.10.10.2 255.255.255.0
 no shutdown
exit
```

```
ip route 0.0.0.0 0.0.0.0 10.10.10.1
```

```
end
write memory
```

---

## 3. Peer-A / R2 (LAN Servidor - VLAN 10)

```
enable
configure terminal
hostname R2
no ip domain-lookup
service password-encryption
banner motd # ACCESO SOLO PERSONAL AUTORIZADO #
enable secret 20251367
```

```
line console 0
 password 20251367
 login
 exec-timeout 5 0
line vty 0 4
 transport input ssh
 exec-timeout 5 0
exit
```

```
ip domain-name miguel.local
crypto key generate rsa modulus 2048
ip ssh version 2
```

```
interface f0/0
 description WAN hacia ISP
 ip address 200.13.67.2 255.255.255.252
 ip nat outside
 no shutdown
exit

interface f0/1
 description LAN Servidor VLAN10 hacia Switch1
 ip address 10.13.67.1 255.255.255.0
 ip nat inside
 no shutdown
exit
```

```
ip route 0.0.0.0 0.0.0.0 200.13.67.1
access-list 1 permit 10.13.67.0 0.0.0.255
ip nat inside source list 1 interface f0/0 overload
```

```
aaa new-model
radius-server host 10.13.67.10 auth-port 1812 acct-port 1813 key miguel2025
aaa authentication login default group radius local
aaa authorization exec default group radius local
aaa authorization network default group radius
username miguel privilege 15 secret 20251367
```

```
end
write memory
```

---

## 4. Peer-B / R3 (LAN Cliente - VLAN 20)

```
enable
configure terminal
hostname R3
no ip domain-lookup
service password-encryption
banner motd # ACCESO SOLO PERSONAL AUTORIZADO #
enable secret 20251367
```

```
line console 0
 password 20251367
 login
 exec-timeout 5 0
line vty 0 4
 transport input ssh
 exec-timeout 5 0
exit
```

```
ip domain-name miguel.local
crypto key generate rsa modulus 2048
ip ssh version 2
```

```
interface f0/0
 description WAN hacia ISP
 ip address 200.13.67.6 255.255.255.252
 ip nat outside
 no shutdown
exit

interface f0/1
 description LAN Cliente VLAN20 hacia Switch2
 ip address 172.13.67.1 255.255.255.0
 ip nat inside
 no shutdown
exit
```

```
ip route 0.0.0.0 0.0.0.0 200.13.67.5
access-list 1 permit 172.13.67.0 0.0.0.255
ip nat inside source list 1 interface f0/0 overload
```

```
ip dhcp excluded-address 172.13.67.1 172.13.67.10
ip dhcp pool VLAN20_POOL
 network 172.13.67.0 255.255.255.0
 default-router 172.13.67.1
 dns-server 10.13.67.10
 domain-name miguel.local
exit
```

```
aaa new-model
radius-server host 10.13.67.10 auth-port 1812 acct-port 1813 key miguel2025
aaa authentication login default group radius local
aaa authorization exec default group radius local
username miguel privilege 15 secret 20251367
```

```
end
write memory
```

---

## 5. Switch1 y Switch2 - seguridad de puertos

### Switch1
```
enable
configure terminal
hostname Switch1
no ip domain-lookup
banner motd # ACCESO SOLO AUTORIZADO #
service password-encryption
enable secret 20251367
```

```
vlan 10
 name LAN_SERVIDOR
exit
ip dhcp snooping
ip dhcp snooping vlan 10
```

```
interface range e0/0 - 1
 switchport mode access
 switchport access vlan 10
 spanning-tree portfast
 spanning-tree bpduguard enable
exit

interface e0/1
 switchport port-security
 switchport port-security maximum 1
 switchport port-security mac-address sticky
 switchport port-security violation shutdown
exit

interface e0/0
 ip dhcp snooping trust
 spanning-tree guard root
exit
```

```
line vty 0 4
 transport input ssh
 login local
exit
username miguel privilege 15 secret 20251367
end
write memory
```

### Switch2
```
enable
configure terminal
hostname Switch2
no ip domain-lookup
banner motd # ACCESO SOLO AUTORIZADO #
service password-encryption
enable secret 20251367
```

```
vlan 20
 name LAN_CLIENTE
exit
ip dhcp snooping
ip dhcp snooping vlan 20
```

```
interface range e0/0 - 1
 switchport mode access
 switchport access vlan 20
 spanning-tree portfast
 spanning-tree bpduguard enable
exit

interface e0/1
 switchport port-security
 switchport port-security maximum 1
 switchport port-security mac-address sticky
 switchport port-security violation shutdown
exit

interface e0/0
 ip dhcp snooping trust
 spanning-tree guard root
exit
```

```
line vty 0 4
 transport input ssh
 login local
exit
username miguel privilege 15 secret 20251367
end
write memory
```

---

## 6. VPN Site-to-Site IPsec + GRE

### En R2 (Peer-A)
```
crypto keyring 2025
 pre-shared-key address 200.13.67.6 key 1367
exit

crypto isakmp policy 10
 encryption aes 256
 hash sha
 authentication pre-share
 group 5
exit

crypto ipsec transform-set TS-GRE esp-aes 256 esp-sha-hmac
 mode transport
exit
crypto ipsec profile PROFILE-GRE
 set transform-set TS-GRE
exit

interface Tunnel0
 ip address 100.13.67.1 255.255.255.252
 tunnel source f0/0
 tunnel destination 200.13.67.6
 tunnel protection ipsec profile PROFILE-GRE
 ip access-group ACL_HACIA_SERVIDOR in
exit
ip route 172.13.67.0 255.255.255.0 Tunnel0
```

### En R3 (Peer-B)
```
crypto keyring 2025
 pre-shared-key address 200.13.67.2 key 1367
exit

crypto isakmp policy 10
 encryption aes 256
 hash sha
 authentication pre-share
 group 5
exit

crypto ipsec transform-set TS-GRE esp-aes 256 esp-sha-hmac
 mode transport
exit
crypto ipsec profile PROFILE-GRE
 set transform-set TS-GRE
exit

interface Tunnel0
 ip address 100.13.67.2 255.255.255.252
 tunnel source f0/0
 tunnel destination 200.13.67.2
 tunnel protection ipsec profile PROFILE-GRE
exit
ip route 10.13.67.0 255.255.255.0 Tunnel0
```

### Verificacion
```
show crypto isakmp sa
show crypto ipsec sa
show ip interface brief | include Tunnel
ping 100.13.67.2 source tunnel0
```
Si el tunel queda en `up/down`, generar trafico interesante:
```
interface tunnel0
 shutdown
exit
interface tunnel0
 no shutdown
exit
```

---

## 7. VPN Site-to-Site L2TPv3 + IPsec

### En R2 (Peer-A)
```
crypto keyring 2025-L2TP
 pre-shared-key address 200.13.67.6 key 1367
exit

crypto isakmp policy 20
 encryption aes 256
 hash sha
 authentication pre-share
 group 5
exit

crypto ipsec transform-set TS-L2TP esp-aes 256 esp-sha-hmac
exit
crypto map MAP-L2TP 10 ipsec-isakmp
 set peer 200.13.67.6
 set transform-set TS-L2TP
 match address ACL_TRAFICO_L2TP
exit

ip access-list extended ACL_TRAFICO_L2TP
 permit udp host 200.13.67.2 host 200.13.67.6 eq 1701
exit

pseudowire-class PW-L2TP
 encapsulation l2tpv3
 protocol l2tpv3
 ip local interface FastEthernet0/0
exit

interface Tunnel1
 ip address 100.13.67.9 255.255.255.252
 tunnel source f0/0
 tunnel destination 200.13.67.6
 tunnel mode l2tpv3 unmanaged
 xconnect 200.13.67.6 100 encapsulation l2tpv3 pw-class PW-L2TP
exit

interface f0/0
 crypto map MAP-L2TP
exit
```

### En R3 (Peer-B)
```
crypto keyring 2025-L2TP
 pre-shared-key address 200.13.67.2 key 1367
exit

crypto isakmp policy 20
 encryption aes 256
 hash sha
 authentication pre-share
 group 5
exit

crypto ipsec transform-set TS-L2TP esp-aes 256 esp-sha-hmac
exit
crypto map MAP-L2TP 10 ipsec-isakmp
 set peer 200.13.67.2
 set transform-set TS-L2TP
 match address ACL_TRAFICO_L2TP
exit

ip access-list extended ACL_TRAFICO_L2TP
 permit udp host 200.13.67.6 host 200.13.67.2 eq 1701
exit

pseudowire-class PW-L2TP
 encapsulation l2tpv3
 protocol l2tpv3
 ip local interface FastEthernet0/0
exit

interface Tunnel1
 ip address 100.13.67.10 255.255.255.252
 tunnel source f0/0
 tunnel destination 200.13.67.2
 tunnel mode l2tpv3 unmanaged
 xconnect 200.13.67.2 100 encapsulation l2tpv3 pw-class PW-L2TP
exit

interface f0/0
 crypto map MAP-L2TP
exit
```

### Verificacion
```
show xconnect all
show crypto isakmp sa
show ip interface brief | include Tunnel1
```

> Esta VPN queda como enlace de respaldo/alterno documentado; el enrutamiento entre LANs (seccion 8) usa el tunel GRE (Tunnel0) como principal.

---

## 8. Comunicacion entre LANs solo por VPN + solo HTTP/ICMP al servidor

```
ip access-list extended ACL_HACIA_SERVIDOR
 permit udp host 172.13.67.1 host 10.13.67.10 eq 1812
 permit udp host 172.13.67.1 host 10.13.67.10 eq 1813
 permit icmp any host 10.13.67.10
 permit tcp any host 10.13.67.10 eq 80
 deny ip any host 10.13.67.10
 permit ip any any
exit
```
Ya aplicada en `interface Tunnel0` del R2 (seccion 6).

Las rutas `ip route 172.13.67.0 255.255.255.0 Tunnel0` (en R2) y `ip route 10.13.67.0 255.255.255.0 Tunnel0` (en R3) garantizan que el trafico entre ambas LAN solo viaje por el tunel VPN.

---

## 9. Windows Server 2022 - AD, IIS, NPS

### 9.1 Direccionamiento
```
Panel de control > Redes e Internet > Centro de redes y recursos compartidos
> Cambiar configuracion del adaptador > Ethernet0 > Propiedades > IPv4

IP:            10.13.67.10
Mascara:       255.255.255.0
Gateway:       10.13.67.1
DNS preferido: 127.0.0.1
```

### 9.2 Instalar AD DS y promover a controlador de dominio
```
Administrador del servidor > Panel > Agregar roles y caracteristicas
> Instalacion basada en caracteristicas o roles > Siguiente
> Seleccionar servidor local > Siguiente
> Roles de servidor: marcar "Servicios de dominio de Active Directory"
  > Agregar caracteristicas > Siguiente
> Caracteristicas: Siguiente
> Confirmacion: Instalar > Cerrar
```
```
Administrador del servidor > icono de notificaciones
> Promover este servidor a controlador de dominio

Configuracion de implementacion:
  Agregar un nuevo bosque
  Nombre de dominio raiz: miguel.local
  Siguiente

Opciones del controlador de dominio:
  Nivel funcional del bosque/dominio: el mas alto disponible
  Contrasena DSRM: Segura2121...
  Siguiente

Opciones de DNS: Siguiente
Opciones adicionales: NetBIOS = MIGUEL > Siguiente
Rutas: por defecto > Siguiente
Revisar opciones: Siguiente
Comprobacion de requisitos: Instalar

El servidor reinicia solo
```

### 9.3 Unidades organizativas
```
dsa.msc
Clic derecho en miguel.local > Nuevo > Unidad organizativa > Nombre: Grupos > Aceptar
Clic derecho en miguel.local > Nuevo > Unidad organizativa > Nombre: Usuarios > Aceptar
```

### 9.4 IIS
```
Administrador del servidor > Panel > Agregar roles y caracteristicas
> Roles de servidor: marcar "Servidor Web (IIS)" > Agregar caracteristicas > Siguiente
> Servicios de rol: por defecto > Siguiente
> Instalar > Cerrar
```
Verificacion: `Herramientas administrativas > IIS > Sitios > Default Web Site` debe estar "Started"; abrir `http://localhost` desde el servidor.

### 9.5 Grupos y usuarios (niveles 15 / 10 / 1)

**Grupos**
```
dsa.msc > clic derecho en OU "Grupos" > Nuevo > Grupo
  ADMINS-15  (Ambito: Global, Tipo: Seguridad)
  OPS-10     (Ambito: Global, Tipo: Seguridad)
  USERS-1    (Ambito: Global, Tipo: Seguridad)
```

**Usuarios**

| Nombre completo | Usuario (SamAccountName) | Grupo | Nivel |
|---|---|---|---|
| Miguel Ramirez Meli | mramirez | ADMINS-15 | 15 |
| Ronald Arcangel Nunez | rarcangel | ADMINS-15 | 15 |
| Yolanda Peralta Cruz | yperalta | OPS-10 | 10 |
| Franklin De la Rosa Gomez | fdelarosa | OPS-10 | 10 |
| Carmen Sosa Bautista | csosa | USERS-1 | 1 |
| Elvis Reynoso Tavarez | ereynoso | USERS-1 | 1 |

```
dsa.msc > clic derecho en OU "Usuarios" > Nuevo > Usuario
  Nombre / Apellidos / Nombre de inicio de sesion: (segun tabla)
  Siguiente
  Contrasena: Segura2121...   Confirmar: Segura2121...
  Desmarcar "Debe cambiar la contrasena"
  Marcar "La contrasena nunca expira"
  Siguiente > Finalizar

Repetir para los 6 usuarios de la tabla
```

**Asignar cada usuario a su grupo**
```
dsa.msc > OU "Usuarios" > doble clic en el usuario > pestana "Miembro de"
> Agregar > escribir el nombre del grupo > Comprobar nombres > Aceptar > Aceptar

mramirez, rarcangel   -> ADMINS-15
yperalta, fdelarosa   -> OPS-10
csosa, ereynoso       -> USERS-1
```

### 9.6 NPS (RADIUS)

**Instalar el rol**
```
Administrador del servidor > Panel > Agregar roles y caracteristicas
> Roles de servidor: marcar "Servicios de directivas y acceso de redes"
  > Agregar caracteristicas > Siguiente
> Servicios de rol: "Servidor de directivas de redes" > Siguiente
> Instalar > Cerrar
```

**Clientes RADIUS**
```
nps.msc > Clientes y servidores RADIUS > Clientes RADIUS > Nuevo

Nombre: Peer-A | IP: 200.13.67.2 | Secreto compartido: miguel2025
Nombre: Peer-B | IP: 200.13.67.6 | Secreto compartido: miguel2025
```

**Directivas de red**
```
nps.msc > Directivas > Directivas de red > Nueva

NPS-ADMINS-15
  Condicion: Windows Groups = ADMINS-15
  Conceder acceso
  Vendor Specific > Cisco > cisco-av-pair = shell:priv-lvl=15

NPS-OPS-10
  Condicion: Windows Groups = OPS-10
  cisco-av-pair = shell:priv-lvl=10

NPS-USERS-1
  Condicion: Windows Groups = USERS-1
  cisco-av-pair = shell:priv-lvl=1
```
```
Ordenar de arriba a abajo: NPS-ADMINS-15, NPS-OPS-10, NPS-USERS-1
```

### 9.7 Verificacion
```powershell
Get-ADDomain
Get-ADUser -Filter * -SearchBase "OU=Usuarios,DC=miguel,DC=local"
Get-ADGroup -Filter * -SearchBase "OU=Grupos,DC=miguel,DC=local"
```

---

## 10. Windows10-1 (Cliente - VLAN 20)

```
Panel de control > Redes e Internet > Centro de redes y recursos compartidos
> Cambiar configuracion del adaptador > NIC1 > Propiedades > IPv4
  Obtener una direccion IP automaticamente
  Obtener la direccion del servidor DNS automaticamente
Aceptar
```

**Verificacion**
```powershell
ipconfig /all
ping 10.13.67.10
```
```
Navegador > http://10.13.67.10
```
