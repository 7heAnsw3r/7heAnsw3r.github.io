---
title: Guía Práctica para Controlar y Optimizar stdout 🐧
date: 2024-11-03
tags:
  - "#Linux"
  - "#grep"
  - "#awk"
  - "#tee"
  - "#sed"
  - "#Control_Flujo"
  - "#Unix"
  - "#Programacion"
description: En esta sección, exploraremos cómo manipular la salida de información en sistemas operativos Linux/Unix. Esta habilidad es esencial para administradores de sistemas, Pentesters, Programadores y cualquier persona que use sistemas basados en Unix como su plataforma principal. Aprender a gestionar y filtrar el flujo de datos en la terminal puede optimizar tareas, facilitar análisis y mejorar la eficiencia en múltiples escenarios.
---
---
En esta sección, exploraremos cómo manipular la salida de información en sistemas operativos Linux/Unix. Esta habilidad es esencial para administradores de sistemas, Pentesters, Programadores y cualquier persona que use sistemas basados en Unix como su plataforma principal. Aprender a gestionar y filtrar el flujo de datos en la terminal puede optimizar tareas, facilitar análisis y mejorar la eficiencia en múltiples escenarios.

---

# Control de stdout

**Descripción General**: La salida estándar, conocida como **stdout**, es el flujo de datos que generan los programas en la línea de comandos. Es una de las tres corrientes de E/S (entrada/salida) predeterminadas en sistemas Unix y similares, junto con **stdin** (entrada estándar) y **stderr** (salida de error estándar). Al ejecutar un comando en la terminal, la salida que ves en pantalla se envía, por defecto, a stdout.

Puedes redirigir stdout a un archivo o a otro programa utilizando operadores como `>` y `|`. Esto te permite almacenar la salida o encadenar comandos para realizar tareas complejas. Por ejemplo:

```zsh
echo "Hola, mundo!" > hola.txt
```

Este comando enviará el texto "Hola, mundo!" al archivo `hola.txt` en lugar de mostrarlo en pantalla.

**Objetivo**: Esta guía tiene como objetivo enseñarte a utilizar comandos de Linux/Unix para redirigir, filtrar y manipular la salida de datos (**stdout**) de forma eficiente, optimizando tu flujo de trabajo y facilitando el procesamiento de información en la terminal.

## Redirección y Pipes

Para redirigir la salida de un comando a un archivo específico, usamos el operador `>`. Este operador envía el resultado de un comando a un archivo que elijamos, sobrescribiendo el contenido si ya existe. Por ejemplo:

```zsh
┌──(7heAnsw3r㉿AllEyezOnMe)-[~]
└─$ echo "Hola Mundo" > hola.txt

┌──(7heAnsw3r㉿AllEyezOnMe)-[~]
└─$ cat hola.txt
Hola Mundo
```

Si queremos añadir información al archivo en lugar de sobrescribirlo, utilizamos `>>`, que agrega el nuevo contenido al final del archivo. Veamos un ejemplo:

```zsh
┌──(7heAnsw3r㉿AllEyezOnMe)-[~]
└─$ echo "Soy 7heAnsw3r" >> hola.txt

┌──(7heAnsw3r㉿AllEyezOnMe)-[~]
└─$ cat hola.txt
Hola Mundo
Soy 7heAnsw3r
```

De esta forma, podemos redirigir y acumular la salida de varios comandos en un solo archivo, lo cual es útil para almacenar y organizar datos sin perder información anterior.

### Uso de Pipes (`|`)

Los pipes (`|`) son otra herramienta esencial en la línea de comandos de Linux. Permiten redirigir la salida de un comando como entrada de otro, lo cual facilita el procesamiento de datos de forma secuencial y permite crear flujos de trabajo complejos sin necesidad de archivos temporales. 

Algunas ventajas de los pipes incluyen:

- **Procesamiento Secuencial**: Permiten ejecutar comandos uno tras otro.
- **Simplificación de Procesos**: Hacen posible crear flujos de trabajo complejos con un solo comando.
- **Evitación de Archivos Temporales**: No es necesario guardar datos intermedios en archivos.
- **Transferencia Continua de Datos**: Los datos fluyen de un comando al siguiente sin interrupción.

Ejemplo básico de uso de pipe:

```zsh
┌──(7heAnsw3r㉿AllEyezOnMe)-[~]
└─$ echo "Hola Mundo" | grep "Hola"
"Hola" Mundo
```

## Uso de `Grep`

El comando **grep** es una herramienta poderosa en la línea de comandos que permite buscar palabras o cadenas de texto en archivos y salidas de comandos. Como vimos en el ejemplo anterior, grep muestra las líneas completas que contienen la información buscada. Su versatilidad y eficacia la convierten en una herramienta esencial para administradores de sistemas, desarrolladores y cualquier persona que trabaje con texto.

A continuación, se presenta una guía detallada sobre las opciones más comunes de **grep**, acompañada de ejemplos prácticos que ilustran su uso en diferentes situaciones.

En el ejemplo anterior, utilizamos **pipes** para trabajar con **grep**, pero en esta ocasión veremos cómo utilizar **grep** sin **pipes**. En este ejemplo, podemos observar que al buscar la palabra **'hola'**, no obtendremos resultados; sin embargo, al buscar la palabra **'Hola'**, sí encontraremos la coincidencia correspondiente. Esto se debe a que **grep** es sensible a mayúsculas y minúsculas (case sensitive). 

Aquí tienes un ejemplo:

```zsh
┌──(7heAnsw3r㉿AllEyezOnMe)-[~]
└─$ grep 'hola' hola.txt

┌──(7heAnsw3r㉿AllEyezOnMe)-[~]
└─$ grep 'Hola' hola.txt
Hola Mundo
```
### Opciones de Control de Salida

- **Ignorar Distinciones**: En el ejemplo anterior, observamos que al buscar con mayúsculas o minúsculas se obtienen diferentes resultados. Para ignorar estas distinciones, podemos utilizar la opción `-i`. Por ejemplo:

	 ```zsh
    ┌──(7heAnsw3r㉿AllEyezOnMe)-[~]
    └─$ grep -i 'hola' hola.txt
    'Hola' Mundo
    ```

- **Mostrar Números de Línea**: Si queremos ver el número de línea donde se encuentra cada coincidencia en el archivo, utilizamos la opción `-n`. Por ejemplo:

	```zsh
    ┌──(7heAnsw3r㉿AllEyezOnMe)-[~]
    └─$ grep -in 'hola' hola.txt
    1:'Hola' Mundo
	```

- **Mostrar Solo Coincidencias**: Para ver únicamente las coincidencias sin mostrar el resto de la línea, utilizamos la opción `-o`. Por ejemplo:

	```zsh
    ┌──(7heAnsw3r㉿AllEyezOnMe)-[~]
    └─$ grep -ion 'hola' hola.txt
    1:Hola
	```

- **Resaltar Coincidencias**: Para resaltar las coincidencias en color, podemos usar la opción `--color`. Por ejemplo:

	```zsh
    ┌──(7heAnsw3r㉿AllEyezOnMe)-[~]
    └─$ grep -i --color 'hola' hola.txt
    'Hola' Mundo
	```

### Opciones de Selección e Interpretación de Patrones

- ***Extended Regular Expressions*** (`-E`): Usar expresiones regulares extendidas 

	```zsh
	┌──(7heAnsw3r㉿AllEyezOnMe)-[~]
	└─$ grep -E 'Ho|So' hola.txt
	'Ho'la Mundo
	'So'y 7heAnsw3r
	```

- ***Basic Regular Expressions*** (`-G`): Usa expresiones regulares básicas

	```zsh
	┌──(7heAnsw3r㉿AllEyezOnMe)-[~]
	└─$ grep -G 'M.n' hola.txt
	Hola 'Mu'ndo
	```

- ***Fixed Strings*** (`-F`): Trata los patrones como cadenas Fijas

	```zsh
	┌──(7heAnsw3r㉿AllEyezOnMe)-[~]
	└─$ grep -F 'Hola' hola.txt
	'Hola' Mundo
	```

- ***Patrones desde Archivos*** (`-f`): Usa patrones desde un archivo

	```zsh
	┌──(7heAnsw3r㉿AllEyezOnMe)-[~]
	└─$ cat patterns.txt 
	Hola
	Soy
	┌──(7heAnsw3r㉿AllEyezOnMe)-[~]
	└─$ grep -f patterns.txt hola.txt 
	Hola Mundo
	Soy 7heAnsw3r
	```

Además de las opciones anteriores, **grep** ofrece varias opciones interesantes que pueden ser muy útiles en diferentes situaciones:

- **Invertir Coincidencias** (`-v`): Esta opción permite mostrar las líneas que **no** coinciden con el patrón especificado. Por ejemplo:

    ```zsh
    ┌──(7heAnsw3r㉿AllEyezOnMe)-[~]
    └─$ grep -v 'hola' hola.txt
    Hola Mundo
	Soy 7heAnsw3r
    ```

- **Mostrar Contexto** (`-C`): Muestra un número específico de líneas de contexto antes y después de cada coincidencia. Por ejemplo, `-C 2` mostrará 2 líneas de contexto:

    ```zsh
    ┌──(7heAnsw3r㉿AllEyezOnMe)-[~]
    └─$ grep -C 2 'Hola' hola.txt
    Hola Mundo
	Soy 7heAnsw3r
	Estoy utilizando grep
    ```

- **Contar Coincidencias** (`-c`): Esta opción cuenta el número de líneas que coinciden con el patrón en lugar de mostrar las líneas. Por ejemplo:

    ```zsh
    ┌──(7heAnsw3r㉿AllEyezOnMe)-[~]
    └─$ grep -c 'hola' hola.txt
    0
    ```

- **Buscar Recursivamente** (`-r`): Permite buscar el patrón en todos los archivos dentro de un directorio y sus subdirectorios. Por ejemplo:

    ```zsh
    ┌──(7heAnsw3r㉿AllEyezOnMe)-[~]
    └─$ grep -r 'Hola' .
    ./hola.txt:Hola Mundo
	./patterns.txt:Hola
    ```

- **Mostrar Líneas Después de la Coincidencia** (`-A`): Esta opción permite mostrar un número específico de líneas después de cada coincidencia. Por ejemplo, `-A 3` mostrará 3 líneas después:

    ```zsh
    ┌──(7heAnsw3r㉿AllEyezOnMe)-[~]
    └─$ grep -A 3 'hola' hola.txt
    ```

- **Mostrar Líneas Antes de la Coincidencia** (`-B`): Similar a `-A`, pero muestra líneas antes de cada coincidencia. Por ejemplo, `-B 2` mostrará 2 líneas antes:

    ```zsh
    ┌──(7heAnsw3r㉿AllEyezOnMe)-[~]
    └─$ grep -B 2 'hola' hola.txt
    ```

Estas opciones permiten a los usuarios personalizar su búsqueda y manejar la salida de manera más efectiva, haciendo de **grep** una herramienta aún más poderosa en la línea de comandos.

## AWK

### ¿Qué es `awk`?

`awk `es una poderosa herramienta de procesamiento de texto en sistemas Unix/Linux que permite analizar y manipular datos de texto mediante patrones y acciones. Su diseño es especialmente útil para trabajar con archivos de texto estructurados en columnas, facilitando la extracción y transformación de datos.

### Principales Características de `awk`

- **Procesamiento de Campos**: Divide las líneas de un archivo en campos utilizando delimitadores (por defecto, el espacio en blanco), lo que permite acceder y manipular datos de manera eficiente.

- **Patrones y Acciones**: Ejecuta acciones específicas en líneas que coinciden con un patrón determinado, ofreciendo flexibilidad en la manipulación de datos.

- **Variables Incorporadas**: `awk` incluye variables integradas como `$0` (que representa la línea completa) y `$1`, `$2`, etc. (que representan campos específicos), lo que simplifica el acceso a los datos.

- **Funciones Incorporadas**: Proporciona una variedad de funciones para trabajar con cadenas, realizar operaciones numéricas e incluso ejecutar cálculos complejos, ampliando su capacidad de análisis.

- **Portabilidad**: `awk` es compatible con casi todos los sistemas Unix/Linux, lo que lo convierte en una herramienta versátil y ampliamente utilizada en la administración y análisis de datos.

### Ejemplo básico de **awk**: Procesando la salida de `netstat`

Comenzaremos ejecutando un comando que genera una gran cantidad de información organizada en columnas: `netstat`. Este comando muestra las conexiones de red activas en el sistema. Para procesar esta salida, utilizaremos **pipes** junto con **awk** para extraer información específica.

**Ejemplo de `netstat` sin `awk`:**

```zsh
netstat
```

La salida mostrará algo similar a esto:

```zsh
Active UNIX domain sockets (w/o servers)
Proto RefCnt Flags       Type       State         I-Node   Path
unix  3      [ ]         STREAM     CONNECTED     20243    
unix  3      [ ]         STREAM     CONNECTED     15443    /run/systemd/journal/stdout
unix  3      [ ]         STREAM     CONNECTED     12950    /run/dbus/system_bus_socket
unix  3      [ ]         DGRAM      CONNECTED     8789     
unix  3      [ ]         STREAM     CONNECTED     17572    
unix  3      [ ]         STREAM     CONNECTED     15035    /run/user/1000/bus
```

**Ejemplo de `netstat` con `awk`:**

Podemos utilizar **awk** para filtrar la salida y mostrar solo la columna que nos interesa. Por ejemplo, para mostrar el estado de las conexiones, ejecutamos:

```zsh
netstat | awk '{print $5}'
```

Esto mostrará:

```zsh
STREAM
STREAM
STREAM
DGRAM
STREAM
STREAM
STREAM
STREAM
STREAM
```

### Filtrando usuarios en `/etc/passwd`

En las pruebas de pentesting, es crucial verificar la integridad del archivo `/etc/passwd`, que contiene información sobre las cuentas de usuario. 

**Ejemplo básico:**

```zsh
cat /etc/passwd
```

Esto mostrará una salida extensa, por lo que podemos utilizar **awk** para filtrar la información y mostrar solo los usuarios válidos:

```zsh
cat /etc/passwd | awk -F ":" '{print $1}'
```

Salida:

```zsh
root
daemon
bin
sys
sync
games
```

### Imprimiendo múltiples columnas

Si queremos imprimir más de una columna, simplemente especificamos los números de las columnas que deseamos imprimir. Por ejemplo, para mostrar el nombre de usuario, ID de usuario y ID de grupo, podemos hacer:

```zsh
cat /etc/passwd | awk -F ":" '{print $1, $3, $4}'
```

Esto imprimirá:

```zsh
root 0 0
daemon 1 1
bin 2 2
sys 3 3
sync 4 65534
games 5 60
```

Si queremos agregar tabulaciones entre los campos, podemos utilizar `"\t"`:

```zsh
cat /etc/passwd | awk -F ":" '{print $1 "\t" $3 "\t" $4}'
```

Esto mostrará:

```zsh
root    0    0
daemon  1    1
bin     2    2
sys     3    3
sync    4    65534
games   5    60
```

### Configurando delimitadores personalizados

Podemos establecer delimitadores personalizados utilizando la sección `BEGIN` en **awk**. Por ejemplo:

```zsh
cat /etc/passwd | awk 'BEGIN {FS=":"; OFS=" ; "} {print $1, $3, $4}'
```

Esto nos dará:

```zsh
root ; 0 ; 0
daemon ; 1 ; 1
bin ; 2 ; 2
sys ; 3 ; 3
sync ; 4 ; 65534
games ; 5 ; 60
```

### Usando expresiones regulares

Para imprimir el último campo delimitado por `/` de una salida, podemos usar la siguiente expresión regular:

```zsh
awk -F "/" '/^\// {print $NF}' /etc/shells
```

Aquí, cada parte del comando hace lo siguiente:
- `-F "/"`: Establece `/` como el delimitador.
- `/^\//`: Coincide con líneas que comienzan con `/`.
- `{print $NF}`: Imprime el último campo (donde `NF` es el número total de campos en la línea).

Al ejecutar este comando, obtenemos:

```zsh
bash
bash
bash
dash
pwsh
screen
tmux
zsh
```

### Comprendiendo `BEGIN` y `NF`

- **BEGIN**: Es una sección especial en **awk** que se ejecuta antes de procesar cualquier línea de entrada. Se usa comúnmente para establecer variables de campo (`FS`) y otras configuraciones globales.
  
- **NF**: Esta variable integrada representa el número total de campos en la línea actual. Por ejemplo, si deseas acceder al último campo sin saber cuántos campos hay, puedes utilizar `$NF`.

### Usando **NF** para imprimir campos

Si no sabemos cuántos campos hay, podemos utilizar `NF` para acceder a campos específicos:

```zsh
cat /etc/passwd | awk -F ":" '{print $NF}'
```

Esto imprimirá el último campo de cada línea, que generalmente contiene la ruta del intérprete de comandos.
### Operar con Líneas de Texto Mediante el Comando `awk`

Hasta ahora, hemos aprendido varios métodos para imprimir columnas específicas de texto. Pero también podemos filtrar líneas completas según lo que necesitemos. ¡Vamos a ver cómo hacerlo!
#### Imprimir Líneas que Sigan un Patrón Específico

Imagina que tienes la salida del comando `df`, que se ve así:

```bash
┌──(7heAnsw3r㉿AllEyezOnMe)-[~]
└─$ df   
Filesystem     1K-blocks     Used Available Use% Mounted on
udev             6109492        0   6109492   0% /dev
tmpfs            1234568     1392   1233176   1% /run
/dev/sda1      101639152 21191960  75238004  22% /
tmpfs            6172832    11692   6161140   1% /dev/shm
tmpfs               5120        0      5120   0% /run/lock
tmpfs               1024        0      1024   0% /run/credentials/systemd-journald.service
tmpfs               1024        0      1024   0% /run/credentials/systemd-tmpfiles-setup-dev-early.service
```

Si solo quieres mostrar las líneas que comienzan con `/`, puedes usar el siguiente comando:

```bash
df | awk '/^\// {print}'
```

Esto te dará solo las líneas que empiezan por `/`:

```
/dev/sda1      101639152 21191960  75238004  22% /
```

**Ejemplo Extra:** Si quieres filtrar aún más y mostrar solo la línea que comienza por `/dev/sda1`, ejecutarías:

```bash
df | awk '/^\/dev\/sda1/ {print}'
```

#### Filtrar Líneas que Empiezan o Terminan con un Patrón

Ya hemos visto cómo mostrar líneas que empiezan con una letra o palabra. Si quieres ver las líneas que empiezan por `tmpfs`, simplemente haz:

```bash
df | awk '/^tmpfs/ {print}'
```

**Para ver solo las líneas que terminan en `/shm`, puedes usar:**

```bash
df | awk '/\/shm$/ {print}'
```

**Y si quieres líneas que empiecen con `tmpfs` y terminen con `/shm`, puedes hacer esto:**

```bash
df | awk '/\/shm$/ && /^tmpfs/ {print}'
```

#### Mostrar Columnas Específicas de Líneas Filtradas

En lugar de mostrar todas las columnas, puedes especificar cuáles quieres ver. Por ejemplo, para mostrar solo las columnas 1, 2 y 3 de las líneas que empiezan por `/`, harías lo siguiente:

```bash
df -h | awk '/^\// {print $1"\t"$2"\t"$3}'
```

**Ejemplo de Sumar Columnas:**

Si quieres sumar las columnas 2 y 3:

```bash
df -h | awk '/^\// {print $1"\t"$2 + $3}'
```

#### Filtrar Líneas por Longitud

Si quieres imprimir las líneas del archivo `/etc/shells` que tienen más de 9 caracteres, el comando es:

```bash
awk 'length($0) > 9' /etc/shells
```

Puedes usar `length($0) < 9`, `length($0) == 9` o `length($0) != 9` para otros filtros.

#### Filtrar con Múltiples Condiciones

Usa el operador `&&` para mostrar líneas que cumplan múltiples condiciones. Por ejemplo, si quieres mostrar líneas de `df -h` que empiecen con `t` y cuya columna 6 tenga más de 8 caracteres:

```bash
sudo df -h | awk '/^t/ && length($6) > 8 {print $0}'
```

#### Ver Longitud de Líneas

Para saber cuántos caracteres tiene cada línea en `/etc/shells`, haz:

```bash
awk '{print length}' /etc/shells
```

**Si también quieres ver el contenido:**

```bash
awk '{print length"\t"$0}' /etc/shells
```

**Y para obtener el número de caracteres de la columna 1 en `df -h`, usa:**

```bash
sudo df -h | awk '{print length($1)"\t"$1}'
```

#### Filtrar por Último Delimitador

Para listar procesos cuyo último delimitador sea `firefox`, puedes combinar lo aprendido con `if`:

```bash
ps -ef | awk '{ if($NF == "obsidian") print $0}'
```

**Si solo quieres el número de proceso y el último delimitador:**

```bash
ps -ef | awk '{ if($NF == "obsidian") print $2" "$NF}'
```

## Operar con Texto Mediante el Comando `sed`

Hasta ahora, hemos aprendido varios métodos para manipular texto en la terminal. Hoy, nos centraremos en `sed`, una poderosa herramienta para editar texto de manera no interactiva. ¡Vamos a ver cómo funciona!
### Reemplazar Texto

Uno de los usos más comunes de `sed` es reemplazar texto en un archivo. Supongamos que tienes un archivo llamado `texto.txt` con el siguiente contenido:

```
Hola, mundo.
Esto es un archivo de ejemplo.
```

Para la creacion pongamos en practica todo lo que hemos aprendido hasta aqui de la siguiente manera:

```zsh
┌──(7heAnsw3r㉿AllEyezOnMe)-[~]
└─$ echo "Hola, mundo.\nEsto es un archivo de ejemplo." > texto.txt 
┌──(7heAnsw3r㉿AllEyezOnMe)-[~]
└─$ cat texto.txt 
Hola, mundo.
Esto es un archivo de ejemplo.
```

Si quieres reemplazar "mundo" por "universo", utilizarías el siguiente comando:

```bash
sed 's/mundo/universo/' texto.txt
```

Esto imprimirá:

```zsh
┌──(7heAnsw3r㉿AllEyezOnMe)-[~]
└─$ sed 's/mundo/universo/' texto.txt
Hola, universo.
Esto es un archivo de ejemplo.
```

Lo cual no altera el archivo en si pero si a la salida del archivo 

Si deseas que el cambio se aplique a todas las líneas, simplemente usa la opción `g`:

```bash
sed 's/mundo/universo/g' texto.txt
```

### Eliminar Líneas

Puedes usar `sed` para eliminar líneas específicas. Por ejemplo, si deseas eliminar la segunda línea de `texto.txt`, ejecutarías:

```bash
sed '2d' texto.txt
```

Esto imprimirá:

```
┌──(7heAnsw3r㉿AllEyezOnMe)-[~]
└─$ sed '2d' texto.txt               
Hola, mundo.
┌──(7heAnsw3r㉿AllEyezOnMe)-[~]
└─$ cat texto.txt 
Hola, mundo.
Esto es un archivo de ejemplo.
```

Si quieres eliminar todas las líneas que contengan la palabra "ejemplo", usa:

```bash
sed '/ejemplo/d' texto.txt
```

### Insertar y Añadir Texto

También puedes insertar o añadir texto en líneas específicas. Para insertar una línea antes de la segunda línea:

```bash
sed '2i Esta línea se inserta antes de la segunda línea.' texto.txt
```

```zsh
┌──(7heAnsw3r㉿AllEyezOnMe)-[~]
└─$ sed '2i Esta línea se inserta antes de la segunda línea.' texto.txt
Hola, mundo.
Esta línea se inserta antes de la segunda línea.
Esto es un archivo de ejemplo.
┌──(7heAnsw3r㉿AllEyezOnMe)-[~]
└─$ cat texto.txt
Hola, mundo.
Esto es un archivo de ejemplo.
```

Para añadir una línea después de la primera línea, utiliza:

```bash
sed '1a Esta línea se añade después de la primera línea.' texto.txt
```

### Reemplazo en Archivos

Para hacer los cambios directamente en el archivo, usa la opción `-i`. Esto reemplazará "mundo" por "universo" en el archivo original:

```bash
sed -i 's/mundo/universo/g' texto.txt
```

Ten en cuenta que este comando no imprimirá nada en la salida estándar.

### Filtrar Líneas por Patrón

Puedes usar `sed` para mostrar solo líneas que coincidan con un patrón. Si quieres mostrar solo las líneas que contienen la palabra "archivo":

```bash
sed -n '/archivo/p' texto.txt
```

La opción `-n` suprime la salida normal, y `p` imprime las líneas coincidentes.

```zsh
┌──(7heAnsw3r㉿AllEyezOnMe)-[~]
└─$ sed -n '/archivo/p' texto.txt
Esto es un archivo de ejemplo.
```
### Modificar Delimitadores

Por defecto, `sed` utiliza la barra `/` como delimitador, pero puedes cambiarlo a otro carácter. Por ejemplo, si quieres reemplazar `:` por `-` en un archivo `fechas.txt`:

```bash
sed 's#:#-#g' fechas.txt
```

### Ejemplos Combinados

Combina varias operaciones en un solo comando. Por ejemplo, para reemplazar "mundo" por "universo" y eliminar la segunda línea:

```bash
sed -e 's/mundo/universo/g' -e '2d' texto.txt
```

### Usar `sed` con Tubos

Puedes usar `sed` en combinación con otros comandos a través de pipes. Por ejemplo, para filtrar la salida de `ps` y reemplazar "bash" por "sh":

```bash
ps aux | sed 's/bash/sh/g'
```

Esta guía es un recurso completo sobre cómo utilizar comandos esenciales como `grep`, `awk`, `sed`, pipes y redirecciones. Estas herramientas son de gran utilidad para administradores de sistemas, equipos de blue team y red team, así como para cualquier persona que trabaje en entornos sin interfaz gráfica, como en servidores, donde el uso de GUI puede ser costoso y menos eficiente. Aquí encontrarás una introducción práctica sobre cómo mejorar y manejar la salida de texto con estos comandos, haciendo tu trabajo más fluido y optimizado.

En el próximo capítulo, exploraremos herramientas adicionales como `less`, `more` y `tee`, y también veremos la generación de scripts para automatizar tareas diarias.


<div style="width:100%;height:0;padding-bottom:57%;position:relative;"><iframe src="https://giphy.com/embed/UFGj6EYw5JhMQ" width="100%" height="100%" style="position:absolute" frameBorder="0" class="giphy-embed" allowFullScreen></iframe></div><p><a href="https://giphy.com/gifs/UFGj6EYw5JhMQ">via GIPHY</a></p>
