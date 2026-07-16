# Configuración VPN — Laboratorio R1 (ISP) + R2/R3/R4

## Esquema de direccionamiento

| Segmento | Red | Router / Gateway | Host |
|---|---|---|---|
| VLAN 10 (Windows10-1, detrás de R2) | 10.13.67.0/24 | R2 f0/1 = 10.13.67.1 | Windows10-1 = 10.13.67.10 |
| VLAN 20 (WindowsServer2022-1, detrás de R3) | 20.13.67.0/24 | R3 f0/1 = 20.13.67.1 | Server = 20.13.67.10 |
| VLAN 30 (Pc1/VPCS, detrás de R4) | 30.13.67.0/24 | R4 f0/1 = 30.13.67.1 | Pc1 = 30.13.67.10 |
| Público (ISP, R1) | 200.13.67.0/24 subneteado en /30 | R1↔R2: 200.13.67.0/30, R1↔R3: 200.13.67.4/30, R1↔R4: 200.13.67.8/30 | — |

**Distribución de las 2 VPN pedidas:**
- **VPN Client-to-Site L2TP/IPsec**: el propio **Windows10-1** se conecta como cliente VPN nativo de Windows hacia **R3** (que actúa como servidor LNS), para entrar a la red del Server (VLAN 20) igual que un usuario remoto.
- **VPN Site-to-Site IKEv1**: túnel permanente entre **R3** y **R4**, para que la VLAN 20 (Server) y la VLAN 30 (Pc1) se comuniquen de forma transparente (Pc1 es un VPCS y no puede correr un cliente VPN).

R1 solo actúa como ISP/tránsito, no es punto final de ninguna VPN.

---

## R1 — ISP (solo enrutamiento, sin VPN)

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

R1 no necesita rutas adicionales: las tres redes públicas están directamente conectadas a él.

---

## R2 — Router del cliente (Windows10-1 / VLAN 10)

Solo da salida a Internet (NAT) para que Windows10-1 pueda llegar a R3 y levantar el túnel L2TP. No participa en ninguna VPN él mismo.

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

! ===================== DHCP para Windows10-1 (VLAN 10) =====================
ip dhcp excluded-address 10.13.67.1

ip dhcp pool POOL_WIN10
 network 10.13.67.0 255.255.255.0
 default-router 10.13.67.1

end
write memory
```

En Windows10-1: Configuración → Red → Ethernet → Propiedades IP → **Obtener dirección IP automáticamente (DHCP)**.

---

## R3 — Server (VLAN 20) + Servidor L2TP (LNS) + extremo IKEv1 hacia R4

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

! ===================== VPN SITE-TO-SITE IKEv1 (R3 <-> R4) =====================
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

! ===================== VPN CLIENT-TO-SITE L2TP/IPsec (Windows10-1 -> R3) ======
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

! El mismo crypto map lleva la entrada estática (site-to-site) y la dinámica (L2TP)
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

---

## R4 — Cliente2 (VLAN 30 / Pc1) — extremo IKEv1 hacia R3

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

! ===================== DHCP para Pc1 / VPCS (VLAN 30) =====================
ip dhcp excluded-address 30.13.67.1

ip dhcp pool POOL_PC1
 network 30.13.67.0 255.255.255.0
 default-router 30.13.67.1

end
write memory
```

En Pc1 (VPCS), obtener IP con:
```
PC1> ip dhcp
```

---

## Switch1 — detrás de R2 (segmento VLAN 10 / Windows10-1)

```
enable
configure terminal
hostname Switch1

vlan 10
 name VLAN10_Windows10-1

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

---

## Switch2 — detrás de R3 (segmento VLAN 20 / WindowsServer2022-1)

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

---

## Switch3 — detrás de R4 (segmento VLAN 30 / Pc1 - VPCS)

```
enable
configure terminal
hostname Switch3

vlan 30
 name VLAN30_Pc1

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

> **Nota:** `e0/0` conecta hacia el router y `e0/1` hacia el host (Windows10-1, Server o Pc1 según el switch). Ambos puertos de cada switch quedan en la misma VLAN (10, 20 o 30 respectivamente) para reflejar el nombre del segmento en el diagrama. Como el enlace hacia el router va en modo access (no trunk), el router no necesita saber nada de la VLAN — simplemente ve su interfaz f0/1 conectada a ese segmento igual que antes.

---

## PCs / hosts

- **Windows10-1**: obtiene IP por **DHCP** desde el pool `POOL_WIN10` de R2 (rango 10.13.67.2–254, gateway 10.13.67.1).
- **WindowsServer2022-1**: IP estática 20.13.67.10/24, gateway 20.13.67.1.
- **Pc1 (VPCS)**: obtiene IP por **DHCP** desde el pool `POOL_PC1` de R4 (rango 30.13.67.2–254, gateway 30.13.67.1):
```
PC1> ip dhcp
```

---

## Conexión L2TP en el cliente Windows (Windows10-1)

Pasos para crear la conexión VPN L2TP/IPsec en Windows10-1, apuntando a R3 (200.13.67.6):

1. **Configuración → Red e Internet → VPN → Agregar una conexión VPN**.
2. **Proveedor de VPN**: Windows (integrado).
3. **Nombre de la conexión**: `Miguel-L2PT`.
4. **Nombre o dirección del servidor**: `200.13.67.6`.
5. **Tipo de VPN**: `L2TP/IPsec con clave previamente compartida`.
6. **Clave precompartida**: `ClaveL2TP_123` (debe coincidir con el `crypto isakmp key` de R3).
7. **Tipo de información de inicio de sesión**: Nombre de usuario y contraseña.
   - Usuario: `2025`
   - Contraseña: `1367`
8. Guardar y luego ir a **Configuración → Red e Internet → VPN**, seleccionar `Miguel-L2PT` → **Conectar**.

Si prefieres el método clásico (Panel de control):
`Panel de control → Redes e Internet → Centro de redes y recursos compartidos → Configurar una nueva conexión → Conectarse a un área de trabajo → Usar mi conexión a Internet (VPN) → 200.13.67.6 → tipo L2TP/IPsec + clave precompartida → usuario/contraseña.`

Al conectar, Windows10-1 recibirá una IP del pool `20.13.67.100–150` (asignada por R3) y podrá alcanzar el Server como si estuviera dentro de la VLAN 20.

---

## Verificación rápida

```
show crypto isakmp sa       ! en R3 y R4 (debe verse QM_IDLE para el túnel S2S)
show crypto ipsec sa        ! ver paquetes encriptados/desencriptados
show vpdn session           ! en R3, ver la sesión L2TP activa de Windows10-1
show ip interface brief     ! confirmar IPs en todos los routers
show interfaces status      ! en cada switch, confirmar que los puertos están up/up en modo access
```

> **Importante:** la lista AAA usada por `ppp authentication ms-chap-v2` debe crearse con `aaa authentication ppp VPDN_AUTH local` (no `aaa authentication login`), de lo contrario aparece el warning `AAA: authentication list "VPDN_AUTH" is not defined for PPP` y el túnel L2TP no autenticará correctamente.

Desde Windows10-1 (una vez conectada la VPN): `ping 20.13.67.10`
Desde Pc1: `ping 20.13.67.10` (debe funcionar automáticamente por el túnel site-to-site, sin cliente VPN).

---

## Windows Server 2022 — Roles AD DS, IIS y NPS (con niveles de acceso 15, 10, 1)

Esta sección es independiente de la configuración de red anterior (routers, switches, VPN, DHCP) — **no se modifica nada de lo ya hecho**, esto solo se agrega sobre el Windows Server 2022 (interfaz en español) que ya tienes como `WindowsServer2022-1` (20.13.67.10) detrás de R3.

### Instalación de roles (Administrador del servidor)

1. Abrir **Administrador del servidor**.
2. En el panel principal → **Agregar roles y características**.
3. Tipo de instalación → **Instalación basada en roles o características** → Siguiente.
4. Selección del servidor → elegir `WindowsServer2022-1` (servidor local) → Siguiente.
5. En **Roles de servidor**, marcar:
   - **Servicios de dominio de Active Directory**
   - **Servidor web (IIS)**
   - **Servicios de acceso y directivas de redes** (aquí vive NPS)
6. Cuando el asistente pida agregar características necesarias (por ejemplo, Herramientas de administración), clic en **Agregar características**.
7. Siguiente en cada pantalla (incluida la de IIS) → **Instalar**.
8. Al terminar, no cerrar la ventana: usa la bandera amarilla de notificaciones en Administrador del servidor para completar la configuración de cada rol (especialmente AD DS).

---

### 1) Active Directory Domain Services (AD DS)

**Promover el servidor a controlador de dominio** (desde la notificación de la bandera amarilla → "Promover este servidor a controlador de dominio"):

1. Operación de implementación → **Agregar un nuevo bosque**.
2. Nombre de dominio raíz: `miguel.local`
3. Opciones del controlador de dominio → nivel funcional del bosque y dominio: `Windows Server 2016` o superior → establecer contraseña de DSRM.
4. Opciones de DNS → puede aparecer una advertencia de delegación DNS, es normal en laboratorio, ignorar.
5. Nombre NetBIOS: `MIGUEL` (se autocompleta).
6. Rutas (base de datos, logs, SYSVOL): dejar las predeterminadas.
7. Revisar opciones → **Instalar** (el servidor reiniciará automáticamente).

**Crear la estructura en "Usuarios y equipos de Active Directory" (dsa.msc):**

1. Clic derecho en el dominio `miguel.local` → Nuevo → **Unidad organizativa** → nombre: `Red`.
2. Dentro de la OU `Red`, clic derecho → Nuevo → **Grupo**, crear los 3 grupos de acceso:

| Grupo | Nivel de acceso Cisco (privilege level) |
|---|---|
| `GG_Nivel15` | 15 (acceso total / modo enable) |
| `GG_Nivel10` | 10 (acceso intermedio) |
| `GG_Nivel1` | 1 (solo lectura / user exec) |

3. Dentro de la misma OU, clic derecho → Nuevo → **Usuario**, crear:

| Nombre completo | Usuario de inicio de sesión | Grupo | Nivel |
|---|---|---|---|
| Miguel Ramírez | `mramirez` | `GG_Nivel15` | 15 |
| Emily Torres | `etorres` | `GG_Nivel10` | 10 |
| Carlos Gómez | `cgomez` | `GG_Nivel1` | 1 |

4. Al crear cada usuario, asignar una contraseña y **desmarcar** "El usuario debe cambiar la contraseña en el siguiente inicio de sesión" (para simplificar el laboratorio).
5. Agregar cada usuario a su grupo: seleccionar el usuario → pestaña **Miembro de** → **Agregar** → escribir el nombre del grupo correspondiente.

**Equivalente en PowerShell** (más rápido si prefieres no usar la GUI):

```powershell
New-ADOrganizationalUnit -Name "Red" -Path "DC=miguel,DC=local"

New-ADGroup -Name "GG_Nivel15" -GroupScope Global -Path "OU=Red,DC=miguel,DC=local"
New-ADGroup -Name "GG_Nivel10" -GroupScope Global -Path "OU=Red,DC=miguel,DC=local"
New-ADGroup -Name "GG_Nivel1"  -GroupScope Global -Path "OU=Red,DC=miguel,DC=local"

New-ADUser -Name "Miguel Ramirez" -SamAccountName "mramirez" `
  -UserPrincipalName "mramirez@miguel.local" -Path "OU=Red,DC=miguel,DC=local" `
  -AccountPassword (ConvertTo-SecureString "Nivel15Pass!" -AsPlainText -Force) -Enabled $true

New-ADUser -Name "Emily Torres" -SamAccountName "etorres" `
  -UserPrincipalName "etorres@miguel.local" -Path "OU=Red,DC=miguel,DC=local" `
  -AccountPassword (ConvertTo-SecureString "Nivel10Pass!" -AsPlainText -Force) -Enabled $true

New-ADUser -Name "Carlos Gomez" -SamAccountName "cgomez" `
  -UserPrincipalName "cgomez@miguel.local" -Path "OU=Red,DC=miguel,DC=local" `
  -AccountPassword (ConvertTo-SecureString "Nivel1Pass!" -AsPlainText -Force) -Enabled $true

Add-ADGroupMember -Identity "GG_Nivel15" -Members "mramirez"
Add-ADGroupMember -Identity "GG_Nivel10" -Members "etorres"
Add-ADGroupMember -Identity "GG_Nivel1"  -Members "cgomez"
```

---

### 2) IIS — Servidor Web

- Ya quedó instalado en el paso de roles (no requiere configuración adicional para un laboratorio básico).
- **Verificación local**: en el propio servidor, abrir un navegador → `http://localhost` → debe cargar la página predeterminada de IIS ("Bienvenido" / página azul de bienvenida en español).
- **Verificación remota** (esto además demuestra que el túnel L2TP realmente da acceso a los recursos internos, no solo ping): desde **Windows10-1**, con la VPN `Miguel-L2PT` conectada, abrir el navegador y visitar `http://20.13.67.10`. Debe mostrar la misma página de bienvenida de IIS.

---

### 3) NPS — Servidor de directivas de redes (RADIUS)

1. Ya se instaló al marcar "Servicios de acceso y directivas de redes".
2. Abrir **Servidor de directivas de redes** (`nps.msc`).
3. Clic derecho en **NPS (Local)** → **Registrar servidor en Active Directory** → Aceptar. (Esto agrega el equipo al grupo `RAS and IAS Servers`, necesario para que NPS pueda leer los grupos del dominio).

**Agregar clientes RADIUS** (los routers que van a delegar el login administrativo en NPS):

Clic derecho en **Clientes RADIUS** → **Nuevo**:

| Nombre descriptivo | Dirección IP | Secreto compartido |
|---|---|---|
| R3 | 20.13.67.1 | `RadiusKey123` |
| R4 | 30.13.67.1 | `RadiusKey123` |

> R3 está en la misma red que el servidor (20.13.67.0/24), así que llega directo. R4 llega a través del túnel IPsec site-to-site ya existente hacia 20.13.67.0/24. R1 y R2 hoy no tienen ruta hacia esa red, así que no se agregan como clientes por ahora.

**Crear las 3 directivas de red** (una por nivel de acceso), usando el atributo específico de proveedor de Cisco:

Para `GG_Nivel15`:
1. **Directivas → Directivas de red** → clic derecho → **Nueva**.
2. Nombre: `Politica_Nivel15`.
3. Tipo de servidor de acceso a redes: sin especificar (o "Servidor de acceso remoto" según la versión).
4. Condiciones → Agregar → **Grupos de Windows** → seleccionar `MIGUEL\GG_Nivel15`.
5. Especificar permisos de acceso: **Conceder acceso**.
6. Métodos de autenticación: marcar **MS-CHAP v2** (agregar **PAP sin cifrar** solo si el equipo Cisco lo requiere).
7. Configurar atributos → **Atributos específicos del proveedor** → Agregar → seleccionar **Cisco** → atributo `Cisco-AV-Pair` → valor:
   ```
   shell:priv-lvl=15
   ```
8. Finalizar.

Repetir el mismo procedimiento para:
- `Politica_Nivel10` → grupo `GG_Nivel10` → `Cisco-AV-Pair = shell:priv-lvl=10`
- `Politica_Nivel1` → grupo `GG_Nivel1` → `Cisco-AV-Pair = shell:priv-lvl=1`

**Orden de las directivas** (NPS evalúa de arriba hacia abajo y aplica la primera que coincide): colocar `Politica_Nivel15` primero, luego `Politica_Nivel10`, luego `Politica_Nivel1`.

---

### Integración RADIUS en los routers (R3 y R4) — AAA para login administrativo

El login administrativo (consola/Telnet/SSH) de **R3** y **R4** ahora se valida contra este NPS en vez de solo localmente. Es una lista AAA **distinta** a `VPDN_AUTH` (que solo usa el túnel L2TP), así que no interfiere con la VPN ni con el resto de la configuración existente.

**En R3** (misma red que el servidor, 20.13.67.0/24 — llega directo):

```
enable
configure terminal

radius server NPS_SERVER
 address ipv4 20.13.67.10 auth-port 1812 acct-port 1813
 key RadiusKey123

aaa group server radius RADIUS_NPS
 server name NPS_SERVER

aaa authentication login VTY_AUTH group RADIUS_NPS local
aaa authorization exec VTY_AUTH group RADIUS_NPS local

line vty 0 4
 login authentication VTY_AUTH
 authorization exec VTY_AUTH

end
write memory
```

**En R4** (llega al NPS a través del túnel IPsec site-to-site ya existente hacia 20.13.67.0/24):

```
enable
configure terminal

radius server NPS_SERVER
 address ipv4 20.13.67.10 auth-port 1812 acct-port 1813
 key RadiusKey123

aaa group server radius RADIUS_NPS
 server name NPS_SERVER

aaa authentication login VTY_AUTH group RADIUS_NPS local
aaa authorization exec VTY_AUTH group RADIUS_NPS local

ip radius source-interface FastEthernet0/1

line vty 0 4
 login authentication VTY_AUTH
 authorization exec VTY_AUTH

end
write memory
```

`ip radius source-interface FastEthernet0/1` fija el origen del tráfico RADIUS en la red 30.13.67.0/24, para que viaje cifrado por el túnel IPsec hacia R3/NPS en vez de salir por la interfaz pública.

> **Recomendación de seguridad:** deja también un usuario local de emergencia con privilegio 15 en cada router (`username admin privilege 15 secret ...`), para no quedarte sin acceso si el NPS se cae — el `local` al final de las listas AAA ya sirve como respaldo automático en ese caso.


---

### Verificación completa

**AD DS:**
```
dcdiag /v
repadmin /replsummary
```
```powershell
Get-ADUser -Filter * -SearchBase "OU=Red,DC=miguel,DC=local" | Select Name, SamAccountName
Get-ADGroupMember -Identity "GG_Nivel15"
Get-ADGroupMember -Identity "GG_Nivel10"
Get-ADGroupMember -Identity "GG_Nivel1"
```

**IIS:**
```powershell
Get-Website
Test-NetConnection -ComputerName 20.13.67.10 -Port 80
```
Y desde el navegador: `http://20.13.67.10` (en local) y también desde **Windows10-1** con la VPN `Miguel-L2PT` conectada.

**NPS / RADIUS** (ya integrado en R3 y R4):
```
! En Windows Server: Visor de eventos → Registros de Windows → Seguridad
! Buscar eventos 6272 (acceso concedido) o 6273 (acceso denegado) generados por NPS
```
Y en el router, después de iniciar sesión con cada usuario (`mramirez`, `etorres`, `cgomez`):
```
show privilege
```
Debe mostrar `Current privilege level is 15` para Miguel Ramírez, `10` para Emily Torres y `1` para Carlos Gómez — confirmando que el `Cisco-AV-Pair` de cada directiva se aplicó correctamente.

**Recordatorio — túnel L2TP (ya funcionando, sin cambios):**
```
show vpdn session          ! en R3
show crypto isakmp sa      ! en R3
```
