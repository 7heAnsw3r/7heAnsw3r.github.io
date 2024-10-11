---
title: "HTB: Nibbles WriteUp 🐧"
date: 2024-10-11
tags:
  - "#Linux"
  - "#Web"
  - "#Plugins"
  - "#OWASP"
  - "#Sudo"
  - "#nmap"
description: Nibbles es una máquina bastante simple, pero la inclusión de una lista negra de inicios de sesión hace que sea un poco más difícil encontrar credenciales válidas. Afortunadamente, se puede enumerar un nombre de usuario y adivinar la contraseña correcta no toma mucho tiempo en la mayoría de los casos.
---
---
# Nibbles

Nibbles es una máquina bastante sencilla donde trabajamos con el framework OWASP. Sin embargo, con la inclusión de una lista negra de inicio de sesión, encontrar credenciales válidas se complica un poco. Lo bueno es que se puede enumerar un nombre de usuario, y para la mayoría, adivinar la contraseña correcta no debería llevar mucho tiempo. Es una buena oportunidad para practicar la búsqueda de vulnerabilidades en la web y mejorar nuestras habilidades en hacking.

---

<figure>
<img src="/assets/img/Nibbles/Nibbles.png" alt="Freelancer">
<figcaption>Fig 1. Nibbles- HTB</figcaption>
</figure>
---

Empezamos la maquina realizando un escaneo de puertos utilizando la herramienta de Nmap, con el escaneo típico escaneo para realizar escaneos en CTFs. Este escaneo es especialmente rápido porque controlamos la velocidad de envío de paquetes por segundo y, además, usamos opciones que ayudan a evadir firewalls potenciales.

```zsh
sudo nmap -p- --open --min-rate 5000 -sS -f -Pn -n 10.10.10.75 -oG puertos
```

| Opción                | Descripción                                                                                                                                                    |
| --------------------- | -------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| **`sudo`**            | Ejecuta Nmap con privilegios de superusuario, necesarios para ciertos tipos de escaneos, especialmente el escaneo SYN (-sS).                                   |
| **`-p-`**             | Especifica un escaneo de todos los puertos, del 1 al 65535. Permite descubrir cualquier puerto que esté abierto, sin limitarse a los puertos comunes.          |
| **`--open`**          | Muestra solo los puertos que están abiertos, simplificando la salida y enfocándose en los puertos útiles para análisis posteriores.                            |
| **`--min-rate 5000`** | Establece la tasa mínima de paquetes enviados a 5000 por segundo, acelerando el escaneo. Ideal para CTFs, donde se busca la mayor velocidad posible.           |
| **`-sS`**             | Realiza un escaneo SYN (scaneo sigiloso), enviando paquetes SYN para identificar puertos abiertos, disminuyendo la posibilidad de ser detectado.               |
| **`-f`**              | Fragmenta los paquetes en fragmentos más pequeños, ayudando a evadir ciertos firewalls que intentan bloquear escaneos detectando paquetes más grandes.         |
| **`-Pn`**             | Omite la fase de descubrimiento de host y asume que el host está activo, útil cuando el objetivo bloquea los paquetes de ping.                                 |
| **`-n`**              | Indica a Nmap que no resuelva nombres de dominio, haciendo el escaneo más rápido al evitar la resolución DNS.                                                  |
| **`10.10.10.75`**     | Dirección IP objetivo. En un contexto de CTF, esta suele ser la dirección del servidor al que se busca acceder.                                                |
| **`-oG puertos`**     | Guarda la salida en un archivo en formato Greppable (nombrado "puertos"). Facilita la búsqueda de resultados específicos y su reutilización en otros comandos. |

En el escaneo tenemos los siguientes resultados:

```zsh
sudo nmap -p- --open --min-rate 5000 -sS -f -Pn -n 10.10.10.75 -oG puertos
[sudo] password for k-dot: 
Starting Nmap 7.94SVN ( https://nmap.org ) at 2024-10-11 03:53 -05
Nmap scan report for 10.10.10.75
Host is up (0.11s latency).
Not shown: 63547 closed tcp ports (reset), 1986 filtered tcp ports (no-response)
Some closed ports may be reported as filtered due to --defeat-rst-ratelimit
PORT   STATE SERVICE
22/tcp open  ssh
80/tcp open  http

Nmap done: 1 IP address (1 host up) scanned in 15.79 seconds

```


---

#### **Análisis de Puertos y Enfoque OWASP:**

Los puertos **22 (SSH)** y **80 (HTTP)** están abiertos. La presencia de solo estos puertos indica que probablemente estemos frente a un entorno de hacking web, dado que el puerto 80, junto con otros como el 443 o el 8080, se asocian comúnmente con servicios web.

A partir de esto, es útil seguir el **marco de referencia de OWASP**. Este marco ofrece una guía estructurada para identificar y mitigar vulnerabilidades específicas de aplicaciones web. Utilizando OWASP, podemos cubrir áreas clave de riesgo, como **inyecciones SQL**, **control de acceso deficiente**, y **gestión inadecuada de sesiones**, entre otros. Esto nos permite orientar nuestro análisis hacia vulnerabilidades comunes en servicios web, maximizando la efectividad de nuestras pruebas.

---

#### **Escaneo Detallado por Servicios:**

Con esta info, pasamos a realizar otro escaneo. Podríamos haberlo incluido en el primer escaneo, pero eso lo habría ralentizado bastante, así que es mejor dividirlo en partes. Para esto, usamos la siguiente combinación de comandos:

```zsh
sudo nmap -p22,80 -sCV -sS -Pn -n 10.10.10.75 -oN objetivos.txt
```

| Opción       | Descripción                                                                                                                                                                                                                                                                                                        |
| ------------ | ------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------ |
| **`p22,80`** | Especifica que vamos a escanear solo los puertos 22 (SSH) y 80 (HTTP), que ya sabemos están abiertos. Esto hace el escaneo más rápido, enfocándonos solo en lo importante.                                                                                                                                         |
| **`-sCV`**   | Habilita el escaneo de versiones y scripts. El **`-sC`** usa scripts NSE (Nmap Scripting Engine) básicos para identificar servicios, y el **`-sV`** intenta determinar las versiones exactas de los servicios que corren en esos puertos. Esto es útil para identificar vulnerabilidades específicas de versiones. |

Resultado del escaneo:

```zsh
sudo nmap -p22,80 -sCV -sS -Pn -n 10.10.10.75 -oN objetivos.txt           
Starting Nmap 7.94SVN ( https://nmap.org ) at 2024-10-11 04:03 -05
Nmap scan report for 10.10.10.75
Host is up (0.094s latency).

PORT   STATE SERVICE VERSION
22/tcp open  ssh     OpenSSH 7.2p2 Ubuntu 4ubuntu2.2 (Ubuntu Linux; protocol 2.0)
| ssh-hostkey: 
|   2048 c4:f8:ad:e8:f8:04:77:de:cf:15:0d:63:0a:18:7e:49 (RSA)
|   256 22:8f:b1:97:bf:0f:17:08:fc:7e:2c:8f:e9:77:3a:48 (ECDSA)
|_  256 e6:ac:27:a3:b5:a9:f1:12:3c:34:a5:5d:5b:eb:3d:e9 (ED25519)
80/tcp open  http    Apache httpd 2.4.18 ((Ubuntu))
|_http-server-header: Apache/2.4.18 (Ubuntu)
|_http-title: Site doesn't have a title (text/html).
Service Info: OS: Linux; CPE: cpe:/o:linux:linux_kernel

Service detection performed. Please report any incorrect results at https://nmap.org/submit/ .
Nmap done: 1 IP address (1 host up) scanned in 10.45 seconds

```

---

##### **Análisis de Vulnerabilidades en el Servicio SSH:**

La versión del servicio en el puerto 22 es **OpenSSH 7.2p2**, que ya es algo antigua. Esto nos da la oportunidad de buscar vulnerabilidades conocidas para esta versión. Usamos **searchsploit** para buscar en la base de datos de Exploit DB si existe algún exploit disponible que nos permita enumerar o explotar este servicio. Aquí está el comando y algunos resultados relevantes:

```zsh
searchsploit OpenSSH 7.2p2
```

**Resultados de Searchsploit:**

| Exploit Title                                                          | Path                   |
| ---------------------------------------------------------------------- | ---------------------- |
| OpenSSH 2.3 < 7.7 - Username Enumeration                               | linux/remote/45233.py  |
| OpenSSH 2.3 < 7.7 - Username Enumeration (PoC)                         | linux/remote/45210.py  |
| OpenSSH 7.2 - Denial of Service                                        | linux/dos/40888.py     |
| OpenSSH 7.2p2 - Username Enumeration                                   | linux/remote/40136.py  |
| OpenSSH < 7.4 - 'UsePrivilegeSeparation Disabled' Privilege Escalation | linux/local/40962.txt  |
| OpenSSH < 7.4 - agent Protocol Arbitrary Library Loading               | linux/remote/40963.txt |
| OpenSSH < 7.7 - User Enumeration (2)                                   | linux/remote/45939.py  |
| OpenSSHd 7.2p2 - Username Enumeration                                  | linux/remote/40113.txt |

Como vemos, existen varios exploits de enumeración de usuarios y uno de Denial of Service (DoS). Podemos probar alguno de estos para obtener nombres de usuario válidos o realizar ataques adicionales, según los permisos y el contexto del CTF. En particular, los scripts de enumeración pueden ser útiles para extraer información sin llamar mucho la atención.

---

##### **Uso de Exploits y Exploración del Sitio Web:**

Para copiar un exploit a nuestro directorio de trabajo, usamos el siguiente comando de **searchsploit**:

```zsh
searchsploit -m 40136
```

Luego, debemos realizar un pequeño ajuste en el script para que funcione correctamente con Python 3. Modificamos la siguiente línea:

```python
# Cambia esta línea
starttime = time.clock()
# Por esta
starttime = time.perf_counter()
```

Esto soluciona una incompatibilidad con Python 3, ya que `time.clock()` fue reemplazado por `time.perf_counter()`. Ahora, ejecutamos el exploit de la siguiente manera:

```zsh
python3 40136.py 10.10.10.75 -U /usr/share/wordlists/metasploit/unix_passwords.txt -s
```

Este comando lanza un escaneo de enumeración de usuarios en el servidor SSH de destino usando el archivo de contraseñas especificado. Pero no tenemos ningún resultado así que es hora de comenzar a enumerar el sitio Web.

---

####  **Análisis del Sitio Web y Exploración de NibbleBlog:**

Como primer paso, utilizamos **whatweb** para obtener información sobre la arquitectura del sitio:

```zsh
whatweb http://10.10.10.75
```

**Salida del Comando:**
```
http://10.10.10.75 [200 OK] Apache[2.4.18], Country[RESERVED][ZZ], HTTPServer[Ubuntu Linux][Apache/2.4.18 (Ubuntu)], IP[10.10.10.75]
```

Esta salida nos indica que el servidor web está utilizando **Apache 2.4.18** sobre un sistema operativo **Ubuntu**. Sin embargo, no encontramos información particularmente interesante aquí, así que decidimos explorar el sitio web. Al acceder, nos recibe un simple mensaje de "Hello World!". Esto sugiere que hay más por descubrir en busca de posibles puntos de entrada.

<figure>
<img src="/assets/img/Nibbles/imagen1.png" alt="Freelancer">
</figure>
Antes de realizar un escaneo de subdirectorios por fuerza bruta con **ffuf**, verificamos el código fuente de la página usando **CTRL+U** y encontramos un mensaje muy interesante.

<figure>
<img src="/assets/img/Nibbles/imagen2.png" alt="Freelancer">
</figure>

Encontramos un subdirectorio el subdirectorio /nibbleblog/ lo cual nos hace pensar que el sitio fue hecho con nibble blog 

<figure>
<img src="/assets/img/Nibbles/imagen3.png" alt="Freelancer">
</figure>
Al acceder al subdirectorio **/nibbleblog/**, confirmamos que efectivamente el sitio está construido con **NibbleBlog**.

<figure>
<img src="/assets/img/Nibbles/imagen4.png" alt="Freelancer">
</figure>

Dado que sabemos que el sitio utiliza NibbleBlog, es prudente utilizar herramientas como **ffuf** para realizar un escaneo de fuerza bruta de subdirectorios. Ejecutamos el siguiente comando

```zsh
ffuf -u http://10.10.10.75/nibbleblog/FUZZ -w /usr/share/wordlists/seclists/Discovery/Web-Content/directory-list-2.3-big.txt 
```

Con el escaneo de ffuf activo encontramos el subdirectorio README el cual es muy común  la version de NIbbleBlog es v4.0.3


<figure>
<img src="/assets/img/Nibbles/imagen5.png" alt="Freelancer">
</figure>

Además de listar los subdirectorios **themes**, **admin**, **README**, **plugins** y **languages**, notamos que podemos acceder a los archivos dentro de esos directorios. Antes de indagar en el índice de cada uno, especificamos a **ffuf** que queremos buscar archivos con las extensiones `.txt`, `.xml`, y `.php`. Esto nos lleva a un área donde podemos iniciar sesión en **admin.php**.


<figure>
<img src="/assets/img/Nibbles/imagen6.png" alt="Freelancer">
</figure>

Supongo que necesitamos encontrar las credenciales para ingresar al sitio. En nuestro escaneo con **ffuf**, descubrimos un subdirectorio llamado **content**. Podríamos haber utilizado **dirb**, que realiza ataques recursivos, pero optaremos por hacerlo manualmente. Tenemos acceso al índice de **nibbleblog/content**. 


<figure>
<img src="/assets/img/Nibbles/imagen7.png" alt="Freelancer">
</figure>

<figure>
<img src="/assets/img/Nibbles/imagen8.png" alt="Freelancer">
</figure>

ingresamos a la carpeta private ya que es obvio que ahi se guarda información privilegiada como no, esperemos tener suerte, encontramos varios archivos con terminación .xml 

<figure>
<img src="/assets/img/Nibbles/imagen9.png" alt="Freelancer">
</figure>
En **users.xml** y **config.xml**, descubrimos que el nombre de usuario es **admin**. Además, en **config.xml** encontramos la siguiente información:

<figure>
<img src="/assets/img/Nibbles/imagen10.png" alt="Freelancer">
</figure>
el correo es admin@nibbles.com asi que voy a intentar con las credenciales admin - nibbles, regresamos al sitio de admin.php y nos logeamos y pues si funciona.

<figure>
<img src="/assets/img/Nibbles/imagen11.png" alt="Freelancer">
</figure>

---

### **Acceso como Administrador y Ejecución de Archivos:**

Ahora que hemos logrado acceder como administrador al sitio web, tenemos la capacidad de subir archivos.

<figure>
<img src="/assets/img/Nibbles/imagen12.png" alt="Freelancer">
</figure>

Recuerda que encontramos que la versión de **NibbleBlog** es **4.0.3**, lo que nos permite buscar exploits que puedan guiarnos sobre las acciones posibles. Efectivamente, podemos realizar una carga de archivos de tipo **Arbitrary Upload File**. Primero, instalamos el plugin **About**.

<figure>
<img src="/assets/img/Nibbles/imagen13.png" alt="Freelancer">
</figure>
Intentaremos subir un archivo `.php` en primer lugar, y después probaremos con archivos `.jpg` y `.png`.

<figure>
<img src="/assets/img/Nibbles/imagen14.png" alt="Freelancer">
</figure>
Nos va a salir un mensaje de warning 

<figure>
<img src="/assets/img/Nibbles/imagen15.png" alt="Freelancer">
</figure>
Recordemos que anteriormente encontramos el subdirectorio **content**, donde suponemos que se guarda la configuración de nuestro plugin.

<figure>
<img src="/assets/img/Nibbles/imagen16.png" alt="Freelancer">
</figure>
Así que navegamos a **private/plugins**, seleccionamos el nombre del plugin que instalamos y hacemos clic en el archivo con extensión `.php`.

<figure>
<img src="/assets/img/Nibbles/imagen17.png" alt="Freelancer">
</figure>
¡Hemos conseguido nuestra **reverse shell**!

<figure>
<img src="/assets/img/Nibbles/imagen18.png" alt="Freelancer">
</figure>
Sin embargo, con esta shell no podemos hacer mucho, por lo que debemos realizar el tratamiento correspondiente para mejorar la terminal. Ejecutamos el siguiente comando:
```zsh
script /dev/null -c bash
```
Luego, enviamos nuestra shell a segundo plano presionando **CTRL+Z** y ejecutamos:
```zsh
stty raw -echo; fg
```

A continuación, ejecutamos la siguiente línea de comandos, configurada para su entorno:

```zsh
restart xterm
export TERM=xterm
stty rows columns
```

y con esta configuración ya tenemos una shell interactiva 

<figure>
<img src="/assets/img/Nibbles/imagen19.png" alt="Freelancer">
</figure>
Ahora ya no tenemos miedo de ejecutar CTRL+C tenemos la primera bandera 

---

### Privileges Escalation 

Como buena práctica, utilizamos `sudo -l` para verificar qué permisos tenemos para convertirnos en usuario root.


<figure>
<img src="/assets/img/Nibbles/imagen20.png" alt="Freelancer">
</figure>
No se nos pidió la contraseña, lo que nos da acceso a root a través del archivo **monitor.sh**. En nuestro directorio actual, encontramos un archivo `.zip` llamado **personal**. Para descomprimirlo, utilizamos la herramienta **unzip**:
```zsh 
unzip personal.zip 
```
Una vez descomprimido, notamos que el archivo tiene permisos de ejecución.

<figure>
<img src="/assets/img/Nibbles/imagen21.png" alt="Freelancer">
</figure>
A continuación, utilizamos **nano** para leer el contenido del archivo. Modificamos el archivo con el siguiente comando para obtener una shell como root:
```zsh
echo "bash -i" > monitor.sh
```

Al ejecutar esto, logramos acceso como root.


<figure>
<img src="/assets/img/Nibbles/imagen22.png" alt="Freelancer">
</figure>

