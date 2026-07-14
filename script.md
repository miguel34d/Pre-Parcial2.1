# Laboratorio Final - Configuracion Completa (desde cero, para GNS3)
**Estudiante:** Miguel Ramirez Meli - Matricula 2025-1367
**Topologia:** R-ISP (ISP) - Peer-A (LAN Servidor, VLAN 10) - VPN - Peer-B (LAN Cliente, VLAN 20)

---

## 1. Plan de direccionamiento IP

| Enlace / Red | Direccion | Interfaz |
|---|---|---|
| R-ISP <-> Peer-A | 200.13.67.0/30 | R-ISP f0/0 = .1 / Peer-A f0/0 = .2 |
| R-ISP <-> Peer-B | 200.13.67.4/30 | R-ISP f0/1 = .5 / Peer-B f0/0 = .6 |
| R-ISP <-> Internet (Cloud1) | 10.10.10.0/24 | R-ISP e0/2 = .2 |
| LAN Servidor (VLAN 10) | 10.13.67.0/24 | Peer-A f0/1 = .1 (gateway) |
| LAN Cliente (VLAN 20) | 172.13.67.0/24 | Peer-B f0/1 = .1 (gateway) |
| WindowsServer2022-2 | 10.13.67.10/24 | gateway 10.13.67.1 |
| Windows10-1 (NIC1) | DHCP (pool 172.13.67.0/24) | gateway 172.13.67.1 |
| Tunel GRE (IPsec+GRE) | 100.13.67.0/30 | Tu0 Peer-A=.1 / Tu0 Peer-B=.2 |
| Credenciales VPN (usuario/clave) | usuario: **2025** / clave: **1367** | (segun matricula 2025-1367) |

> **Nota tecnica sobre "usuario/clave" en la VPN:** IPsec site-to-site con llave pre-compartida (PSK) no tiene un campo de "usuario" como tal - solo una clave compartida entre los dos routers. Para que el **2025** quede reflejado de forma explicita en la configuracion, se usa un `crypto keyring` llamado **2025** que contiene la clave **1367**. Ver seccion 5.A.

### Claves de acceso de los routers y switches

| Dispositivo | Enable secret | Password consola | Usuario SSH/VTY | Clave SSH/VTY |
|---|---|---|---|---|
| R-ISP | *(no configurado - ver nota abajo)* | *(no configurado)* | *(no tiene SSH habilitado)* | *(no tiene SSH habilitado)* |
| Peer-A | 20251367 | 20251367 | miguel | 20251367 |
| Peer-B | 20251367 | 20251367 | miguel | 20251367 |
| Switch1 | 20251367 | *(no configurado)* | miguel | 20251367 |
| Switch2 | 20251367 | *(no configurado)* | miguel | 20251367 |

> En Peer-A y Peer-B, `miguel` / `20251367` es el usuario local de respaldo (fallback) configurado con `username miguel privilege 15 secret 20251367`. Si el servidor RADIUS (NPS) esta arriba y respondiendo, tambien puedes entrar con cualquier usuario de AD que hayas creado en la seccion 7.4 (ADMINS-15, OPS-10, USERS-1), segun el nivel de privilegio que le hayas asignado ahi.

> **R-ISP no tiene `enable secret` ni password de consola/SSH en esta guia**, porque en la seccion 2 solo se le puso `banner motd` y las interfaces - no se penso como dispositivo administrable remotamente, solo como transito IP puro del ISP. Si tu profesor pide que R-ISP tambien tenga clave de enable/consola, dimelo y lo agrego (por consistencia, seria `20251367` igual que los demas).

---

## 2. R-ISP (ISP - solo IP publica, sin protocolo de enrutamiento)

```
enable
configure terminal
hostname R-ISP
no ip domain-lookup
banner motd # ACCESO SOLO AUTORIZADO - ISP #
```

```
interface f0/0
 description Enlace a Peer-A
 ip address 200.13.67.1 255.255.255.252
 no shutdown
exit
```

```
interface f0/1
 description Enlace a Peer-B
 ip address 200.13.67.5 255.255.255.252
 no shutdown
exit
```

```
! No se configura ningun protocolo de enrutamiento RIP OSPF EIGRP
! Al tener ambos enlaces directamente conectados, R-ISP puede
! reenviar paquetes entre Peer-A y Peer-B sin rutas adicionales
```

### Salida a Internet (e0/2)
> R-ISP (Router1) usa unicamente interfaces e0/x (e0/0 hacia Peer-A, e0/1 hacia Peer-B, e0/2 hacia Internet/Cloud1), segun la topologia real. Requiere modulo NM-4E (o similar) en Configure > Slots si el dispositivo no trae 3+ puertos Ethernet integrados.

```
interface f0/2
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

## 3. Peer-A (lado Servidor - VLAN 10)

### Parte 1 - basicos
```
enable
configure terminal
hostname Peer-A
no ip domain-lookup
service password-encryption
banner motd # ACCESO SOLO PERSONAL AUTORIZADO - Peer-A #
enable secret 20251367
```

### Parte 2 - lineas (nota el "exit" final, es clave para el paso siguiente)
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

### Parte 3 - SSH (requiere que domain-name se aplique en modo config global, por eso el exit de arriba)
```
ip domain-name miguel.local
crypto key generate rsa modulus 2048
ip ssh version 2
```

### Parte 4 - interfaces
```
interface f0/0
 description WAN hacia R-ISP
 ip address 200.13.67.2 255.255.255.252
 ip nat outside
 no shutdown
exit
```

```
interface f0/1
 description LAN Servidor VLAN10 hacia Switch1
 ip address 10.13.67.1 255.255.255.0
 ip nat inside
 no shutdown
exit
```

### Parte 5 - ruta por defecto y NAT
```
ip route 0.0.0.0 0.0.0.0 200.13.67.1
access-list 1 permit 10.13.67.0 0.0.0.255
ip nat inside source list 1 interface f0/0 overload
```

### Parte 6 - RADIUS (sintaxis clasica, compatible con IOS 12.4)
```
aaa new-model
radius-server host 10.13.67.10 auth-port 1812 acct-port 1813 key MiguelRadius2025!
aaa authentication login default group radius local
aaa authorization exec default group radius local
aaa authorization network default group radius
username miguel privilege 15 secret 20251367
```

### Parte 7 - ACL (solo HTTP + ICMP al servidor, resto bloqueado)
```
ip access-list extended ACL_HACIA_SERVIDOR
 remark Excepciones RADIUS AD desde Peer-B
 permit udp host 172.13.67.1 host 10.13.67.10 eq 1812
 permit udp host 172.13.67.1 host 10.13.67.10 eq 1813
 remark Trafico de usuario permitido solo HTTP e ICMP
 permit icmp any host 10.13.67.10
 permit tcp any host 10.13.67.10 eq 80
 deny ip any host 10.13.67.10
 permit ip any any
exit
```
> Esta ACL se aplica sobre la interfaz del tunel (Tunnel0), ver seccion 5. Ahi llega el trafico que viene de la LAN cliente.

### Parte 8 - guardar
```
end
write memory
```

---

## 4. Peer-B (lado Cliente - VLAN 20)

### Parte 1 - basicos
```
enable
configure terminal
hostname Peer-B
no ip domain-lookup
service password-encryption
banner motd # ACCESO SOLO PERSONAL AUTORIZADO - Peer-B #
enable secret 20251367
```

### Parte 2 - lineas (nota el "exit" final)
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

### Parte 3 - SSH
```
ip domain-name miguel.local
crypto key generate rsa modulus 2048
ip ssh version 2
```

### Parte 4 - interfaces
```
interface f0/0
 description WAN hacia R-ISP
 ip address 200.13.67.6 255.255.255.252
 ip nat outside
 no shutdown
exit
```

```
interface f0/1
 description LAN Cliente VLAN20 hacia Switch2
 ip address 172.13.67.1 255.255.255.0
 ip nat inside
 no shutdown
exit
```

### Parte 5 - ruta por defecto y NAT
```
ip route 0.0.0.0 0.0.0.0 200.13.67.5
access-list 1 permit 172.13.67.0 0.0.0.255
ip nat inside source list 1 interface f0/0 overload
```

### Parte 6 - DHCP para la LAN cliente
```
ip dhcp excluded-address 172.13.67.1 172.13.67.10
ip dhcp pool VLAN20_POOL
 network 172.13.67.0 255.255.255.0
 default-router 172.13.67.1
 dns-server 10.13.67.10
 domain-name miguel.local
exit
```

### Parte 7 - RADIUS (sintaxis clasica)
```
aaa new-model
radius-server host 10.13.67.10 auth-port 1812 acct-port 1813 key MiguelRadius2025!
aaa authentication login default group radius local
aaa authorization exec default group radius local
username miguel privilege 15 secret 20251367
```

### Parte 8 - guardar
```
end
write memory
```

---

## 5. VPN Site-to-Site (entre Peer-A y Peer-B)

> La variante L2TPv3 pseudowire no es soportada por el feature-set adventerprisek9/advipservicesk9 estandar sin soporte SP. Da `% Invalid input detected` al intentar `interface Pseudowire1`. Se usa **IPsec + GRE** como la VPN funcional para la entrega (5.A). El intento L2TPv3 queda documentado en 5.C solo como referencia teorica.

> **Nota sobre crypto (importante):** tu imagen `c3725-adventerprisek9-mz.124-25d` es de la rama **mainline** de IOS (12.4), no de la rama **T**. La rama mainline no soporta `hash sha256` ni `group 14` en `crypto isakmp policy` (esas features llegaron en 15.1(2)T+). Por eso esta guia usa `hash sha` (SHA-1) y `group 5` (Diffie-Hellman de 1536 bits), que si estan disponibles en 12.4. El transform-set tambien se ajusto a `esp-sha-hmac` en vez de `esp-sha256-hmac` por la misma razon. Si en tu equipo tambien falla `group 5`, baja a `group 2` (1024 bits, soportado practicamente en toda version de IOS).

> **Nota sobre "usuario 2025 / clave 1367":** IPsec PSK site-to-site no soporta un campo de usuario independiente (eso existe en VPN de acceso remoto con XAUTH, que es otro escenario). Para dejar el **usuario (2025)** visible y verificable en la configuracion, en vez de un `crypto isakmp key` global se usa un `crypto keyring` llamado **2025**, que contiene la clave **1367**. Al hacer `show crypto isakmp key` o `show running-config | section crypto keyring` se vera claramente el nombre `2025` asociado a la clave `1367`, cumpliendo con el par usuario/clave de la matricula.

### 5.A. IPsec + GRE (la que se entrega)

**En Peer-A:**
```
crypto keyring 2025
 pre-shared-key address 200.13.67.6 key 1367
exit
```

```
crypto isakmp policy 10
 encryption aes 256
 hash sha
 authentication pre-share
 group 5
exit
```

```
crypto ipsec transform-set TS-GRE esp-aes 256 esp-sha-hmac
 mode transport
exit
crypto ipsec profile PROFILE-GRE
 set transform-set TS-GRE
exit
```

```
interface Tunnel0
 ip address 100.13.67.1 255.255.255.252
 tunnel source f0/0
 tunnel destination 200.13.67.6
 tunnel protection ipsec profile PROFILE-GRE
 ip access-group ACL_HACIA_SERVIDOR in
exit
ip route 172.13.67.0 255.255.255.0 Tunnel0
```

**En Peer-B:**
```
crypto keyring 2025
 pre-shared-key address 200.13.67.2 key 1367
exit
```

```
crypto isakmp policy 10
 encryption aes 256
 hash sha
 authentication pre-share
 group 5
exit
```

```
crypto ipsec transform-set TS-GRE esp-aes 256 esp-sha-hmac
 mode transport
exit
crypto ipsec profile PROFILE-GRE
 set transform-set TS-GRE
exit
```

```
interface Tunnel0
 ip address 100.13.67.2 255.255.255.252
 tunnel source f0/0
 tunnel destination 200.13.67.2
 tunnel protection ipsec profile PROFILE-GRE
exit
ip route 10.13.67.0 255.255.255.0 Tunnel0
```

> Nota: la clave `1367` (dentro del keyring `2025`) se dejo sin caracteres especiales al final porque en algunas versiones de IOS eso puede cortar el comando o interpretarse como fin de linea de comentario al pegar rapido en la consola de GNS3. Asegurate de que la clave (`1367`) y el nombre del keyring (`2025`) coincidan exactamente en ambos routers.

### 5.B. Verificacion del tunel
```
show crypto isakmp sa
show crypto ipsec sa
show interfaces tunnel0
show crypto keyring 2025
ping 100.13.67.2 source tunnel0
```
`show crypto isakmp sa` debe mostrar `QM_IDLE`, y `show interfaces tunnel0` debe reportar `line protocol is up`. `show crypto keyring 2025` debe mostrar la direccion del peer y la clave `1367`. Si no levanta, revisa que las pre-shared keys coincidan exactamente en ambos routers (`show run | section crypto keyring`).

### 5.C. (Descartado) L2TPv3 + IPsec - no aplicar, solo referencia
```
! No soportado en feature-set estandar sin SP Services.
! Da % Invalid input detected en interface Pseudowire1.
! Ver documento anterior para el bloque completo de referencia.
```

---

## 6. Switch1 (lado Servidor) y Switch2 (lado Cliente)

El puerto que da hacia el router (Peer-A o Peer-B) es el que hayas cableado ahi en GNS3 - ajusta el numero de puerto segun tu topologia real si no es `e0/0`/`e0/1`.

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
```

```
interface e0/1
 switchport port-security
 switchport port-security maximum 1
 switchport port-security mac-address sticky
 switchport port-security violation shutdown
exit
```

```
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

### Switch2 (mismo bloque, VLAN 20)
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
```

```
interface e0/1
 switchport port-security
 switchport port-security maximum 1
 switchport port-security mac-address sticky
 switchport port-security violation shutdown
exit
```

```
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

## 7. Windows Server 2022 (roles: AD DS, IIS, NPS)

### 7.1 Direccionamiento
- IP: `10.13.67.10 /24`, gateway `10.13.67.1`, DNS: `127.0.0.1` (se vuelve su propio DNS al instalar AD).

### 7.2 Active Directory Domain Services
1. Administrador del servidor -> Agregar roles y caracteristicas -> **AD DS**.
2. Promover a controlador de dominio -> Nuevo bosque -> dominio `miguel.local`.
3. Reiniciar.

### 7.3 IIS
1. Agregar rol **Servidor Web (IIS)**.
2. Deja el sitio predeterminado en el puerto 80 (coincide con el ACL que solo permite HTTP).

### 7.4 Grupos y usuarios para niveles de acceso (NPS)
En **Usuarios y equipos de AD**, crea:

| Grupo | Nivel de privilegio |
|---|---|
| ADMINS-15 | 15 |
| OPS-10 | 10 |
| USERS-1 | 1 |

Crea un usuario de prueba en cada grupo.

### 7.5 Instalar y configurar NPS (RADIUS)
1. Agregar rol **Servidor de directivas de redes (NPS)**.
2. **Clientes RADIUS** -> agregar dos clientes:
   - Nombre: Peer-A, IP: 200.13.67.2 (o 10.13.67.1 si el origen del paquete es la LAN), secreto compartido: `MiguelRadius2025!`
   - Nombre: Peer-B, IP: 200.13.67.6, mismo secreto compartido.
3. **Directivas de red** -> crear 3 directivas (una por grupo), condicion = "Windows Groups" = ADMINS-15/OPS-10/USERS-1.
4. En cada directiva, pestana **Configuracion** -> *Atributos RADIUS estandar* -> agrega el atributo **Vendor Specific**:
   - Vendor: Cisco (codigo 9)
   - Atributo: `cisco-av-pair`
   - Valor: `shell:priv-lvl=15` (o `=10`, `=1` segun el grupo).
5. En autenticacion, habilita **PAP** (o MS-CHAPv2 si prefieres cifrado, pero requiere el certificado - ver 7.6).

### 7.6 Importar certificado (por que y como)
El certificado es necesario si usas NPS con PEAP/EAP, o IPsec con autenticacion por certificado (alternativa a pre-shared key).

Pasos:
1. `certlm.msc` (Administrador de certificados del equipo local).
2. Si tienes una CA propia, solicita un certificado via **Plantillas de certificado -> Servidor Web/Autenticacion de equipo**, o
3. Si te dieron un `.pfx`: clic derecho en **Personal -> Certificados -> Todas las tareas -> Importar**, selecciona el `.pfx`, ingresa la contrasena.
4. En **NPS -> Directivas de red -> [tu directiva] -> Restricciones -> Metodos de autenticacion -> EAP -> Microsoft: EAP protegido (PEAP)**, selecciona el certificado importado.
5. Verifica que el certificado aparezca tambien en **Entidades de certificacion raiz de confianza** si es autofirmado.

---

## 8. Checklist contra el enunciado

| Requisito | Donde se cubre |
|---|---|
| Port-Security, MOTD | Seccion 6 (port-security); MOTD en todas las secciones |
| "Interfaces pasivas" | No aplica: no hay protocolo de enrutamiento dinamico (solo rutas estaticas, y R-ISP/ISP no debe tener ninguno). Si tu profesor lo exige literal, avisame y agrego OSPF + passive-interface |
| Medidas de seguridad ante ataques vistos en clase | Seccion 6: DHCP snooping, port-security, root guard, bpduguard |
| Direccionamiento IP | Seccion 1 |
| DHCP para la LAN | Seccion 4, parte 6 |
| Ruta por defecto | Secciones 2-4 |
| NAT | Secciones 3-4 |
| Autenticacion por RADIUS | Secciones 3-4, parte RADIUS + seccion 7.5 |
| Comunicacion entre LANs solo por VPN | Seccion 5.A (rutas apuntan a Tunnel0) |
| Solo HTTP + ICMP al servidor | ACL_HACIA_SERVIDOR (seccion 3, parte 7), aplicada en Tunnel0 de Peer-A |
| VPN L2TP + IPsec | Seccion 5.C (documentada, no aplicable en este feature-set) |
| VPN IPsec + GRE | Seccion 5.A (la que se entrega) |
| ISP sin protocolo de enrutamiento | Seccion 2 |
| Windows Server: AD, IIS, NPS | Seccion 7 |
| NPS niveles 15/10/1 | Seccion 7.4 y 7.5 |
| Direccionamiento por matricula | Seccion 1 |
| Credenciales VPN por matricula | Seccion 1 y 5.A (usuario=2025, clave=1367) |

---

## 9. Notas finales / correcciones aplicadas en esta version

- **Credenciales unificadas**: se cambio el enable secret y la password de consola (antes `Cisco123!`) a `20251367` en Peer-A, Peer-B, Switch1 y Switch2, para que coincidan con el usuario `miguel` / clave `20251367` que ya se usaba en SSH.
- **Interfaces LAN de Peer-A y Peer-B**: se uso `f0/1` en vez de `e0/0`, porque el c3725 base en GNS3 no trae `e0/0` integrado (requiere modulo NM-1E/NM-4E agregado manualmente en Configure > Slots). Si ya agregaste ese modulo, reemplaza `f0/1` por `e0/0` en las secciones 3 y 4.
- **RADIUS**: se cambio a la sintaxis clasica de una linea (`radius-server host ...`) porque el bloque moderno (`radius server NAME` + subcomandos) no es compatible con IOS 12.4(25d).
- **`crypto key generate rsa`**: ahora funciona porque se agrego un `exit` explicito despues de configurar `line vty 0 4`, así el `ip domain-name` se aplica en modo de configuracion global (antes se estaba ejecutando por error dentro de `(config-line)#`).
- **VPN usuario/clave (2025/1367)**: se cambio de un `crypto isakmp key` global anonimo a un `crypto keyring 2025` con clave `1367`, para que el "usuario" de la matricula quede visible y verificable directamente en la configuracion (`show crypto keyring`), no solo en la tabla de la seccion 1.
- Todas las claves (`20251367`, `MiguelRadius2025!`, etc.) son ejemplos - cambialas si tu institucion exige un estandar distinto.
