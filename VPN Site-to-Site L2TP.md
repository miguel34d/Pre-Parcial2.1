# Laboratorio Final - VPN Site-to-Site L2TP + IPsec
**Estudiante:** Miguel Ramirez Meli - Matricula 2025-1367
**Topologia:** ISP - R1(Peer-A, LAN Servidor) - VPN - R2(Peer-B, LAN Cliente)

> **Aviso tecnico importante:** en IOS clasico (12.4, feature-set `adventerprisek9`), el mecanismo `VPDN` para L2TP fue disenado para acceso remoto (dial-in de clientes PPP), no para tuneles router-a-router. La configuracion de esta seccion es el enfoque estandar mas cercano a "L2TP site-to-site" disponible en este feature-set (LNS/LAC con `vpdn-group` + `Virtual-Template`), pero en topologias GNS3 con c3725 puede no negociar la fase PPP de forma limpia sin un medio de marcado (dialer) real. Si al verificar no sube, es un limite de plataforma, no un error de sintaxis - documentalo asi ante tu profesor y ten `Laboratorio_VPN_IPsec_GRE.md` como la VPN funcional de respaldo.

---

## 1. Plan de direccionamiento IP

| Enlace / Red | Direccion | Interfaz |
|---|---|---|
| R-ISP <-> Peer-A | 200.13.67.0/30 | R-ISP f0/0 = .1 / Peer-A f0/0 = .2 |
| R-ISP <-> Peer-B | 200.13.67.4/30 | R-ISP f0/1 = .5 / Peer-B f0/0 = .6 |
| LAN Servidor (VLAN 10) | 10.13.67.0/24 | Peer-A f0/1 = .1 (gateway) |
| LAN Cliente (VLAN 20) | 172.13.67.0/24 | Peer-B f0/1 = .1 (gateway) |
| WindowsServer2022 | 10.13.67.10/24 | gateway 10.13.67.1 |
| Windows10-1 | DHCP (pool 172.13.67.0/24) | gateway 172.13.67.1 |
| Pool direcciones tunel L2TP | 100.13.67.8/29 (.9-.14 asignables) | asignado por Peer-A via `ip local pool` |
| Credenciales VPN | usuario/hostname L2TP: **2025** / clave/tunnel-password: **1367** | segun matricula 2025-1367 |

> El L2TP tunnel no usa `crypto isakmp key` global; el nombre local VPDN (`local name 2025`) y el `l2tp tunnel password 1367` dejan el usuario/clave de la matricula visible y verificable en la config (`show run | section vpdn`).

---

## 2. R-ISP (solo IP publica, sin protocolo de enrutamiento)

```
enable
configure terminal
hostname R-ISP
no ip domain-lookup
banner motd # ACCESO SOLO AUTORIZADO - ISP #
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
! Sin RIP/OSPF/EIGRP: ambos enlaces estan directamente conectados
end
write memory
```

---

## 3. Peer-A (R1 - lado Servidor / LNS, VLAN 10)

```
enable
configure terminal
hostname Peer-A
no ip domain-lookup
service password-encryption
banner motd # ACCESO SOLO PERSONAL AUTORIZADO - Peer-A #
enable secret 20251367
line console 0
 password 20251367
 login
 exec-timeout 5 0
line vty 0 4
 transport input ssh
 exec-timeout 5 0
exit
ip domain-name miguel.local
crypto key generate rsa modulus 2048
ip ssh version 2
username miguel privilege 15 secret 20251367
```

```
interface f0/0
 description WAN hacia R-ISP
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
```

**ACL - solo HTTP + ICMP hacia el servidor:**
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
> Se aplica sobre `Virtual-Template1` (seccion 5). Razon por la que Windows10-1 no se une al dominio (verificacion en seccion 8).

```
end
write memory
```

---

## 4. IPsec (proteccion de transporte para el trafico L2TP/UDP 1701)

> Rama mainline 12.4: usar `hash sha` y `group 5` (bajar a `group 2` si falla).

**En Peer-A:**
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
crypto ipsec transform-set TS-L2TP esp-aes 256 esp-sha-hmac
 mode transport
exit
crypto map CMAP-L2TP 10 ipsec-isakmp
 set peer 200.13.67.6
 set transform-set TS-L2TP
 match address ACL-L2TP-TRAFFIC
exit
ip access-list extended ACL-L2TP-TRAFFIC
 permit udp host 200.13.67.2 host 200.13.67.6 eq 1701
exit
interface f0/0
 crypto map CMAP-L2TP
exit
```

**En Peer-B:**
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
crypto ipsec transform-set TS-L2TP esp-aes 256 esp-sha-hmac
 mode transport
exit
crypto map CMAP-L2TP 10 ipsec-isakmp
 set peer 200.13.67.2
 set transform-set TS-L2TP
 match address ACL-L2TP-TRAFFIC
exit
ip access-list extended ACL-L2TP-TRAFFIC
 permit udp host 200.13.67.6 host 200.13.67.2 eq 1701
exit
interface f0/0
 crypto map CMAP-L2TP
exit
```

---

## 5. Tunel L2TP (VPDN) - Peer-A como LNS

```
vpdn enable
vpdn-group L2TP-GRP
 accept-dialin
  protocol l2tp
  virtual-template 1
 terminate-from hostname Peer-B
 local name 2025
 l2tp tunnel password 0 1367
exit
ip local pool L2TP-POOL 100.13.67.9 100.13.67.14
interface Virtual-Template1
 ip unnumbered f0/1
 peer default ip address pool L2TP-POOL
 ppp authentication chap
 ip access-group ACL_HACIA_SERVIDOR in
exit
ip route 172.13.67.0 255.255.255.0 Virtual-Template1
username 2025 password 0 1367
end
write memory
```

---

## 6. Peer-B (R2 - lado Cliente / LAC, VLAN 20)

```
enable
configure terminal
hostname Peer-B
no ip domain-lookup
service password-encryption
banner motd # ACCESO SOLO PERSONAL AUTORIZADO - Peer-B #
enable secret 20251367
line console 0
 password 20251367
 login
 exec-timeout 5 0
line vty 0 4
 transport input ssh
 exec-timeout 5 0
exit
ip domain-name miguel.local
crypto key generate rsa modulus 2048
ip ssh version 2
username miguel privilege 15 secret 20251367
```

```
interface f0/0
 description WAN hacia R-ISP
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

**Tunel L2TP (LAC - inicia la marcacion):**
```
vpdn enable
vpdn-group L2TP-GRP
 request-dialin
  protocol l2tp
  ip address 200.13.67.2
 initiate-to ip 200.13.67.2
 local name Peer-B
 l2tp tunnel password 0 1367
exit
interface Dialer1
 ip address negotiated
 encapsulation ppp
 dialer pool 1
 ppp authentication chap
 ppp chap hostname 2025
 ppp chap password 0 1367
exit
interface f0/0
 dialer pool-member 1
exit
ip route 10.13.67.0 255.255.255.0 Dialer1
```

```
aaa new-model
radius-server host 10.13.67.10 auth-port 1812 acct-port 1813 key miguel2025
aaa authentication login default group radius local
aaa authorization exec default group radius local
end
write memory
```

---

## 7. Verificacion del tunel

```
show vpdn tunnel
show vpdn session
show interfaces virtual-access 1
show crypto isakmp sa
show crypto ipsec sa
ping 172.13.67.1 source f0/1
```
`show vpdn tunnel` debe listar la sesion establecida entre `200.13.67.2` y `200.13.67.6`; `show crypto isakmp sa` debe mostrar `QM_IDLE`/`ACTIVE` sobre UDP/1701. Si el tunel no negocia (queda en "no session"), confirma el aviso tecnico al inicio del documento y usa `Laboratorio_VPN_IPsec_GRE.md` como respaldo funcional para la entrega.

---

## 8. Switch1 (Servidor) y Switch2 (Cliente)

```
enable
configure terminal
hostname Switch1
no ip domain-lookup
banner motd # ACCESO SOLO AUTORIZADO #
service password-encryption
enable secret 20251367
vlan 10
 name LAN_SERVIDOR
exit
ip dhcp snooping
ip dhcp snooping vlan 10
no ip dhcp snooping information option
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
line vty 0 4
 transport input ssh
 login local
exit
username miguel privilege 15 secret 20251367
end
write memory
```

```
enable
configure terminal
hostname Switch2
no ip domain-lookup
banner motd # ACCESO SOLO AUTORIZADO #
service password-encryption
enable secret 20251367
vlan 20
 name LAN_CLIENTE
exit
ip dhcp snooping
ip dhcp snooping vlan 20
no ip dhcp snooping information option
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
line vty 0 4
 transport input ssh
 login local
exit
username miguel privilege 15 secret 20251367
end
write memory
```

---

## 9. Windows Server 2022 (AD DS, DNS, IIS, NPS) - Configuracion detallada

**Direccionamiento:** IP 10.13.67.10 /24, gateway 10.13.67.1, DNS 127.0.0.1 (se autoapunta una vez instalado el rol DNS).

```
Administrador del servidor > Panel local > Ethernet > Propiedades
> Protocolo de Internet version 4 (TCP/IPv4) > Propiedades
  IP: 10.13.67.10
  Mascara: 255.255.255.0
  Puerta de enlace: 10.13.67.1
  DNS preferido: 127.0.0.1
```
```powershell
Get-NetIPConfiguration
Test-Connection 10.13.67.1 -Count 2
```

### 9.1 Instalar el rol AD DS

```
Administrador del servidor > Panel > Agregar roles y caracteristicas
> Instalacion basada en caracteristicas o en roles > Siguiente
> Seleccionar el servidor del grupo > Siguiente
> Roles de servidor: marcar "Servicios de dominio de Active Directory"
  (aparece un dialogo pidiendo agregar caracteristicas dependientes -> Agregar caracteristicas)
> Siguiente > Siguiente > Siguiente (pantalla informativa de AD DS) > Instalar
> Esperar a que termine (NO reiniciar todavia) > Cerrar
```

### 9.2 Promover el servidor a controlador de dominio

```
Administrador del servidor > icono de notificaciones (bandera amarilla)
> "Promover este servidor a controlador de dominio"
```
Asistente de configuracion de Servicios de dominio de Active Directory:
```
1) Operacion de implementacion
   Seleccionar: Agregar un nuevo bosque
   Nombre de dominio raiz: miguel.local

2) Opciones del controlador de dominio
   Nivel funcional del bosque y del dominio: Windows Server 2016 (o el mas alto disponible)
   Dejar marcado: DNS Server
   (Catalogo global ya viene marcado por ser el primer DC del bosque)
   Contrasena de restauracion de servicios de directorio (DSRM): Segura2121...

3) Opciones DNS
   Ignorar la advertencia de delegacion (normal: no hay DNS padre)

4) Opciones adicionales
   Nombre NetBIOS del dominio: MIGUEL (se autogenera, confirmar)

5) Rutas
   Dejar las rutas por defecto para NTDS, SYSVOL y logs

6) Revisar opciones > Verificar requisitos previos
   Las advertencias amarillas son normales; no debe haber errores en rojo

7) Instalar
   El servidor reinicia solo al terminar
```

Verificacion tras el reinicio:
```powershell
Get-ADDomain
Get-ADForest
dcdiag /v
Get-Service NTDS, DNS, Netlogon, kdc
nslookup miguel.local 127.0.0.1
```
Todos los servicios deben quedar `Running`; `dcdiag` no debe reportar fallas en las pruebas Connectivity/Advertising/Services.

### 9.3 Verificar el rol DNS (se instala junto con AD DS)

```
Administrador del servidor > Herramientas > DNS
> Expandir SERVIDOR > Zonas de busqueda directa > miguel.local
  Deben existir el registro SOA, los NS y los registros A del propio DC
```
```powershell
Get-DnsServerZone
Resolve-DnsName miguel.local
```

### 9.4 Instalar el rol IIS (Servidor Web)

```
Administrador del servidor > Agregar roles y caracteristicas
> Roles de servidor: marcar "Servidor web (IIS)"
> Siguiente (sin caracteristicas extra) > Siguiente (servicios de rol por defecto) > Instalar
```
```powershell
Get-WindowsFeature Web-Server
Invoke-WebRequest http://localhost -UseBasicParsing | Select-Object StatusCode
```
Desde Windows10-1 (con la VPN arriba), `http://10.13.67.10` debe mostrar la pagina de bienvenida de IIS (coincide con `permit tcp any host 10.13.67.10 eq 80` de la ACL).

### 9.5 Crear la estructura de OUs, grupos y usuarios

**Por GUI (dsa.msc):**
```
Herramientas > Usuarios y equipos de Active Directory
> clic derecho en miguel.local > Nuevo > Unidad organizativa > "Grupos"
> repetir > Nuevo > Unidad organizativa > "Usuarios"

> clic derecho OU "Grupos" > Nuevo > Grupo
  Nombre: ADMINS-15 | Ambito: Global | Tipo: Seguridad
  (repetir para OPS-10 y USERS-1)

> clic derecho OU "Usuarios" > Nuevo > Usuario
  Nombre / Apellidos / Inicio de sesion segun la tabla de abajo
  Asignar contrasena, marcar "La contrasena nunca expira" (uso academico)
  (repetir para los 6 usuarios)

> por cada usuario: clic derecho > Agregar a un grupo > escribir el nombre del grupo > Aceptar
```

**Equivalente en PowerShell (mismo resultado, mas rapido de reproducir):**
```powershell
New-ADOrganizationalUnit -Name "Grupos"   -Path "DC=miguel,DC=local"
New-ADOrganizationalUnit -Name "Usuarios" -Path "DC=miguel,DC=local"

New-ADGroup -Name "ADMINS-15" -GroupScope Global -GroupCategory Security -Path "OU=Grupos,DC=miguel,DC=local"
New-ADGroup -Name "OPS-10"    -GroupScope Global -GroupCategory Security -Path "OU=Grupos,DC=miguel,DC=local"
New-ADGroup -Name "USERS-1"   -GroupScope Global -GroupCategory Security -Path "OU=Grupos,DC=miguel,DC=local"

$pw = ConvertTo-SecureString "Segura2121..." -AsPlainText -Force

New-ADUser -Name "Miguel Ramirez Meli"       -SamAccountName mramirez   -UserPrincipalName mramirez@miguel.local   -Path "OU=Usuarios,DC=miguel,DC=local" -AccountPassword $pw -Enabled $true
New-ADUser -Name "Ronald Arcangel Nunez"     -SamAccountName rarcangel  -UserPrincipalName rarcangel@miguel.local  -Path "OU=Usuarios,DC=miguel,DC=local" -AccountPassword $pw -Enabled $true
New-ADUser -Name "Yolanda Peralta Cruz"      -SamAccountName yperalta   -UserPrincipalName yperalta@miguel.local   -Path "OU=Usuarios,DC=miguel,DC=local" -AccountPassword $pw -Enabled $true
New-ADUser -Name "Franklin De la Rosa Gomez" -SamAccountName fdelarosa  -UserPrincipalName fdelarosa@miguel.local  -Path "OU=Usuarios,DC=miguel,DC=local" -AccountPassword $pw -Enabled $true
New-ADUser -Name "Carmen Sosa Bautista"      -SamAccountName csosa      -UserPrincipalName csosa@miguel.local      -Path "OU=Usuarios,DC=miguel,DC=local" -AccountPassword $pw -Enabled $true
New-ADUser -Name "Elvis Reynoso Tavarez"     -SamAccountName ereynoso   -UserPrincipalName ereynoso@miguel.local   -Path "OU=Usuarios,DC=miguel,DC=local" -AccountPassword $pw -Enabled $true

Add-ADGroupMember -Identity ADMINS-15 -Members mramirez, rarcangel
Add-ADGroupMember -Identity OPS-10    -Members yperalta, fdelarosa
Add-ADGroupMember -Identity USERS-1   -Members csosa, ereynoso
```

| Usuario | SamAccountName | Grupo | Nivel priv-lvl |
|---|---|---|---|
| Miguel Ramirez Meli | mramirez | ADMINS-15 | 15 |
| Ronald Arcangel Nunez | rarcangel | ADMINS-15 | 15 |
| Yolanda Peralta Cruz | yperalta | OPS-10 | 10 |
| Franklin De la Rosa Gomez | fdelarosa | OPS-10 | 10 |
| Carmen Sosa Bautista | csosa | USERS-1 | 1 |
| Elvis Reynoso Tavarez | ereynoso | USERS-1 | 1 |

Verificacion:
```powershell
Get-ADOrganizationalUnit -Filter *
Get-ADGroup -Filter * -SearchBase "OU=Grupos,DC=miguel,DC=local" | Select Name
Get-ADUser  -Filter * -SearchBase "OU=Usuarios,DC=miguel,DC=local" | Select Name, SamAccountName
Get-ADGroupMember ADMINS-15
Get-ADGroupMember OPS-10
Get-ADGroupMember USERS-1
```

### 9.6 Instalar el rol NPS

```
Administrador del servidor > Agregar roles y caracteristicas
> Roles de servidor: marcar "Servicios de directivas y acceso de redes"
> Siguiente > Siguiente > Instalar
```

### 9.7 Registrar los clientes RADIUS (los dos routers)

```
Herramientas > Servidor de directivas de redes (nps.msc)
> Clientes y servidores RADIUS > Clientes RADIUS > clic derecho > Nuevo
  Nombre descriptivo: Peer-A
  Direccion (IP o DNS): 200.13.67.2
  Secreto compartido manual: miguel2025

  (repetir) Nombre descriptivo: Peer-B
  Direccion: 200.13.67.6
  Secreto compartido manual: miguel2025
```

### 9.8 Crear las 3 directivas de red (una por nivel de privilegio)

```
NPS > Directivas > Directivas de red > clic derecho > Nueva directiva de red

Directiva 1 - "ADMINS-15"
  Condiciones: Agregar > Grupos de Windows > MIGUEL\ADMINS-15
  Permiso de acceso: Acceso concedido
  Metodos de autenticacion: CHAP (o PAP segun lo visto en clase)
  Configuracion > Atributos RADIUS > Vendor Specific > Agregar
    Proveedor: Cisco
    Atributo: cisco-av-pair
    Formato: Cadena
    Valor: shell:priv-lvl=15
  Finalizar

Directiva 2 - "OPS-10"
  Condicion: Grupos de Windows = MIGUEL\OPS-10
  cisco-av-pair = shell:priv-lvl=10

Directiva 3 - "USERS-1"
  Condicion: Grupos de Windows = MIGUEL\USERS-1
  cisco-av-pair = shell:priv-lvl=1
```

Orden de evaluacion (arrastrar en el panel "Directivas de red" hasta quedar asi, de arriba hacia abajo):
```
1. ADMINS-15
2. OPS-10
3. USERS-1
```
NPS evalua de arriba hacia abajo y aplica la primera directiva que haga match; el orden importa porque un usuario podria calificar para mas de un grupo si la jerarquia no se respeta.

### 9.9 Verificacion completa NPS + RADIUS de extremo a extremo

```powershell
Get-WindowsFeature NPAS
Get-Service IAS      # nombre real del servicio de NPS
```
```
Visor de eventos > Registros de Windows > Seguridad
> filtrar por ID 6272 (acceso concedido) o 6273 (acceso denegado)
```
Prueba real desde un router, autenticando con un usuario de cada grupo (ejemplo con yperalta, grupo OPS-10):
```
Peer-A> ssh -l yperalta 10.13.67.1
```
Debe autenticar contra RADIUS y el nivel de privilegio resultante debe coincidir con el del grupo (15/10/1). Si falla, revisar en el router:
```
show radius statistics
debug radius authentication
```
y en el NPS, el Visor de eventos: el motivo de rechazo mas comun es un secreto compartido (`miguel2025`) que no coincide entre el `radius-server host` del router y el cliente RADIUS registrado en NPS.


---

## 10. Windows10-1 (Cliente - VLAN 20)

**1) NIC en DHCP**
```powershell
ipconfig /release
ipconfig /renew
ipconfig /all
```
Debe recibir IP entre `172.13.67.11` y `172.13.67.254`, gateway `172.13.67.1`, DNS `10.13.67.10`.

**2) Trafico permitido (debe funcionar, cruza la VPN)**
```powershell
ping 10.13.67.10
```
```
Navegador > http://10.13.67.10
```

**3) Trafico bloqueado (debe fallar - confirma la ACL)**
```powershell
Test-NetConnection 10.13.67.10 -Port 445
Test-NetConnection 10.13.67.10 -Port 389
nslookup miguel.local 10.13.67.10
```

**4) Intento de union al dominio (debe fallar, confirma "bloquear el resto")**
```powershell
Add-Computer -DomainName "miguel.local" -Credential (Get-Credential) -Restart
```
Falla por DNS/Kerberos/LDAP bloqueados en `ACL_HACIA_SERVIDOR`. Windows10-1 permanece en **workgroup**.

**5) Resumen**
```powershell
Test-NetConnection 10.13.67.10 -InformationLevel Detailed   # ping -> True
Test-NetConnection 10.13.67.10 -Port 80                     # HTTP -> True
Test-NetConnection 10.13.67.10 -Port 445                    # SMB -> False
```

---

## 11. Checklist contra el enunciado

| Requisito | Cumplido en |
|---|---|
| Port-Security, MOTD | Seccion 8 / todas las secciones |
| Medidas ante ataques vistos en clase | Seccion 8 (DHCP snooping, port-security, root guard, bpduguard) |
| Direccionamiento IP por matricula | Seccion 1 |
| DHCP para la LAN | Seccion 6; verificado en 10 |
| Ruta por defecto | Secciones 2, 3, 6 |
| NAT | Secciones 3, 6 |
| Autenticacion RADIUS | Secciones 3, 6, 9 |
| Comunicacion entre LANs solo por VPN | Secciones 5-6 (rutas via tunel L2TP) |
| Solo HTTP+ICMP al servidor | ACL_HACIA_SERVIDOR, seccion 3; verificado en 10 |
| VPN Site-to-Site L2TP + IPsec | Secciones 4-5 (ver aviso tecnico de plataforma) |
| ISP sin protocolo de enrutamiento | Seccion 2 |
| Windows Server: AD, IIS, NPS | Seccion 9 |
| NPS niveles 15/10/1 | Seccion 9 |
| Credenciales VPN = matricula (usuario 2025 / clave 1367) | Seccion 1 y 5 |
